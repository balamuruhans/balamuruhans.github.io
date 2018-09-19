Introduction to KVM Migration

KVM migration is the movement of a virtual machine (guest) from one
physical host to another physical host, where both hosts are using Qemu (userspace system emulator) as hypervisor in KVM hosting mode.

In KVM hosting mode, Qemu uses kernel module KVM (Kernel based Virtual Machine) as accelerator to virtualize guest if both host and guest are in same
architecture. Qemu emulates the hardware and execution of instructions from
guests are done on host using KVM.

KVM Guest migration is used mainly for,

Load balancing – If host gets overloaded, guests can be moved to other host
which are not utilized much.

Energy saving – guests can be moved from multiple host to single host and power
off hosts which are not used.

Maintenance – If host have to be shut-off for maintenance or to upgrade etc.,

Migration can categorized as live, online and offline based on the state in which guest remains in source and destination host during migration.

Live migration – guest remains to in running state in source, guest is booted and remains to be in paused state in destination host while migration is in progress. Once the migration is completed guest starts its execution from destination host without downtime.

Online migration – guest remains in paused state in source and in destination host, once  migration completes it resumes in destination host. It takes comparatively less time to migrate as the there is no memory dirtying during migration as guest remains to be in paused state but there will be downtime equal to the migration time.

Offline migration – guest can be in running or in shutoff state in source, offline migration would just define a guest in destination in the destination host and it would remain in shutoff state.

Functionally KVM migration is of two types,

1. Precopy migration

– Mark all the memory pages as dirty and copy it to the destination
– Copy memory pages dirtied during previous copy and iterate till the
the convergence (it is the condition for migration with `downtime limit`, based on the network bandwidth and number of dirty pages yet to be transferred ETA [expected downtime] is calculated and it should be with in downtime limit to handover the VM to destination. we have other configurable migration parameters based on customer Service Level Agreement (SLA) that can affect this convergence).
– Stop the vm after convergence is reached in that particular instance to copy rest of memory pages and other vm states
– Resume the VM in the destination

Precopy migration can never end, if the rate at which pages gets dirtied in source host is faster than the pages copied to the destination host. It can happen for memory sensitive loads or due to lower network bandwidth. So to address this drawback Postcopy migration got implemented.

2. Postcopy migration (It works only on Live migration)

In Postcopy migration, minimum states and resources required for the guest to
start is copied to destination host and started in destination. so that the pages that gets dirtied will be on destination and other memory pages are copied in background on demand.


– Stop the vm in the source and copy non memory vm states to destination
to resume in destination
– Copy memory pages on demand/in background using Asynchrounous Page
faults

Asynchronous Page Fault:
If a page that vcpu is trying to access is swapped out KVM sends async PF to the vcpu and continues vcpu execution. Requested page is swapped in by another thread in parallel. When vcpu gets async PF it puts the task that faulted to sleep until “wake up” interrupt is delivered. When page is brought to host memory KVM sends “wake up” interrupt and the guest task resumes execution.

userfaultfd() system call is implemented to handle page faults from userspace for the KVM guest migration. The running guest can move to a new host while leaving its memory behind, speeding the migration. When that guest starts faulting in the missing pages, the user-space mechanism can pull them across the net and store them in the guest’s address space. The result is quick migration without the need to put any sort of page-migration protocol into the kernel.
