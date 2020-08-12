TP3: Multitarea con desalojo
============================

env_return
----------

1. al terminar un proceso su función umain() ¿dónde retoma la ejecución el kernel? Describir la secuencia de llamadas desde que termina umain() hasta que el kernel dispone del proceso.

Al terminar un proceso, la "clib" de jos llama a la sycall env_destroy. (Si esto no se hiciera probablemente terminaria en un page fault).
Al llamar esta syscall, el kernel toma el control de la CPU, que destruye el env y llama al scheduler otra vez.

La secuencia de llamadas es:
1. umain termina, libmain llama a exit().
2. exit llama a sys_env_destroy(0) (0 para indicar que se destruya el proceso actual).
3. sys_env_destroy llama a return syscall de lib, con los parametros correspondientes a la syscall env destroy.
4. syscall ejecuta la instruccion int, que genera una interrupcion por software.
5. el CPU, al llegar la interrupcion, se fija el codigo a ejecutar en la IDT.
6. este codigo es codigo del kernel, configurado en el tp pasado.

---

2. ¿en qué cambia la función env_destroy() en este TP, respecto al TP anterior?

En este tp se llama a sched_yield, para que el planificador lance un nuevo proceso. En el tp anterior se iba directamente al monitor del kernel, ya que no habia soporte para multiples procesos.

sys_yield
---------

Leer y estudiar el código del programa user/yield.c. Cambiar la función i386_init() para lanzar tres instancias de dicho programa, y mostrar y explicar la salida de make qemu-nox.

```
[00000000] new env 00001000
[00000000] new env 00001001
[00000000] new env 00001002
Hello, I am environment 00001000.
Hello, I am environment 00001001.
Hello, I am environment 00001002.
Back in environment 00001000, iteration 0.
Back in environment 00001001, iteration 0.
Back in environment 00001002, iteration 0.
Back in environment 00001000, iteration 1.
Back in environment 00001001, iteration 1.
Back in environment 00001002, iteration 1.
Back in environment 00001000, iteration 2.
Back in environment 00001001, iteration 2.
Back in environment 00001002, iteration 2.
Back in environment 00001000, iteration 3.
Back in environment 00001001, iteration 3.
Back in environment 00001002, iteration 3.
Back in environment 00001000, iteration 4.
All done in environment 00001000.
[00001000] exiting gracefully
[00001000] free env 00001000
Back in environment 00001001, iteration 4.
All done in environment 00001001.
[00001001] exiting gracefully
[00001001] free env 00001001
Back in environment 00001002, iteration 4.
All done in environment 00001002.
[00001002] exiting gracefully
[00001002] free env 00001002
```

Las primeras lineas las imprime el kernel. Se crean los envs.
Luego comienza el codigo de usuario. El programa imprime el "Hello, ..." y luego llama a sys_yield. Por eso se puede ver que los tres prints salen seguidos: El primer env llama a sys_yield, que ejecuta el segundo env, que imprime y luego llama a sys_yield, y asi.

Cuando el 3er env (00001002) llama a sys_yield, se vuelve a ejecutar el 1er env, segun la logica del planificador robinson, ya que no hay mas envs creados. Luego se van ejecutando secuencialmente los envs creados. Esto corresponde a robinson, que va cambiando siempre de env, y solo vuelve a ejecutar un env si ya se ejecutaron todos los RUNNABLE desde la ultima ve que se ejecuto.

Luego, una vez que paso 4 veces por cada env, todos terminan en el mismo orden.

ipc_recv
--------

Por mas que se guarde o no se guarde el id del sender a traves del primer parametro de ipc_recv, el usuario siempre tendra acceso a la variable global thisenv. Si el campo env_ipc_from es 0, entonces hubo un error en la llamada.

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
Continuing.
The target architecture is assumed to be i386
=> 0xf0100174 <boot_aps+92>:    mov    %esi,%ecx

Thread 1 hit Hardware watchpoint 1: mpentry_kstack

Old value = (void *) 0x0
New value = (void *) 0xf0255000 <percpu_kstacks+65536>
boot_aps () at kern/init.c:109
109                     lapic_startap(c->cpu_id, PADDR(code));
(gdb) bt
#0  boot_aps () at kern/init.c:109
#1  0xf010022a in i386_init () at kern/init.c:55
#2  0xf0105ee6 in ?? ()
#3  0xf0100047 in entry () at kern/entry.S:86
(gdb) info threads
  Id   Target Id                    Frame
* 1    Thread 1.1 (CPU#0 [running]) boot_aps () at kern/init.c:109
  2    Thread 1.2 (CPU#1 [halted ]) 0x000fd0ae in ?? ()
  3    Thread 1.3 (CPU#2 [halted ]) 0x000fd0ae in ?? ()
  4    Thread 1.4 (CPU#3 [halted ]) 0x000fd0ae in ?? ()
(gdb) continue
Continuing.
=> 0xf0100174 <boot_aps+92>:    mov    %esi,%ecx

Thread 1 hit Hardware watchpoint 1: mpentry_kstack

Old value = (void *) 0xf0255000 <percpu_kstacks+65536>
New value = (void *) 0xf025d000 <percpu_kstacks+98304>
boot_aps () at kern/init.c:109
109                     lapic_startap(c->cpu_id, PADDR(code));
(gdb) info threads
  Id   Target Id                    Frame
* 1    Thread 1.1 (CPU#0 [running]) boot_aps () at kern/init.c:109
  2    Thread 1.2 (CPU#1 [running]) spin_lock (lk=0x0) at kern/spinlock.c:71
  3    Thread 1.3 (CPU#2 [halted ]) 0x000fd0ae in ?? ()
  4    Thread 1.4 (CPU#3 [halted ]) 0x000fd0ae in ?? ()
(gdb) thread 2
[Switching to thread 2 (Thread 1.2)]
#0  spin_lock (lk=0x0) at kern/spinlock.c:71
71              while (xchg(&lk->locked, 1) != 0)
(gdb) bt
#0  spin_lock (lk=0x0) at kern/spinlock.c:71
#1  0x00000000 in ?? ()
(gdb) p cpunum()
$1 = 1
(gdb) thread 1
[Switching to thread 1 (Thread 1.1)]
#0  boot_aps () at kern/init.c:111
111                     while(c->cpu_status != CPU_STARTED)
(gdb) p cpunum()
$2 = 0
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

No solo es una direccion virtual, cuando al momento de ejecutarse esta instruccion, la memoria virtual no esta activada, si no que ademas el codigo ubicado aqui se copia a la direccion fisica `MPENTRY_PADDR` (0x7000). Por eso un una ejecucion de gdb no se va a detener si se pone un breakpoint en este simbolo.

5. Con GDB, mostrar el valor exacto de %eip y mpentry_kstack cuando se ejecuta la instrucción anterior en el último AP. Se recomienda usar, por ejemplo:

Al momento de ejecutar la instruccion
```S
movl $(RELOC(entry_pgdir)), %eax
```
- eip: 0x703b
- mpentry_kstack: 0x0

y luego de activar memoria virtual:
- eip: 0x704e
- mpentry_kstack: 0xf0265000 <percpu_kstacks+131072>


5. Responder qué ocurre:

en JOS, si un proceso llama a sys_env_destroy(0), sys_env_destroy(-1)

Al hacer sys_env_destroy(0), se destruye el environment actual, y luego lanza un nuevo environment, sin retornar al caller.
Al hacer sys_env_destroy(-1), deberia retornar -E_BAD_ENV, ya que los envids menores a 0 son invalidos.

en Linux, si un proceso llama a kill(0, 9) o kill(-1, 9)

Si se llama a kill(n,9), se le envia al pid especificado por n la senal 9 (KILL).

En el caso n=0, se le envia a todos los procesos en el grupo de procesos actual.
En el caso n=-1, se le envia a todos los procesos con pid mayor que 1
