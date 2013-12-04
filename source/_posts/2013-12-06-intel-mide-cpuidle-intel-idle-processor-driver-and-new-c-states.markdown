---
layout: post
title: "Intel_mide: CPUIDLE: Intel_idle processor driver and new C-States"
date: 2013-12-06 17:34
comments: true
categories: [kernel, power] 
---
When there is nothing left to do, the CPUs will go into the idle state to wait until it is needed again. These idle modes, which go by names like "C states," vary in the amount of power saved, but also in the amount of ancillary information which may be lost and the amount of time required to get back into a fully-functional mode.  

Intel_idle driver actually performs idle state power management, and regsiters into existing CPU Idle subsystem and extends driver for Merrifield CPU (Slivermont). It also introduces a new platform idle states C7x deeper than traditional C6 state. The overall idea is that CPU C7x-states are extended to devices and rest of the platform, hence puting the platform to S0ix states.  

On intel mid platform (Merrifield), the PMU driver communicates with the Intel_idle driver, platform device drivers, pci drivers, and the PMU firmwares (NC: Punit, SC:SCU) to coordinate platform power state transitions.  

1. The PMU driver provides a platform-specific callback to the Intel_idle driver so that long periods of idleness can be extended to the entire platform. 
	* soc_s0ix_idle()->enter_s0ix_state() // intel_idle driver defined at /drivers/idle/intel_idle.c   
	* mid_s0ix_enter()		      // PMU driver defined at /arch/x86/platform/intel-mid/intel_soc_pmu.c  

	I could find out cpuidle_state->.enter() associated with soc_s0ix_idle() for C6 state on medfield platform, and the call stack looks like as above shown. However, I don't figure out the similar assoication between intel_idle and pmu driver on merrifield platform. I am curious if this statement is also appliciable on merrifield platform ??

2. Based on hint of idleness, the PMU driver extends CPU idleness to the reset of the platform via standard Linux PM_QoS, Runtime PM, and PCI PM calls.

3. Once the CPU and devices are all idle, the PMU driver programs the North and South Complex PMUs to implement the required power transitiion for S0ix to eumlate C7-x states.

Here is very good reference to understand the cpuidle governor and subsystem
http://lwn.net/Articles/384146/ before digging into how intel_idle driver fits with current cpuilde intrastructure.

The below is code trace for intel_idle driver located at "/drivers/idle/intel_idle.c"
- Comment in intel_idle driver
```
/*
 * intel_idle is a cpuidle driver that loads on specific Intel processors
 * in lieu of the legacy ACPI processor_idle driver.  The intent is to
 * make Linux more efficient on these processors, as intel_idle knows
 * more than ACPI, as well as make Linux more immune to ACPI BIOS bugs.
 */
```
- Driver Init.  
	* intel_idle_probe() starts to match current CPU with array of x86_cpu_ids via and assign corresponding default cpuidle C-state table  
	* intel_idle_cpuidle_driver_init() checks real C-state table and update its state when needed (eg, target_residency)
	* register the intel_dile driver with the cpudile subsystem through cpuidle_register_driver()  
	* intel_idle_cpu_init() allocates, initializes, and registers cpuidle_device for each CPU
	* register cpu_hotplug_notifer to know about CPUs going up/dow

``` c intel_idle_init()
static int __init intel_idle_init(void)                                                                                             
{
        int retval, i;

        /* Do not load intel_idle at all for now if idle= is passed */
        if (boot_option_idle_override != IDLE_NO_OVERRIDE)
                return -ENODEV;

        retval = intel_idle_probe();
        if (retval)
                return retval;

        intel_idle_cpuidle_driver_init();
        retval = cpuidle_register_driver(&intel_idle_driver);
        if (retval) {
                struct cpuidle_driver *drv = cpuidle_get_driver();
                printk(KERN_DEBUG PREFIX "intel_idle yielding to %s",
                        drv ? drv->name : "none");
                return retval;
        }

        intel_idle_cpuidle_devices = alloc_percpu(struct cpuidle_device);
        if (intel_idle_cpuidle_devices == NULL)
                return -ENOMEM;

        for_each_online_cpu(i) {
                retval = intel_idle_cpu_init(i);
                if (retval) {
                        cpuidle_unregister_driver(&intel_idle_driver);
                        return retval;
                }

                if (platform_is(INTEL_ATOM_BYT)) {
                        /* Disable automatic core C6 demotion by PUNIT */
                        if (wrmsr_on_cpu(i, CLPU_CR_C6_POLICY_CONFIG,
                                        DISABLE_CORE_C6_DEMOTION, 0x0))
                                pr_err("Error to disable core C6 demotion");
                        /* Disable automatic module C6 demotion by PUNIT */
                        if (wrmsr_on_cpu(i, CLPU_MD_C6_POLICY_CONFIG,
                                        DISABLE_MODULE_C6_DEMOTION, 0x0))
                                pr_err("Error to disable module C6 demotion");
                }
        }
        register_cpu_notifier(&cpu_hotplug_notifier);

        return 0;
}
```  

- Default cpuidle C-states for merrifield
	* "flags" field describes the characteristics of this sleep state  
		* CPUIDLE_FLAG_TIME_VALID should be set if it is possible to accurately measure the amount of time spent in this particular idle state.  
		* CPUIDLE_FLAG_TLB_FLUSHED is set to inidicate the HW flushes the TLB for this state.  
		* MWAIT takes an 8-bit "hint" in EAX "suggesting" the C-state (top nibble) and sub-state (bottom nibble). 0x00 means "MWAIT(C1)", 0x10 means "MWAIT(C2)" etc.  
	* "exit_latency" in US says how long it takes to get back to a fully functional state.  
	* "target_residency" in US is the minimum amount of time the processor should spend in this state to make the transition worth the effort.  
	* The "enter()" function will be called when the current governor decides to put the CPU into the given state  
``` c Merrifield CPUidle C states
#if defined(CONFIG_REMOVEME_INTEL_ATOM_MRFLD_POWER)
static struct cpuidle_state mrfld_cstates[CPUIDLE_STATE_MAX] = {
        { /* MWAIT C1 */
                .name = "C1-ATM",
                .desc = "MWAIT 0x00",
                .flags = MWAIT2flg(0x00) | CPUIDLE_FLAG_TIME_VALID,
                .exit_latency = 1,
                .target_residency = 4,
                .enter = &intel_idle },
        { /* MWAIT C4 */
                .name = "C4-ATM",
                .desc = "MWAIT 0x30",
                .flags = MWAIT2flg(0x30) | CPUIDLE_FLAG_TIME_VALID | CPUIDLE_FLAG_TLB_FLUSHED,
                .exit_latency = 100,
                .target_residency = 400,
                .enter = &intel_idle },
        { /* MWAIT C6 */
                .name = "C6-ATM",
                .desc = "MWAIT 0x52",
                .flags = MWAIT2flg(0x52) | CPUIDLE_FLAG_TIME_VALID | CPUIDLE_FLAG_TLB_FLUSHED,
                .exit_latency = 140,
                .target_residency = 560,
                .enter = &intel_idle },
        { /* MWAIT C7-S0i1 */
                .name = "S0i1-ATM",
                .desc = "MWAIT 0x60",
                .flags = MWAIT2flg(0x60) | CPUIDLE_FLAG_TIME_VALID | CPUIDLE_FLAG_TLB_FLUSHED,
                .exit_latency = 1200,
                .target_residency = 4000,
                .enter = &intel_idle },
        { /* MWAIT C9-S0i3 */
                .name = "S0i3-ATM",
                .desc = "MWAIT 0x64",
                .flags = MWAIT2flg(0x64) | CPUIDLE_FLAG_TIME_VALID | CPUIDLE_FLAG_TLB_FLUSHED,
                .exit_latency = 10000,
                .target_residency = 20000,
                .enter = &intel_idle },
        {
                .enter = NULL }
};
#else
#define mrfld_cstates atom_cstates
#endif

```
The actual performs given C-state transitions implemented by intel_idle(). As we saw, this is done through "enter()" functions associated with each state. The decision as to which idle state makes sense in a given situation is very much a policy issue implemented by the cpuidle "governors".

A call to enter() is a request from the current governor to put the CPU associated with dev into the given state. Note that enter() is free to choose a different state if there is a good reason to do so, but it should store the actual state used in the device's last_state field.
``` c intel_idle
/**                                                                                                                                 
 * intel_idle
 * @dev: cpuidle_device
 * @drv: cpuidle driver
 * @index: index of cpuidle state
 *
 * Must be called under local_irq_disable().
 */
static int intel_idle(struct cpuidle_device *dev,
                struct cpuidle_driver *drv, int index)
{
        unsigned long ecx = 1; /* break on interrupt flag */
        struct cpuidle_state *state = &drv->states[index];
        unsigned long eax = flg2MWAIT(state->flags);
        unsigned int cstate;
        int cpu = smp_processor_id();

#if (defined(CONFIG_REMOVEME_INTEL_ATOM_MRFLD_POWER) && \
        defined(CONFIG_PM_DEBUG))
        {
                /* Get Cstate based on ignore table from PMU driver */
                unsigned int ncstate;
                cstate =
                (((eax) >> MWAIT_SUBSTATE_SIZE) & MWAIT_CSTATE_MASK) + 1;
                ncstate = pmu_get_new_cstate(cstate, &index);
                eax     = flg2MWAIT(drv->states[index].flags);
        }
#endif
        cstate = (((eax) >> MWAIT_SUBSTATE_SIZE) & MWAIT_CSTATE_MASK) + 1;

        /*
         * leave_mm() to avoid costly and often unnecessary wakeups
         * for flushing the user TLB's associated with the active mm.
         */
        if (state->flags & CPUIDLE_FLAG_TLB_FLUSHED)
                leave_mm(cpu);

        if (!(lapic_timer_reliable_states & (1 << (cstate))))
                clockevents_notify(CLOCK_EVT_NOTIFY_BROADCAST_ENTER, &cpu);

        if (!need_resched()) {

                __monitor((void *)&current_thread_info()->flags, 0, 0);
                smp_mb();
                if (!need_resched())
                        __mwait(eax, ecx);
#if defined(CONFIG_REMOVEME_INTEL_ATOM_MDFLD_POWER) || \
        defined(CONFIG_REMOVEME_INTEL_ATOM_CLV_POWER)
                if (!need_resched() && is_irq_pending() == 0)
                        __get_cpu_var(update_buckets) = 0;
#endif
        }

        if (!(lapic_timer_reliable_states & (1 << (cstate))))
                clockevents_notify(CLOCK_EVT_NOTIFY_BROADCAST_EXIT, &cpu);

        return index;
}

```
