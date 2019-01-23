---
title: Integración Continua: Construyendo un blog con Hugo + Travis + GitHub Pages
date: 2019-01-22T14:00:00Z
---

## Integración Continua: Construyendo un blog con Hugo + Travis + GitHub Pages
En esta entrada veremos cómo crear un blog como este mismo, usando integración continua con las herramientas Hugo, Travis y GitHub Pages.

Primero, creamos el ([repositorio GitHub](https://github.com/alvarobele/blog-ic)) y en él crearemos la rama _gh-pages_, donde estarán los ficheros generados por Hugo, mientras que en la rama _master_ será donde tendremos los ficheros Markdown, así como el fichero _.travis.yml_ y el _config.toml_ que exportaremos a Travis en el proceso de Integración Contínua.

El fichero _.travis.yml_ será el siguiente:
```
install:
  - wget https://github.com/gohugoio/hugo/releases/download/v0.53/hugo_0.53_Linux-64bit.deb && sudo dpkg -i hugo*.deb
  - git clone https://github.com/alvarobrod/blog.git

script:
  - rm hugo*.deb
  - hugo new site sitio
  - mkdir -p sitio/content/post
  - mv blog/content/post/* sitio/content/post
  - mv blog/config.toml sitio/
  - cd sitio/themes && git clone https://github.com/nirocfz/arabica && cd ..
  - hugo -t arabica
  - rm -rf themes/arabica

deploy:
  provider: pages
  local-dir: sitio/public
  skip-cleanup: true
  github-token: $GH_TOKEN
  keep-history: true
  on:
    branch: master
```
  

Este fichero será el que le diga a Travis el proceso a seguir.

Y el fichero _config.toml_ será el siguiente:
```
baseURL = "/blog"
title = "Álvaro Beleño"
author = "Álvaro Beleño"
paginate = 3
theme = "arabica"

[params]
description = "Administración de sistemas"
twitter = "alvarobrod"
```

Una vez hecho esto, crearemos la carpeta _post_  en el repositorio GitHub, donde guardaremos los ficheros Markdown que Hugo convertirá en nuestros posts.

Ahora, para hacer que Travis pueda hacer _commits_ a nuestro repositorio GitHub, tendremos que crear un Token de autenticación. Para esto, en GitHub debemos irnos a la sección  _Settings_ y desde ahí a _Developer settings_ y _Personal access tokens_. Una vez tengamos creado este  Token, añadiremos el mismo como una variable de sistema en Travis, yendo a la página del repositorio y a _Settings_  (Habiendo activado nuestro repositorio en Travis primero).
