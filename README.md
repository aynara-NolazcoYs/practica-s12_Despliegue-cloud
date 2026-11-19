# Practica S12: Kubernetes, AWS y seguridad

Laboratorio para desplegar una aplicacion nginx sobre Kubernetes, aplicar un contexto de seguridad non-root y exponerla mediante un AWS Load Balancer.

## Requisitos

- Cuenta de AWS con permisos para administrar EKS o un clúster Kubernetes administrado.
- `kubectl` configurado para el clúster.
- AWS Load Balancer Controller o integración de AWS para crear un servicio `LoadBalancer`.
- GitHub para publicar el repositorio.

## Arquitectura

1. Namespace aislado: `practica-s12`.
2. Deployment con dos réplicas de nginx en el puerto 8080.
3. SecurityContext non-root con UID/GID 101 y perfil seccomp `RuntimeDefault`.
4. Secret Kubernetes Base64 para datos de base de datos.
5. ConfigMap para configuracion de nginx y pagina HTML.
6. Sondas HTTP `/health`.
7. Servicio `LoadBalancer` en el puerto 80.

## Despliegue

```bash
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/01-security-configmap.yaml
kubectl apply -f k8s/02-db-secret.yaml
kubectl apply -f k8s/03-app-deployment.yaml
kubectl apply -f k8s/04-app-service.yaml
```

Verificar recursos:

```bash
kubectl get all -n practica-s12
kubectl get pods -n practica-s12 -w
kubectl get svc app-service -n practica-s12
```

Esperar un `EXTERNAL-IP` o hostname y probar:

```bash
EXTERNAL_IP=$(kubectl get svc app-service -n practica-s12 \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl -i "http://${EXTERNAL_IP}/"
curl -i "http://${EXTERNAL_IP}/health"
```

Si AWS entrega una IP:

```bash
EXTERNAL_IP=$(kubectl get svc app-service -n practica-s12 \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -i "http://${EXTERNAL_IP}/"
```

## Seguridad

El contenedor se ejecuta con:

- `runAsNonRoot: true`
- UID/GID `101`
- `allowPrivilegeEscalation: false`
- sistema de archivos raíz de solo lectura
- todas las capacidades Linux eliminadas
- perfil seccomp `RuntimeDefault`
- volúmenes temporales para `/var/cache/nginx` y `/var/run`

Validar:

```bash
POD=$(kubectl get pod -n practica-s12 -l app=app-practica \
  -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it "$POD" -n practica-s12 -- id
kubectl exec -it "$POD" -n practica-s12 -- whoami
```

## Diagnostico

```bash
kubectl get pods -n practica-s12
kubectl logs <pod-app> -n practica-s12
kubectl describe pod <pod-app> -n practica-s12
kubectl get events -n practica-s12 --sort-by=.lastTimestamp
```

La bitácora completa está en [docs/troubleshooting-log.md](docs/troubleshooting-log.md).

## Evidencias para el informe

1. `kubectl get pods -n practica-s12`: antes y despues.
2. `kubectl exec ... -- id` y `whoami`: usuario non-root.
3. `kubectl logs ...` y `kubectl describe pod ...`: diagnostico.
4. `curl -i http://<EXTERNAL-IP>/`: respuesta HTTP exitosa.
