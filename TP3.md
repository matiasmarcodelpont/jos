TP3: Multitarea con desalojo
============================

multicore_init
--------------

1. ¿Qué código copia, y a dónde, la siguiente línea de la función boot_aps()?

```c
memmove(code, mpentry_start, mpentry_end - mpentry_start);
```

La linea copia la memoria ubicada en mpentry_start, que es el codigo assembler con el entrypoint de los CPUs non-boot a la direccion `MPENTRY_PADDR` (0x7000), que esta en la memoria del kernel.

2. ¿Para qué se usa la variable global mpentry_kstack? ¿Qué ocurriría si el espacio para este stack se reservara en el archivo kern/mpentry.S, de manera similar a bootstack en el archivo kern/entry.S?

La variable `mpentry_kstack` guarda la direccion del stack del CPU que esta siendo inicializado. El kernel que corre en el primer CPU va cambiando el valor de esta variable a medida que inicializa los CPUs con el valor de su stack respectivo.

En el momento de compilacion, no se sabe cuantos CPUs se dispone (o si se supiera el codigo no seria portable). Por eso, las direcciones de los stacks de cada CPU debe ser definido en runtime. Si esta variable se definiera en el archivo kern/mpentry.S, todos los CPUs usarian el mismo stack, y pisarian sus datos.

3. Cuando QEMU corre con múltiples CPUs, éstas se muestran en GDB como hilos de ejecución separados. Mostrar una sesión de GDB en la que se muestre cómo va cambiando el valor de la variable global `mpentry_kstack`:

```
$ make qemu-nox-gdb CPUS=4
...

// En otra terminal:
$ make gdb
(gdb) watch mpentry_kstack
Hardware watchpoint 1: mpentry_kstack
(gdb) continue
...
(gdb) bt
...
(gdb) info threads
...
(gdb) continue
...
(gdb) info threads
...
(gdb) thread 2
...
(gdb) bt
...
(gdb) p cpunum()
...
(gdb) thread 1
...
(gdb) p cpunum()
...
(gdb) continue
```

4. En el archivo kern/mpentry.S se puede leer:

    ```S
    # We cannot use kern_pgdir yet because we are still
    # running at a low EIP.
    movl $(RELOC(entry_pgdir)), %eax
    ```

    - ¿Qué valor tendrá el registro %eip cuando se ejecute esa línea?

      Responder con redondeo a 12 bits, justificando desde qué región de memoria se está ejecutando este código.

    - ¿Se detiene en algún momento la ejecución si se pone un breakpoint en mpentry_start? ¿Por qué?

Cuando se inicializa un CPU, el codigo que corre esta en la direcion fisica `MPENTRY_PADDR` (0x7000). Este es el valor (redondeado hacia abajo 12 bits) que tendra el `%eip` al correr esta instruccion.

Sin embargo, el simbolo `mpentry_start` tiene el valor f01052a0:

```
$ nm obj/kern/kernel | grep mpentry_start
f01052a0 T mpentry_start
```

No solo es una direccion virtual, cuando al momento de ejecutarse esta instruccion, la memoria cirtual no esta activada, si no que ademas el codigo ubicado aqui se copia a la direccion fisica `MPENTRY_PADDR` (0x7000). Por eso un una ejecucion de gdb no se va a detener si se pone un breakpoint en este simbolo.
