# KVM Migration using Libvirt:

This document explains how to perform KVM migration of guest from source host
to target host using Libvirt (virsh API) with qcow2 guest image placed in
NFS shared location so that source and target host could access the guest
image from same path before and after migration.

__Presetups:__

Some presetup configurations have to be done for the migrating the guest,

1. Install and place guest image qcow2 with shared storage in NFS
2. Mount the image location in source and in target host.

    *source*

    ```
    # mount -t nfs <nfs-server-ip>:/home/bala/NFS/ /home/bala/sharing/
    ```

    *target*

    ```
    # mount -t nfs <nfs-server-ip>:/home/bala/NFS/ /home/bala/sharing/
    ```

    so that both the source and target host could access the image from
    `/home/bala/sharing`

3. Enabled ports 49152:49216 in iptables in target host, as this
allows firewall to permit Libvirtd to use the ports for migration,

    use `firewall-cmd` if it is available else use `iptables`

    ```
    # firewall-cmd --add-port=49152-49216/tcp --zone=public --permanent
    # firewall-cmd --reload
    ```

    or

    ```
    # iptables -I INPUT -p tcp -m tcp --dport 49152:49216 -j ACCEPT
    ```

4. Permit selinux in source and target host to allow Libvirt for using NFS
by setting virt_use_nfs boolean to 1,

    ```
    # setsebool virt_use_nfs 1
    ```

5. Passwordless ssh can be configured between source and target host to avoid
password prompt or password can be given while triggering the migration

__Perform *online migration* using virsh command__

```
# virsh migrate guest-vm-name qemu+ssh://<target_host_ipaddress>/system --verbose
root@<target_host_ipaddress>'s password:
Migration: [100 %]
```

__Perform *live migration* using virsh command__

```
# virsh migrate guest-vm-name qemu+ssh://<target_host_ipaddress>/system --live --verbose
root@<target_host_ipaddress>'s password:
Migration: [100 %]
```

__Perform *offline migration* using virsh command__

```
# virsh migrate guest-vm-name qemu+ssh://<target_host_ipaddress>/system --offline --persistent --verbose
root@<target_host_ipaddress>'s password:
Migration: [100 %]
```

* Live migration, online migration and offline migration types are explained in
[Introduction To KVM Migration](https://balamuruhans.github.io/2018/11/13/introduction-to-kvm-migration.html)
