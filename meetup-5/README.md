# Meetup 5: Helm y GitOps con Flux

## Helm

### Comandos utilizados

```sh
# Crear un nuevo chart de Helm
helm create hello-world-chart

# Eliminar los templates por defecto
templates/*
rm -rf templates/*

# Validar el chart
templates/
cd templates
helm lint hello-world-chart

# Renderizar los manifiestos
templates/
helm template .

# Instalar el chart en modo dry-run
helm install --dry-run hello-release hello-world-chart

# Instalar el chart
helm install hello-release hello-world-chart

# Instalar nuevamente 
helm install hello-release hello-world-chart 

# Actualizar el release
helm upgrade hello-release hello-world-chart

# Hacer rollback del release
helm rollback hello-release
helm rollback <release-name> <revision-number>

# Empaquetar el chart
helm package hello-world-chart/
```

---

## Flux

### Instalación y primeros pasos

```sh
# Ver versión de Flux
flux --version

# Instalar Flux en el clúster
flux install 

# Verificar pods de Flux
kubectl get pods -n flux-system

El repositorio fuente gitops fue actualizado al siguiente: https://github.com/Cloud-Native-Arequipa/flux-cncf-aqp-gitops

# Variables de entorno para bootstrap con GitHub
export GITHUB_USER="cloud-native-arequipa"
export GITHUB_REPO="flux-cncf-aqp-gitops"
export GITHUB_BRANCH="main"
export FLUX_PATH="clusters/dev"

# Bootstrap de Flux con GitHub
flux bootstrap github \
  --owner=${GITHUB_USER} \
  --repository=${GITHUB_REPO} \
  --branch=${GITHUB_BRANCH} \
  --path=${FLUX_PATH} \
  --personal

# Verificar pods en todos los namespaces
kubectl get po -A

# Consultar fuentes y kustomizations de Flux
flux get sources git -A
flux get kustomizations -A
```

### Reconciliaciones manuales

```sh
flux reconcile source helm bitnami-oci -n flux-system
flux reconcile helmrelease wordpress -n bitnami
flux reconcile source helm bitnami-oci -n flux-system
```

---

> Este README documenta los comandos y pasos principales realizados durante el Meetup 5 para trabajar con Helm y GitOps usando Flux.
