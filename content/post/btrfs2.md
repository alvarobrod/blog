---
title: Btrfs, parte 2 
date: 2019-03-20T08:00:00Z
tags: ["brtfs", "raid", "mdadm"]
---

Esta es la segunda parte de mis posts sobre Btrfs, para leer la primera, ve [aquí](https://alvarobrod.github.io/blog/post/btrfs1/)

# Otras funcionalidades
## Compresión
Cuando montamos el disco de manera normal, no usará ningún tipo de compresión por defecto. Para poder empezar a utilizar la compresión, tendremos que indicar qué método de compresión es el que deseamos utilizar. Esto lo haremos de la siguiente manera:
```
mount -o compress=<método de compresión> /dev/sdb /mnt
```
Contamos con varios métodos de compresión en Btrfs:
* ZLIB: Gran ratio de compresión, pero lento.
* LZO: Menor ratio de compresión, pero mayor velocidad a la hora de comprimir/descomprimir (Diseñado para ser rápido).
* ZSTD: Cuenta con un ratio de compresión equiparable al de ZLIB, pero con mayores velocidades de compresión/descompresión.

Vamos a hacer pruebas, por ejemplo, con ZLIB:
Primero, montamos nuestro disco (Recordamos que montando un disco, nos montará directamente los demás al estar en un pool de almacenamiento):
```
mount -o compress=zlib /dev/sdb /mnt
```

### ZLIB
He usado la utilidad dd para crear un fichero de gran tamaño de forma fácil:
```
dd if=/dev/zero of=/mnt/prueba bs=2048

root@arca:~# dd if=/dev/zero of=/mnt/prueba bs=2048
dd: error writing '/mnt/prueba': No space left on device
88499663+0 records in
88499662+0 records ou
181247307776 bytes (181 GB, 169 GiB) copied, 513.136 s, 353 MB/s

root@arca:~# ls -hal /mnt/prueba 
-rw-r--r-- 1 root root 169G Feb 11 18:18 /mnt/prueba

root@arca:~# btrfs fi usage /mnt
Overall:
    [...]
    Used: 5.71GiB
    [...]
```
Como podemos comprobar, este método tiene un gran ratio de compresión, pudiendo alojar un fichero de 169 GiB en 5,71 GiB.

### LZO
Activaremos este método de compresión al montar el disco de la siguiente manera:
```
mount -o compress=lzo /dev/sdb /mnt
```
A continuación, volvemos a ejecutar el comando dd que ejecutamos anteriormente y podremos ver que este método de compresión tiene un ratio menor que ZLIB, aunque también comprime con un ratio bastante bueno:
```
root@arca:~# dd if=/dev/zero of=/mnt/prueba bs=2048
dd: error writing '/mnt/prueba': No space left on device
80101981+0 records in
80101980+0 records out
164048855040 bytes (164 GB, 153 GiB) copied, 267.435 s, 613 MB/s

root@arca:~# ls -hal /mnt/prueba 
-rw-r--r-- 1 root root 153G Feb 11 22:07 /mnt/prueba

root@arca:~# btrfs fi usage /mnt
Overall:
    [...]
    Used: 5.18GiB
    [...]
```

### ZSTD
Para poder utilizar este método de compresión, tendremos que tenerlo activado en nuestro kernel. Para ver si disponemos del mismo, miraremos si las opciones CONFIG_ZSTD_COMPRESS y CONFIG_ZSTD_DECOMPRESS están habilitadas.
Se utilizaría de la misma manera que hemos utilizado los métodos anteriores.

## COW (Copy-on-write)
_Btrfs_ es un sistema de ficheros _Copy-on-write_ (COW, para abreviar). Esto básicamente quiere decir que, cuando el sistema accede a un fichero y lo modifica, en vez de sobreescribirlo, _Btrfs_ hace una copia del fichero, lo modifica y lo guarda en una localización diferente en el disco. Después actualiza los metadatos para que reflejen esta nueva localización (En esta actualización de los metadatos, estos son tratados de la misma manera que se han tratado los datos anteriormente).
Para probar el COW, he creado un fichero de 200 MiB con el siguiente comando:
```
dd if=/dev/zero of=/mnt/prueba bs=2048 count=100k
```
Y he creado una copia en el mismo directorio con el comando que sigue:
```
cp --reflink=always /mnt/prueba /mnt/prueba2
```
La opción _--reflink=always_ nos permite controlar las copias COW. Si especificamos always, se ejecuta una copia donde sólo se copian los bloques de datos cuando estos son modificados. Si esto no es posible, la copia fallará. En cambio, especificando auto, si esto falla, la copia se hará de forma estándar.
Tras hacer la copia, veremos que no ocupa espacio en nuestro disco, ya que no se ha modificado ningún bloque del fichero:
```
root@arca:~# dd if=/dev/zero of=/mnt/prueba bs=2048 count=100k
102400+0 records in
102400+0 records out
209715200 bytes (210 MB, 200 MiB) copied, 0.277714 s, 755 MB/s
root@arca:~# btrfs fi usage /mnt
Overall:
    [...]
    Free (estimated):          1.33GiB  (min: 687.81MiB)
    [...]
root@arca:~# cp --reflink=always /mnt/prueba /mnt/prueba2
root@arca:~# btrfs fi usage /mnt
Overall:
    [...]
    Free (estimated): 1.33GiB (min: 687.88MiB)
    [...]
```

## Deduplicación
Como hemos visto, _Btrfs_ es un sistema de ficheros _copy-on-write_, que significa que una nueva copia de un fichero se crearía cuando se modifica el original.
La deduplicación va un paso más allá, identificando cuándo los datos se han escrito dos veces y combinándolos en una misma extensión del disco.
Hay varias herramientas, llamadas “_deduplicators_”, como _duperemove_, _bedup_, o _bees_.

## Subvolúmenes
Los subvolúmenes en _Btrfs_ son partes del sistema de ficheros con su jerarquía de directorios y ficheros propia e independiente. Un subvolumen parece un directorio normal y corriente, pero tiene funcionalidades añadidas.
Un sistema de ficheros recién creado también se considera un subvolumen, llamado _top-level_ (Alto nivel, en español). Este subvolumen no puede ser borrado o reemplazado por otro subvolumen.
Para crear un subvolumen, ejecutaremos el siguiente comando:
```
btrfs subvolume create subvolumen
```

_Nota: El comando anterior tendremos que ejecutarlo estando en el directorio del sistema de ficheros Btrfs._

Como hemos dicho antes y podemos comprobar ahora, el subvolumen es una especie de directorio:
```
root@arca:/mnt# ls -hal
total 20K
drwxr-xr-x  1 root root   20 Feb 13 11:29 .
drwxr-xr-x 23 root root 4.0K Feb 13 11:22 ..
drwxr-xr-x  1 root root    0 Feb 13 11:33 subvolumen
root@arca:/mnt# touch subvolumen/prueba
root@arca:/mnt# tree
.
└── subvolumen
    └── prueba

1 directory, 1 file
```
Aunque el subvolumen se comporte de manera casi idéntica a un directorio, tiene diferencias con este último. Por ejemplo, podríamos montar el subvolumen en otro punto de montaje diferente.
Primero obtendremos el ID del subvolumen con el siguiente comando:
```
btrfs subvolume list /mnt

root@arca:/mnt# btrfs subvolume list /mnt
ID 256 gen 10 top level 5 path subvolumen
```
Y ahora, utilizando el ID (256 en este caso), podremos montar dicho subvolumen aparte:
```
mount -o subvolid=256 /dev/sdb /punto_montaje

root@arca:~# tree /punto_montaje/
/punto_montaje/
└── prueba

0 directories, 1 file
```
Con esto, conseguimos poder tratar a cada subvolumen como un sistema de ficheros independiente.
Podemos borrar subvolúmenes con el siguiente comando, aunque este subvolumen debe estar vacío antes de proceder al borrado:
```
btrfs subvolume delete <ruta del subvolumen>
```

## Snapshots
Podríamos definir los _snapshots_ como subvolúmenes que son copias _copy-on-write_ de otro subvolumen. Esto nos permitiría hacer una especie de copia de seguridad del subvolumen de una manera que no consume apenas recursos adicionales.
Para hacer pruebas, he creado dos subvolúmenes, _sub1_ y _sub2_, el primero conteniendo dos ficheros de 200 MiB cada uno, _prueba1_ y _prueba2_ y un directorio, prueba_dir:
```
.
├── sub1
│   ├── prueba1
│   ├── prueba2
│   └── prueba_dir
└── sub2

root@arca:/mnt# btrfs fi usage /mnt
Overall:
    [...]
    Used:            401.45MiB
    [...]
```
Ahora, queremos crear un _snapshot_ del subvolumen sub1. Para ello, ejecutaremos el siguiente comando:
```
btrfs subvolume snapshot /mnt/sub1 /mnt/snapshot
```
Y tendremos ya un _snapshot_ del subvolumen sub1.
```
.
├── snapshot
│   ├── prueba1
│   ├── prueba2
│   └── prueba_dir
├── sub1
│   ├── prueba1
│   ├── prueba2
│   └── prueba_dir
└── sub2
```
Como podremos también comprobar, este snapshot no ocupa espacio en disco, ya que, al ser una copia _copy-on-write_, sólo se efectuará la copia cuando los ficheros sean modificados. 
```
root@arca:/mnt# btrfs fi usage /mnt
Overall:
    [...]
    Used:            401.63MiB
    [...]
```

## Balanceo de los datos
Al copiar/crear ficheros, a veces ocupan espacio de sólo disco. Podemos hacer que se reparta dicho peso entre los discos que tengamos, ejecutando el siguiente comando:
```
btrfs fi balance --full-balance /mnt
```

Y hasta aquí los dos posts sobre _Btrfs_, gracias por leer :)
