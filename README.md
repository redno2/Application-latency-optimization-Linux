# Application-latency-optimization-Linux

This is the technical part from my medium article "TODO url medium"
For more information about the options below, please see the references at the end of the medium article.


## Technical Recommendations

The are the resulting of technical recommendations.

### Bios
The modification of the bios parameters was done by following the Lenovo SR-630 documentation and other online documentation. If you want go deeper please read the documentation at the references part on the medium atricle.

```Hyper-Threading (HT) → DISABLED 
VT-d (Intel virtualization technologies) → DISABLED
Intel VT for directed I/O (VT-d) → DISABLED
Patrol Scrubbing → OFF
Power workload configuration → I/O sensitive
Turbo mode → ENABLED
CPU P-States → DISABLED
C-States → DISABLED
Prefetcher → ENABLED
UPI Link speed → Maximum performance
Memory speed → Maximum performance
```

### Grub parameters
Testing was done based on kernel v.5.3.0–40, for both generic and low latency. And configuration to isolate CPU 12 to 23
The specific configuration to be added in `/etc/default/grub`


Time Stamp Counter (TSC) which is the default clocksource=tsc, but fix it

`clocksource=tsc`

Prevents the clock from entering in deeper C-states

`processor.max_cstate=1`

Disabling CPU Power Saving States

`intel_idle.max_cstate=0`

Prevents the Intel idle driver from managing power state and CPU frequency

`intel_pstate=disable`

Isolate a given set of CPUs from disturbance isolcpus=12–23 (deprecated, but currently cpuset doesn't work correctly)

`isolcpus=12-23`

Turns off the timer tick on an idle CPU

`nohz=on`

Turns off the timer tick on a CPU when there is only one runnable task on that CPU; needs nohz to beset to on

`nohz_full=12–23`

To allow the user to move all RCU offload threads to a housekeepingCPU;

`rcu_nocbs=12–23`

Reduces contention on these kernel locks. The parameterensures that the ticks per CPU do not occur simultaneously by making their start times 'skewed'.Skewing the start times of the per-CPU timer ticks decreases the potential for lock conflicts, reducing system jitter for interrupt response times.

`skew_tick=1`

Forces the clock to avoid entering the idle state

`idle=poll`

Prevents the kernel from detecting soft lockups in user threads

`nosoftlockup`

#### Example 
set GRUB_CMDLINE_LINUX_DEFAULT in /etc/default/grub at the following for the configuration for CPU 5 to 23.

`GRUB_CMDLINE_LINUX_DEFAULT="console=tty1 console=ttyS1,115200n8 nomodeset net.ifnames=0 biosdevname=0 clocksource=tsc processor.max_cstate=1 intel_idle.max_cstate=0 intel_pstate=disable isolcpus=5-23 nohz=on nohz_full=5-23 rcu_nocbs=5-23 skew_tick=1 idle=poll nosoftlockup"`

And update grub

`root@host:~# update-grub`


### System configuration
The specific configuration to be added in `/etc/sysctl.conf`

The total bandwidth available to all real-time tasks. The default values is 950,000 μs (0.95 s) or, in other words, 95% of the CPU bandwidth. Setting the value to -1 means that real-time tasks may use up to 100% of CPU times. This is only adequate when the real-time tasks are well engineered and have no obvious caveats such as unbounded polling loops. 
So to be safe, is set more than the default but not unlimited

`kernel.sched_rt_runtime_us = 1000000`

When a task in D state did not get scheduled for more than this value report a warning. This file shows up if CONFIG_DETECT_HUNG_TASK is enabled.
0: means infinite timeout - no checking done. Possible values to set are in range {0..LONG_MAX/HZ}.

`kernel.hung_task_timeout_secs = 600`

The hard lockup detector monitors each CPU for its ability to respond to timer interrupts. The mechanism utilizes CPU performance counter registers that are programmed to generate Non-Maskable Interrupts (NMIs) periodically while a CPU is busy. Hence, the alternative name 'NMI watchdog'.

`kernel.nmi_watchdog = 0`

Enables/disables automatic NUMA memory balancing. On NUMA machines, there is a performance penalty if remote memory is accessed by a CPU. When this feature is enabled the kernel samples what task thread is accessing memory by periodically unmapping pages and later trapping a page fault. At the time of the page fault, it is determined if the data being accessed should be migrated to a local memory node.

`kernel.numa_balancing=0` 

The time interval between which vm statistics are updated. The default is 1 second. On-demand vmstat workers commit prevents OS jitter due to vmstat_update()

`vm.stat_interval = 10`

This is used to force the Linux VM to keep a minimum number of kilobytes free. The VM uses this number to compute a watermark[WMARK_MIN] value for each lowmem zone in the system. Each lowmem zone gets a number of reserved free pages based proportionally on its size.

`vm.min_free_kbytes=1024000`

#### IRQ Balance
Remove isolate CPUs from irqbalance in /etc/default/irqbalance
`IRQBALANCE_BANNED_CPUS=FFF000 #Convert bin to hex cpu number`

#### Network
And the first network part added in `/etc/sysctl.conf`
One basic tweak for network. More incoming in a next article.
Disable the TCP selective acks option for better throughput
There is some research available in the networking community which shows enabling SACK on high-bandwidth links can cause unnecessary CPU cycles to be spent calculating SACK values, reducing overall efficiency of TCP connections. This research implies these links are so fast, the overhead of retransmitting small amounts of data is less than the overhead of calculating the data to provide as part of a Selective Acknowledgment.Unless there is high latency or high packet loss, it is most likely better to keep SACK turned off over a high performance network.

`net.ipv4.tcp_sack=0`

## How to use it
Install some packages

`apt install schedtool numactl util-linux numad linux-tools-$(uname -r) linux-tools-generic`

This setup imply you to use a scheduler and affinity for your specific process. For that I use "chrt" and "numactl" or "taskset"
Start your process get the PID and apply config on it. Below it set the SCHED_FIFO on it and it's children, and set the priority at 99 (RT).

`chrt -f -a -p 99 $PID`

And to push the affinity on this PID you have to specify which CPU need to be used. Like with taskset.

`taskset -a -cp 12-23 $PID`

Or use numactl to control that it's, more performing.



We set out to reduce jitter/latency on our Forex Trading applications to the max, the focus was on the operating system level as well as the JVM configuration and choice itself.

