TP1: Memoria virtual en JOS
===========================

page2pa
-------

...


boot_alloc_pos
--------------
a)

Num:    Valor  Tam  Tipo    Unión  Vis      Nombre Ind
110: f0117950     0 NOTYPE  GLOBAL DEFAULT    6 end

resultado = f0117950

resultado = roundup(f0117950,4096) = f0118000


b)
Breakpoint 1, boot_alloc (n=65684) at kern/pmap.c:89
89	{
(gdb) print (void*) end
$1 = (void *) 0xf01006f6 <cons_init>
(gdb) print (void*) nextfree
$2 = (void *) 0x0
(gdb) finish
Correr hasta la salida desde #0  boot_alloc (n=65684) at kern/pmap.c:89
Could not fetch register "orig_eax"; remote failure reply 'E14'
(gdb) print (void*) end
$3 = (void *) 0xf01006f6 <cons_init>
(gdb) print (void*) nextfree
$4 = (void *) 0xf0119000

...


page_alloc
----------
page2pa devuelve la physical address asociada a una struct PageInfo.
page2kva devuelve la virtual adress del kernel asociada a una physical address, que a su vez esta asociada a una struct PageInfo.
...

map_region_large
----------------
Responder: ¿cuánta memoria se ahorró de este modo? ¿Es una cantidad fija, o depende de la memoria física de la computadora?

Se ahorra xactamente una pagina (4KiB) por cada large page (4MiB) que se aloca.
En el caso puntual de boot_map_region, va a depender de la memoria fisica de la computadora, ya que se mapea el array `PageInfo* pages`, cuyo tamaño depende de la memoria fisica de la computadora.
Luego se mappea el stack, que solo ocupa 8 paginas, entonces no se usan large pages.
Por ultimo se mappean los 256MiB usados por el kernel, en donde entran 64 large pages. Entonces se ahorran 64 * 4096B = 256KiB.
