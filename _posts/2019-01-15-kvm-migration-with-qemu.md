# KVM Migration using Qemu:

Generally VMs would be migrated from one host to other for various reasons
explained in [introduction To KVM Migration](https://balamuruhans.github.io/2018/11/13/introduction-to-kvm-migration.html)
but VMs can be migrated with in local host as well using Qemu which is used
for *Qemu live patching*, testing & debugging etc.,

Let us discuss about the Qemu live patching feature, how to perform Qemu Local
host migration in this document.

__Qemu live patching:__

  Qemu live patching is a feature to upgrade the Qemu without bringing down the
  running VM instance, it can be achieved using postcopy live migration.
  Migration can be triggered from [qemu monitor interface](https://balamuruhans.github.io/2019/01/16/qemu-monitor-interface.html),

  Source:
  ```
  # qemu-system-ppc64 --enable-kvm --nographic -vga none -machine pseries -m 4G,slots=32,maxmem=32G -smp 16,maxcpus=32 -device virtio-blk-pci,drive=rootdisk -drive file=/var/lib/libvirt/images/avocado/data/avocado-vt/images/rhel72-ppc64le.qcow2,if=none,cache=none,format=qcow2,id=rootdisk -monitor telnet:127.0.0.1:1234,server,nowait -net nic,model=virtio -net user -redir tcp:2000::22
  ```
  Fire another Qemu commanline with new qemu binary as target with `-incoming`
  option, set migrate_set_capability postcopy-ram on in both source and target before
  triggering the migration from source

  ```
  # telnet 127.0.0.1 1234
  Trying 127.0.0.1...
  Connected to 127.0.0.1.
  Escape character is '^]'.
  QEMU 2.6.0 monitor - type 'help' for more information
  (qemu) migrate_set_capability postcopy-ram on
  (qemu) migrate -d tcp:localhost:4444
  (qemu) migrate_start_postcopy
  ```
  Target:
  1. Use newer qemu binary that have to be live patched
  2. Use -incoming option to make qemu listen for vm to be migrated
  3. use different port for monitor

  ```
  # /home/qemu-softmmu <same options in source> -monitor telnet:127.0.0.1:1235,server,nowait -incoming tcp:0:4444
  ```

  set migrate_set_capability postcopy-ram in target,

  ```
  # telnet 127.0.0.1 1235
  Trying 127.0.0.1...
  Connected to 127.0.0.1.
  Escape character is '^]'.
  QEMU 2.6.0 monitor - type 'help' for more information
  (qemu) migrate_set_capability postcopy-ram on
  ```

  once the migration is success, the VM is live and runs in patched QEMU.


  __localhost migration__:

  If the same above mentioned migration is used with same Qemu binary in source
  and target then it is localhost postcopy migration. If migrate_set_capability
  postcopy-ram is not enabled then precopy migration is done as below,

  Source:
  ```
  # qemu-system-ppc64 --enable-kvm --nographic -vga none -machine pseries -m 4G,slots=32,maxmem=32G -smp 16,maxcpus=32 -device virtio-blk-pci,drive=rootdisk -drive file=/var/lib/libvirt/images/avocado/data/avocado-vt/images/rhel72-ppc64le.qcow2,if=none,cache=none,format=qcow2,id=rootdisk -monitor telnet:127.0.0.1:1234,server,nowait -net nic,model=virtio -net user -redir tcp:2000::22
  ```

  Fire another Qemu commanline with new qemu binary as target with `-incoming`
  option,

  ```
  # telnet 127.0.0.1 1234
  Trying 127.0.0.1...
  Connected to 127.0.0.1.
  Escape character is '^]'.
  QEMU 2.6.0 monitor - type 'help' for more information
  (qemu) migrate -d tcp:localhost:4444
  (qemu) migrate_start_postcopy
  ```

  Target:
  1. Use -incoming option to make qemu listen for vm to be migrated
  2. use different port for monitor

  ```
  # qemu-system-ppc64 <same options in source> -monitor telnet:127.0.0.1:1235,server,nowait -incoming tcp:0:4444
  ```

Local host migration from Libvirt is not supported which is explained in
libvirt mailing list discussion series,
`https://www.redhat.com/archives/libvir-list/2017-September/msg00441.html`
so suggested way to perform Qemu upgrade in production environment using Libvirt
is,
  1. live migrate the VMs to different host (Postcopy is always the better option)
  2. upgrade the Qemu in the source host
  3. live migrate back the VMs to source host (Qemu always ensures to support
    migration from older Qemu to newer Qemu)
