---
title: Crear un CronJob para reiniciar deployment en AKS 
date: 2021-01-22T15:28:00Z
tags: ["aks", "kubernetes", "k8s", "cronjob"]
---

Tras estar bastante tiempo sin subir un post, he decidido subir cosas interesantes que haga en mi trabajo que considere que pueden ser útiles o que, simplemente, quiera tener en algún sitio y así no perderlo de vista :)

Hoy os hablaré de cómo crear un recurso _cronJob_ en AKS para manejar el reinicio de uno (O varios) _deployments_. Realmente esto se podría hacer en cualquier instancia de Kubernetes, con lo cual no aplica sólo a la plataforma de Azure. Aunque es en servicios gestionados como este donde esta solución llega a ser más interesante.

Lo que haremos será crear un recurso _cronJob_ que reinicie el _deployment_ indicado a la hora indicada. Este _cronJob_ lanzará un _pod_ que ejecutará _kubectl_ para llevar a cabo el reinicio.

# Creación del serviceAccount
Para esto, lo primero que haremos será crear un recurso _serviceAccount_. Este será usado por el _pod_ para autenticarse y poder hacer uso de la API interna del _cluster_ de k8s. Ejecutaremos el siguiente comando para su creación:
```
kubectl create serviceaccount deployment-restart-sa
```

# Creación del role
Una vez tengamos creado este _serviceAccount_, necesitaremos un recurso _role_ que definirá los permisos que tendrá dicho _serviceAccount_.
Para crear este _role_, generamos un fichero YAML que contenga lo siguiente:
```
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-restart-role
  namespace: <nombre-del-namespace>
rules:
  - apiGroups: ["apps", "extensions"]
    resources: ["deployments"]
    resourceNames: ["<deployment-a-reiniciar>"] # Podemos añadir varios deployments separando con comas
    verbs: ["get", "patch"]
...
```

Aquí, el apartado _verbs_ definirá las acciones que podrá realizar el _serviceAccount_. En este caso _get_ y _patch_ serán las acciones que nos permitirán obtener información sobre el _deployment_ y modificarlo.
Teniendo listo el fichero, crearemos el _role_ con:
```
kubectl apply -f <nombre-del-fichero>
```

# Creación del roleBinding
Acto seguido, tenemos que definir un recurso _roleBinding_ para relacionar el _role_ con el _serviceAccount_ anteriormente creados.
**IMPORTANTE: Si no creamos este _roleBinding_, el _serviceAccount_ no tendrá asignados los permisos necesarios para poder reiniciar el _deployment_, con lo cual el _cronJob_ no funcionará.**

Crearemos el _roleBinding_ con el siguiente comando:
```
kubectl create rolebinding deployment-restart-rb --role=deployment-restart-role --serviceaccount=<namespace>:deployment-restart-sa
```

Aquí es importante ver que al especificar el _serviceAccount_, tenemos que especificar también el _namespace_ en el que reside.

# Creación del cronJob
Por último, crearemos el recurso _cronJob_ que se encargará del reinicio. Para esto, crearemos otro fichero YAML con el siguiente contenido:
```
---
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: deployment-restart
  namespace: <nombre-del-namespace>
spec:
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 1
  schedule: '0 2 * * *' # Aquí definimos la hora a la que se ejecuta, en este caso a la 2:00 AM
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 100
      template:
        spec:
          serviceAccountName: deployment-restart-sa
          restartPolicy: Never
          containers:
            - name: kubectl
              image: bitnami/kubectl
              command:
                - 'kubectl'
                - 'rollout'
                - 'restart'
                - 'deployment/<deployment-a-reiniciar>'
...
```

En este fichero, algunas partes a destacar son:
* Cuando establecemos la hora a la que queremos que se ejecute el _job_, tendremos que tener en cuenta la zona horaria del _cluster_ AKS, que por defecto es UTC.
* Tener  _currencyPolicy_ en _Forbid_ significa que no se permiten _jobs_ concurrentes. Es decir, si cuando el _job_ se va a ejecutar todavía hay otro ejecutándose, este _job_ nuevo no se ejecuta.
* El parámetro _backoffLimit_ define el número de intentos que se llevarán a cabo antes de considerar el _job_ como "fallido".
* El parámetro _activeDeadlineSeconds_ es una forma de terminar el _job_, definiendo el tiempo en segundos que pasará hasta la terminación de este.
* _successfulJobsHistoryLimit_ define el número de ejecuciones exitosas que se retendrán. Estando en "1", significa que persistirán los recursos creados por la última ejecución exitosa del _job_. Si queremos que se borren automáticamente los recursos correspondientes (Normalmente _pods_), simplemente tendremos que ponerlo a "0".

Teniendo ya el fichero completo, crearemos el recurso utilizando de nuevo el comando:
```
kubectl apply -f <nombre-del-fichero>
```

Finalmente, podremos utilizar el siguiente comando para verificar que nuestro _cronJob_ se ha creado:
```
kubectl get cronjobs
``
* El parámetro _backoffLimit_ define el número de intentos que se llevarán a cabo antes de considerar el _job_ como "fallido".
* El parámetro _activeDeadlineSeconds_ es una forma de terminar el _job_, definiendo el tiempo en segundos que pasará hasta la terminación de este.
* _successfulJobsHistoryLimit_ define el número de ejecuciones exitosas que se retendrán. Estando en "1", significa que persistirán los recursos creados por la última ejecución exitosa del _job_. Si queremos que se borren automáticamente los recursos correspondientes (Normalmente _pods_), simplemente tendremos que ponerlo a "0".

Teniendo ya el fichero completo, crearemos el recurso utilizando de nuevo el comando:
```
kubectl apply -f <nombre-del-fichero>
```

Finalmente, podremos utilizar el siguiente comando para verificar que nuestro _cronJob_ se ha creado:
```
kubectl get cronjobs
````
