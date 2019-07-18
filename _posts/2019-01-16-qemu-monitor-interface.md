# Qemu monitor interfaces

Qemu provides monitor interface to user and higher management layers like
Libvirt to interact, communicate and perform operations with VM, there are
two types of Qemu monitor interface

QMP (Qemu monitor protocol)

It provides to interface using commands that adhere to json based syntax
(dictionary like), application and management programs are suggested to use
this interface as it is considered to be stable

use qmp option in Qemu commandline,
```
# /usr/libexec/qemu-kvm -qmp telnet:127.0.0.1:1234,server,nowait
```
connect to QMP monitor using telnet,

```
# telnet 127.0.0.1 1234
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
{"QMP": {"version": {"qemu": {"micro": 0, "minor": 12, "major": 2}, "package": "qemu-kvm-2.12.0-74.module+el8.1.0+3227+57d66ad3"}, "capabilities": []}}
```
HMP (Human monitor protocol)

It provides to interface using commands that are similar to bash commands,
mostly used for testing, debugging and analysis of operations with Qemu.

use monitor option in Qemu command line,
```
# /usr/libexec/qemu-kvm -monitor telnet:127.0.0.1:1234,server,nowait
```
connect to HMP monitor using telnet,
```
# telnet 127.0.0.1 1234
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
QEMU 2.12.0 monitor - type 'help' for more information
(qemu)
```
