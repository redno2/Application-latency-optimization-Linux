# Linux-latency-optimization

This is a technical part from my medium article

We set out to reduce jitter/latency on our Forex Trading applications to the max, the focus was on the operating system level as well as the JVM configuration and choice itself.

For example set GRUB_CMDLINE_LINUX_DEFAULT in /etc/default/grub
`GRUB_CMDLINE_LINUX_DEFAULT="console=tty1 console=ttyS1,115200n8 nomodeset net.ifnames=0 biosdevname=0 clocksource=tsc processor.max_cstate=1 intel_idle.max_cstate=0 intel_pstate=disable isolcpus=5-23 nohz=on nohz_full=5-23 rcu_nocbs=5-23 skew_tick=1 idle=poll nosoftlockup"`