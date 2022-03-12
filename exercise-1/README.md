# Ejercicio 1
Crear un `deployment` en el cual la imagen utilizada sea una personalizada en donde se muestren sus datos, agregar un `service`, `ingress-controller` y con la herramienta [external dns](https://github.com/kubernetes-sigs/external-dns) agregar un DNS a la pagina. 

1. Para tener un contenedor el cual despliegue nuestros datos la crearemos. En la carpeta `customImage` esta el Dockerfile utilizado. Basicamente lo que hacemos es usar la imagen base de nginx, reemplazar el contenido que trae por defecto por uno que muestre nuestros datos por eso tenemos un index.html. Posteriormente creamos la imagen y la submios a dockerhub para poder acceder a ella luego.

2. Creamos el archivo `deployment.yml` el cual como su nombre indica es un deployment que correra 1 replica de nuestra imagen creada previamente. 
    ```
    kubectl apply -f deployment.yml
    ```

3. Creamos el archivo `service.yml` el cual es un servicio que apunta a nuestros pods creados previamente por nuestro deployment
    ```
    kubectl apply -f service.yml
    ```

4. Service Account para el ingress-controller 
    - Referencia: https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/
    - Crear un rol 
    - El tipo entidad de rol a usar es `Web identity` (Para usar este tipo debemos de tener un identity provider para nuestro cluster).
    - Volvemos a IAM Console, seleccionmos la opcion de `Identity Providers`
      + Seleccionamos la opcion de `Add provider`
      + Seleccionamos la opcion de `OpenID Connect`
      + Y la URL es la que se encuentra en la configuracion de nuestro cluster creado en la seccion de detalles
      + Audience: `sts.amazonaws.com`
    - Seleccionamos el identity provider creado para nuestro cluster
    - Los permisos a asignar los podemos ver en la referencia brindada 
      + Creamos la policy el json brindado
    - Le asignamos esa policy al rol
    - Le asignamos un nombre: `aws-load-balancer-controller-rx`
    - Luego de crear el rol en la seccion de 'Trust relationships' editamos la condicion de la 'trusted entities'
        ```
        #Actual: 
        "StringEquals": {
            "oidc.eks.us-east-1.amazonaws.com/id/C001D3B51A8E80E7635ABA01E026A525:aud": "sts.amazonaws.com"
        }
        #Nuevo
        "StringEquals": {
            "oidc.eks.us-east-1.amazonaws.com/id/C001D3B51A8E80E7635ABA01E026A525:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller-rx"
        }
        "system:serviceaccount:NAMESPACE:SERVICE_ACCOUNT_NAME"
        ```

    - Se crea el archivo `service-account.yml` 
        + El nombre del service account tinene que ser el mismo que el que le asignamos arriba (kube-system:aws-load-balancer-controller-rx)
        + El namespace tambien tiene que ser el mismo que indicamos arriba (kube-system)
        + El rol que va a usar es el que creamos previamente. El ARN se puede obtener en la consola de aws buscando el rol.

    - Agregaremos el controller al cluster utilizando `helm`
        ```
        helm install ingress eks/aws-load-balancer-controller -n kube-system --set clusterName=training-rx --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller-rx
        ```
    Para verificar que nuestro ingress fue instalado correctamente podemos ver los pods del namespace kube-system (ya que ese fue el especificado).
    
    <p align="center">
        <img src="https://user-images.githubusercontent.com/30850990/158033392-93591304-06c4-471f-9ea9-37f196e907ff.png"/>
    </p>

5. Crear el ingress
    - Para ello tenemos el archivo `ingress.yml` en el cual indicamos las reglas de ruteo. De momento solo tenemos la principal `/` la cual redirigera a nuestro pod creado previamente. 
        ```
        kubectl apply -f ingress.yml
        ```
    - Al momento de crearlo, el cluster notara la necesidad de un balanceador de carga y lo creara por nosotros. 

    <p align="center">
        <img src="https://user-images.githubusercontent.com/30850990/158033566-4fde248b-cb2f-4d3e-b5c1-786a49685a12.png"/>
    </p>

    <p align="center">
        <img src="https://user-images.githubusercontent.com/30850990/158033713-f0a45239-0248-491b-8307-b8b88bbde748.png"/>
    </p>

    - Si vamos al DNS brindado, nos mostrara la pagina del contenedor creado:
    <p align="center">
        <img src="https://user-images.githubusercontent.com/30850990/158033747-4ad03382-4648-4fe6-90f3-d815c1d8e6c1.png"/>
    </p>
    
6. Si bien el dns de balanceador de carga brindado no es muy amigable podemos registrar un record  bajo el dominio de una hosted zone ya creada en Route53. Para esto podemos usar `external-dns`.
    - Referencia: https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md
    - Crear una policy segun los permisos necesarios brindados en la referencia.
    - Crear un rol de tipo identity provider y asignarle la policy anterior
    - De igual manera que se hizo con el ingress controller, ya que usara un serviceAccount modificar la `trust relationship` del rol.
    - Crear un manifest (utilizar el brindado en la referencia y cambiarle los campos necesarios) llamado `external-dns.yaml`
        ```
        kubectl apply -f external-dns.yaml
        ```
    - Para verificar que esto fue exitoso podemos ir a Route53 y verificar los registros:

    <p align="center">
        <img src="https://user-images.githubusercontent.com/30850990/158034071-d6ba7868-7484-4942-80fa-a723a2cbcf71.png"/>
    </p>


    - De igual manera si vamos a la url `raulx.training.trambo.cloud` nos muestra lo mismo que la dns del balanceador.

    <p align="center">
        <img src="https://user-images.githubusercontent.com/30850990/158034105-a75767d0-9b68-48da-84c3-50d246638764.png"/>
    </p>
    