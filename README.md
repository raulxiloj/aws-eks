# Amazon EKS

<p>
<img src="https://user-images.githubusercontent.com/30850990/158027444-c2d8186f-1eba-4753-8294-03149f55f201.png" width="400">
</p>


Amazon Elastic Kubernetes Service (EKS) es un servicio administrado para ejecutar Kubernetes en AWS sin necesidad de instalar, operar o mantener el cluster como tal o los nodos de k8s. Kubernetes es un orquestador para automatizar la implementacion, escalado y administracion de las aplicaciones en contenedores.

Este repositorio contiene una serie de ejercicios en los cuales se practica eks.

## Cluster
Antes de iniciar con los ejercicios necesitamos crear un cluster.

1. Crear un rol el cual sera responsable de manejar los recursos de kubernetes en aws.
    - Dirigirse a la consola IAM
    - Seleccionar la opcion crear rol
    - En la seccion caso de uso seleccionar EKS
    - Por ultimo seleccionar **EKS - Cluster**
    - Por defecto este rol ya tiene asignado una 'policy' para administrar un cluster de eks *AmazonEKSClusterPolicy*

<p align="center">
<img src="https://user-images.githubusercontent.com/30850990/158028533-f1975a2a-8695-44d6-bb9a-52bcfcba94f4.png" height="375px" />
</p>

2. Crear el cluster
    - Asignarle un nombre `training-rx`
    - Agregar el rol creado
    - Seleccionar la vpc en la que se quiera crear el cluster
    - Seleccionar las subnets en donde se deseen crear los nodos
        + Se necesitan al menos dos subnets en diferente AZ

<p align="center">
    <img src="https://user-images.githubusercontent.com/30850990/158028947-b8295f18-4153-45d4-8713-93bf4e98e00d.png" height="150px">
</p>


3. Agregar el cluster a nuestro kube config.
    - Para agregar el contexto del cluster a nuestro kube config local debemos ejecutar el siguiente comando indicando la region.
    ```
    aws eks update-kubeconfig --name training-rx --region=us-east-1
    ```

    - Para verificarlo podemos obtener los contextos y confirmar que tengamos el que agregamos.
<p align="center">
    <img src="https://user-images.githubusercontent.com/30850990/158029189-e2e9d401-9ea0-48f2-81cc-118c6a0ab5ce.png"  />
</p>

4. Worker Nodes: Por defecto cuando creamos un cluster lo unico que creamos es el nodo master al cual no tenemos acceso ya que este servicio es administrado, los nodos donde desplegaremos nuestros pods se llaman `worker nodes` y esos los tenemos que crear. 
    1. Crear un rol el cual manejara los node groups
        - IAM Console
        - Seleccionar la opcion crear rol
        - Seleccionar AWS Service
        - Seleccionar EC2
        - Agregar las siguientes policies: **AmazonEKSWorkerNodePolicy**, **AmazonEKS_CNI_Policy**, **AmazonEC2ContainerRegistryReadOnly**
    2. Dentro del cluster, irse a la opcion de configuracion y seleccionar la opcion de `compute`. 
        - Agregar un Node Group
        - Asignarle un nombre
        - Agregarle el rol previamente creado
        - Elegir el tipo de instancia para los nodos
        - El tamano del disco
        - Cantidad de nodos 
        - Subnets en donde se van a crar los nodos (instancias)

<p align="center">
    <img src="https://user-images.githubusercontent.com/30850990/158029925-129813ac-be9f-475a-9cd7-433beb1ac67e.png" height="150" />
</p>

Si obtenemos los nodos desde la linea de comandos `kubectl` nos mostrara los nodos creados. 
<p align="center">
    <img src="https://user-images.githubusercontent.com/30850990/158029994-119adb70-f360-4f70-a52b-c7cdfb571ce9.png">
</p>

Y tenemos listo nuestro cluster para poder ejecutar deployments, pods, services, etc. 🙂

## Ejercicios
- [Desplegar un contenedor que contenga tus datos y agregar un dns.](exercise-1/README.md)


## ⚙️ Requerimientos
- Una cuenta de aws
- aws cli
- kubectl 
- Conocimiento en kubernetes