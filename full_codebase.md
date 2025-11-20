# Estructura de la Codebase

```
app/
├── README.md
├── api-gateway
│   ├── configmap.yaml
│   ├── deployment.yaml
│   └── service.yaml
├── docproc-service
│   ├── configmap.yaml
│   ├── deployment.yaml
│   └── service.yaml
├── embedding-service
│   ├── configmap.yaml
│   ├── deployment.yaml
│   └── service.yaml
├── export_codebase.py
├── ingest-service
│   ├── configmap.yaml
│   ├── deployment-api.yaml
│   ├── deployment-worker.yaml
│   └── service-api.yaml
├── postgresql
│   ├── persistent-volume-claim.yaml
│   ├── service.yaml
│   └── statefulset.yaml
├── query-service
│   ├── configmap.yaml
│   ├── deployment.yaml
│   └── service.yaml
├── reranker-service
│   ├── configmap.yaml
│   ├── deployment.yaml
│   └── service.yaml
├── reranker_gpu-service
│   └── reranker-gpu.yaml
└── sparse-search-service
    ├── configmap.yaml
    ├── cronjob.yaml
    ├── deployment.yaml
    ├── service.yaml
    └── serviceaccount.yaml
```

# Codebase: `app`

## File: `README.md`
```md
# Manifiestos — Atenex Platform (atenex)

Este documento resume los manifiestos Kubernetes disponibles en este directorio `atenex-manifests`. Está escrito en español y provee un inventario por microservicio, namespaces usados, secretos y ConfigMaps esperados, puertos, y recomendaciones técnicas para despliegue.

## Visión general

- Namespace principal utilizado por los manifiestos: `atenex` (asegúrate de crearlo antes de aplicar los recursos).
- Tipo de acceso a los servicios: la mayoría usan `Service` tipo `ClusterIP` (acceso interno). Cambia a `LoadBalancer` o configura un `Ingress` si se requiere acceso externo.
- Pipeline CI/CD: muchas imágenes incluidas son placeholders (ej. `ghcr.io/dev-nyro/...:develop-...`) y se espera que el pipeline reemplace tags por versiones apropiadas.

## Inventario de microservicios y recursos

Cada entrada incluye los recursos principales (Deployment/StatefulSet, Service, ConfigMap, otros). Para cada servicio se listan variables clave que deben estar presentes.

### api-gateway
- Archivos: `api-gateway/deployment.yaml`, `api-gateway/service.yaml`, `api-gateway/configmap.yaml`
- Namespace: `atenex`
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
- Namespace: `atenex`
- Recursos creados: Deployments `ingest-api-deployment`, `ingest-worker-deployment`, Service `ingest-api-service`, ConfigMap `ingest-service-config`.
- Puertos: API container 8000 -> Service 80 (targetPort `http`). Worker no expone servicio.
- ConfigMaps/Secrets/Volumes:
  - ConfigMap `ingest-service-config` (milvus, postgres, celery, GCS bucket, embedding settings).
  - Secret `ingest-service-secrets` (referenciado; contiene credenciales como ZILLIZ_API_KEY, GCS keys si aplica).
  - Secret `gcs-worker-sa-key` montado como volumen en `/etc/gcp-secrets` (clave JSON para GCS).
- Recomendaciones:
  - Verifica la URL de Milvus/Zilliz y los timeouts.
  - Ajustar recursos del worker (actualmente altos: requests 1 CPU / 3Gi memoria).
  - Asegurar un broker Redis accesible en `redis-service-master.atenex.svc.cluster.local:6379`.

### embedding-service
- Archivos: `embedding-service/deployment.yaml`, `embedding-service/service.yaml`, `embedding-service/configmap.yaml`
- Namespace: `atenex`
- Recursos creados: Deployment `embedding-service-deployment`, Service `embedding-service`, ConfigMap `embedding-service-config`.
- Puertos: container 8003 -> Service 80 (targetPort `http`).
- Secrets:
  - Secret `embedding-service-secrets` (contiene `EMBEDDING_OPENAI_API_KEY`).
- Recomendaciones:
  - Usa readinessProbe para evitar tráfico a instancias no listas (ya incluida).
  - Controla retries y timeouts hacia OpenAI en ConfigMap.

### docproc-service
- Archivos: `docproc-service/deployment.yaml`, `docproc-service/service.yaml`, `docproc-service/configmap.yaml`
- Namespace: `atenex`
- Recursos creados: Deployment `docproc-service-deployment`, Service `docproc-service`, ConfigMap `docproc-service-config`.
- Puertos: container 8005 -> Service 80.
- Consideraciones:
  - Diseñado para procesar documentos grandes (memoria y timeout configurados)
  - Probes liveness/readiness configuradas.

### query-service
- Archivos: `query-service/deployment.yaml`, `query-service/service.yaml`, `query-service/configmap.yaml`
- Namespace: `atenex`
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
- Namespace: `atenex`
- Recursos creados: Deployment `reranker-service-deployment` (nota: replicas: 0 en manifiesto), Service `reranker-service`, ConfigMap `reranker-service-config`.
- Puertos: container 8004 -> Service 80.
- Consideraciones:
  - Replica 0 sugiere que no debe iniciarse por defecto (activar en despliegue si se necesita).
  - Usa cache HF (`/app/.cache/huggingface`) con emptyDir por defecto; considerar PVC para modelos grandes.

### reranker_gpu (endpoints)
- Archivo: `reranker_gpu-service/reranker-gpu.yaml`
- Namespace: `atenex`
- Observaciones:
  - Define un Service `reranker-gpu` y Endpoints apuntando a una IP local (por ejemplo Docker Desktop gateway). Útil para enrutar hacia un servicio externo (GPU host) o un proceso fuera del cluster.
  - Ajustar la IP en `Endpoints.subsets.addresses.ip` a la correcta en producción.

### sparse-search-service
- Archivos: `sparse-search-service/configmap.yaml`, `sparse-search-service/deployment.yaml`, `sparse-search-service/service.yaml`, `sparse-search-service/cronjob.yaml`, `sparse-search-service/serviceaccount.yaml`
- Namespace: `atenex`
- Recursos creados: Deployment `sparse-search-service`, Service `sparse-search-service`, CronJob `sparse-search-index-builder`, ServiceAccount `sparse-search-builder-sa`, ConfigMap `sparse-search-service-config`.
- Funcionalidad: servicio BM25 (sparse retrieval) y job periódico que reconstruye índices en GCS.
- Secrets/Volumes:
  - Secret `sparse-search-service-secrets` para credenciales de Postgres y MinIO (`SPARSE_POSTGRES_PASSWORD`, `SPARSE_MINIO_ACCESS_KEY`, `SPARSE_MINIO_SECRET_KEY`).
  - CronJob usa `serviceAccountName: sparse-search-builder-sa` — crear RBAC si es necesario.
- Recomendaciones:
  - Verifica los valores de MinIO (`SPARSE_STORAGE_BACKEND`, endpoint, bucket y credenciales) antes de desplegar.
  - Ajustar schedule del CronJob para producción/test.

### postgresql
- Archivos: `postgresql/statefulset.yaml`, `postgresql/service.yaml`, `postgresql/persistent-volume-claim.yaml`
- Namespace: `atenex`
- Recursos creados: StatefulSet `postgresql`, Service `postgresql-service`, PersistentVolumeClaim `postgresql-pvc`.
- Secrets esperados:
  - Secret `postgresql-secrets` con claves `POSTGRES_USER` y `POSTGRES_PASSWORD`.
- PVC:
  - `postgresql-pvc` solicita 5Gi, `ReadWriteOnce`.
- Recomendaciones:
  - Para producción, aumentar replicas y usar un operador/ha solution (Patroni, Crunchy, Cloud SQL, etc.).
  - Asegurar backups y políticas de retención.

## Recursos globales y dependencias externas

- Redis: `redis-service-master.atenex.svc.cluster.local:6379` (usado por Celery en ingest).
- MinIO: varias apps consumen `minio.minio.svc.cluster.local:9000`; mantén sincronizadas las credenciales en los Secrets correspondientes.
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

- Todos los manifiestos apuntan al namespace `atenex`. Crear el namespace antes de aplicar:

  kubectl create namespace atenex

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
kubectl create namespace atenex
```

2) Crear Secrets y ConfigMaps necesarios (ejemplos simplificados):

```powershell
# Crear secret para Postgres
kubectl create secret generic postgresql-secrets --namespace atenex --from-literal=POSTGRES_USER=postgres --from-literal=POSTGRES_PASSWORD="changeme"

# Crear secret con GCS key (archivo key.json en C:\keys\key.json)
kubectl create secret generic gcs-worker-sa-key --namespace atenex --from-file=key.json=C:\keys\key.json
```

3) Aplicar recursos en orden recomendado (Postgres -> infra -> servicios que dependen de ellos):

```powershell
kubectl apply -f postgresql/ -n atenex
# Esperar que postgres esté listo
kubectl rollout status statefulset/postgresql -n atenex

kubectl apply -f sparse-search-service/ -n atenex
kubectl apply -f ingest-service/ -n atenex
kubectl apply -f embedding-service/ -n atenex
kubectl apply -f docproc-service/ -n atenex
kubectl apply -f query-service/ -n atenex
kubectl apply -f reranker-service/ -n atenex
kubectl apply -f api-gateway/ -n atenex
```

4) Verificar servicios y endpoints:

```powershell
kubectl get pods,svc,deploy,sts -n atenex
```

## File: `api-gateway/configmap.yaml`
```yaml
# api-gateway/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-gateway-config
  namespace: atenex
data:
  GATEWAY_LOG_LEVEL: "INFO"
  GATEWAY_INGEST_SERVICE_URL: "http://ingest-api-service.atenex.svc.cluster.local:80"
  GATEWAY_QUERY_SERVICE_URL: "http://query-service.atenex.svc.cluster.local:80"
  GATEWAY_POSTGRES_SERVER: "postgresql-service.atenex.svc.cluster.local"
  GATEWAY_POSTGRES_PORT: "5432"  
  GATEWAY_POSTGRES_USER: "postgres"
  GATEWAY_POSTGRES_DB: "atenex"
  GATEWAY_DEFAULT_COMPANY_ID: "550e8400-e29b-41d4-a716-446655440000"
  GATEWAY_HTTP_CLIENT_TIMEOUT: "30"
  GATEWAY_HTTP_CLIENT_MAX_KEEPALIVE_CONNECTIONS: "100" # Corregido
  GATEWAY_HTTP_CLIENT_MAX_CONNECTIONS: "200"
  GATEWAY_JWT_ALGORITHM: "HS256"
```

## File: `api-gateway/deployment.yaml`
```yaml
# api-gateway/deployment.yaml
# --- Deployment para API Gateway (SIN Liveness/Readiness Probes) ---
apiVersion: apps/v1
kind: Deployment
metadata:
  # Nombre del Deployment
  name: api-gateway-deployment
  # Namespace donde se desplegará (¡ASEGÚRATE QUE EXISTA!)

  namespace: atenex
  labels:
    app: api-gateway
spec:
  # Número de réplicas (considera 2+ para alta disponibilidad)
  replicas: 1
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
        - name: api-gateway
          # La imagen será actualizada por el pipeline CI/CD

          # Ejemplo de imagen inicial, el pipeline la reemplazará
          image: ghcr.io/marcksdbgg/api-gateway:main-fa3bb46
          # Política de pull de imagen (Always para asegurar la última, IfNotPresent si prefieres caché)

          imagePullPolicy: Always
          ports:
            # Nombre del puerto para referencia interna (ej. en Service)
            - name: http
              # Puerto que expone tu aplicación DENTRO del contenedor (Dockerfile/Gunicorn)

              containerPort: 8080
              protocol: TCP
          # No se necesita command/args si el CMD/ENTRYPOINT del Dockerfile es correcto y usa el puerto 8080

          # env: # La variable PORT ya debería estar configurada en tu app o Dockerfile si es necesaria

          #   - name: PORT

          #     value: "8080"

          # Carga variables de entorno desde ConfigMaps y Secrets
          envFrom:
            - configMapRef:
                # ¡IMPORTANTE! Este ConfigMap DEBE existir en el namespace 'atenex'
                name: api-gateway-config
            - secretRef:
                # ¡IMPORTANTE! Este Secret DEBE existir en el namespace 'atenex'
                name: api-gateway-secrets
          # Límites y solicitudes de recursos (ajustar según monitorización)

          resources:
            requests: # Mínimo garantizado
              cpu: "100m" # 10% de un vCPU
              memory: "128Mi" # 128 Mebibytes
            limits: # Máximo permitido
              cpu: "500m" # 50% de un vCPU
              memory: "512Mi" # 512 Mebibytes

```

## File: `api-gateway/service.yaml`
```yaml
# api-gateway/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-gateway-service
  namespace: atenex
  labels:
    app: api-gateway
spec:
  type: ClusterIP # O LoadBalancer/NodePort si necesitas acceso externo directo
  selector:
    app: api-gateway # Selecciona los Pods del Deployment del Gateway
  ports:
    - name: http
      protocol: TCP
      port: 80 # Puerto por el que otros servicios (o Ingress) accederán al Gateway
      targetPort: http # Nombre del puerto en el Pod (definido en Deployment, que es 8080)
```

## File: `docproc-service/configmap.yaml`
```yaml
# docproc-service/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: docproc-service-config
  namespace: atenex
data:
  DOCPROC_LOG_LEVEL: "INFO"
  DOCPROC_PORT: "8005" # Puerto interno del contenedor
  DOCPROC_CHUNK_SIZE: "1000"
  DOCPROC_CHUNK_OVERLAP: "200"
  DOCPROC_SUPPORTED_CONTENT_TYPES: '["application/pdf","application/vnd.openxmlformats-officedocument.wordprocessingml.document","application/msword","text/plain","text/markdown","text/html","application/vnd.openxmlformats-officedocument.spreadsheetml.sheet","application/vnd.ms-excel"]'
  # DOCPROC_GUNICORN_WORKERS: "4" # Gunicorn workers se configuran en el CMD del Dockerfile, pero podría ser una variable aquí si se desea
```

## File: `docproc-service/deployment.yaml`
```yaml
# docproc-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: docproc-service-deployment
  namespace: atenex
  labels:
    app: docproc-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: docproc-service
  template:
    metadata:
      labels:
        app: docproc-service
    spec:
      containers:
        - name: docproc-service
          image: ghcr.io/dev-nyro/docproc-service:develop-f6fa2b8
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8005 # Debe coincidir con DOCPROC_PORT y el CMD del Dockerfile
              protocol: TCP
          # El CMD ya está en el Dockerfile, no es necesario aquí a menos que se quiera sobrescribir.
          # command: ["gunicorn"]
          # args: [
          #     "-k", "uvicorn.workers.UvicornWorker",
          #     "-w", "4", # Usar DOCPROC_GUNICORN_WORKERS si se define en ConfigMap
          #     "-b", "0.0.0.0:8005",
          #     "-t", "300", # Timeout más largo para procesamiento de archivos grandes
          #     "--log-level", "info",
          #     "app.main:app"
          #   ]
          envFrom:
            - configMapRef:
                name: docproc-service-config
          # No se necesitan secretos específicos para este servicio por ahora
          resources:
            requests:
              cpu: "500m" # PyMuPDF puede ser intensivo en CPU
              memory: "1Gi" # Y también en memoria para archivos grandes
            limits:
              cpu: "2000m"
              memory: "4Gi"
          readinessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 15
            periodSeconds: 20
            timeoutSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 30
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 3
      # imagePullSecrets: # Descomentar si usas registro privado
      # - name: your-registry-secret

```

## File: `docproc-service/service.yaml`
```yaml
# docproc-service/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: docproc-service
  namespace: atenex
  labels:
    app: docproc-service
spec:
  type: ClusterIP
  selector:
    app: docproc-service
  ports:
    - name: http
      protocol: TCP
      port: 80 # Puerto que otros servicios usarán para llamar a docproc-service
      targetPort: http # Nombre del puerto en el Deployment (que es 8005)
```

## File: `embedding-service/configmap.yaml`
```yaml
# embedding-service/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: embedding-service-config
  namespace: atenex
data:
  EMBEDDING_LOG_LEVEL: "INFO"
  EMBEDDING_PORT: "8003"

  EMBEDDING_ACTIVE_EMBEDDING_PROVIDER: "openai"

  # OpenAI Embedding Model Configuration
  EMBEDDING_OPENAI_EMBEDDING_MODEL_NAME: "text-embedding-3-small"
  EMBEDDING_EMBEDDING_DIMENSION: "1536"
  
  EMBEDDING_OPENAI_TIMEOUT_SECONDS: "30"
  EMBEDDING_OPENAI_MAX_RETRIES: "3"
```

## File: `embedding-service/deployment.yaml`
```yaml
# embedding-service/k8s/embedding-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: embedding-service-deployment
  namespace: atenex
  labels:
    app: embedding-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: embedding-service
  template:
    metadata:
      labels:
        app: embedding-service
    spec:
      # volumes: # ELIMINADO: dshm ya no es necesario para OpenAI
      #   - name: dshm
      #     emptyDir:
      #       medium: Memory
      #       sizeLimit: 256Mi
      containers:
        - name: embedding-service
          image: ghcr.io/dev-nyro/embedding-service:develop-3792e69
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8003 # Debe coincidir con EMBEDDING_PORT del ConfigMap
              protocol: TCP
          envFrom:
            - configMapRef:
                name: embedding-service-config
          env:
            - name: EMBEDDING_OPENAI_API_KEY
              valueFrom:
                secretKeyRef:
                  name: embedding-service-secrets
                  key: EMBEDDING_OPENAI_API_KEY
          # volumeMounts: # ELIMINADO: dshm ya no es necesario
          #   - name: dshm
          #     mountPath: /dev/shm
          resources:
            requests:
              cpu: "250m" # Reducido, ya que es I/O bound a OpenAI
              memory: "512Mi" # Reducido, no hay modelo en memoria
            limits:
              cpu: "1000m"
              memory: "1Gi"
          readinessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 15 # Puede ser menor sin descarga de modelo local
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
            # livenessProbe: # ELIMINADO según solicitud
            #   httpGet:
            #     path: /health
            #     port: http
            #   initialDelaySeconds: 60
            #   periodSeconds: 15
            #   timeoutSeconds: 5
            #   failureThreshold: 3

```

## File: `embedding-service/service.yaml`
```yaml
# embedding-service/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: embedding-service # DNS: embedding-service.atenex.svc.cluster.local
  namespace: atenex
  labels:
    app: embedding-service
spec:
  type: ClusterIP # Default, good for internal services
  selector:
    app: embedding-service # Must match labels in the Deployment's template
  ports:
    - name: http
      protocol: TCP
      port: 80 # Port the K8s Service will listen on
      targetPort: http # Target port name on the Pod (which is 8003)
```


## File: `ingest-service/configmap.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingest-service-config
  namespace: atenex
data:
  # General
  INGEST_LOG_LEVEL: "INFO"

  # Celery
  INGEST_CELERY_BROKER_URL: "redis://redis-service-master.atenex.svc.cluster.local:6379/0"
  INGEST_CELERY_RESULT_BACKEND: "redis://redis-service-master.atenex.svc.cluster.local:6379/1"

  # PostgreSQL
  INGEST_POSTGRES_SERVER: "postgresql-service.atenex.svc.cluster.local"
  INGEST_POSTGRES_PORT: "5432"
  INGEST_POSTGRES_USER: "postgres"
  INGEST_POSTGRES_DB: "atenex"
  # INGEST_POSTGRES_PASSWORD is in a Secret

  # Milvus/Zilliz
  INGEST_MILVUS_URI: "https://in03-0afab716eb46d7f.serverless.gcp-us-west1.cloud.zilliz.com"
  INGEST_MILVUS_COLLECTION_NAME: "atenex_collection"
  INGEST_MILVUS_GRPC_TIMEOUT: "20"
  
  # MinIO S3-Compatible Storage
  INGEST_MINIO_ENDPOINT: "minio.minio.svc.cluster.local:9000"
  INGEST_MINIO_BUCKET_NAME: "atenex"
  INGEST_MINIO_SECURE: "false" # Use 'true' if you configure TLS on MinIO
  INGEST_MINIO_ACCESS_KEY: "admin123"
  INGEST_MINIO_SECRET_KEY: "admin123"

  # Embedding Settings
  INGEST_EMBEDDING_DIMENSION: "1536" 

  # Tokenizer Settings
  INGEST_TIKTOKEN_ENCODING_NAME: "cl100k_base"

  # URLs for dependent services
  INGEST_EMBEDDING_SERVICE_URL: "http://embedding-service.atenex.svc.cluster.local:80/api/v1/embed"
  INGEST_DOCPROC_SERVICE_URL: "http://docproc-service.atenex.svc.cluster.local:80/api/v1/process"
```

## File: `ingest-service/deployment-api.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingest-api-deployment
  namespace: atenex
  labels:
    app: ingest-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ingest-api
  template:
    metadata:
      labels:
        app: ingest-api
    spec:
      containers:
        - name: ingest-api
          image: ghcr.io/marcksdbgg/ingest-service:main-932a4ef # Placeholder, CI/CD to update
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8000
              protocol: TCP
          command: ["gunicorn"]
          args: ["-k", "uvicorn.workers.UvicornWorker", "-w", "4", "-b", "0.0.0.0:8000", "-t", "120", "--log-level", "info", "app.main:app"]
          envFrom:
            - configMapRef:
                name: ingest-service-config
            - secretRef:
                name: ingest-service-secrets # Reference to the secret
          env:
            - name: INGEST_ZILLIZ_API_KEY # MODIFIED: Added Zilliz API Key from Secret
              valueFrom:
                secretKeyRef:
                  name: ingest-service-secrets # Name of the k8s Secret
                  key: ZILLIZ_API_KEY # Key within the Secret
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1500m"
              memory: "2Gi"

```

## File: `ingest-service/deployment-worker.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingest-worker-deployment
  namespace: atenex
  labels:
    app: ingest-worker
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ingest-worker
  template:
    metadata:
      labels:
        app: ingest-worker
    spec:
      containers:
        - name: ingest-worker
          image: ghcr.io/marcksdbgg/ingest-service:main-932a4ef
          imagePullPolicy: Always
          command: ["celery"]
          args: ["-A", "app.tasks.celery_app", "worker", "--loglevel=INFO", "-P", "prefork", "-c", "4", # Number of worker processes
          ]
          envFrom:
            - configMapRef:
                name: ingest-service-config
            - secretRef:
                name: ingest-service-secrets # Reference to the secret
          env:
            - name: INGEST_ZILLIZ_API_KEY # MODIFIED: Added Zilliz API Key from Secret
              valueFrom:
                secretKeyRef:
                  name: ingest-service-secrets # Name of the k8s Secret
                  key: ZILLIZ_API_KEY # Key within the Secret
          resources:
            requests:
              cpu: "1000m" # 1 vCPU
              memory: "3Gi"
            limits:
              cpu: "4000m" # 4 vCPU
              memory: "8Gi"

```

## File: `ingest-service/service-api.yaml`
```yaml
# ingest-api/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: ingest-api-service
  namespace: atenex
  labels:
    app: ingest-api
spec:
  type: ClusterIP # Default, suitable for internal access
  selector:
    app: ingest-api # Selects pods from the API deployment
  ports:
    - name: http
      protocol: TCP
      port: 80 # Standard port for the service
      targetPort: http # Matches the containerPort name 'http' (which is 8000)
```

## File: `postgresql/persistent-volume-claim.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgresql-pvc
  namespace: atenex
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi

```

## File: `postgresql/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgresql-service
  namespace: atenex
  labels:
    app: postgresql
spec:
  type: ClusterIP
  ports:
    - name: postgres
      port: 5432
      targetPort: postgres
      protocol: TCP
  selector:
    app: postgresql

```

## File: `postgresql/statefulset.yaml`
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
  namespace: atenex
  labels:
    app: postgresql
spec:
  serviceName: "postgresql-service"
  replicas: 1
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
        - name: postgresql
          image: postgres:16-alpine
          ports:
            - name: postgres
              containerPort: 5432
              protocol: TCP
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgresql-secrets
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgresql-secrets
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_DB
              value: "atenex"
            - name: POSTGRES_INITDB_ARGS
              value: "--encoding=UTF8 --locale=en_US.UTF-8"
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
          volumeMounts:
            - name: postgresql-data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: postgresql-data
          persistentVolumeClaim:
            claimName: postgresql-pvc

```

## File: `query-service/configmap.yaml`
```yaml
# query-service/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: query-service-config
  namespace: atenex
data:
  # General
  QUERY_LOG_LEVEL: "INFO"

  # PostgreSQL
  QUERY_POSTGRES_SERVER: "postgresql-service.atenex.svc.cluster.local"
  QUERY_POSTGRES_PORT: "5432"
  QUERY_POSTGRES_USER: "postgres"
  QUERY_POSTGRES_DB: "atenex"
  # QUERY_POSTGRES_PASSWORD -> Proveniente de Secret

  # Milvus/Zilliz
  QUERY_MILVUS_URI: "https://in03-0afab716eb46d7f.serverless.gcp-us-west1.cloud.zilliz.com"
  QUERY_MILVUS_COLLECTION_NAME: "atenex_collection" #"atenex_e5_collection"
  QUERY_MILVUS_EMBEDDING_FIELD: "embedding"
  QUERY_MILVUS_CONTENT_FIELD: "content"
  QUERY_MILVUS_COMPANY_ID_FIELD: "company_id"
  QUERY_MILVUS_DOCUMENT_ID_FIELD: "document_id"
  QUERY_MILVUS_FILENAME_FIELD: "file_name"
  QUERY_MILVUS_GRPC_TIMEOUT: "20" 
  # QUERY_MILVUS_METADATA_FIELDS: (...) # Uses default from config.py

  # Embedding Service (Remote)
  QUERY_EMBEDDING_DIMENSION: "1536" #"768" 
  QUERY_EMBEDDING_SERVICE_URL: "http://embedding-service.atenex.svc.cluster.local:80" #"http://host.docker.internal:8003"  
  QUERY_EMBEDDING_CLIENT_TIMEOUT: "30"

  # Sparse Search Service (Remote BM25)
  QUERY_BM25_ENABLED: "true" # Controla si se usa el paso de búsqueda dispersa (llamada al servicio remoto)
  QUERY_SPARSE_SEARCH_SERVICE_URL: "http://sparse-search-service.atenex.svc.cluster.local:80"
  QUERY_SPARSE_SEARCH_CLIENT_TIMEOUT: "30"

  # Reranker Service (Remote)
  QUERY_RERANKER_ENABLED: "true"
  QUERY_RERANKER_SERVICE_URL: "http://reranker-service.atenex.svc.cluster.local:80"
  QUERY_RERANKER_CLIENT_TIMEOUT: "30"

  # LLM (llama.cpp + Granite) Settings
  # Nota: este endpoint debe ser accesible desde el clúster de Kubernetes. Cambia el host/IP
  # por la dirección expuesta de tu servidor llama.cpp externo (por ejemplo, un túnel, IP LAN o DNS público).
  QUERY_LLM_API_BASE_URL: "http://192.168.1.43:9090"
  QUERY_LLM_MODEL_NAME: "granite-3.2-2b-instruct-q4_k_m.gguf"
  QUERY_LLM_MAX_OUTPUT_TOKENS: "1024"

  # (Legacy) Gemini settings can remain defined via secrets if needed
  # QUERY_GEMINI_MODEL_NAME: "gemini-2.0-flash"

  # RAG Pipeline Control & Parameters
  QUERY_RETRIEVER_TOP_K: "50" # Nº de chunks a pedir a dense y sparse retrievers inicialmente
  QUERY_MAX_CONTEXT_CHUNKS: "16" # Nº máx de chunks para el LLM después de reranking/filtering
  QUERY_MAX_PROMPT_TOKENS: "22000" 
  QUERY_MAX_TOKENS_PER_CHUNK: "900"
  QUERY_MAX_CHARS_PER_CHUNK: "4800"
  QUERY_DIVERSITY_FILTER_ENABLED: "true"
  QUERY_DIVERSITY_LAMBDA: "0.5" # Para MMR si está habilitado

  # HTTP Client Settings (Globales, usados por http_client en UseCase para Reranker y otros futuros)
  QUERY_HTTP_CLIENT_TIMEOUT: "180" 
  QUERY_HTTP_CLIENT_MAX_RETRIES: "2"
  QUERY_HTTP_CLIENT_BACKOFF_FACTOR: "1.0"

  # Chat history settings
  QUERY_MAX_CHAT_HISTORY_MESSAGES: "6"
  QUERY_NUM_SOURCES_TO_SHOW: "7"
  
  # MapReduce (Opcional, actualmente gestionado en config.py pero podría moverse aquí)
  QUERY_MAPREDUCE_ENABLED: "true"
  QUERY_MAPREDUCE_CHUNK_BATCH_SIZE: "3"
  QUERY_MAPREDUCE_ACTIVATION_THRESHOLD_CHUNKS: "16"
  QUERY_TIKTOKEN_ENCODING_NAME: "cl100k_base"
```

## File: `query-service/deployment.yaml`
```yaml
# query-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: query-service-deployment
  namespace: atenex
  labels:
    app: query-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: query-service
  template:
    metadata:
      labels:
        app: query-service
    spec:
      containers:
        - name: query-service
          image: ghcr.io/marcksdbgg/query-service:main-ad44860
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8001
              protocol: TCP
          command: ["gunicorn", "-k", "uvicorn.workers.UvicornWorker"]
          args: ["-w", "2", "-b", "0.0.0.0:8001", "-t", "120", "app.main:app"]
          envFrom:
            - configMapRef:
                name: query-service-config
            - secretRef:
                name: query-service-secrets
          env:
            - name: QUERY_ZILLIZ_API_KEY # MODIFIED: Added Zilliz API Key from Secret
              valueFrom:
                secretKeyRef:
                  name: query-service-secrets # Name of the k8s Secret
                  key: ZILLIZ_API_KEY # Key within the Secret
          resources:
            requests:
              cpu: "1500m"
              memory: "3Gi"
            limits:
              cpu: "3000m"
              memory: "8Gi"

```

## File: `query-service/service.yaml`
```yaml
# query-service/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: query-service # Nombre del Service (interno al cluster)
  namespace: atenex
  labels:
    app: query-service
spec:
  type: ClusterIP # Tipo de servicio (accesible dentro del cluster)
  selector:
    app: query-service # Selecciona los Pods con esta etiqueta (del Deployment)
  ports:
    - name: http
      protocol: TCP
      port: 80 # Puerto por el que otros servicios accederán a este (ej: API Gateway)
      targetPort: http # Nombre del puerto en el Pod (definido en el Deployment, que es 8001)
```

## File: `reranker-service/configmap.yaml`
```yaml
# reranker-service/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: reranker-service-config
  namespace: atenex
data:
  RERANKER_LOG_LEVEL: "INFO" # DEBUG, INFO, WARNING, ERROR, CRITICAL
  RERANKER_MODEL_NAME: "BAAI/bge-reranker-base" # Model from Hugging Face
  RERANKER_MODEL_DEVICE: "cpu" # Device for inference (e.g., "cpu", "cuda:0")
  RERANKER_HF_CACHE_DIR: "/app/.cache/huggingface" # Internal container path for model cache
  RERANKER_BATCH_SIZE: "128" # Batch size for reranker predictions
  RERANKER_MAX_SEQ_LENGTH: "512" # Max sequence length for the model
  # RERANKER_PORT is typically set in the Deployment's container spec or defaults in Dockerfile
  # RERANKER_WORKERS is typically set in the Deployment's container spec or defaults in Dockerfile
```

## File: `reranker-service/deployment.yaml`
```yaml
# reranker-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reranker-service-deployment
  namespace: atenex
  labels:
    app: reranker-service
spec:
  replicas: 0
  selector:
    matchLabels:
      app: reranker-service
  template:
    metadata:
      labels:
        app: reranker-service
    spec:
      containers:
        - name: reranker-service
          image: ghcr.io/dev-nyro/reranker-service:develop-74d7e4d # Placeholder: replace with actual image path and tag
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8004 # Port the application listens on (matches RERANKER_PORT default)
              protocol: TCP
          # Command and args are usually taken from the Dockerfile's CMD.
          # If you need to override or ensure specific values, uncomment and adjust:
          # command: ["gunicorn", "-k", "uvicorn.workers.UvicornWorker"]
          # args: [
          #   "app.main:app",
          #   "--bind", "0.0.0.0:$(RERANKER_PORT)", # Ensure RERANKER_PORT is available
          #   "--workers", "$(RERANKER_WORKERS)"    # Ensure RERANKER_WORKERS is available
          # ]
          envFrom:
            - configMapRef:
                name: reranker-service-config
          env:
            - name: RERANKER_PORT
              value: "8004" # Explicitly set port, aligns with containerPort
            - name: RERANKER_WORKERS
              value: "2" # Explicitly set Gunicorn workers
              # Hugging Face token might be needed for private models or to avoid rate limits
              # - name: HUGGING_FACE_HUB_TOKEN
              #   valueFrom:
              #     secretKeyRef:
              #       name: huggingface-secrets # Example, create this secret if needed
              #       key: hub_token
          resources:
            requests:
              cpu: "1000m" # 1 vCPU
              memory: "2Gi" # Initial memory request
            limits:
              cpu: "2000m" # 2 vCPU
              memory: "4Gi" # Max memory
          volumeMounts:
            - name: hf-cache-volume
              mountPath: "/app/.cache/huggingface" # Matches RERANKER_HF_CACHE_DIR
          livenessProbe:
            httpGet:
              path: /health
              port: http # Refers to the named port 'http' (8004)
            initialDelaySeconds: 60 # Time for model to load
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 65 # Slightly after liveness, ensuring model is ready
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
      volumes:
        - name: hf-cache-volume
          emptyDir: {} # Stores model cache, ephemeral with pod.
          # For persistent model cache across pod restarts (recommended for large models):
          # persistentVolumeClaim:
          #   claimName: reranker-hf-cache-pvc # Ensure this PVC is created
      # imagePullSecrets: # Uncomment if using a private container registry
      # - name: your-registry-secret

```

## File: `reranker-service/service.yaml`
```yaml
# reranker-service/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: reranker-service
  namespace: atenex
  labels:
    app: reranker-service
spec:
  type: ClusterIP # Internal service
  selector:
    app: reranker-service # Matches labels of the Pods in the Deployment
  ports:
    - name: http
      protocol: TCP
      port: 80 # Port the service will be available on within the cluster
      targetPort: http # Named port on the Pods (which is 8004)
```

## File: `reranker_gpu-service/reranker-gpu.yaml`
```yaml
# reranker-gpu.yaml
apiVersion: v1
kind: Service
metadata:
  name: reranker-gpu
  namespace: atenex
spec:
  ports:
    - name: http
      port: 8004        # puerto que usarán los pods
      protocol: TCP
---
apiVersion: v1
kind: Endpoints
metadata:
  name: reranker-gpu      # <-- mismo nombre que el Service
  namespace: atenex
subsets:
  - addresses:
      - ip: 192.168.65.1  # ← Cambiado a la puerta de enlace de Docker Desktop para WSL2
    ports:
      - port: 8004
```

## File: `sparse-search-service/configmap.yaml`
```yaml
# sparse-search-service/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sparse-search-service-config
  namespace: atenex
  labels:
    app.kubernetes.io/name: sparse-search-service
    app.kubernetes.io/part-of: atenex-platform
data:
  SPARSE_LOG_LEVEL: "INFO"
  PORT: "8004"
  SPARSE_SERVICE_VERSION: "1.0.0"

  SPARSE_POSTGRES_SERVER: "postgresql-service.atenex.svc.cluster.local"
  SPARSE_POSTGRES_PORT: "5432"
  SPARSE_POSTGRES_DB: "atenex"
  SPARSE_POSTGRES_USER: "postgres"
  SPARSE_DB_POOL_MIN_SIZE: "2"
  SPARSE_DB_POOL_MAX_SIZE: "10"
  SPARSE_DB_CONNECT_TIMEOUT: "30"
  SPARSE_DB_COMMAND_TIMEOUT: "60"

  # MinIO object storage (replaces previous GCS bucket usage)
  SPARSE_STORAGE_BACKEND: "minio"
  SPARSE_INDEX_BUCKET_NAME: "atenex"
  SPARSE_MINIO_ENDPOINT: "http://minio.minio.svc.cluster.local:9000"
  SPARSE_MINIO_REGION: "us-east-1"
  SPARSE_MINIO_SECURE: "false"

  SPARSE_INDEX_CACHE_MAX_ITEMS: "50"
  SPARSE_INDEX_CACHE_TTL_SECONDS: "3600" # 1 hora

```

## File: `sparse-search-service/cronjob.yaml`
```yaml
# sparse-search-service/cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: sparse-search-index-builder
  namespace: atenex
  labels:
    app: sparse-search-service
    component: index-builder
    app.kubernetes.io/name: sparse-search-index-builder
    app.kubernetes.io/part-of: atenex-platform
spec:
  schedule: "0 */6 * * *" # O "*/10 * * * *" para pruebas más rápidas
  jobTemplate:
    spec:
      ttlSecondsAfterFinished: 3600
      backoffLimit: 2
      template:
        metadata:
          labels:
            app: sparse-search-service
            component: index-builder-pod
            app.kubernetes.io/name: sparse-search-index-builder-pod
        spec:
          serviceAccountName: sparse-search-builder-sa # <--- ¡ASEGÚRATE QUE ESTA LÍNEA ESTÉ PRESENTE!
          containers:
            - name: index-builder-container
              image: ghcr.io/dev-nyro/sparse-search-service:develop-48c4f60 # Asegúrate que CI/CD actualice esto
              imagePullPolicy: Always
              command: ["python", "-m", "app.jobs.index_builder_cronjob"]
              args:
                - "--company-id"
                - "ALL"
              envFrom:
                - configMapRef:
                    name: sparse-search-service-config
                - secretRef:
                    name: sparse-search-service-secrets
              env:
                - name: SPARSE_POSTGRES_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: sparse-search-service-secrets
                      key: SPARSE_POSTGRES_PASSWORD
                - name: SPARSE_MINIO_ACCESS_KEY
                  valueFrom:
                    secretKeyRef:
                      name: sparse-search-service-secrets
                      key: SPARSE_MINIO_ACCESS_KEY
                - name: SPARSE_MINIO_SECRET_KEY
                  valueFrom:
                    secretKeyRef:
                      name: sparse-search-service-secrets
                      key: SPARSE_MINIO_SECRET_KEY
                - name: SPARSE_INDEX_STORAGE_ACCESS_KEY
                  valueFrom:
                    secretKeyRef:
                      name: sparse-search-service-secrets
                      key: SPARSE_MINIO_ACCESS_KEY
                - name: SPARSE_INDEX_STORAGE_SECRET_KEY
                  valueFrom:
                    secretKeyRef:
                      name: sparse-search-service-secrets
                      key: SPARSE_MINIO_SECRET_KEY
                - name: INDEX_STORAGE_ACCESS_KEY
                  valueFrom:
                    secretKeyRef:
                      name: sparse-search-service-secrets
                      key: SPARSE_MINIO_ACCESS_KEY
                - name: INDEX_STORAGE_SECRET_KEY
                  valueFrom:
                    secretKeyRef:
                      name: sparse-search-service-secrets
                      key: SPARSE_MINIO_SECRET_KEY
              resources:
                requests:
                  memory: "1Gi"
                  cpu: "500m"
                limits:
                  memory: "2Gi"
                  cpu: "1"
          restartPolicy: OnFailure

```

## File: `sparse-search-service/deployment.yaml`
```yaml
# sparse-search-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sparse-search-service
  namespace: atenex
  labels:
    app: sparse-search-service
    app.kubernetes.io/name: sparse-search-service
    app.kubernetes.io/version: "1.0.0"
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: atenex-platform
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sparse-search-service
  template:
    metadata:
      labels:
        app: sparse-search-service
        app.kubernetes.io/name: sparse-search-service
        app.kubernetes.io/version: "1.0.0"
    spec:
      # serviceAccountName: sparse-search-runtime-sa # Alternativa si usas Workload Identity
      containers:
        - name: sparse-search-service-container
          image: ghcr.io/dev-nyro/sparse-search-service:develop-48c4f60 # Reemplaza con tu imagen
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8004
              protocol: TCP
          envFrom:
            - configMapRef:
                name: sparse-search-service-config
            - secretRef:
                name: sparse-search-service-secrets # Para SPARSE_POSTGRES_PASSWORD
          env:
            - name: SPARSE_POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: sparse-search-service-secrets
                  key: SPARSE_POSTGRES_PASSWORD
            - name: SPARSE_MINIO_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: sparse-search-service-secrets
                  key: SPARSE_MINIO_ACCESS_KEY
            - name: SPARSE_MINIO_SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: sparse-search-service-secrets
                  key: SPARSE_MINIO_SECRET_KEY
            - name: SPARSE_INDEX_STORAGE_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: sparse-search-service-secrets
                  key: SPARSE_MINIO_ACCESS_KEY
            - name: SPARSE_INDEX_STORAGE_SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: sparse-search-service-secrets
                  key: SPARSE_MINIO_SECRET_KEY
            - name: INDEX_STORAGE_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: sparse-search-service-secrets
                  key: SPARSE_MINIO_ACCESS_KEY
            - name: INDEX_STORAGE_SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: sparse-search-service-secrets
                  key: SPARSE_MINIO_SECRET_KEY
          resources:
            requests:
              memory: "1Gi"
              cpu: "500m"
            limits:
              memory: "4Gi"
              cpu: "2000m"
          readinessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 30
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 3
            successThreshold: 1
          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 60
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 3


```

## File: `sparse-search-service/service.yaml`
```yaml
# sparse-search-service/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: sparse-search-service 
  namespace: atenex
  labels:
    app: sparse-search-service 
    app.kubernetes.io/name: sparse-search-service
    app.kubernetes.io/part-of: atenex-platform
spec:
  type: ClusterIP 
  selector:
    app: sparse-search-service 
  ports:
  - name: http
    port: 80 
    targetPort: http 
    protocol: TCP
```

## File: `sparse-search-service/serviceaccount.yaml`
```yaml
# sparse-search-service/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sparse-search-builder-sa # El nombre exacto que usa el CronJob
  namespace: atenex
  labels:
    app: sparse-search-service
    component: index-builder-sa 
```
