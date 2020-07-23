TP2: Procesos de usuario
========================

env_alloc
---------

...


env_init_percpu
---------------

...


env_pop_tf
----------

...


gdb_hello
---------

```
(qmu) info registers
EAX=003bc000 EBX=00010094 ECX=f03bc000 EDX=000001fe
ESI=00010094 EDI=00000000 EBP=f0119fd8 ESP=f0119fbc
EIP=f0102f6f EFL=00000092 [--S-A--] CPL=0 II=0 A20=1 SMM=0 HLT=0
ES =0010 00000000 ffffffff 00cf9300 DPL=0 DS   [-WA]
CS =0008 00000000 ffffffff 00cf9a00 DPL=0 CS32 [-R-]
...
```

```
(gdb) p *tf
{
  tf_regs = {
    reg_edi = 0,
    reg_esi = 0,
    reg_ebp = 0,
    reg_oesp = 0,
    reg_ebx = 0,
    reg_edx = 0,
    reg_ecx = 0,
    reg_eax = 0
  },
  tf_es = 35,
  tf_padding1 = 0,
  tf_ds = 35,
  tf_padding2 = 0,
  tf_trapno = 0,
  tf_err = 0,
  tf_eip = 4005548032,
  tf_cs = 27,
  tf_padding3 = 0,
  tf_eflags = 0,
  tf_esp = 4005552128,
  tf_ss = 35,
  tf_padding4 = 0
}
(gdb) x/17x tf
0xf01c8000:     0x00000000      0x00000000      0x00000000      0x00000000
0xf01c8010:     0x00000000      0x00000000      0x00000000      0x00000000
0xf01c8020:     0x00000023      0x00000023      0x00000000      0x00000000
0xf01c8030:     0xeebfd000      0x0000001b      0x00000000      0xeebfe000
0xf01c8040:     0x00000023
```

```
(gdb) x/68x $sp
0xf01c8000:     0x00000000      0x00000000      0x00000000      0x00000000
0xf01c8010:     0x00000000      0x00000000      0x00000000      0x00000000
0xf01c8020:     0x00000023      0x00000023      0x00000000      0x00000000
0xf01c8030:     0xeebfd000      0x0000001b      0x00000000      0xeebfe000
0xf01c8040:     0x00000023
```

Se ve que los contenidos son los mismos.

Las primeros 8 valores son los registros de proposito general. Son nulos porque el environment que va a ejecutarse se esta corriendo por primera vez.

Luego vienen tf_es y tf_ds, que son registros de configuracion de permisos que se configuran en la funcion env_alloc. Lo mismo con tf_ss y tf_cs, que aparecen mas adelante.

A contnuacion estan tf_trapno y tf_err.

Luego viene tf_eip, el intruction pointer que configuramos en la funcion load_icode con el entrypoint del ejecutable.

Luego tf_cs. Luego tf_eflags que es 0.

A continuacion esta tf_esp, el stack pointer que se configuro en env_alloc con la direccion del stack del environment, que mappeamos en load_icode.

Por ultimo esta tf_ss.

```
(qemu) info registers
EAX=00000000 EBX=00000000 ECX=00000000 EDX=00000000
ESI=00000000 EDI=00000000 EBP=00000000 ESP=f01c8030
EIP=f0102f82 EFL=00000096 [--S-AP-] CPL=0 II=0 A20=1 SMM=0 HLT=0
ES =0023 00000000 ffffffff 00cff300 DPL=3 DS   [-WA]
CS =0008 00000000 ffffffff 00cf9a00 DPL=0 CS32 [-R-]
...
```

Basicamente, se restauran todos los registros de proposito general, es y ds con los valores de tf.

```
(gdb) p $pc
$2 = (void (*)()) 0xeebfd000
```

```
(gdb) p $pc
$2 = (void (*)()) 0x800020 <_start>
```

```
EAX=00000000 EBX=00000000 ECX=00000000 EDX=00000000
ESI=00000000 EDI=00000000 EBP=00000000 ESP=eebfe000
EIP=eebfd000 EFL=00000002 [-------] CPL=3 II=0 A20=1 SMM=0 HLT=0
ES =0023 00000000 ffffffff 00cff300 DPL=3 DS   [-WA]
CS =001b 00000000 ffffffff 00cffa00 DPL=3 CS32 [-R-]
```

Cambio esp, ya que ahora se esta usando la pila del env, que se configuro antes y se toma en la instruccion iret. Lo mismo pasa con eip.

Antes de realizar la interrupcion, se ejecutan las siguientes instrucciones.

```s
0x00800a3e <+17>:    mov    0x8(%ebp),%ecx
0x00800a41 <+20>:    mov    0xc(%ebp),%ebx
0x00800a44 <+23>:    mov    0x10(%ebp),%edi
0x00800a47 <+26>:    mov    0x14(%ebp),%esi
```

Estas instrucciones ponen ciertos valores en los registros de proposito general, que sirven como parametros para la syscall.

En la primera, el valor de *($ebp + 0x8) es 13, que es el numero de syscall que se quiere ejecutar.

Los siguientes valores son todos 0.

user_evilhello
--------------

Es distinto, ya que copia el char que lee de la memoria del kernel, e intenta imprimir el char que fue copiado al segmento de datos del usuario.

Con este mecanismo, cuando se ejecuta la syscall, y luego el chequeo de memoria, la direccion de memoria es una direccion del segmento de datos del usuario, por lo tanto el kernel no sospecharia nada.

Sin embargo, al intentar ejecutarlo se produce un page fault. Esto pasa porque las paginas mapeadas al codigo del kernel no tienen permisos de lectura para el usuario. Entonces es el procesador el que hace este chequeo de proteccion, y no hay un problema de seguridad.
