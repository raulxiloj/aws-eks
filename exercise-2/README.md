# Ejercicio 2 (HPA)
Hacer que un deployment incremente sus replicas debido al alto uso de cpu.

HorizontalPodAutoscaler (HPA como abreviatura) automaticamente actualiza un recurso con el fin de escalar bajo demanda. Que sea horizontal se refiere a que se crean mas pods. 

Cuando la carga de trabajo baje y el numero de pods configurados siga arriba del necesario el HPA manda a escalar hacia abajo. 

Para este ejemplo se utilizara una imagen de un apache que lo unico que hace es un `for` y esto con el fin de estar aumentando el cpu. 

1. Se crea un deployment utilizando una imagen llamada `k8s.gcr.io/hpa-example`.  Se crea un servicio y se expone el apache.

2.  Desde la linea de comandos el autoscaling se realiza de la siguiente manera:
    ```
    kubectl autoscale deployment hpa-deployment --cpu-percent=50 --min=1 --max=20
    ```

    para verificar que se creo de manera correcta podemos verificar con el siguiente comando: 
    <p align="center">
        <img src="https://user-images.githubusercontent.com/30850990/158038531-992f3fee-2449-49f1-ba64-0c16f11ea9a3.png"/>
    </p>
    
3. Como se puede observar en las metricas se muestra `<unknown>/50%` y esto es debido a que el cluster como tal no tiene una herramienta la cual ayude a brindar ciertas metricas. Por lo cual hay que instalar alguna, existen varias herramientas como [Prometheus](https://prometheus.io/), [metrics-server](https://artifacthub.io/packages/helm/metrics-server/metrics-server), entre otras.

    - Primero agregamos el repo a helm
        ```
        helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
        ```
    - Luego instalamos el chart de helm
        ```
        helm upgrade --install metrics-server metrics-server/metrics-server
        ```
    *Nota: Esta herramienta no necesita de un serviceAccount ya que no utiliza recursos de aws.*

<p align="center">
    <img src="https://user-images.githubusercontent.com/30850990/158039271-fc4baa8a-4884-4fd6-8bf0-e2bf62ed6909.png"/>
</p>

- Si volvemos a verificar nuestro hpa sigue mostrando `<unknown>/50%` si ejecutamos el comando `describe` hacia el hpa nos mostrara los eventos que estan ocurriendo y se puede identificar de mejor manera el problema. 
<p align="center">
    <img src="https://user-images.githubusercontent.com/30850990/158039339-267b410e-4f56-4867-93ee-c6553c5dbf9f.png"/>
</p>

- Como se puede observar el error es diferente, antes no existia el recurso `pods.metrics.k8s.io` ahora el error cambio a `failed to get cpu utilization: missing request for cpu`. Basicamente el error es que los pods no tiene un limite de cpu definido, por lo mismo k8s no puede hacer el calculo sobre el porcentaje establecido en el hpa. Esto se especifica en el deployment en la seccion de `resources` a nivel de containers.

- Luego de aplicar las nuevas configuraciones, volvemos a ejecutar el comando `get hpa` para verificar que ya no haya inconveniente y nos muestra `0%/50%` lo cual indica que esta correcto y esta pudiendo calcular el uso del cpu.
<p align="center">
    <img src="https://user-images.githubusercontent.com/30850990/158039868-1508884f-3083-4feb-9e52-2ce0610b72bf.png"/>
</p>


4. Para incrementar la carga de cpu, podemos hacer multiples peticiones hacia ese ese pod. Para ello crearemos un pod temporal y desde ahi haremos la llamada.

    ```
    kubectl run -it test --image=busybox -- sh
    ```

    - Ahora con el comando wget podemos hacer la llamada hasta nuestro hpa 
    ```
    wget -q -O- http://hpa-service
    ```

- Video de ejemplo:

https://user-images.githubusercontent.com/30850990/158040605-30b453fa-fc62-456a-9711-5cbaf8989e21.mov

# Referencia:
- https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/