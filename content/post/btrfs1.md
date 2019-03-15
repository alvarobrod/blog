---
title: Btrfs, parte 1 
date: 2019-03-15T108:00:00Z
tags: ["brtfs", "raid", "mdadm"]
---

Btrfs

# Escenario
Para estas pruebas vamos a disponer de una máquina Debian con tres discos asociados a la misma. Estos discos tienen diferentes tamaños, siendo uno de 1 GB, otro de 2 GB y el último de 3 GB.

# Gestión de los discos adicionales
Para formatear un disco con el sistema de ficheros _Btrfs_, ejecutaremos el siguiente comando:
```
mkfs.btrfs /dev/sdc
```
Como podemos ver, he formateado el disco sin hacer particiones previamente. Podemos hacerlo así o hacer particiones de antemano y así formatear una en concreto.
Podremos montar dicho disco de la manera tradicional (Por ejemplo, en _/mnt_):
```
mount /dev/sdb /mnt
```
O bien, añadiendo una entrada al fichero _/etc/fstab_.
Con el siguiente comando, podremos ver cuánto espacio de disco se está utilizando y cuánto podría estar disponible:
```
btrfs fi usage /mnt
```
Aquí vemos que tenemos que indicar la ruta donde hemos montado nuestro disco formateado con _Btrfs_, y no el disco en sí.
A continuación, mostraré la salida de dicho comando para el disco que acabamos de formatear y montar:
```
root@arca:~# btrfs fi usage /mnt
Overall:
    Device size:                1.00GiB
    Device allocated:               126.38MiB
    Device unallocated:     897.62MiB
    Device missing:             0.00B
    Used:                   256.00KiB
    Free (estimated):           905.62MiB   (min: 456.81MiB)
    Data ratio:                 1.00
    Metadata ratio:             2.00
    Global reserve:         16.00MiB    (used: 0.00B)

Data,single: Size:8.00MiB, Used:0.00B
   /dev/sdb    8.00MiB

Metadata,DUP: Size:51.19MiB, Used:112.00KiB
   /dev/sdb  102.38MiB

System,DUP: Size:8.00MiB, Used:16.00KiB
   /dev/sdb   16.00MiB

Unallocated:
   /dev/sdb  897.62MiB
```
Ahora, si quisiéramos añadir un nuevo disco a _/mnt_, ejecutaríamos el siguiente comando:
```
btrfs device add -f /dev/sdc /mnt
```
Como podemos ver, estamos añadiendo a _/mnt_ el disco _/dev/sdc_, que estará previamente formateado con _Btrfs_. Tenemos que añadirle el parámetro _-f_ ya que _/dev/sdc_ ya contenía un sistema de ficheros _Btrfs_, teniendo así que forzar la acción.
Si volvemos a ejecutar el comando que hemos visto anteriormente para ver el espacio del que disponemos, veremos que ahora el tamaño del dispositivo, es decir, el tamaño de _/mnt_ es de 3 GiB, de los cuales dispondremos de 2,88 GiB (Aproximadamente).
```
root@arca:~# btrfs fi usage /mnt
Overall:
    Device size:                3.00GiB
    Device allocated:           126.38MiB
    Device unallocated:     2.88GiB
    Device missing:             0.00B
    Used:                   256.00KiB
    Free (estimated):           2.88GiB (min: 1.45GiB)
    Data ratio:                 1.00
    Metadata ratio:             2.00
    Global reserve:         16.00MiB    (used: 0.00B)

Data,single: Size:8.00MiB, Used:0.00B
   /dev/sdb    8.00MiB

Metadata,DUP: Size:51.19MiB, Used:112.00KiB
   /dev/sdb  102.38MiB

System,DUP: Size:8.00MiB, Used:16.00KiB
   /dev/sdb   16.00MiB

Unallocated:
   /dev/sdb  897.62MiB
   /dev/sdc    2.00GiB
```
Entonces, como podemos ver, al añadir un nuevo volúmen, lo que estamos haciendo es crear un pool con un tamaño total de 3 GiB. Este pool será tratado como si fuera un único disco.
Crearemos un fichero de 2GB con el siguiente comando:
```
fallocate -l 2G /mnt/prueba
```
Ejecutando el comando que hemos visto anteriormente, podremos ver que el pool de almacenamiento tiene ahora unos 903 MiB de espacio libre disponibles (Aproximadamente).
Acto seguido, ejecutamos el siguiente comando:
```
root@arca:~# df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            488M     0  488M   0% /dev
tmpfs           100M  3.1M   97M   4% /run
/dev/sda1       8.7G  2.1G  6.2G  26% /
tmpfs           499M     0  499M   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           499M     0  499M   0% /sys/fs/cgroup
tmpfs           100M     0  100M   0% /run/user/1000
/dev/sdb        3.0G  2.1G  904M  70% /mnt
```
Y podremos ver cómo nuestro sistema interpreta _/dev/sdb_ como un único disco de 3 GB. Esto se debe a que _/dev/sdb_ y _/dev/sdc_ están en el mismo pool, tal y como comentábamos antes.
Si queremos retirar un volumen de dicho pool, ejecutaremos el siguiente comando:
```
btrfs device del /dev/sdc /mnt
```
Y, si ejecutamos el siguiente comando, veremos que el disco _/dev/sdc_ ya no forma parte de nuestro pool de almacenamiento.
```
root@arca:~# btrfs fi show
Label: none  uuid: 28073a9a-7d0a-41f3-908a-7c8942852a91
    Total devices 1 FS bytes used 192.00KiB
    devid    1 size 1.00GiB used 230.38MiB path /dev/sdb
```
_Nota: Aquí también podríamos ejecutar el comando btrfs fi usage /mnt y nos mostraría información similar, aunque más detallada, como hemos visto anteriormente._
Si contamos con el pool de los dos discos, podemos convertirlo directamente a un RAID 1. Esto lo haremos con el siguiente comando:
```
btrfs balance start -v -mconvert=raid1 -dconvert=raid1 /mnt
```
Al ejecutar este comando, habremos convertido nuestro pool de almacenamiento en un RAID 1, reduciendo a la mitad el tamaño total pero teniendo una redundancia total de los datos. Con la opción _-mconvert_ indicaríamos el tipo de RAID que utilizamos para los metadatos, y con _-dconvert_, indicamos el tipo de RAID para los datos.
_Nota: Los metadatos en Btrfs incluyen por ejemplo datos sobre las estructuras internas del sistema de ficheros, estructuras de directorios, nombres de ficheros, permisos de estos, sumas de comprobación (checksums), etc._
```
root@arca:~# btrfs fi usage /mnt
Overall:
    Device size:     3.00GiB
    Device allocated:    1.19GiB
    Device unallocated: 1.81GiB
    Device missing: 0.00B
    Used: 512.00KiB
    Free (estimated):    1.22GiB (min: 1.22GiB)
    Data ratio: 2.00
    Metadata ratio: 2.00
    Global reserve: 16.00MiB (used: 0.00B)

Data,RAID1: Size:320.00MiB, Used:128.00KiB
   /dev/sdb  320.00MiB
   /dev/sdc  320.00MiB

Metadata,RAID1: Size:256.00MiB, Used:112.00KiB
   /dev/sdb  256.00MiB
   /dev/sdc  256.00MiB

System,RAID1: Size:32.00MiB, Used:16.00KiB
   /dev/sdb   32.00MiB
   /dev/sdc   32.00MiB

Unallocated:
   /dev/sdb  416.00MiB
   /dev/sdc    1.41GiB
```

# RAID
## RAID 1
Podemos ver que disponemos de un RAID con el siguiente comando:
```
btrfs fi df /mnt
```
La salida del comando en nuestro escenario sería la siguiente:
```
root@arca:~# btrfs fi df /mnt
Data, RAID1: total=320.00MiB, used=128.00KiB
System, RAID1: total=32.00MiB, used=16.00KiB
Metadata, RAID1: total=256.00MiB, used=112.00KiB
GlobalReserve, single: total=16.00MiB, used=0.00B
```
Podríamos volver a convertir de RAID 1 a un pool normal utilizando el siguiente comando:
```
btrfs balance start -dconvert=single -mconvert=raid1 /mnt
```
También podremos añadir un nuevo disco al RAID utilizando el comando que ya hemos utilizado anteriormente:
```
btrfs device add -f /dev/sdd /mnt
```
## RAID 10
Para crear un RAID 10 a partir de nuestro pool de almacenamiento, ejecutaremos el siguiente comando:
```
btrfs balance start -v -mconvert=raid10 -dconvert=raid10 /mnt

root@arca:~# btrfs fi df /mnt
Data, RAID10: total=736.00MiB, used=192.00KiB
System, RAID10: total=64.00MiB, used=16.00KiB
Metadata, RAID10: total=196.88MiB, used=112.00KiB
GlobalReserve, single: total=16.00MiB, used=0.00B
```
## RAID 5
Para crear un RAID 5, ejecutaremos el siguiente comando:
```
btrfs balance start -v -mconvert=raid5 -dconvert=raid5 /mnt

root@arca:~# btrfs fi df /mnt
Data, RAID5: total=720.00MiB, used=192.00KiB
System, RAID5: total=96.00MiB, used=16.00KiB
Metadata, RAID5: total=288.00MiB, used=112.00KiB
GlobalReserve, single: total=16.00MiB, used=0.00B
```
## RAID 6
Para crear un RAID 6, ejecutamos el siguiente comando:
```
btrfs balance start -v -mconvert=raid6 -dconvert=raid6 /mnt

root@arca:~# btrfs fi df /mnt
Data, RAID6: total=736.00MiB, used=192.00KiB
System, RAID6: total=64.00MiB, used=16.00KiB
Metadata, RAID6: total=196.88MiB, used=112.00KiB
GlobalReserve, single: total=16.00MiB, used=0.00B
```
## Comparación de RAID Btrfs con RAID mdadm
Poniendo un ejemplo, si hacemos un RAID 1 _mdadm_, tendremos que tener dos discos de tamaño idéntico, escribiéndose en uno de ellos una copia exacta del otro.
En un RAID 1 _Btrfs_, por otro lado, podremos tener varios discos e incluso, de distinto tamaño, y _Btrfs_ se encargará de distribuir los bloques de datos de manera uniforme entre todos los discos. Si más adelante quisiéramos añadir un nuevo disco al RAID, podríamos hacerlo y, de nuevo, _Btrfs_ se encargaría de “alinear” los datos del nuevo disco con los datos del RAID ya existente.
_Nota: Hay que decir que, obviamente, la implementación de RAID de Btrfs no es igual a la de mdadm._
Como podemos ver, entonces, con la implementación de RAID de _Btrfs_ tendremos mayor flexibilidad, con mayores posibilidades de expansión.
También debemos apuntar que, al día de escritura de esta entrada, los RAID 5 y 6 todavía no son estables en la implementación _Btrfs_.
## Reparación de RAID
Para hacer las pruebas de degradación del RAID, he montado un RAID 1 con dos discos, uno de 1 GB (_/dev/sdb_) y otro de 3 GB (_/dev/sdc_).
Si uno de nuestros discos falla, podríamos montar el sistema de ficheros en modo degraded (“Degradado”). Esto lo haremos con el siguiente comando:
```
mount -o degraded /dev/sdb /mnt
```
Cuando tengamos un disco nuevo para sustituir el que ha fallado, ejecutaremos el siguiente comando:
```
btrfs replace start /dev/sdb /dev/sdd /mnt
```
Este comando leerá los ficheros del disco original (Si es que sigue accesible) y de los otros discos del RAID, usando esta información para poblar el nuevo disco. Si no queremos que el sistema lea los ficheros del disco original, añadiremos la opción _-r_.
También podríamos reemplazar los discos borrando el dañado y añadiendo posteriormente el nuevo.
