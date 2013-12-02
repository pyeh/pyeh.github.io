---
layout: post
title: "Intel_mid: Introduction of android watchdogs and kernel watchdogs"
date: 2013-12-02 17:29
comments: true
categories: [android, kernel, intel]
---
Watchdogs monitor software and hardware device and prevent whole system from hanging. After looking into android BSP running on intel mid, merrifield, platform, I will try to classfiy watchdog into the following types. The first two belong to native Android supporting, and the last three are specified to intel-mid platform.

##1. Android framework's Java* watchdog
Deal with cases when any of the following locks is held for more than a minute or when ServerThread is busy.

-- ThermalManagerService  
-- PowerManagerService  
-- WindowMangerService  
-- MountService  
-- NetworkManagementService  
-- ActivityMangerService  

If one of above services hangs for one minute, the java watchdog kills it and results in restarting android's framework by killing the SystemServer.
##2. Device-critical services
Critical services are declared as "critical" in the corresponding rc files (eg, ueventd, servicemanager). If critical services exist or crash more than four times in four minutes, the device will reboot into recovery mode. This feature is handled by the init process.

##3. Kernel watchdog leads to COLD_RESET
The kernel watchdog prvents the operating system from hanging. The System Control Unit (SCU firmware) resets the platform when the kernel cannot schedule the watchdog daemon (/usr/bin/ia_watchdogd).

The driver located at /drivers/watchdog/intel_scu_watchdog_evo.c provides a /dev/watchdog device to access the kernel watchdog and ioctls to configure the timer. Since the SCU provides the functionality, all access to watchdog features are routed to the SCU via an IPC (see more PIC regitrations at arch/x86/platform/intel-mid/intel_mid_scu.c)

* Init code installs the driver
``` c watchdog_rpmsg_init()
static struct rpmsg_driver watchdog_rpmsg = {
        .drv.name       = KBUILD_MODNAME,
        .drv.owner      = THIS_MODULE,
        .id_table       = watchdog_rpmsg_id_table,
        .probe          = watchdog_rpmsg_probe,
        .callback       = watchdog_rpmsg_cb,
        .remove         = watchdog_rpmsg_remove,
};

static int __init watchdog_rpmsg_init(void)
{
        if (intel_mid_identify_cpu() == INTEL_MID_CPU_CHIP_TANGIER)
                return register_rpmsg_driver(&watchdog_rpmsg);
        else {                                                                                                                      
                pr_err("%s: watchdog driver: bad platform\n", __func__);
                return -ENODEV;
        }
}

#ifdef MODULE
module_init(watchdog_rpmsg_init);
#else
rootfs_initcall(watchdog_rpmsg_init);
#endif
```

* create the /dev/watchdog only if the disabled_kernel_watchdog module parameter is not set. It gets the timer's configuration, registers reboot notifier, registers dump handler to irq#15, and adds sysfs/debugfs entries. 
``` c watchdog_rpmsg_probe()->intel_scu_watchdog_init()
/* Init code */
static int intel_scu_watchdog_init(void)
{
        int ret = 0;

        watchdog_device.normal_wd_action   = SCU_COLD_RESET_ON_TIMEOUT;
        watchdog_device.reboot_wd_action   = SCU_COLD_RESET_ON_TIMEOUT;
        watchdog_device.shutdown_wd_action = SCU_COLD_OFF_ON_TIMEOUT;

#ifdef CONFIG_DEBUG_FS
        watchdog_device.panic_reboot_notifier = false;
#endif /* CONFIG_DEBUG_FS */

        /* Initially, we are not in shutdown mode */
        watchdog_device.shutdown_flag = false;

        /* Check timeouts boot parameter */
        if (check_timeouts(pre_timeout, timeout)) {
                pr_err("%s: Invalid timeouts\n", __func__);
                return -EINVAL;
        }

        /* Reboot notifier */
        watchdog_device.reboot_notifier.notifier_call = reboot_notifier;
        watchdog_device.reboot_notifier.priority = 1;
        ret = register_reboot_notifier(&watchdog_device.reboot_notifier);
        if (ret) {
                pr_crit("cannot register reboot notifier %d\n", ret);
                goto error_stop_timer;
        }

        /* Do not publish the watchdog device when disable (TO BE REMOVED) */
        if (!disable_kernel_watchdog) {
                watchdog_device.miscdev.minor = WATCHDOG_MINOR;
                watchdog_device.miscdev.name = "watchdog";
                watchdog_device.miscdev.fops = &intel_scu_fops;

                ret = misc_register(&watchdog_device.miscdev);
                watchdog_device.miscdev.fops = &intel_scu_fops;

                ret = misc_register(&watchdog_device.miscdev);
                if (ret) {
                        pr_crit("Cannot register miscdev %d err =%d\n",
                                WATCHDOG_MINOR, ret);
                        goto error_reboot_notifier;
                }
        }

        /* MSI #15 handler to dump registers */
        handle_mrfl_dev_ioapic(EXT_TIMER0_MSI);
        ret = request_irq((unsigned int)EXT_TIMER0_MSI,
                watchdog_warning_interrupt,
                IRQF_SHARED|IRQF_NO_SUSPEND, "watchdog",
                &watchdog_device);
        if (ret) {
                pr_err("error requesting warning irq %d\n",
                       EXT_TIMER0_MSI);
                pr_err("error value returned is %d\n", ret);
                goto error_misc_register;
        }

#ifdef CONFIG_INTEL_SCU_SOFT_LOCKUP
        init_timer(&softlock_timer);
#endif

        if (disable_kernel_watchdog) {
                pr_err("%s: Disable kernel watchdog\n", __func__);

                /* Make sure timer is stopped */
                ret = watchdog_stop();
                if (ret != 0)
                        pr_debug("cant disable timer\n");
        }

#ifdef CONFIG_DEBUG_FS
        ret = create_debugfs_entries();  
        if (ret) {
                pr_err("%s: Error creating debugfs entries\n", __func__);
                goto error_debugfs_entry;
        }
#endif

        watchdog_device.started = false;

        ret = create_watchdog_sysfs_files();
        if (ret) {
                pr_err("%s: Error creating debugfs entries\n", __func__);
                goto error_sysfs_entry;
        }

        return ret;

error_sysfs_entry:
        /* Nothing special to do */
#ifdef CONFIG_DEBUG_FS
error_debugfs_entry:
        /* Remove entries done by create function */
#endif

error_misc_register:
        misc_deregister(&watchdog_device.miscdev);

error_reboot_notifier:
        unregister_reboot_notifier(&watchdog_device.reboot_notifier);

error_stop_timer:
        watchdog_stop();

        return ret;
}
```

* interrupt handler related to pre-timeout dumps kernel backtraces.
``` c watchdog_warning_interrupt
/* warning interrupt handler */ 
static irqreturn_t watchdog_warning_interrupt(int irq, void *dev_id) 
{ 
        pr_warn("[SHTDWN] %s, WATCHDOG TIMEOUT!\n", __func__); 
 
        /* Let's reset the platform after dumping some data */ 
        trigger_all_cpu_backtrace(); 
        panic("Kernel Watchdog"); 
 
        /* This code should not be reached */ 
        return IRQ_HANDLED; 
} 

```

* When power transisition happens, reboot_notifier is called for re-configuring watchdog timeouts and its default behavior.
COLD_RESET is set to reboot, and COLD_OFF is set to poewr halt and off. In case of a stucking rebooting or shutdown procedure, the platform will still could execute reset or power-off seperately.
``` c reboot_notifier
/* Reboot notifier */
static int reboot_notifier(struct notifier_block *this,
                           unsigned long code,
                           void *another_unused)
{
        int ret;

        if (code == SYS_RESTART || code == SYS_HALT || code == SYS_POWER_OFF) {
                pr_warn("Reboot notifier\n");

                if (watchdog_set_appropriate_timeouts())
                        pr_crit("reboot notifier cant set time\n");

                switch (code) {
                case SYS_RESTART:
                        ret = watchdog_set_reset_type(
                                watchdog_device.reboot_wd_action);
                        break;

                case SYS_HALT:
                case SYS_POWER_OFF:
                        ret = watchdog_set_reset_type(
                                watchdog_device.shutdown_wd_action);
                        break;
                }
                if (ret)
                        pr_err("%s: could not set reset type\n", __func__);

#ifdef CONFIG_DEBUG_FS
                /* debugfs entry to generate a BUG during
                any shutdown/reboot call */
                if (watchdog_device.panic_reboot_notifier)
                        BUG();
#endif
                /* Don't do instant reset on close */
                reset_on_release = false;

                /* Kick once again */
                if (disable_kernel_watchdog == false) {
                        ret = watchdog_keepalive();
                        if (ret)
                                pr_warn("%s: no keep alive\n", __func__);

                        /* Don't allow any more keep-alives */
                        watchdog_device.shutdown_flag = true;
                }
        }
        return NOTIFY_DONE;
}
```

##4. Userspace watchdog daemon
Source codes are located at /hardware/ia_watchdog/watchdog_daemon folder) and target location is /usr/bin/ia_watchdogd. This daemon is declared as one-shot service in the rc file (init.watchdog.rc) and perform the following steps:

-- open the watchdog device /dev/watchdog.  
-- configure the pre_timeout with 75 seconds and timeout with 90 seconds.  
-- Loop forever. In the loop, kick the watchdog device (by writing to 'R' to /dev/watchdog) every 60 seconds.  
##5. SCU watchdog leads to PLATFORM_RESET(deep reset)
This prevents the platform stucking on SCU by issuing a PLATFORM_RESET because the interface between the SCU and PMIC is broken.
