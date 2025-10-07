# Manifiestos — Atenex Platform (nyro-develop)

Este documento resume los manifiestos Kubernetes disponibles en este directorio `atenex-manifests`. Está escrito en español y provee un inventario por microservicio, namespaces usados, secretos y ConfigMaps esperados, puertos, y recomendaciones técnicas para despliegue.

## Visión general

- Namespace principal utilizado por los manifiestos: `nyro-develop` (asegúrate de crearlo antes de aplicar los recursos).
- Tipo de acceso a los servicios: la mayoría usan `Service` tipo `ClusterIP` (acceso interno). Cambia a `LoadBalancer` o configura un `Ingress` si se requiere acceso externo.
- Pipeline CI/CD: muchas imágenes incluidas son placeholders (ej. `ghcr.io/dev-nyro/...:develop-...`) y se espera que el pipeline reemplace tags por versiones apropiadas.

## Inventario de microservicios y recursos

Cada entrada incluye los recursos principales (Deployment/StatefulSet, Service, ConfigMap, otros). Para cada servicio se listan variables clave que deben estar presentes.

### api-gateway
- Archivos: `api-gateway/deployment.yaml`, `api-gateway/service.yaml`, `api-gateway/configmap.yaml`
- Namespace: `nyro-develop`
- Recursos creados: Deployment `api-gateway-deployment`, Service `api-gateway-service`, ConfigMap `api-gateway-config`.
- Puertos: container 8080, Service expone 80 (targetPort `http`).
- ConfigMaps/Secrets esperados:
  - ConfigMap `api-gateway-config` (incluye URLs a ingest/query/postgres y parámetros HTTP/JWT).
  - Secret `api-gateway-secrets` (referenciado en el Deployment; crear con credenciales JWT o DB si es necesario).
- Recomendaciones:
  - Aumentar replicas >=2 en producción.
  - Añadir probes (liveness/readiness) si la app lo soporta.
  - Si GHCR es privado, añadir `imagePullSecrets`.

### ingest-service
- Archivos: `ingest-service/deployment-api.yaml`, `ingest-service/deployment-worker.yaml`, `ingest-service/service-api.yaml`, `ingest-service/configmap.yaml`
- Namespace: `nyro-develop`
- Recursos creados: Deployments `ingest-api-deployment`, `ingest-worker-deployment`, Service `ingest-api-service`, ConfigMap `ingest-service-config`.
- Puertos: API container 8000 -> Service 80 (targetPort `http`). Worker no expone servicio.
- ConfigMaps/Secrets/Volumes:
  - ConfigMap `ingest-service-config` (milvus, postgres, celery, GCS bucket, embedding settings).
  - Secret `ingest-service-secrets` (referenciado; contiene credenciales como ZILLIZ_API_KEY, GCS keys si aplica).
  - Secret `gcs-worker-sa-key` montado como volumen en `/etc/gcp-secrets` (clave JSON para GCS).
- Recomendaciones:
  - Verifica la URL de Milvus/Zilliz y los timeouts.
  - Ajustar recursos del worker (actualmente altos: requests 1 CPU / 3Gi memoria).
  - Asegurar un broker Redis accesible en `redis-service-master.nyro-develop.svc.cluster.local:6379`.

### embedding-service
- Archivos: `embedding-service/deployment.yaml`, `embedding-service/service.yaml`, `embedding-service/configmap.yaml`
- Namespace: `nyro-develop`
- Recursos creados: Deployment `embedding-service-deployment`, Service `embedding-service`, ConfigMap `embedding-service-config`.
- Puertos: container 8003 -> Service 80 (targetPort `http`).
- Secrets:
  - Secret `embedding-service-secrets` (contiene `EMBEDDING_OPENAI_API_KEY`).
- Recomendaciones:
  - Usa readinessProbe para evitar tráfico a instancias no listas (ya incluida).
  - Controla retries y timeouts hacia OpenAI en ConfigMap.

### docproc-service
- Archivos: `docproc-service/deployment.yaml`, `docproc-service/service.yaml`, `docproc-service/configmap.yaml`
- Namespace: `nyro-develop`
- Recursos creados: Deployment `docproc-service-deployment`, Service `docproc-service`, ConfigMap `docproc-service-config`.
- Puertos: container 8005 -> Service 80.
- Consideraciones:
  - Diseñado para procesar documentos grandes (memoria y timeout configurados)
  - Probes liveness/readiness configuradas.

### query-service
- Archivos: `query-service/deployment.yaml`, `query-service/service.yaml`, `query-service/configmap.yaml`
- Namespace: `nyro-develop`
- Recursos creados: Deployment `query-service-deployment`, Service `query-service`, ConfigMap `query-service-config`.
- Puertos: container 8001 -> Service 80.
- ConfigMaps/Secrets:
  - ConfigMap `query-service-config` (parámetros para Postgres, Milvus/Zilliz, Embedding, Sparse Search, Reranker, LLMs, RAG settings).
  - Secret `query-service-secrets` (Zilliz/Gemini/DB passwords esperados).
- Recomendaciones:
  - Ajustar `QUERY_RETRIEVER_TOP_K` y límites de tokens según uso.
  - Configurar correctamente endpoints para reranker/sparse-search y sus secrets.

### reranker-service
- Archivos: `reranker-service/deployment.yaml`, `reranker-service/service.yaml`, `reranker-service/configmap.yaml`
- Namespace: `nyro-develop`
- Recursos creados: Deployment `reranker-service-deployment` (nota: replicas: 0 en manifiesto), Service `reranker-service`, ConfigMap `reranker-service-config`.
- Puertos: container 8004 -> Service 80.
- Consideraciones:
  - Replica 0 sugiere que no debe iniciarse por defecto (activar en despliegue si se necesita).
  - Usa cache HF (`/app/.cache/huggingface`) con emptyDir por defecto; considerar PVC para modelos grandes.

### reranker_gpu (endpoints)
- Archivo: `reranker_gpu-service/reranker-gpu.yaml`
- Namespace: `nyro-develop`
- Observaciones:
  - Define un Service `reranker-gpu` y Endpoints apuntando a una IP local (por ejemplo Docker Desktop gateway). Útil para enrutar hacia un servicio externo (GPU host) o un proceso fuera del cluster.
  - Ajustar la IP en `Endpoints.subsets.addresses.ip` a la correcta en producción.

### sparse-search-service
- Archivos: `sparse-search-service/configmap.yaml`, `sparse-search-service/deployment.yaml`, `sparse-search-service/service.yaml`, `sparse-search-service/cronjob.yaml`, `sparse-search-service/serviceaccount.yaml`
- Namespace: `nyro-develop`
- Recursos creados: Deployment `sparse-search-service`, Service `sparse-search-service`, CronJob `sparse-search-index-builder`, ServiceAccount `sparse-search-builder-sa`, ConfigMap `sparse-search-service-config`.
- Funcionalidad: servicio BM25 (sparse retrieval) y job periódico que reconstruye índices en GCS.
- Secrets/Volumes:
  - Secret `sparse-search-gcs-key` (montado para acceso a GCS) y `sparse-search-service-secrets` para la contraseña de Postgres.
  - CronJob usa `serviceAccountName: sparse-search-builder-sa` — crear RBAC si es necesario.
- Recomendaciones:
  - Verifica `SPARSE_INDEX_GCS_BUCKET_NAME` y permisos de la service account.
  - Ajustar schedule del CronJob para producción/test.

### postgresql
- Archivos: `postgresql/statefulset.yaml`, `postgresql/service.yaml`, `postgresql/persistent-volume-claim.yaml`
- Namespace: `nyro-develop`
- Recursos creados: StatefulSet `postgresql`, Service `postgresql-service`, PersistentVolumeClaim `postgresql-pvc`.
- Secrets esperados:
  - Secret `postgresql-secrets` con claves `POSTGRES_USER` y `POSTGRES_PASSWORD`.
- PVC:
  - `postgresql-pvc` solicita 5Gi, `ReadWriteOnce`.
- Recomendaciones:
  - Para producción, aumentar replicas y usar un operador/ha solution (Patroni, Crunchy, Cloud SQL, etc.).
  - Asegurar backups y políticas de retención.

## Recursos globales y dependencias externas

- Redis: `redis-service-master.nyro-develop.svc.cluster.local:6379` (usado por Celery en ingest).
- GCS: se usan secretos/keys montadas en múltiples servicios (`gcs-worker-sa-key`, `sparse-search-gcs-key`). Asegura que las claves estén protegidas y que el bucket exista.
- Milvus / Zilliz Cloud: URLs apuntan a instancias serverless de Zilliz (ajustar si usas instancia propia).
- Secrets esperados (resumen):
  - `api-gateway-secrets`
  - `ingest-service-secrets`
  - `gcs-worker-sa-key` (Secret con `key.json`)
  - `embedding-service-secrets` (EMBEDDING_OPENAI_API_KEY)
  - `query-service-secrets`
  - `reranker-service`/`huggingface-secrets` (opcional si modelo privado)
  - `sparse-search-service-secrets` (SPARSE_POSTGRES_PASSWORD)
  - `postgresql-secrets` (POSTGRES_USER, POSTGRES_PASSWORD)

## Namespaces y RBAC

- Todos los manifiestos apuntan al namespace `nyro-develop`. Crear el namespace antes de aplicar:

  kubectl create namespace nyro-develop

- El CronJob `sparse-search-index-builder` usa `serviceAccountName: sparse-search-builder-sa`; considera crear roles/rolebindings necesarios para acceso a Secrets/PVC/GCS si el clúster lo requiere.

## Buenas prácticas y notas de despliegue

- Validación previa: usar `kubectl apply --dry-run=client -f <file>` o `kustomize` antes de aplicar en staging.
- Secrets: nunca almacenar claves en ConfigMaps; usa Secrets y, si es posible, soluciones como Vault o KMS.
- Imágenes: actualizar tags desde el pipeline CI/CD. Considera usar `imagePullPolicy: IfNotPresent` en entornos controlados para ahorrar ancho de banda.
- Probes: agregar liveness/readiness donde falten (ej. api-gateway) para mejorar la resiliencia.
- Autoescalado: valorar HPA para servicios con carga variable (ingest-worker, query-service).
- Persistencia: para modelos grandes o caches, usar PVCs en vez de `emptyDir` si quieres persistencia entre reinicios.

## Cómo aplicar los manifiestos (ejemplo mínimo)

1) Crear namespace:

```powershell
kubectl create namespace nyro-develop
```

2) Crear Secrets y ConfigMaps necesarios (ejemplos simplificados):

```powershell
# Crear secret para Postgres
kubectl create secret generic postgresql-secrets --namespace nyro-develop --from-literal=POSTGRES_USER=postgres --from-literal=POSTGRES_PASSWORD="changeme"

# Crear secret con GCS key (archivo key.json en C:\keys\key.json)
kubectl create secret generic gcs-worker-sa-key --namespace nyro-develop --from-file=key.json=C:\keys\key.json
```

3) Aplicar recursos en orden recomendado (Postgres -> infra -> servicios que dependen de ellos):

```powershell
kubectl apply -f postgresql/ -n nyro-develop
# Esperar que postgres esté listo
kubectl rollout status statefulset/postgresql -n nyro-develop

kubectl apply -f sparse-search-service/ -n nyro-develop
kubectl apply -f ingest-service/ -n nyro-develop
kubectl apply -f embedding-service/ -n nyro-develop
kubectl apply -f docproc-service/ -n nyro-develop
kubectl apply -f query-service/ -n nyro-develop
kubectl apply -f reranker-service/ -n nyro-develop
kubectl apply -f api-gateway/ -n nyro-develop
```

4) Verificar servicios y endpoints:

```powershell
kubectl get pods,svc,deploy,sts -n nyro-develop