---
title: Diferencias de PostgreSQL y MariaDB respecto a ORACLE en cuanto a la gestión del almacenamiento 
date: 2019-01-26T15:28:00Z
tags: ["oracle", "postgresql", "mariadb"]
---

# MariaDB
A partir de la versión 4.0 de MySQL se incluye InnoDB, que es un motor de almacenamiento, cuyo predecesor fue MyISAM (actualmente en desuso debido a la gran superioridad de InnoDB).
En MariaDB no tenemos disponible la opción de crear _tablespaces_.

En MySQL no tenemos muchas de las cláusulas de almacenamiento con las que sí contamos en ORACLE entre ellas, las relacionadas con el número de extensiones y el tamaño de las mismas, como por ejemplo NEXT, MINEXTENTS, MAXEXTENTS, PCTINCREASE, etc. Lo que si podemos hacer es establecer el tamaño inicial y máximo de la extensión con INITIAL_SIZE y MAX_SIZE.

Al contrario que en ORACLE, en MariaDB no tenemos nada similar al parámetro PCTINCREASE anteriormente mencionado, por lo que esto nos resta bastante flexibilidad a la hora de gestionar el almacenamiento de nuestra base de datos.

En resumen, las cláusulas de almacenamiento de MySQL con respecto a las ORACLE están bastante limitadas, ya que nos encontramos con una funcionalidad muy reducida a la que nos encontraríamos al trabajar con este último sistema gestor de bases de datos.

# PostgreSQL
En PostgreSQL no existe o no hemos sido capaces de encontrar nada con respecto a la gestión de almacenamiento. 
