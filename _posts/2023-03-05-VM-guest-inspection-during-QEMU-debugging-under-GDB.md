---
title:  "VM guest inspection during QEMU debugging under GDB"
layout: post
long: true
date: 2023-03-05 23:59:00 +0300
---

QEMU has good debugging capabilities, such as [gdbstub](https://qemu.readthedocs.io/en/latest/system/gdb.html). But sometimes we have to run a QEMU process under GDB and set breakpoints in the QEMU source code. When the process breaks, we can easily inspect the QEMU state. But what about the guest state, how can we inspect it? For example, how to read the guest memory by virtual address when a GDB watchpoint is triggered?



The problem is that the QEMU monitor is stopped as well as QMP by the breakpoint. Fortunately, the GDB's ability to call a debugee function and QEMU's ability to pause a virtual machine can help us.

The GDB `call` command allows us to execute a function in the program being debugged. The syntax for the command is:
```
(gdb) call <function_name>(<arg1>, <arg2>, ...)
```
But which function to call?

The answer is `vm_stop(0)`. After this call, the VM will be paused and we can continue the QEMU process with `cont` and obtain an access to the monitor. Then we can read guest virtual memory with `x`, dump memory with `dump-guest-memory` and so on.

![image](https://user-images.githubusercontent.com/8286747/223064735-167e2a92-7976-46cb-a12b-e0f632188462.png)

![image](https://user-images.githubusercontent.com/8286747/223065122-807e1bb5-0a98-453a-b81b-327ed9c5e1fe.png)
