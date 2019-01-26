---
title: Diferencias de PostgreSQL y MariaDB respecto a ORACLE en cuanto a la gestión del almacenamiento 
date: 2019-01-26T15:28:00Z
tags: ["oracle", "postgresql", "mariadb", "mysql"]
---

# MariaDB, MySQL
En MariaDB no tenemos disponible la opción de crear _tablespaces_.
A partir de la versión 4.0 de MySQL se incluye InnoDB, que es un motor de almacenamiento, cuyo predecesor fue MyISAM (actualmente en desuso debido a la gran superioridad de InnoDB), el cual sí incluye esta funcionalidad.

En MySQL tenemos dos opciones para crear los _tablespaces_; con el motor de almacenamiento InnoDB y, por otro lado, con NDB. 
InnoDB nos permite establecer el tamaño de bloque y la encriptación:
```
CREATE [UNDO] TABLESPACE tablespace_name
    [ADD DATAFILE 'file_name']
    [FILE_BLOCK_SIZE = value]
    [ENCRYPTION [=] {'Y' | 'N'}]
    [ENGINE [=] InnoDB]
;
```
Por otra parte, NDB nos aporta varios parámetros más, aunque la mayoría son ignorados, ya que provocan errores.
```
CREATE [UNDO] TABLESPACE tablespace_name
    [ADD DATAFILE 'file_name']
    USE LOGFILE GROUP logfile_group
    [EXTENT_SIZE [=] extent_size]
    [INITIAL_SIZE [=] initial_size]
    [AUTOEXTEND_SIZE [=] autoextend_size]
    [MAX_SIZE [=] max_size]
    [NODEGROUP [=] nodegroup_id]
    [WAIT]
    [COMMENT [=] 'string']
    [ENGINE [=] NDB]
;
``` 
El parámetro USE LOGFILE GROUP es necesario para la creación del _tablespace_ usando el motor NDB. Previamente, habrá que crearlo mediante la siguiente sentencia:
```
CREATE LOGFILE GROUP logfile_group;
```
Los demás parámetros son opcionales, aunque la mayoría no están soportados:

- **EXTENT_SIZE**: Establece el tamaño en bytes de las extensiones utilizadas por los ficheros que pertenecen al espacio de tablas. Por defecto tiene el valor de 1M, el tamaño mínimo es 32k y el máximo teórico es de 2GB (Depende de varios factores).
- **INITIAL_SIZE**: Establece el tamaño en bytes del fichero que fue especificado con la cláusula **ADD DATAFILE**. Por defecto tiene el valor de 128M.
- **AUTOEXTEND_SIZE**, **MAX_SIZE**, **NODEGROUP**, **WAIT**, **COMMENT**: Actualmente son ignorados. Se reservan para un posible uso futuro.

En MySQL no tenemos muchas de las cláusulas de almacenamiento con las que sí contamos en ORACLE entre ellas, las relacionadas con el número de extensiones y el tamaño de las mismas, como por ejemplo NEXT, MINEXTENTS, MAXEXTENTS, PCTINCREASE, etc. Lo que si podemos hacer es establecer el tamaño inicial y máximo de la extensión con INITIAL_SIZE y MAX_SIZE.

Al contrario que en ORACLE, en MySQL no tenemos nada similar al parámetro PCTINCREASE anteriormente mencionado, por lo que esto nos resta bastante flexibilidad a la hora de gestionar el almacenamiento de nuestra base de datos.

# PostgreSQL
En PostgreSQL no existe o no hemos sido capaces de encontrar nada con respecto a la gestión de almacenamiento. 

## Conclusiones
En resumen, las cláusulas de almacenamiento de MySQL con respecto a las de ORACLE están **bastante** limitadas, ya que nos encontramos con una funcionalidad muy reducida a la que nos encontraríamos al trabajar con este último sistema gestor de bases de datos.
Además, podemos ver que la mayoría de opciones provocan errores con el motor de almacenamiento por defecto de MySQL. Por lo tanto, sería como si no tuviéramos estas opciones. 
