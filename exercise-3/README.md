# Ejercicio 3 (Storage)

## Persistent Volumes y PersistenVolumeClaim
Un PersistentVolume (PV) es una pieza de almacenamiento en el cluster brindado por algun administrador y dinamicamente usando algun `storage classes`. 

Un PersistentVolumeClaim (PVC) es una solicitud de almacenamiento por un usuario. Similar a un pod (Un pod consume recursos de los nodos), PVCs consume recursos de PV.

1. Iniciamos con la creacion la 'policy' y el rol.
    - Policy: https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/v1.3.2/docs/iam-policy-example.json
    - Creacion de un rol por medio de un web identity seleccionando como proveedor el id de nuestro cluster
    - Arreglar el trust-policy.json

2. Crear un service account que tenga una `annotation` con el ARN del role previamente creado. 
    - `efs-service-account.yml`
    ```
    kubectl apply -f efs-service-account.yml
    ```

3. Instalar el driver de Amazon EFS
    - Para ello utilizaremos `helm`
    - Agregamos el repo
    ```
    helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
    ```
    - Instalamos la helm chart 
    ```
    helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
    --namespace kube-system \
    --set image.repository=602401143452.dkr.ecr.us-east-1.amazonaws.com/eks/aws-efs-csi-driver \
    --set controller.serviceAccount.create=false \
    --set controller.serviceAccount.name=efs-csi-controller-sa
    ```
    * en el parametro image.repository usar el correspondiente a la region (https://docs.aws.amazon.com/eks/latest/userguide/add-ons-images.html)

4. Creacion de un EFS file system en amazon
    - Se necesita el id de la vpc y el CIDR de la vpc. Estos parametros se pueden obtener por consola con los siguientes comandos o desde la interfaz.
        ```
        vpc_id=$(aws eks describe-cluster \
        --name k8s-training-rx \
        --query "cluster.resourcesVpcConfig.vpcId" \
        --output text \
        --region us-east-1)
        ```

        ```
        cidr_range=$(aws ec2 describe-vpcs \
        --vpc-ids $vpc_id \
        --query "Vpcs[].CidrBlock" \
        --output text \
        --region us-east-1)
        ```
    - Creacion de un grupo de seguridad con una regla de entrada que permita el trafico NFS entrante 

    - Crear el EFS en amazon (desde la linea de comandos o desde la interfaz). *Guardar el ID ya que se usara posteriormente*

    - Obtener la IP de los nodos del cluster
        ```
        kubectl get nodes
        ```
    <p align="center">
        <img src="https://user-images.githubusercontent.com/30850990/158442329-2d1fc004-6291-402d-9cb8-5205324ef4f6.png">
    </p>

    - Obtener los IDs de las subnets de la vpc en la que estamos trabajando nuestro cluster y en AZ se encuentran
        ```
        aws ec2 describe-subnets \
        --filters "Name=vpc-id,Values=$vpc_id" \
        --query 'Subnets[*].{SubnetId: SubnetId,AvailabilityZone: AvailabilityZone,CidrBlock: CidrBlock}' \
        --output table \
        --region us-east-1
        ```
        <p align="center">
            <img src="https://user-images.githubusercontent.com/30850990/158442681-1b5e3198-ceb9-4995-9039-cc02f96b3aee.png">
        </p>

    - Agregar destinos de montaje para las subredes en las que se encuentran los nodos.
        - En este ejemplo, el cluster tiene dos nodos. 1 con la IP `10.0.1.5` y otro con la IP `10.0.2.172`. El primero se encuentra dentro del bloque del CIDR de la subred que se encuentra en la AZ `1a` y el segundo en la AZ `1b`. Por consecuente debemos de crear un montaje para ambas subredes.
        - El siguiente comando crea un montaje indicandole en que subred (por lo mismo se ejecuta dos veces cambiando el paranetro `subnet-id`)
            ```
            aws efs create-mount-target \
            --file-system-id fs-01a833986ab2c80db \
            --subnet-id subnet-0f3e9ef9134c03fc7 \
            --security-groups sg-00d5a2af763edbe87 \
            --region us-east-1

            aws efs create-mount-target \
            --file-system-id fs-01a833986ab2c80db \
            --subnet-id subnet-0f0170d8d31ab86f7 \
            --security-groups sg-00d5a2af763edbe87 \
            --region us-east-1
            ```

5. Prueba de ejemplo, de manera dinamica.
    - Ya que se utilizara `dyanmic provisioning` se necesita un `StorageClass`
        - `storageclass.yml`
        ```
        kubectl apply -f storageclass.yaml
        ```
    - Se crea un PersistentVolumeClaim (pvc) 
        - `pvc.yaml`
        ```
        kubectl apply -f pvc.yaml
        ```
    - Se crea un deployment en el cual en la definicion del *template* se indica que se estara usando el volumen.
        - `deployment.yaml`
        ```
        kubectl apply -f deployment.yaml
        ```  

    - Se crea un PersistentVolume `pv.yml` en el cual se indica que se utilizara el EFS creado previamente
        ```
        kubectl apply -f pv.yml
        ```

    - Para verificar que nuestro pvc se este utilizando por los pods creados, podemos ejecutar el comando:
        ```
        kubectl describe pvc efs-claim
        ```
    <p align="center">
        <img src="https://user-images.githubusercontent.com/30850990/158451242-f98ad05e-49cc-499d-ba55-1ba36c7f537a.png">
    </p>

    - Si abrimos una terminal en pods creados en diferentes nodos, podremos comprobar que comparten el mismo volumen.

    ![Screen Shot 2022-03-15 at 13 29 25](https://user-images.githubusercontent.com/30850990/158456635-81dbb63d-13ab-47fc-9f06-c797b67fae5e.png)

    ![Screen Shot 2022-03-15 at 13 29 39](https://user-images.githubusercontent.com/30850990/158456642-b323a56e-460c-4807-8b45-cc658b335258.png)

    - Creamos un archivo desde el pod en el nodo 1
    ![Screen Shot 2022-03-15 at 13 30 05](https://user-images.githubusercontent.com/30850990/158456648-09045f7c-092c-40be-87cd-833b36d40517.png)

    - Verificamos desde el pod en el nodo 2 tambien se vea reflejado :) 
    ![Screen Shot 2022-03-15 at 13 30 17](https://user-images.githubusercontent.com/30850990/158456655-62915c14-1778-4552-a33b-117479d1a12d.png)

- Video de ejemplo:
https://user-images.githubusercontent.com/30850990/158456061-b26af9d4-a6b3-4831-b3f2-35bd18715ada.mov

## Referencias
- https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html