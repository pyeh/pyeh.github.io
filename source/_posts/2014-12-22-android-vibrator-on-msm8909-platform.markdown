---
layout: post
title: "Android vibrator on MSM8909 platform"
date: 2014-12-22 23:38
comments: true
categories: [ android, kernel] 
---
QPNP (Qualcomm Plug Plug N Play) vibrator is a peripheral on Qualcomm PMICs. The PMIC is connected to MSM8909 via SPMI bus.
Its driver uses the timed-output framework to interface with the userspace.

###1. MSM8909 Vibrator driver implementation and DTS file
The vibrator is connected on the VIB_DRV_N line of the QPNP PMIC, and the PMIC vibrator driver can be used to program the voltage from 1.2 V to 3.1 V in 100 mV step through VIB_DRV_N pin.
####DTSI documentation  
``` c kernel/arch/arm/boot/dts/qcom/msm-pm8909.dtsi
                pm8909_vib: qcom,vibrator@c000 {
                        compatible = "qcom,qpnp-vibrator";
                        reg = <0xc000 0x100>;
                        label = "vibrator";
                        status = "disabled";
                };
```
####Documentation/devicetree/bindings/platform/msm/qpnp-vibrator.txt

#####Required Properties:  
	- status: default status is set to "disabled.  Must be "okay"  
	- compatible: must be "qcom,qpnp-vibrator"  
	- label: name which describes the device  
	- reg: address of device  

#####Optional Properties:  
	- qcom,vib-timeout-ms: timeout of vibrator, in ms.  Default 15000 ms  
	- qcom,vib-vtg-level-mV: voltage level, in mV.  Default 3100 mV  

####vibrator driver source  
QPNP vibrator operates in two modes - manual and PWM (Pulse Width Modulation). PWM is supported
through one of dtest lines connected to the vibrator. Manual mode is used on MSM8909 platform.

``` c qpnp_vibrator_prboe() under kernel/drivers/platform/msm/qpnp-vibrator.c
static int qpnp_vibrator_probe(struct spmi_device *spmi)
{
        struct qpnp_vib *vib;
        struct resource *vib_resource;
        int rc;

        vib = devm_kzalloc(&spmi->dev, sizeof(*vib), GFP_KERNEL);
        if (!vib)
                return -ENOMEM;

        vib->spmi = spmi;

        vib_resource = spmi_get_resource(spmi, 0, IORESOURCE_MEM, 0);
        if (!vib_resource) {
                dev_err(&spmi->dev, "Unable to get vibrator base address\n");
                return -EINVAL;
        }
        vib->base = vib_resource->start;

        rc = qpnp_vib_parse_dt(vib);
        if (rc) {
                dev_err(&spmi->dev, "DT parsing failed\n");
                return rc;
        }

        rc = qpnp_vibrator_config(vib);
        if (rc) {
                dev_err(&spmi->dev, "vib config failed\n");
                return rc;
        }

        mutex_init(&vib->lock);
        INIT_WORK(&vib->work, qpnp_vib_update);

        hrtimer_init(&vib->vib_timer, CLOCK_MONOTONIC, HRTIMER_MODE_REL);
        vib->vib_timer.function = qpnp_vib_timer_func;

        vib->timed_dev.name = "vibrator";
        vib->timed_dev.get_time = qpnp_vib_get_time;
        vib->timed_dev.enable = qpnp_vib_enable;

        dev_set_drvdata(&spmi->dev, vib);

        rc = timed_output_dev_register(&vib->timed_dev);
        if (rc < 0)
                return rc;

        return rc;
}
```
####Uses timed output framework
callback *.enable* and *.get_time* of *struct timed_output_dev* is hooked by `qpnp_vib_enable()` and `qpnp_vib_get_time()`.
``` c Implementation of qnp_vib_enable() and qpnp_vib_get_time()
static void qpnp_vib_enable(struct timed_output_dev *dev, int value)
{
        struct qpnp_vib *vib = container_of(dev, struct qpnp_vib,
                                         timed_dev);

        mutex_lock(&vib->lock);
        hrtimer_cancel(&vib->vib_timer);

        if (value == 0)
                vib->state = 0;
        else {
                value = (value > vib->timeout ?
                                 vib->timeout : value);
                vib->state = 1;
                hrtimer_start(&vib->vib_timer,
                              ktime_set(value / 1000, (value % 1000) * 1000000),
                              HRTIMER_MODE_REL);
        }
        mutex_unlock(&vib->lock);
        schedule_work(&vib->work);
}

static int qpnp_vib_get_time(struct timed_output_dev *dev)
{
        struct qpnp_vib *vib = container_of(dev, struct qpnp_vib,
                                                         timed_dev);

        if (hrtimer_active(&vib->vib_timer)) {
                ktime_t r = hrtimer_get_remaining(&vib->vib_timer);
                return (int)ktime_to_us(r);
        } else
                return 0;
}
```

####wq is scheduled to execute work of qpnp_vib_update() and turns off vibrator via qpnp_vibrator_suspend() at suspend callback.
Inside `qpnp_vib_update()` and `qpnp_vibrator_suspend()`, they both call `qpnp_vib_set()` to turn on/off vibrator.
``` c Implementation of qpnp_vib_set() to control vibrator via PWM or Manual Mode
static int qpnp_vib_set(struct qpnp_vib *vib, int on)
{
        int rc;
        u8 val;

        if (on) {
                if (vib->mode != QPNP_VIB_MANUAL)
                        pwm_enable(vib->pwm_info.pwm_dev);
                else {
                        val = vib->reg_en_ctl;
                        val |= QPNP_VIB_EN;
                        rc = qpnp_vib_write_u8(vib, &val,
                                        QPNP_VIB_EN_CTL(vib->base));
                        if (rc < 0)
                                return rc;
                        vib->reg_en_ctl = val;
                }
        } else {
                if (vib->mode != QPNP_VIB_MANUAL)
                        pwm_disable(vib->pwm_info.pwm_dev);
                else {
                        val = vib->reg_en_ctl;
                        val &= ~QPNP_VIB_EN;
                        rc = qpnp_vib_write_u8(vib, &val,
                                        QPNP_VIB_EN_CTL(vib->base));
                        if (rc < 0)
                                return rc;
                        vib->reg_en_ctl = val;
                }
        }

        return 0;
}
```

###2. Timed output class driver
Timed output is a class drvier to allow changing a state and restore is automatically after a specfied timeout.
This exposes a user space interface under */sys/class/timed_output/vibrator/enable* used by vibrator code.
``` c kernel/drivers/staging/android/timed_output.c
static ssize_t enable_show(struct device *dev, struct device_attribute *attr,
                char *buf)
{
        struct timed_output_dev *tdev = dev_get_drvdata(dev);
        int remaining = tdev->get_time(tdev);

        return sprintf(buf, "%d\n", remaining);
}

static ssize_t enable_store(
                struct device *dev, struct device_attribute *attr,
                const char *buf, size_t size)
{
        struct timed_output_dev *tdev = dev_get_drvdata(dev);
        int value;

        if (sscanf(buf, "%d", &value) != 1)
                return -EINVAL;

        tdev->enable(tdev, value);

        return size;
}
int timed_output_dev_register(struct timed_output_dev *tdev)
{
        int ret;

        if (!tdev || !tdev->name || !tdev->enable || !tdev->get_time)
                return -EINVAL;

        ret = create_timed_output_class();
        if (ret < 0)
                return ret;

        tdev->index = atomic_inc_return(&device_count);
        tdev->dev = device_create(timed_output_class, NULL,
                MKDEV(0, tdev->index), NULL, tdev->name);
        if (IS_ERR(tdev->dev))
                return PTR_ERR(tdev->dev);

        ret = device_create_file(tdev->dev, &dev_attr_enable);
        if (ret < 0)
                goto err_create_file;

        dev_set_drvdata(tdev->dev, tdev);
        tdev->state = 0;
        return 0;

err_create_file:
        device_destroy(timed_output_class, MKDEV(0, tdev->index));
        pr_err("failed to register driver %s\n",
                        tdev->name);

        return ret;
}
EXPORT_SYMBOL_GPL(timed_output_dev_register);
```
###3. Android vibrator HAL
Its interface to linux device drivers call with the specified timeout in millisecond via `vibrator_on()`.

``` c hardware/libhardware_legacy/vibrator/vibrator.c

#define THE_DEVICE "/sys/class/timed_output/vibrator/enable"
static int sendit(int timeout_ms)
{
    int nwr, ret, fd;
    char value[20];

#ifdef QEMU_HARDWARE
    if (qemu_check()) {
        return qemu_control_command( "vibrator:%d", timeout_ms );
    }
#endif

    fd = open(THE_DEVICE, O_RDWR);
    if(fd < 0)
        return errno;

    nwr = sprintf(value, "%d\n", timeout_ms);
    ret = write(fd, value, nwr);

    close(fd);

    return (ret == nwr) ? 0 : -1;
}

int vibrator_on(int timeout_ms)
{
    /* constant on, up to maximum allowed time */
    return sendit(timeout_ms);
}

int vibrator_off()
{
    return sendit(0);
}
```

###4. Native layer through JNI between HAL and vibrator service
Controls of vibrator service reaches via `vibratorOn()`, `vibratorOff()`, and `vibratorExists()`.

``` c frameworks//base/services/core/jni/com_android_server_VibratorService.cpp
static void vibratorOn(JNIEnv *env, jobject clazz, jlong timeout_ms)
{
    // ALOGI("vibratorOn\n");
    vibrator_on(timeout_ms);
}

static void vibratorOff(JNIEnv *env, jobject clazz)
{
    // ALOGI("vibratorOff\n");
    vibrator_off();
}

static JNINativeMethod method_table[] = {
    { "vibratorExists", "()Z", (void*)vibratorExists },
    { "vibratorOn", "(J)V", (void*)vibratorOn },
    { "vibratorOff", "()V", (void*)vibratorOff }
};

int register_android_server_VibratorService(JNIEnv *env)
{
    return jniRegisterNativeMethods(env, "com/android/server/VibratorService",
            method_table, NELEM(method_table));
}
```

###5. Java Vibrator Service
In application layer before controling vibrator, applications have to get the access to the vibrator service.
The control goes into the vibrator service (*Framework Layer*) thru binder inteface.
For example, App. creates Vibrator object and start the vibration via `startVibrationLocked(vib)`.

Inside `startVibrationLocked()`:

If mTimeout != 0, then call the JNI function `vibratorOn()`.
else, call the code, which handle the rhythmic vibration pattern, which intern call the `vibratorOn()` through `VibrateThread()`.

``` java frameworks/base/services/core/java/com/android/server/VibratorService.java
    // Lock held on mVibrations
    private void startVibrationLocked(final Vibration vib) {
        try {
            if (mLowPowerMode
                    && vib.mUsageHint != AudioAttributes.USAGE_NOTIFICATION_RINGTONE) {
                return;
            }

            int mode = mAppOpsService.checkAudioOperation(AppOpsManager.OP_VIBRATE,
                    vib.mUsageHint, vib.mUid, vib.mOpPkg);
            if (mode == AppOpsManager.MODE_ALLOWED) {
                mode = mAppOpsService.startOperation(AppOpsManager.getToken(mAppOpsService),
                    AppOpsManager.OP_VIBRATE, vib.mUid, vib.mOpPkg);
            }
            if (mode != AppOpsManager.MODE_ALLOWED) {
                if (mode == AppOpsManager.MODE_ERRORED) {
                    Slog.w(TAG, "Would be an error: vibrate from uid " + vib.mUid);
                }
                mH.post(mVibrationRunnable);
                return;
            }
        } catch (RemoteException e) {
        }
        if (vib.mTimeout != 0) {
            doVibratorOn(vib.mTimeout, vib.mUid, vib.mUsageHint);
            mH.postDelayed(mVibrationRunnable, vib.mTimeout);
        } else {
            // mThread better be null here. doCancelVibrate should always be
            // called before startNextVibrationLocked or startVibrationLocked.
            mThread = new VibrateThread(vib);
            mThread.start();
        }
    }

    private void doVibratorOn(long millis, int uid, int usageHint) {
        synchronized (mInputDeviceVibrators) {
            if (DEBUG) {
                Slog.d(TAG, "Turning vibrator on for " + millis + " ms.");
            }
            try {
                mBatteryStatsService.noteVibratorOn(uid, millis);
                mCurVibUid = uid;
            } catch (RemoteException e) {
            }
            final int vibratorCount = mInputDeviceVibrators.size();
            if (vibratorCount != 0) {
                final AudioAttributes attributes = new AudioAttributes.Builder().setUsage(usageHint)
                        .build();
                for (int i = 0; i < vibratorCount; i++) {
                    mInputDeviceVibrators.get(i).vibrate(millis, attributes);
                }
            } else {
                vibratorOn(millis);
            }
        }
    }

```
