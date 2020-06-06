TP0: Introducción a JOS
=======================

backtrace_func_names
--------------------

Salida del comando `backtrace`:

```
K> backtrace
  ebp f0110f48 eip f0100a53  args 00000001 f0110f70 00000000 kern/monitor.c:60: mon_backtrace+0
  ebp f0110fc8 eip f0100a9c  args kern/monitor.c:123: runcmd+261
  ebp f0110fd8 eip f01000e9  args 00000000 kern/monitor.c:141: monitor+66
  ebp f0110ff8 eip f010003e  args kern/init.c:43: i386_init+81
```
