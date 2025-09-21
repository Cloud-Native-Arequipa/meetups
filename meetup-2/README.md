
# Guía Paso a Paso: Desplegando Aplicaciones con Kind, Helm y HPA en Kubernetes Local

Esta guía te muestra cómo levantar un clúster local con Kind, instalar herramientas necesarias, desplegar una app con Helm, y configurar autoescalado usando HPA.

----------

## 1. Instalación de herramientas (MacOS)
Se instalan `kind` (para crear clústers locales), `kubectl` (cliente de Kubernetes) y se verifica la versión del cliente kubectl.
### MacOS
```
brew install kind
brew install kubectl
kubectl version --client
```


### Instalación de Kind en Linux
Descarga el binario de Kind, lo vuelve ejecutable y lo instala en el sistema.
```
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

Verifica que Kind se haya instalado correctamente.

```
kind version
```

Kind también está disponible para otros sistemas operativos como Windows. Puedes consultar las versiones más recientes y las instrucciones oficiales en: 👉 https://kind.sigs.k8s.io/

----------

## 2. Crear el clúster Kind

Se crea un clúster local con Docker como base. "my-k8s-app" es el nombre del clúster.

```
kind create cluster --name my-k8s-app
```
Lista todos los recursos en todos los namespaces. Útil para verificar que el clúster está funcionando.
```
kubectl get all -A
```

----------


## 3. Desplegar WordPress con Helm

### 3.1 Instalar Helm y desplegar WordPress
 Instala Helm, el gestor de paquetes de Kubernetes.
```
brew install helm
```
Se agrega y actualiza el repositorio de charts Bitnami.
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```
Despliega WordPress con una base de datos, todo gestionado por Helm.
```
helm install myapp bitnami/wordpress
```
Lista los charts desplegados en el clúster.


```
helm list
```


### 3.2. Acceder a la aplicación
Se obtiene el nombre del Pod para poder exponerlo.
```
kubectl get pods
```
Redirecciona el puerto del Pod a localhost para acceder vía navegador: http://localhost:8080
```
kubectl port-forward [POD_NAME] 8080:8080
```

### 3.3. Limpieza del entorno
Elimina la app, el repositorio Helm y el clúster Kind.
```
helm uninstall myapp
helm repo remove bitnami
kind delete cluster --name my-k8s-app
```

----------

## 4. Pruebas de Stress y Autoescalado con HPA

### 4.1. Configurar Metrics Server (requisito para HPA)
Despliega el metrics-server desde el repo oficial. Necesario para medir uso de recursos.
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
Se editan los flags del container para evitar errores con TLS:
```
kubectl -n kube-system edit deployment metrics-server
```

> 

```
- --kubelet-insecure-tls
- --kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP
```
Reinicia el deployment del metrics-server con la nueva config.
```
kubectl -n kube-system rollout restart deployment metrics-server
``` 

### 4.2. Desplegar Apache con HPA
Se crea un namespace para aislar los recursos.
```
kubectl create namespace apache
```

#### Manifest YAML (Deployment, Service, HPA)

Desplegar con:

```
kubectl apply -f apache-deployment-hpa.yaml
```

Contenido del archivo:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apache-deployment
  namespace: apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: apache
  template:
    metadata:
      labels:
        app: apache
    spec:
      containers:
      - name: apache
        image: httpd:2.4
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
          limits:
            cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: apache-service
  namespace: apache
spec:
  selector:
    app: apache
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: apache-hpa
  namespace: apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: apache-deployment
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 5
```

### 4.3. Exponer y probar autoescalado
Se expone el servicio para pruebas y se observa el estado del HPA.
```
kubectl port-forward svc/apache-service -n apache 8081:80 --address 0.0.0.0 &
kubectl get hpa -n apache
```
Dentro del pod ejecuta:

```
kubectl run -i --tty load-generator --image=busybox /bin/sh
```

Esto genera carga continua para que el HPA escale los pods según el uso de CPU.

```
while true; do wget -q -O- http://apache-service.apache.svc.cluster.local; done
```
