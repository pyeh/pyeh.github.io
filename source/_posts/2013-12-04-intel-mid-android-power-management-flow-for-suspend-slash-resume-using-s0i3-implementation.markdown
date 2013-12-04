---
layout: post
title: "Intel_mid: Android power management flow for suspend/resume using S0i3 implementation"
date: 2013-12-04 11:19
comments: true
categories: [android, kernel, power, intel] 
---

The main PMU driver (arch/x86/platform/intel-mid/intel_soc_pmu.c) hooks to support Linux PM suspend/resume flows as follows.  
The S0ix states are low-power active idle states that platform can be transitioned into.
- Register PMU driver as PCI device  
The PMU driver registers mid_suspend_ops via suspend_set_ops().
```c mid_pci_register_init 
/**
 * mid_pci_register_init - register the PMU driver as PCI device
 */
static struct pci_driver driver = {                                                                                                 
        .name = PMU_DRV_NAME,
        .id_table = mid_pm_ids,
        .probe = mid_pmu_probe,
        .remove = mid_pmu_remove,
        .shutdown = mid_pmu_shutdown
};

static int __init mid_pci_register_init(void)
{
        int ret;

        mid_pmu_cxt = kzalloc(sizeof(struct mid_pmu_dev), GFP_KERNEL);

        if (mid_pmu_cxt == NULL)
                return -ENOMEM;

        mid_pmu_cxt->s3_restrict_qos =
                kzalloc(sizeof(struct pm_qos_request), GFP_KERNEL);
        if (mid_pmu_cxt->s3_restrict_qos) {
                pm_qos_add_request(mid_pmu_cxt->s3_restrict_qos,
                         PM_QOS_CPU_DMA_LATENCY, PM_QOS_DEFAULT_VALUE);
        } else {
                return -ENOMEM;
        }

        init_nc_device_states();

        mid_pmu_cxt->nc_restrict_qos =
                kzalloc(sizeof(struct pm_qos_request), GFP_KERNEL);
        if (mid_pmu_cxt->nc_restrict_qos == NULL)
                return -ENOMEM;

        /* initialize the semaphores */
        sema_init(&mid_pmu_cxt->scu_ready_sem, 1);

        /* registering PCI device */
        ret = pci_register_driver(&driver);
        suspend_set_ops(&mid_suspend_ops);

        return ret;
} 
```
- mid_suspend_enter() performs the required S3 state over standby_enter()  
check_nc_sc_status() hooked by mrfld_nc_sc_status_check (defind at arch/x86/platform/intel-mid/intel_soc_mrfld.c) is to check north complex (NC) and soutch complex (SC) device status. Return true if all NC and SC devices are in D0i3.
``` c mid_suspend_enter() 
static const struct platform_suspend_ops mid_suspend_ops = {                                                                        
        .begin = mid_suspend_begin,
        .valid = mid_suspend_valid,
        .prepare = mid_suspend_prepare,
        .prepare_late = mid_suspend_prepare_late,
        .enter = mid_suspend_enter,
        .end = mid_suspend_end,
};

static int mid_suspend_enter(suspend_state_t state)                                                                                 
{
        int ret;

        if (state != PM_SUSPEND_MEM)
                return -EINVAL;

        /* one last check before entering standby */
        if (pmu_ops->check_nc_sc_status) {
                if (!(pmu_ops->check_nc_sc_status())) {
                        trace_printk("Device d0ix status check failed! Aborting Standby entry!\n");
                        WARN_ON(1);
                }
        }

        trace_printk("s3_entry\n");
        ret = standby_enter();
        trace_printk("s3_exit %d\n", ret);
        if (ret != 0)
                dev_dbg(&mid_pmu_cxt->pmu_dev->dev,
                                "Failed to enter S3 status: %d\n", ret);

        return ret;
}

```

- standby_enter() performs requried S3 state 
	- mid_state_to_sys_state() maps power states to driver's internal indexes.  
	- mid_s01x_enter() performs required S3 state
	- __mwait(mid_pmu_cxt->s3_hint, 1) is issued with a hint to enter S0i3(S3 emulation using S0i3, as I observed that MRFLD_S3_HINT with 0x64 is same as MID_S0I3_STATE with 0x64). When both core issue an mwait C7, it is a hint provided by the idle driver to enter an S0ix state.
	- This triggers S0i3 entry, but the decision and policy is selected by SCU FW.
``` c standby_enter()
static int standby_enter(void)
{
        u32 temp = 0;
        int s3_state = mid_state_to_sys_state(MID_S3_STATE);

        if (mid_s0ix_enter(MID_S3_STATE) != MID_S3_STATE) {
                pmu_set_s0ix_complete();
                return -EINVAL;
        }

        /* time stamp for end of s3 entry */
        time_stamp_for_sleep_state_latency(s3_state, false, true);

        __monitor((void *) &temp, 0, 0);
        smp_mb();
        __mwait(mid_pmu_cxt->s3_hint, 1);

        /* time stamp for start of s3 exit */
        time_stamp_for_sleep_state_latency(s3_state, true, false);

        pmu_set_s0ix_complete();

        /*set wkc to appropriate value suitable for s0ix*/
        writel(mid_pmu_cxt->ss_config->wake_state.wake_enable[0],
                       &mid_pmu_cxt->pmu_reg->pm_wkc[0]);
        writel(mid_pmu_cxt->ss_config->wake_state.wake_enable[1],
                       &mid_pmu_cxt->pmu_reg->pm_wkc[1]);

        mid_pmu_cxt->camera_off = 0;
        mid_pmu_cxt->display_off = 0;

        if (platform_is(INTEL_ATOM_MRFLD))
                up(&mid_pmu_cxt->scu_ready_sem);

        return 0;
}
```
- mid_s01x_enter() 
	- pmu_prepare_wake() will mask wakeup from AONT timers for s3. If s3 is aborted for any reason, we don't want to leave AONT timers masked until next suspend, otherwise if next to happen is s0ix, no timer could wakeup SoC from s0ix and we might miss to kick the kernel watchdog.
	- enter() hooked by mrfld_pmu_enter (defined at arch/x86/platform/intel-mid/intel_soc_mrfld.c). Compared to Medfield and Covertail platform, PM_CMD is not required to send to SCU. 
``` c mid_s01x_enter()
int mid_s0ix_enter(int s0ix_state)
{
        int ret = 0;

        if (unlikely(!pmu_ops || !pmu_ops->enter))
                goto ret;

        /* check if we can acquire scu_ready_sem
         * if we are not able to then do a c6 */
        if (down_trylock(&mid_pmu_cxt->scu_ready_sem))
                goto ret;

        /* If PMU is busy, we'll retry on next C6 */
        if (unlikely(_pmu_read_status(PMU_BUSY_STATUS))) {
                up(&mid_pmu_cxt->scu_ready_sem);
                pr_debug("mid_pmu_cxt->scu_read_sem is up\n");
                goto ret;
        }

        pmu_prepare_wake(s0ix_state);

        /* no need to proceed if schedule pending */
        if (unlikely(need_resched())) {
                pmu_stat_clear();
                up(&mid_pmu_cxt->scu_ready_sem);
                goto ret;
        }

        /* entry function for pmu driver ops */
        if (pmu_ops->enter(s0ix_state))
                ret = s0ix_state;

ret:
        return ret;
} 


```
