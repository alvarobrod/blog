---
title: Diferencias de PostgreSQL y MariaDB respecto a ORACLE en cuanto a la gestión del almacenamiento 
date: 2019-01-26T15:28:00Z
tags: ["oracle", "postgresql", "mariadb"]
---

# MariaDB
A partir de la versión 4.0 de MySQL se incluye InnoDB, que es un motor de almacenamiento, cuyo predecesor fue MyISAM (actualmente en desuso debido a la gran superioridad de InnoDB).
En MariaDB no tenemos disponible la opción de crear _tablespaces_.

En MySQL no tenemos muchas de las cláusulas de almacenamiento con las que sí contamos en ORACLE:

- El parámetro NEXT no está en MySQL.
- El parámetro MINEXTENTS no está en MySQL.
- Tampoco está MAXEXTENTS, sólo podremos definir el tamaño máximo de la extensión.
- No podremos establecer un _tablespace_ de "sólo lectura".
- No tendremos la opción de hacer que un _tablespace_ se vaya incrementando de manera porcentual. 

En resumen, las cláusulas de almacenamiento de MySQL con respecto a las ORACLE están bastante limitadas, ya que nos encontramos con una funcionalidad muy reducida a la que nos encontraríamos al trabajar con este último sistema gestor de bases de datos.

# PostgreSQL
En PostgreSQL no existe o no hemos sido capaces de encontrar nada con respecto a la gestión de almacenamiento. 
