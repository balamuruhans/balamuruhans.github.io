# MultiVM KVM migration using Avocado-VT

In a real time cloud scenarios, there would be multiple VMs running with
production load with different operating systems/distributions and VM migration
occurs in bulk, random manner in order to handle load balancing, maintenance and
power saving. This environment can be simulated and tested with Avocado-VT.

Pre-requisite:

Below links gives better understanding on Avocado and Avocado-VT
1. [Test KVM with Avocado-VT](https://sathnaga86.com/2018/05/17/testing-kvm-through-libvirt-environment.html)
2. [MultiVM KVM test with Avocado-VT](https://sathnaga86.com/2018/05/18/kvm-libvirt-multivm-tests-using-avocado.html)

Test design:

The framework have the capability to run the tests for days and with any number
of VMs (we use 10 VMs),

- stress Qemu/Libvirt by migrating VMs one by one, parallel, cross migration
(i.e. migrating VMs from source to target & target to source at sametime) along
with to and fro continuously 5 times (configurable to any number of times).

- stress Kernel in different distros by running CPU/memory/IO stress tools in
VMs, in hosts(source/target/both) and also in both VMs and Hosts.

Code:

* [Test script](https://github.com/autotest/tp-libvirt/blob/master/libvirt/tests/src/virsh_cmd/domain/virsh_migrate_stress.py)
* [Test config](https://github.com/autotest/tp-libvirt/blob/master/libvirt/tests/cfg/virsh_cmd/domain/virsh_migrate_stress.cfg)

Test run:
```
# avocado run --vt-type libvirt --vt-config /home/workspace/runAvocadoFVTTest/avocado-fvt-wrapper/data/avocado-vt/backends/libvirt/cfg/migrate_stress.cfg --vt-only-filter "RHEL.8.0.ppc64le qcow2 virtio_scsi"

JOB ID     : 74df5dcfa787162f36d27488fd7edb91bb267c05
JOB LOG    : /home/workspace/runAvocadoFVTTest/avocado-fvt-wrapper/results/job-2019-07-08T04.04-74df5dc/job.log
 (1/1) guest_import.qemu.qcow2.virtio_scsi.smp2.virtio_net.Guest.RHEL.8.0.ppc64le.powerkvm-qemu.unattended_install.import.import.default_install.aio_native:  PASS (3123.31 s)
RESULTS    : PASS 1 | ERROR 0 | FAIL 0 | SKIP 0 | WARN 0 | INTERRUPT 0 | CANCEL 0
JOB TIME   : 3126.89 s
JOB HTML   : /home/workspace/runAvocadoFVTTest/avocado-fvt-wrapper/results/job-2019-07-08T04.04-74df5dc/results.html

```

`--vt-config` is [config file](https://raw.githubusercontent.com/balamuruhans/avocado-vt/master/test_configs/migrate_stress.cfg) provided to avocado for picking up tests and to override test
parameters from `Test config` mentioned  in *Code* section above.


Test Flow:

1. It initially clones the guest qcow2 images from base image for every VM
according to the OS distributions.

  config file provided in --vt-config have respective parameters set for Avocado
  to perform it [a]
  ```
  # No of VMs to be used for tests
  main_vm = "vms1"
  vms = "vms1 vms2 vms3 vms4 vms5 vms6 vms7 vms8 vms9 vms10"

  # image name to be used after cloning
  image_name_vms1 = "images/rhel77-ppc64le-vms1"
  image_name_vms2 = "images/rhel76-ppc64le-vms2"
  image_name_vms3 = "images/rhel80-ppc64le-vms3"
  image_name_vms4 = "images/rhel80-ppc64le-vms4"
  image_name_vms5 = "images/rhel77-ppc64le-vms5"
  image_name_vms6 = "images/rhel77-ppc64le-vms6"
  image_name_vms7 = "images/rhel76alt-ppc64le-vms7"
  image_name_vms8 = "images/rhel76alt-ppc64le-vms8"
  image_name_vms9 = "images/rhel76-ppc64le-vms9"
  image_name_vms10 = "images/rhel76-ppc64le-vms10"

  # source master base image
  master_images_clone = "rhel80-ppc64le"
  master_image_name_vms1 = "images/rhel77-ppc64le"
  master_image_name_vms2 = "images/rhel76-ppc64le"
  master_image_name_vms3 = "images/rhel80-ppc64le"
  master_image_name_vms4 = "images/rhel80-ppc64le"
  master_image_name_vms5 = "images/rhel77-ppc64le"
  master_image_name_vms6 = "images/rhel77-ppc64le"
  master_image_name_vms7 = "images/rhel76alt-ppc64le"
  master_image_name_vms8 = "images/rhel76alt-ppc64le"
  master_image_name_vms9 = "images/rhel76-ppc64le"
  master_image_name_vms10 = "images/rhel76-ppc64le"
  ```

2. Bring up 10 VMs by importing using virt-install commandline after creating
  NFS setups and copying all the guest images to NFS location [b]
  ```
  # import VMs using virt-install
  create_vm_libvirt = "yes"
  kill_vm_libvirt = "yes"
  kill_vm = "yes"

  # NFS related configurations
  disk_target = sda
  setup_local_nfs = yes
  storage_type = nfs
  nfs_mount_dir=/home/bala/sharing
  images_base_dir = "/home/bala/sharing"
  nfs_mount_options="rw"
  export_options=rw,sync,no_root_squash
  nfs_mount_src=/home/bala/NFS
  export_dir=/home/bala/NFS  
  ```

3. Create passwordless ssh with local host and remote host to mount NFS shared
  path in target host and to migrate seamlessly without password prompt
  ```
  # params to set local and remote server credentials
  local_ip = "192.168.5.3"
  local_pwd = "password"
  remote_user = root
  remote_ip = "192.168.5.2"
  remote_pwd = "password"
  ```

4. Allow selinux to permit libvirt to use NFS shared location in source and
  target host.

5. Open ports 49152~49216 using firewall-cmd in target host while migrating
  from source to target and in source host while migrating back to source.

6. Configure, install and run stress inside all the VMs,
  ```
  # nohup stress --cpu 4 --quiet --timeout 3600 > /dev/null &
  ```

7. As pre-migration check, VM network connection with pings and records
  uptime of each VM before migration.

8. Performs 5 times to and fro Postcopy migration one by one from source to target and
  from target to source.

  Source -> Target:
  ```
  2019-07-08 04:11:27,448 libvirt_vm       L2043 INFO | Migrating VM vms10 from qemu:///system to qemu+ssh://192.168.5.3/system
  2019-07-08 04:11:27,449 virsh            L0655 DEBUG| Running virsh command: migrate --live --persistent --postcopy --postcopy-after-precopy --timeout 3600 --domain vms10 --desturi qemu+ssh://192.168.5.3/system
  2019-07-08 04:11:27,450 process          L0626 INFO | Running '/bin/virsh -c 'qemu:///system' migrate --live --persistent --postcopy --postcopy-after-precopy --timeout 3600 --domain vms10 --desturi qemu+ssh://192.168.5.3/system'
  2019-07-08 04:11:27,458 process          L0705 INFO | Command /bin/virsh -c 'qemu:///system' migrate --live --persistent --postcopy --postcopy-after-precopy --timeout 3600 --domain vms10 --desturi qemu+ssh://192.168.5.3/system running on a thread
  2019-07-08 04:11:44,595 process          L0458 DEBUG| [stdout]
  2019-07-08 04:11:44,596 process          L0714 INFO | Command '/bin/virsh -c 'qemu:///system' migrate --live --persistent --postcopy --postcopy-after-precopy --timeout 3600 --domain vms10 --desturi qemu+ssh://192.168.5.3/system' finished with 0 after 17.1387369633s
  2019-07-08 04:11:44,597 virsh            L0706 DEBUG| status: 0
  2019-07-08 04:11:44,598 virsh            L0707 DEBUG| stdout:
  2019-07-08 04:11:44,598 virsh            L0708 DEBUG| stderr:

  :::
  :::
  ```

  Target -> Source:
  ```
  2019-07-08 04:17:56,819 libvirt_vm       L2043 INFO | Migrating VM vms10 from qemu+ssh://192.168.5.3/system to qemu:///system
  2019-07-08 04:17:56,819 virsh            L0655 DEBUG| Running virsh command: migrate --live --persistent --postcopy --postcopy-after-precopy --timeout 3600 --domain vms10 --desturi qemu:///system
  2019-07-08 04:17:56,820 process          L0626 INFO | Running '/bin/virsh -c 'qemu+ssh://192.168.5.3/system' migrate --live --persistent --postcopy --postcopy-after-precopy --timeout 3600 --domain vms10 --desturi qemu:///system'
  2019-07-08 04:17:56,820 libvirt          L1531 DEBUG| start_time:1562573876, eclipse_time:0
  2019-07-08 04:17:56,831 process          L0705 INFO | Command /bin/virsh -c 'qemu+ssh://192.168.5.3/system' migrate --live --persistent --postcopy --postcopy-after-precopy --timeout 3600 --domain vms10 --desturi qemu:///system running on a thread
  2019-07-08 04:18:13,645 process          L0458 DEBUG| [stdout]
  2019-07-08 04:18:13,647 process          L0714 INFO | Command '/bin/virsh -c 'qemu+ssh://192.168.5.3/system' migrate --live --persistent --postcopy --postcopy-after-precopy --timeout 3600 --domain vms10 --desturi qemu:///system' finished with 0 after 16.8178260326s
  2019-07-08 04:18:13,648 virsh            L0706 DEBUG| status: 0
  2019-07-08 04:18:13,648 virsh            L0707 DEBUG| stdout:
  2019-07-08 04:18:13,648 virsh            L0708 DEBUG| stderr:
  :::
  :::
  ```

9. As post-migration check we perform,
  - VMs are active with running state in Target host
  - VMs network connectivity by pinging
  - uptime of each VM is greater than recorded value in pre migration check
  so that there is no reboot inside VM during migration
  - check dmesg of every VM for any call trace, oops, warnings

10. Finally it performs all the cleanup of configuration setting, VMs in Source
  and target host.

11. Same repeats in to and fro migration for 5 times (configurable to any no of
  times) and in this particular test we perform one by one VMs are migrated but
  `Test config` in *Code* section have other scenarios with parallel migration,
  cross migration, stress in VMs, stress in source, stress in target host,
  stress on both the hosts, stress on all VMs source host and target hosts,
  CPU stress, Memory stress, precopy & postcopy migration etc., that constitutes around 328 scenarios

Note:

[a] [patch 1](https://github.com/avocado-framework/avocado-vt/pull/2146) yet to
be merged upstream in Avocado-VT

[b] [patch 2](https://github.com/avocado-framework/avocado-vt/pull/2147) yet to
be merged upstream in Avocado-VT
