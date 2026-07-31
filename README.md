# InvenTrack en Kubernetes

Manifiestos para desplegar InvenTrack en un cluster local Minikube con Ingress Nginx bajo el dominio de laboratorio `conjunta3p.espe.edu.ec`.

## Arquitectura

- Namespace: `inventrack-prod`.
- Frontend React servido por Nginx no privilegiado.
- Backend Node.js/Express disponible internamente en el puerto 4000.
- MySQL 8 con almacenamiento persistente.
- Services de tipo `ClusterIP`.
- Ingress para `/`, `/api` y `/uploads`.
- NetworkPolicies para restringir el ingreso entre componentes.
- Secret real creado directamente en el cluster y excluido de Git.

## Archivos

| Archivo | Recurso |
|---|---|
| `namespace.yml` | Namespace `inventrack-prod` |
| `configmap.yml` | Configuracion no sensible |
| `mysql-init-configmap.yml` | Esquema inicial de MySQL |
| `mysql.yml` | PVC, Deployment y Service de MySQL |
| `backend.yml` | PVC de uploads, Deployment y Service del backend |
| `frontend.yml` | Deployment y Service del frontend |
| `ingress.yml` | Rutas del dominio de laboratorio |
| `networkpolicy.yml` | Restricciones de ingreso entre pods |
| `secret.yml` | Plantilla con marcadores; no debe aplicarse con esos valores |
| `kustomization.yml` | Conjunto aplicable; excluye `secret.yml` |

## Requisitos

Ejecutar Docker, kubectl y Minikube desde **WSL Ubuntu**:

```bash
docker version
kubectl version --client
minikube version
```

Se necesita tambien una copia de la aplicacion:

```bash
git clone https://github.com/agcudco/conjunta-desarrollo-seguro.git
git clone https://github.com/gamurigm/inventrack-k8s.git
```

## 1. Construir las imagenes

Desde la raiz de `conjunta-desarrollo-seguro`:

```bash
docker build -t inventrack-backend:1.0.0 ./backend
docker build -t inventrack-frontend:1.0.0 ./frontend
docker build -t inventrack-mysql:1.0.0 ./mysql-init
```

Verificar:

```bash
docker images 'inventrack-*'
```

## 2. Iniciar Minikube e Ingress

```bash
minikube start --driver=docker --cpus=4 --memory=6144
minikube addons enable ingress
kubectl config current-context
kubectl get nodes
kubectl -n ingress-nginx rollout status deployment/ingress-nginx-controller --timeout=180s
```

El contexto activo debe ser `minikube` y el nodo debe aparecer `Ready`.

## 3. Cargar las imagenes en Minikube

```bash
minikube image load inventrack-backend:1.0.0
minikube image load inventrack-frontend:1.0.0
minikube image load inventrack-mysql:1.0.0
minikube image ls | grep inventrack
```

## 4. Crear el Secret real sin versionarlo

Desde `inventrack-k8s`, crear primero el namespace:

```bash
kubectl apply -f namespace.yml
```

Capturar los valores sin imprimirlos en terminal:

```bash
read -rsp 'DB password: ' LAB_DB_PASSWORD; echo
read -rsp 'JWT secret: ' LAB_JWT_SECRET; echo
read -rsp 'Groq API key: ' LAB_GROQ_API_KEY; echo

kubectl -n inventrack-prod create secret generic inventrack-secret \
  --from-literal=DB_PASSWORD="$LAB_DB_PASSWORD" \
  --from-literal=JWT_SECRET="$LAB_JWT_SECRET" \
  --from-literal=GROQ_API_KEY="$LAB_GROQ_API_KEY" \
  --dry-run=client -o yaml | kubectl apply -f -

unset LAB_DB_PASSWORD LAB_JWT_SECRET LAB_GROQ_API_KEY
```

`secret.yml` contiene unicamente marcadores documentales. No se debe introducir ni confirmar alli ningun valor real.

## 5. Desplegar los manifiestos

```bash
kubectl apply -k .
kubectl -n inventrack-prod rollout status deployment/mysql --timeout=180s
kubectl -n inventrack-prod rollout status deployment/backend --timeout=180s
kubectl -n inventrack-prod rollout status deployment/frontend --timeout=180s
```

Verificar todos los recursos:

```bash
kubectl get all -n inventrack-prod -o wide
kubectl get ingress,pvc,configmap,secret,networkpolicy -n inventrack-prod
kubectl get events -n inventrack-prod --sort-by='.lastTimestamp'
```

Resultado esperado: los tres pods aparecen `1/1 Running`, MySQL conserva su PVC y el Ingress publica el host requerido.

## 6. Resolver el dominio local

Obtener la IP:

```bash
minikube ip
```

Agregar una unica entrada en `/etc/hosts` dentro de WSL Ubuntu o Kali:

```text
<MINIKUBE_IP> conjunta3p.espe.edu.ec
```

Confirmar que no se esta apuntando a infraestructura publica:

```bash
getent ahostsv4 conjunta3p.espe.edu.ec
test "$(getent ahostsv4 conjunta3p.espe.edu.ec | awk 'NR==1{print $1}')" = "$(minikube ip)"
```

## 7. Validar la aplicacion

```bash
curl -i http://conjunta3p.espe.edu.ec/api/health
curl -I http://conjunta3p.espe.edu.ec/
```

Abrir en el navegador:

```text
http://conjunta3p.espe.edu.ec
```

Si Windows no alcanza directamente la IP de Minikube, usar el puente local documentado en el informe PTES.

## Diagnostico rapido

```bash
kubectl get pods -n inventrack-prod
kubectl describe pod -n inventrack-prod <POD>
kubectl logs -n inventrack-prod deployment/backend --tail=100
kubectl logs -n inventrack-prod deployment/mysql --tail=100
kubectl describe ingress inventrack-ingress -n inventrack-prod
```

- `ImagePullBackOff`: comprobar las etiquetas y repetir `minikube image load`.
- Backend sin conexion: revisar `DB_HOST`, existencia del Secret y readiness de MySQL.
- Ingress sin respuesta: confirmar el addon, la IP local y la entrada de `hosts`.
- `init.sql` no vuelve a ejecutarse: MySQL solo inicializa un directorio de datos vacio; no borrar el PVC sin respaldo.

## Seguridad

- No confirmar `.env`, tokens, contrasenas ni API keys.
- No aplicar `secret.yml` con los marcadores sin reemplazarlos mediante el procedimiento seguro.
- MySQL no se publica mediante Ingress ni NodePort.
- Las pruebas PTES se limitan al dominio resuelto al laboratorio local.
- No probar API Server, kubelet, etcd ni otros servicios internos sin una autorizacion adicional expresa.

## Informe y evidencias

El informe tecnico, el PDF final y las evidencias PTES se encuentran en:

<https://github.com/gamurigm/inventrack-ptes-report>
