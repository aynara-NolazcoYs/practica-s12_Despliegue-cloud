# Bitacora de troubleshooting

## Resumen de fallas corregidas

| # | Falla inicial | Diagnostico | Correccion |
|---|---|---|---|
| 1 | El contenedor intentaba iniciarse como `root` o con capacidades adicionales | Las reglas del Pod pedian `runAsNonRoot` pero el UID no estaba definido de forma compatible | Se fijo `runAsNonRoot: true`, `runAsUser: 101`, `runAsGroup: 101`, `seccompProfile: RuntimeDefault` y se eliminaron todas las capacidades |
| 2 | Las sondas apuntaban a un puerto o endpoint inexistente | El contenedor expone el puerto 8080 y `/health`, pero la configuracion defectuosa usaba otro puerto/ruta | Liveness y Readiness usan HTTP GET a `/health` en el puerto 8080 |
| 3 | Error de red o de Secret al cargar datos de base de datos | Las claves tenian nombres inconsistentes o el recurso faltaba en `practica-s12` | Se crearon `DB_HOST`, `DB_NAME`, `DB_USER` y `DB_PASSWORD` en el Secret y se inyectaron con `envFrom.secretRef` |

## Procedimiento recomendado

### 1. Namespace y configuracion

```bash
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/01-security-configmap.yaml
kubectl apply -f k8s/02-db-secret.yaml
kubectl apply -f k8s/03-app-deployment.yaml
kubectl apply -f k8s/04-app-service.yaml
```

### 2. Evidencia antes de corregir

En esta evidencia se debe capturar el estado inicial del laboratorio. Ejemplos esperados:

```bash
kubectl get pods -n practica-s12
# Ejemplos posibles antes de la correccion:
# app-practica-...   CrashLoopBackOff
# app-practica-...   Error
# app-practica-...   Running pero con Readiness probe failed
```

```bash
kubectl describe pod <pod-app> -n practica-s12
kubectl logs <pod-app> -n practica-s12
```

Registrar aqui:

- Estado y nombre del Pod defectuoso.
- Mensaje de `kubectl describe pod`.
- Mensaje de `kubectl logs`.
- Puerto y ruta que produjo el fallo.

### 3. Evidencia despues de corregir

```bash
kubectl get pods -n practica-s12
# NAME                             READY   STATUS    RESTARTS
# app-practica-<id>                1/1     Running   0
```

### 4. Validacion del usuario

```bash
kubectl exec -it <pod-app> -n practica-s12 -- id
kubectl exec -it <pod-app> -n practica-s12 -- whoami
```

La salida debe mostrar `uid=101` y un nombre diferente de `root`. Puede mostrar `uid=101 gid=101`.

### 5. Validacion de sondas y servicio

```bash
kubectl get deploy,pods,svc -n practica-s12
kubectl get endpoints -n practica-s12 app-service
kubectl describe pod <pod-app> -n practica-s12
```

Cuando AWS asigna el `EXTERNAL-IP`:

```bash
EXTERNAL_IP=$(kubectl get svc app-service -n practica-s12 \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl -i "http://${EXTERNAL_IP}/"
curl -i "http://${EXTERNAL_IP}/health"
```

En clusters AWS que usan una IP en lugar de hostname, sustituir `hostname` por `ip`.
