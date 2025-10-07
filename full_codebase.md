# Estructura de los manifiestos de Kubernetes

```
manifests-nyro/
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
├── full_codebase.md
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

## File: `api-gateway\configmap.yaml`
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
  GATEWAY_DEFAULT_COMPANY_ID: "d7387dc8-0312-4b1c-97a5-f96f7995f36c"
  GATEWAY_HTTP_CLIENT_TIMEOUT: "30"
  GATEWAY_HTTP_CLIENT_MAX_KEEPALIVE_CONNECTIONS: "100" # Corregido
  GATEWAY_HTTP_CLIENT_MAX_CONNECTIONS: "200"
  GATEWAY_JWT_ALGORITHM: "HS256"
```

## File: `api-gateway\deployment.yaml`
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
          image: ghcr.io/dev-nyro/api-gateway:develop-4099116
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
              # --- Secciones livenessProbe y readinessProbe eliminadas ---
# imagePullSecrets: # Descomentar si GHCR es privado y necesitas un secreto para autenticar
# - name: ghcr-secret # Nombre del secreto tipo 'kubernetes.io/dockerconfigjson'

```

## File: `api-gateway\service.yaml`
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

## File: `docproc-service\configmap.yaml`
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
  DOCPROC_SUPPORTED_CONTENT_TYPES: '["application/pdf", "application/vnd.openxmlformats-officedocument.wordprocessingml.document", "application/msword", "text/plain", "text/markdown", "text/html"]'
  # DOCPROC_GUNICORN_WORKERS: "4" # Gunicorn workers se configuran en el CMD del Dockerfile, pero podría ser una variable aquí si se desea
```

## File: `docproc-service\deployment.yaml`
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
          image: ghcr.io/dev-nyro/docproc-service:develop-74d7e4d
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

## File: `docproc-service\service.yaml`
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

## File: `embedding-service\configmap.yaml`
```yaml
# embedding-service/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: embedding-service-config
  namespace: atenex
data:
  EMBEDDING_LOG_LEVEL: "INFO"
  EMBEDDING_PORT: "8003" # Gunicorn/Uvicorn will listen on this port inside the container

  # OpenAI Embedding Model Configuration
  EMBEDDING_OPENAI_EMBEDDING_MODEL_NAME: "text-embedding-3-small"
  # EMBEDDING_OPENAI_EMBEDDING_DIMENSIONS_OVERRIDE: "256" # Optional: uncomment to reduce dimension
  EMBEDDING_EMBEDDING_DIMENSION: "1536" # Must match the model's output, or OPENAI_EMBEDDING_DIMENSIONS_OVERRIDE if set.
                                        # For "text-embedding-3-small" default is 1536.
                                        # If overriding, change this value accordingly.

  # Optional: OpenAI API Base URL (e.g., for Azure OpenAI)
  # EMBEDDING_OPENAI_API_BASE: "https://YOUR_AZURE_OPENAI_RESOURCE.openai.azure.com/"

  EMBEDDING_OPENAI_TIMEOUT_SECONDS: "30"
  EMBEDDING_OPENAI_MAX_RETRIES: "3"
```

## File: `embedding-service\deployment.yaml`
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
          image: ghcr.io/dev-nyro/embedding-service:develop-8e590ba
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

## File: `embedding-service\service.yaml`
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

## File: `export_codebase.py`
```py
from pathlib import Path

# Carpetas que queremos excluir dentro de /app
EXCLUDED_DIRS = {'.git', '__pycache__', '.venv', '.idea', '.mypy_cache', '.vscode', '.github', 'node_modules'}

def build_tree(directory: Path, prefix: str = "") -> list:
    """
    Genera una representación en árbol de la estructura de directorios y archivos,
    excluyendo las carpetas especificadas en EXCLUDED_DIRS.
    """
    # Filtrar y ordenar los elementos del directorio
    entries = sorted(
        [entry for entry in directory.iterdir() if entry.name not in EXCLUDED_DIRS],
        key=lambda e: e.name
    )
    tree_lines = []
    for index, entry in enumerate(entries):
        connector = "└── " if index == len(entries) - 1 else "├── "
        tree_lines.append(prefix + connector + entry.name)
        if entry.is_dir():
            extension = "    " if index == len(entries) - 1 else "│   "
            tree_lines.extend(build_tree(entry, prefix + extension))
    return tree_lines

def generate_codebase_markdown(base_path: str = ".", output_file: str = "full_codebase.md"):
    base = Path(base_path).resolve()
    app_dir = base / "."

    if not app_dir.exists():
        print(f"[ERROR] La carpeta 'app' no existe en {base}")
        return

    lines = []

    # Agregar la estructura de directorios al inicio del Markdown
    lines.append("# Estructura de la Codebase")
    lines.append("")
    lines.append("```")
    lines.append("app/")
    tree_lines = build_tree(app_dir)
    lines.extend(tree_lines)
    lines.append("```")
    lines.append("")

    # Agregar el contenido de la codebase en Markdown
    lines.append("# Codebase: `app`")
    lines.append("")

    # Recorrer solo la carpeta app
    for path in sorted(app_dir.rglob("*")):
        # Ignorar directorios excluidos
        if any(part in EXCLUDED_DIRS for part in path.parts):
            continue

        if path.is_file():
            rel_path = path.relative_to(base)
            lines.append(f"## File: `{rel_path}`")
            try:
                content = path.read_text(encoding='utf-8')
            except UnicodeDecodeError:
                lines.append("_[Skipped: binary or non-UTF8 file]_")
                continue
            except Exception as e:
                lines.append(f"_[Error al leer el archivo: {e}]_")
                continue
            ext = path.suffix.lstrip('.')
            lang = ext if ext else ""
            lines.append(f"```{lang}")
            lines.append(content)
            lines.append("```")
            lines.append("")

    # Agregar pyproject.toml si existe en la raíz
    toml_path = base / "pyproject.toml"
    if toml_path.exists():
        lines.append("## File: `pyproject.toml`")
        try:
            content = toml_path.read_text(encoding='utf-8')
        except UnicodeDecodeError:
            lines.append("_[Skipped: binary or non-UTF8 file]_")
        except Exception as e:
            lines.append(f"_[Error al leer el archivo: {e}]_")
        else:
            lines.append("```toml")
            lines.append(content)
            lines.append("```")
            lines.append("")

    output_path = base / output_file
    try:
        output_path.write_text("\n".join(lines), encoding='utf-8')
        print(f"[OK] Código exportado a Markdown en: {output_path}")
    except Exception as e:
        print(f"[ERROR] Error al escribir el archivo de salida: {e}")

# Si se corre el script directamente
if __name__ == "__main__":
    generate_codebase_markdown()

```

## File: `full_codebase.md`
```md
# Estructura de los manifiestos de Kubernetes

```
manifests-nyro/
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
├── full_codebase.md
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
│   └── service.yaml
└── sparse-search-service
    ├── configmap.yaml
    ├── cronjob.yaml
    ├── deployment.yaml
    ├── service.yaml
    └── serviceaccount.yaml
```

# Codebase: `app`

## File: `api-gateway\configmap.yaml`
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
  GATEWAY_DEFAULT_COMPANY_ID: "d7387dc8-0312-4b1c-97a5-f96f7995f36c"
  GATEWAY_HTTP_CLIENT_TIMEOUT: "30"
  GATEWAY_HTTP_CLIENT_MAX_KEEPALIVE_CONNECTIONS: "100" # Corregido
  GATEWAY_HTTP_CLIENT_MAX_CONNECTIONS: "200"
  GATEWAY_JWT_ALGORITHM: "HS256"
```

## File: `api-gateway\deployment.yaml`
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
          image: ghcr.io/dev-nyro/api-gateway:develop-d60bccd
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
          # --- Secciones livenessProbe y readinessProbe eliminadas ---
# imagePullSecrets: # Descomentar si GHCR es privado y necesitas un secreto para autenticar
# - name: ghcr-secret # Nombre del secreto tipo 'kubernetes.io/dockerconfigjson'

```

## File: `api-gateway\service.yaml`
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

## File: `docproc-service\configmap.yaml`
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
  DOCPROC_SUPPORTED_CONTENT_TYPES: '["application/pdf", "application/vnd.openxmlformats-officedocument.wordprocessingml.document", "application/msword", "text/plain", "text/markdown", "text/html"]'
  # DOCPROC_GUNICORN_WORKERS: "4" # Gunicorn workers se configuran en el CMD del Dockerfile, pero podría ser una variable aquí si se desea
```

## File: `docproc-service\deployment.yaml`
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
          image: ghcr.io/dev-nyro/docproc-service:develop-5be78d9
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

## File: `docproc-service\service.yaml`
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

## File: `embedding-service\configmap.yaml`
```yaml
# embedding-service/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: embedding-service-config
  namespace: atenex
data:
  EMBEDDING_LOG_LEVEL: "INFO"
  EMBEDDING_PORT: "8003" # Gunicorn/Uvicorn will listen on this port inside the container

  # OpenAI Embedding Model Configuration
  EMBEDDING_OPENAI_EMBEDDING_MODEL_NAME: "text-embedding-3-small"
  # EMBEDDING_OPENAI_EMBEDDING_DIMENSIONS_OVERRIDE: "256" # Optional: uncomment to reduce dimension
  EMBEDDING_EMBEDDING_DIMENSION: "1536" # Must match the model's output, or OPENAI_EMBEDDING_DIMENSIONS_OVERRIDE if set.
                                        # For "text-embedding-3-small" default is 1536.
                                        # If overriding, change this value accordingly.

  # Optional: OpenAI API Base URL (e.g., for Azure OpenAI)
  # EMBEDDING_OPENAI_API_BASE: "https://YOUR_AZURE_OPENAI_RESOURCE.openai.azure.com/"

  EMBEDDING_OPENAI_TIMEOUT_SECONDS: "30"
  EMBEDDING_OPENAI_MAX_RETRIES: "3"
```

## File: `embedding-service\deployment.yaml`
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
          image: ghcr.io/dev-nyro/embedding-service:develop-5be78d9
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
              cpu: "250m"  # Reducido, ya que es I/O bound a OpenAI
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

## File: `embedding-service\service.yaml`
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

## File: `export_codebase.py`
```py
from pathlib import Path

# Carpetas que queremos excluir dentro de /app
EXCLUDED_DIRS = {'.git', '__pycache__', '.venv', '.idea', '.mypy_cache', '.vscode', '.github', 'node_modules'}

def build_tree(directory: Path, prefix: str = "") -> list:
    """
    Genera una representación en árbol de la estructura de directorios y archivos,
    excluyendo las carpetas especificadas en EXCLUDED_DIRS.
    """
    # Filtrar y ordenar los elementos del directorio
    entries = sorted(
        [entry for entry in directory.iterdir() if entry.name not in EXCLUDED_DIRS],
        key=lambda e: e.name
    )
    tree_lines = []
    for index, entry in enumerate(entries):
        connector = "└── " if index == len(entries) - 1 else "├── "
        tree_lines.append(prefix + connector + entry.name)
        if entry.is_dir():
            extension = "    " if index == len(entries) - 1 else "│   "
            tree_lines.extend(build_tree(entry, prefix + extension))
    return tree_lines

def generate_codebase_markdown(base_path: str = ".", output_file: str = "full_codebase.md"):
    base = Path(base_path).resolve()
    app_dir = base / "."

    if not app_dir.exists():
        print(f"[ERROR] La carpeta 'app' no existe en {base}")
        return

    lines = []

    # Agregar la estructura de directorios al inicio del Markdown
    lines.append("# Estructura de la Codebase")
    lines.append("")
    lines.append("```")
    lines.append("app/")
    tree_lines = build_tree(app_dir)
    lines.extend(tree_lines)
    lines.append("```")
    lines.append("")

    # Agregar el contenido de la codebase en Markdown
    lines.append("# Codebase: `app`")
    lines.append("")

    # Recorrer solo la carpeta app
    for path in sorted(app_dir.rglob("*")):
        # Ignorar directorios excluidos
        if any(part in EXCLUDED_DIRS for part in path.parts):
            continue

        if path.is_file():
            rel_path = path.relative_to(base)
            lines.append(f"## File: `{rel_path}`")
            try:
                content = path.read_text(encoding='utf-8')
            except UnicodeDecodeError:
                lines.append("_[Skipped: binary or non-UTF8 file]_")
                continue
            except Exception as e:
                lines.append(f"_[Error al leer el archivo: {e}]_")
                continue
            ext = path.suffix.lstrip('.')
            lang = ext if ext else ""
            lines.append(f"```{lang}")
            lines.append(content)
            lines.append("```")
            lines.append("")

    # Agregar pyproject.toml si existe en la raíz
    toml_path = base / "pyproject.toml"
    if toml_path.exists():
        lines.append("## File: `pyproject.toml`")
        try:
            content = toml_path.read_text(encoding='utf-8')
        except UnicodeDecodeError:
            lines.append("_[Skipped: binary or non-UTF8 file]_")
        except Exception as e:
            lines.append(f"_[Error al leer el archivo: {e}]_")
        else:
            lines.append("```toml")
            lines.append(content)
            lines.append("```")
            lines.append("")

    output_path = base / output_file
    try:
        output_path.write_text("\n".join(lines), encoding='utf-8')
        print(f"[OK] Código exportado a Markdown en: {output_path}")
    except Exception as e:
        print(f"[ERROR] Error al escribir el archivo de salida: {e}")

# Si se corre el script directamente
if __name__ == "__main__":
    generate_codebase_markdown()

```

## File: `full_codebase.md`
```md
# Estructura de la Codebase

```
mannifests-nyro/
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
├── full_codebase.md
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
│   └── service.yaml
└── sparse-search-service
    ├── configmap.yaml
    ├── cronjob.yaml
    ├── deployment.yaml
    ├── service.yaml
    └── serviceaccount.yaml
```

# Codebase: `app`

## File: `api-gateway\configmap.yaml`
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
  GATEWAY_DEFAULT_COMPANY_ID: "d7387dc8-0312-4b1c-97a5-f96f7995f36c"
  GATEWAY_HTTP_CLIENT_TIMEOUT: "30"
  GATEWAY_HTTP_CLIENT_MAX_KEEPALIVE_CONNECTIONS: "100" # Corregido
  GATEWAY_HTTP_CLIENT_MAX_CONNECTIONS: "200"
  GATEWAY_JWT_ALGORITHM: "HS256"
```

## File: `api-gateway\deployment.yaml`
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
          image: ghcr.io/dev-nyro/api-gateway:develop-d60bccd
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
          # --- Secciones livenessProbe y readinessProbe eliminadas ---
# imagePullSecrets: # Descomentar si GHCR es privado y necesitas un secreto para autenticar
# - name: ghcr-secret # Nombre del secreto tipo 'kubernetes.io/dockerconfigjson'

```

## File: `api-gateway\service.yaml`
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

## File: `docproc-service\configmap.yaml`
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
  DOCPROC_SUPPORTED_CONTENT_TYPES: '["application/pdf", "application/vnd.openxmlformats-officedocument.wordprocessingml.document", "application/msword", "text/plain", "text/markdown", "text/html"]'
  # DOCPROC_GUNICORN_WORKERS: "4" # Gunicorn workers se configuran en el CMD del Dockerfile, pero podría ser una variable aquí si se desea
```

## File: `docproc-service\deployment.yaml`
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
          image: ghcr.io/dev-nyro/docproc-service:develop-d60bccd
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

## File: `docproc-service\service.yaml`
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

## File: `embedding-service\configmap.yaml`
```yaml
# embedding-service/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: embedding-service-config
  namespace: atenex
data:
  EMBEDDING_LOG_LEVEL: "INFO"
  EMBEDDING_PORT: "8003" # Gunicorn/Uvicorn will listen on this port inside the container

  # FastEmbed Model Configuration
  EMBEDDING_FASTEMBED_MODEL_NAME: "sentence-transformers/all-MiniLM-L6-v2"
  EMBEDDING_EMBEDDING_DIMENSION: "384" # Must match the model

  # Optional: Cache directory for FastEmbed models inside the container
  # If set, consider if this path needs a persistent volume or if ephemeral is fine.
  # For small models, ephemeral might be okay as download is quick.
  # EMBEDDING_FASTEMBED_CACHE_DIR: "/app/.cache/fastembed"
  EMBEDDING_FASTEMBED_CACHE_DIR: "/dev/shm/fastembed_cache" # Use in-memory /dev/shm for faster startup if models are small

  # Optional: Number of threads for FastEmbed tokenization
  # EMBEDDING_FASTEMBED_THREADS: "2" # Example: limit to 2 threads

  # Optional: Max sequence length for the model
  EMBEDDING_FASTEMBED_MAX_LENGTH: "512"
```

## File: `embedding-service\deployment.yaml`
```yaml
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
      volumes:
        - name: dshm # Nombre del volumen
          emptyDir:
            medium: Memory # Usar memoria para este emptyDir
            sizeLimit: 256Mi # Límite de tamaño para el tmpfs
      containers:
        - name: embedding-service
          image: ghcr.io/dev-nyro/embedding-service:develop-d60bccd
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8003
              protocol: TCP
          envFrom:
            - configMapRef:
                name: embedding-service-config
          volumeMounts:
            - name: dshm # Referencia al volumen definido arriba
              mountPath: /dev/shm # CORREGIDO: Montar en /dev/shm
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "1500m"
              memory: "3Gi"
          readinessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 30 # Dar tiempo suficiente para la descarga y carga del modelo
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

## File: `embedding-service\service.yaml`
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

## File: `export_codebase.py`
```py
from pathlib import Path

# Carpetas que queremos excluir dentro de /app
EXCLUDED_DIRS = {'.git', '__pycache__', '.venv', '.idea', '.mypy_cache', '.vscode', '.github', 'node_modules'}

def build_tree(directory: Path, prefix: str = "") -> list:
    """
    Genera una representación en árbol de la estructura de directorios y archivos,
    excluyendo las carpetas especificadas en EXCLUDED_DIRS.
    """
    # Filtrar y ordenar los elementos del directorio
    entries = sorted(
        [entry for entry in directory.iterdir() if entry.name not in EXCLUDED_DIRS],
        key=lambda e: e.name
    )
    tree_lines = []
    for index, entry in enumerate(entries):
        connector = "└── " if index == len(entries) - 1 else "├── "
        tree_lines.append(prefix + connector + entry.name)
        if entry.is_dir():
            extension = "    " if index == len(entries) - 1 else "│   "
            tree_lines.extend(build_tree(entry, prefix + extension))
    return tree_lines

def generate_codebase_markdown(base_path: str = ".", output_file: str = "full_codebase.md"):
    base = Path(base_path).resolve()
    app_dir = base / "."

    if not app_dir.exists():
        print(f"[ERROR] La carpeta 'app' no existe en {base}")
        return

    lines = []

    # Agregar la estructura de directorios al inicio del Markdown
    lines.append("# Estructura de la Codebase")
    lines.append("")
    lines.append("```")
    lines.append("app/")
    tree_lines = build_tree(app_dir)
    lines.extend(tree_lines)
    lines.append("```")
    lines.append("")

    # Agregar el contenido de la codebase en Markdown
    lines.append("# Codebase: `app`")
    lines.append("")

    # Recorrer solo la carpeta app
    for path in sorted(app_dir.rglob("*")):
        # Ignorar directorios excluidos
        if any(part in EXCLUDED_DIRS for part in path.parts):
            continue

        if path.is_file():
            rel_path = path.relative_to(base)
            lines.append(f"## File: `{rel_path}`")
            try:
                content = path.read_text(encoding='utf-8')
            except UnicodeDecodeError:
                lines.append("_[Skipped: binary or non-UTF8 file]_")
                continue
            except Exception as e:
                lines.append(f"_[Error al leer el archivo: {e}]_")
                continue
            ext = path.suffix.lstrip('.')
            lang = ext if ext else ""
            lines.append(f"```{lang}")
            lines.append(content)
            lines.append("```")
            lines.append("")

    # Agregar pyproject.toml si existe en la raíz
    toml_path = base / "pyproject.toml"
    if toml_path.exists():
        lines.append("## File: `pyproject.toml`")
        try:
            content = toml_path.read_text(encoding='utf-8')
        except UnicodeDecodeError:
            lines.append("_[Skipped: binary or non-UTF8 file]_")
        except Exception as e:
            lines.append(f"_[Error al leer el archivo: {e}]_")
        else:
            lines.append("```toml")
            lines.append(content)
            lines.append("```")
            lines.append("")

    output_path = base / output_file
    try:
        output_path.write_text("\n".join(lines), encoding='utf-8')
        print(f"[OK] Código exportado a Markdown en: {output_path}")
    except Exception as e:
        print(f"[ERROR] Error al escribir el archivo de salida: {e}")

# Si se corre el script directamente
if __name__ == "__main__":
    generate_codebase_markdown()

```

## File: `full_codebase.md`
```md
# Estructura de la Codebase

```
manifestes-nyro/
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
├── full_codebase.md
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
│   ├── endpoints.yaml
│   └── service.yaml
└── sparse-search-service
    ├── configmap.yaml
    ├── cronjob.yaml
    ├── deployment.yaml
    ├── service.yaml
    └── serviceaccount.yaml
```

# Codebase: `app`

## File: `api-gateway\configmap.yaml`
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
  GATEWAY_DEFAULT_COMPANY_ID: "d7387dc8-0312-4b1c-97a5-f96f7995f36c"
  GATEWAY_HTTP_CLIENT_TIMEOUT: "30"
  GATEWAY_HTTP_CLIENT_MAX_KEEPALIVE_CONNECTIONS: "100" # Corregido
  GATEWAY_HTTP_CLIENT_MAX_CONNECTIONS: "200"
  GATEWAY_JWT_ALGORITHM: "HS256"
```

## File: `api-gateway\deployment.yaml`
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
          image: ghcr.io/dev-nyro/api-gateway:develop-d60bccd
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
          # --- Secciones livenessProbe y readinessProbe eliminadas ---
# imagePullSecrets: # Descomentar si GHCR es privado y necesitas un secreto para autenticar
# - name: ghcr-secret # Nombre del secreto tipo 'kubernetes.io/dockerconfigjson'

```

## File: `api-gateway\service.yaml`
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

## File: `docproc-service\configmap.yaml`
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
  DOCPROC_SUPPORTED_CONTENT_TYPES: '["application/pdf", "application/vnd.openxmlformats-officedocument.wordprocessingml.document", "application/msword", "text/plain", "text/markdown", "text/html"]'
  # DOCPROC_GUNICORN_WORKERS: "4" # Gunicorn workers se configuran en el CMD del Dockerfile, pero podría ser una variable aquí si se desea
```

## File: `docproc-service\deployment.yaml`
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
          image: ghcr.io/dev-nyro/docproc-service:develop-d60bccd
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

## File: `docproc-service\service.yaml`
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

## File: `embedding-service\configmap.yaml`
```yaml
# embedding-service/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: embedding-service-config
  namespace: atenex
data:
  EMBEDDING_LOG_LEVEL: "INFO"
  EMBEDDING_PORT: "8003" # Gunicorn/Uvicorn will listen on this port inside the container

  # FastEmbed Model Configuration
  EMBEDDING_FASTEMBED_MODEL_NAME: "sentence-transformers/all-MiniLM-L6-v2"
  EMBEDDING_EMBEDDING_DIMENSION: "384" # Must match the model

  # Optional: Cache directory for FastEmbed models inside the container
  # If set, consider if this path needs a persistent volume or if ephemeral is fine.
  # For small models, ephemeral might be okay as download is quick.
  # EMBEDDING_FASTEMBED_CACHE_DIR: "/app/.cache/fastembed"
  EMBEDDING_FASTEMBED_CACHE_DIR: "/dev/shm/fastembed_cache" # Use in-memory /dev/shm for faster startup if models are small

  # Optional: Number of threads for FastEmbed tokenization
  # EMBEDDING_FASTEMBED_THREADS: "2" # Example: limit to 2 threads

  # Optional: Max sequence length for the model
  EMBEDDING_FASTEMBED_MAX_LENGTH: "512"
```

## File: `embedding-service\deployment.yaml`
```yaml
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
      volumes:
        - name: dshm # Nombre del volumen
          emptyDir:
            medium: Memory # Usar memoria para este emptyDir
            sizeLimit: 256Mi # Límite de tamaño para el tmpfs
      containers:
        - name: embedding-service
          image: ghcr.io/dev-nyro/embedding-service:develop-d60bccd
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8003
              protocol: TCP
          envFrom:
            - configMapRef:
                name: embedding-service-config
          volumeMounts:
            - name: dshm # Referencia al volumen definido arriba
              mountPath: /dev/shm # CORREGIDO: Montar en /dev/shm
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "1500m"
              memory: "3Gi"
          readinessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 30 # Dar tiempo suficiente para la descarga y carga del modelo
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

## File: `embedding-service\service.yaml`
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

## File: `export_codebase.py`
```py
from pathlib import Path

# Carpetas que queremos excluir dentro de /app
EXCLUDED_DIRS = {'.git', '__pycache__', '.venv', '.idea', '.mypy_cache', '.vscode', '.github', 'node_modules'}

def build_tree(directory: Path, prefix: str = "") -> list:
    """
    Genera una representación en árbol de la estructura de directorios y archivos,
    excluyendo las carpetas especificadas en EXCLUDED_DIRS.
    """
    # Filtrar y ordenar los elementos del directorio
    entries = sorted(
        [entry for entry in directory.iterdir() if entry.name not in EXCLUDED_DIRS],
        key=lambda e: e.name
    )
    tree_lines = []
    for index, entry in enumerate(entries):
        connector = "└── " if index == len(entries) - 1 else "├── "
        tree_lines.append(prefix + connector + entry.name)
        if entry.is_dir():
            extension = "    " if index == len(entries) - 1 else "│   "
            tree_lines.extend(build_tree(entry, prefix + extension))
    return tree_lines

def generate_codebase_markdown(base_path: str = ".", output_file: str = "full_codebase.md"):
    base = Path(base_path).resolve()
    app_dir = base / "."

    if not app_dir.exists():
        print(f"[ERROR] La carpeta 'app' no existe en {base}")
        return

    lines = []

    # Agregar la estructura de directorios al inicio del Markdown
    lines.append("# Estructura de la Codebase")
    lines.append("")
    lines.append("```")
    lines.append("app/")
    tree_lines = build_tree(app_dir)
    lines.extend(tree_lines)
    lines.append("```")
    lines.append("")

    # Agregar el contenido de la codebase en Markdown
    lines.append("# Codebase: `app`")
    lines.append("")

    # Recorrer solo la carpeta app
    for path in sorted(app_dir.rglob("*")):
        # Ignorar directorios excluidos
        if any(part in EXCLUDED_DIRS for part in path.parts):
            continue

        if path.is_file():
            rel_path = path.relative_to(base)
            lines.append(f"## File: `{rel_path}`")
            try:
                content = path.read_text(encoding='utf-8')
            except UnicodeDecodeError:
                lines.append("_[Skipped: binary or non-UTF8 file]_")
                continue
            except Exception as e:
                lines.append(f"_[Error al leer el archivo: {e}]_")
                continue
            ext = path.suffix.lstrip('.')
            lang = ext if ext else ""
            lines.append(f"```{lang}")
            lines.append(content)
            lines.append("```")
            lines.append("")

    # Agregar pyproject.toml si existe en la raíz
    toml_path = base / "pyproject.toml"
    if toml_path.exists():
        lines.append("## File: `pyproject.toml`")
        try:
            content = toml_path.read_text(encoding='utf-8')
        except UnicodeDecodeError:
            lines.append("_[Skipped: binary or non-UTF8 file]_")
        except Exception as e:
            lines.append(f"_[Error al leer el archivo: {e}]_")
        else:
            lines.append("```toml")
            lines.append(content)
            lines.append("```")
            lines.append("")

    output_path = base / output_file
    try:
        output_path.write_text("\n".join(lines), encoding='utf-8')
        print(f"[OK] Código exportado a Markdown en: {output_path}")
    except Exception as e:
        print(f"[ERROR] Error al escribir el archivo de salida: {e}")

# Si se corre el script directamente
if __name__ == "__main__":
    generate_codebase_markdown()

```

## File: `full_codebase.md`
```md
# Estructura del repositorio de manifiestos de Kubernetes

```
manifests-nyro/
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
├── full_codebase.md
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
└── sparse-search-service
    ├── configmap.yaml
    ├── cronjob.yaml
    ├── deployment.yaml
    ├── service.yaml
    └── serviceaccount.yaml
```

# Codebase: `app`

## File: `api-gateway\configmap.yaml`
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
  GATEWAY_DEFAULT_COMPANY_ID: "d7387dc8-0312-4b1c-97a5-f96f7995f36c"
  GATEWAY_HTTP_CLIENT_TIMEOUT: "30"
  GATEWAY_HTTP_CLIENT_MAX_KEEPALIVE_CONNECTIONS: "100" # Corregido
  GATEWAY_HTTP_CLIENT_MAX_CONNECTIONS: "200"
  GATEWAY_JWT_ALGORITHM: "HS256"
```

## File: `api-gateway\deployment.yaml`
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
          image: ghcr.io/dev-nyro/api-gateway:develop-d60bccd
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
          # --- Secciones livenessProbe y readinessProbe eliminadas ---
# imagePullSecrets: # Descomentar si GHCR es privado y necesitas un secreto para autenticar
# - name: ghcr-secret # Nombre del secreto tipo 'kubernetes.io/dockerconfigjson'

```

## File: `api-gateway\service.yaml`
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

## File: `docproc-service\configmap.yaml`
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
  DOCPROC_SUPPORTED_CONTENT_TYPES: '["application/pdf", "application/vnd.openxmlformats-officedocument.wordprocessingml.document", "application/msword", "text/plain", "text/markdown", "text/html"]'
  # DOCPROC_GUNICORN_WORKERS: "4" # Gunicorn workers se configuran en el CMD del Dockerfile, pero podría ser una variable aquí si se desea
```

## File: `docproc-service\deployment.yaml`
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
          image: ghcr.io/dev-nyro/docproc-service:develop-d60bccd
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

## File: `docproc-service\service.yaml`
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

## File: `embedding-service\configmap.yaml`
```yaml
# embedding-service/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: embedding-service-config
  namespace: atenex
data:
  EMBEDDING_LOG_LEVEL: "INFO"
  EMBEDDING_PORT: "8003" # Gunicorn/Uvicorn will listen on this port inside the container

  # FastEmbed Model Configuration
  EMBEDDING_FASTEMBED_MODEL_NAME: "sentence-transformers/all-MiniLM-L6-v2"
  EMBEDDING_EMBEDDING_DIMENSION: "384" # Must match the model

  # Optional: Cache directory for FastEmbed models inside the container
  # If set, consider if this path needs a persistent volume or if ephemeral is fine.
  # For small models, ephemeral might be okay as download is quick.
  # EMBEDDING_FASTEMBED_CACHE_DIR: "/app/.cache/fastembed"
  EMBEDDING_FASTEMBED_CACHE_DIR: "/dev/shm/fastembed_cache" # Use in-memory /dev/shm for faster startup if models are small

  # Optional: Number of threads for FastEmbed tokenization
  # EMBEDDING_FASTEMBED_THREADS: "2" # Example: limit to 2 threads

  # Optional: Max sequence length for the model
  EMBEDDING_FASTEMBED_MAX_LENGTH: "512"
```

## File: `embedding-service\deployment.yaml`
```yaml
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
      volumes:
        - name: dshm # Nombre del volumen
          emptyDir:
            medium: Memory # Usar memoria para este emptyDir
            sizeLimit: 256Mi # Límite de tamaño para el tmpfs
      containers:
        - name: embedding-service
          image: ghcr.io/dev-nyro/embedding-service:develop-d60bccd
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8003
              protocol: TCP
          envFrom:
            - configMapRef:
                name: embedding-service-config
          volumeMounts:
            - name: dshm # Referencia al volumen definido arriba
              mountPath: /dev/shm # CORREGIDO: Montar en /dev/shm
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "1500m"
              memory: "3Gi"
          readinessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 30 # Dar tiempo suficiente para la descarga y carga del modelo
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

## File: `embedding-service\service.yaml`
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

## File: `export_codebase.py`
```py
from pathlib import Path

# Carpetas que queremos excluir dentro de /app
EXCLUDED_DIRS = {'.git', '__pycache__', '.venv', '.idea', '.mypy_cache', '.vscode', '.github', 'node_modules'}

def build_tree(directory: Path, prefix: str = "") -> list:
    """
    Genera una representación en árbol de la estructura de directorios y archivos,
    excluyendo las carpetas especificadas en EXCLUDED_DIRS.
    """
    # Filtrar y ordenar los elementos del directorio
    entries = sorted(
        [entry for entry in directory.iterdir() if entry.name not in EXCLUDED_DIRS],
        key=lambda e: e.name
    )
    tree_lines = []
    for index, entry in enumerate(entries):
        connector = "└── " if index == len(entries) - 1 else "├── "
        tree_lines.append(prefix + connector + entry.name)
        if entry.is_dir():
            extension = "    " if index == len(entries) - 1 else "│   "
            tree_lines.extend(build_tree(entry, prefix + extension))
    return tree_lines

def generate_codebase_markdown(base_path: str = ".", output_file: str = "full_codebase.md"):
    base = Path(base_path).resolve()
    app_dir = base / "."

    if not app_dir.exists():
        print(f"[ERROR] La carpeta 'app' no existe en {base}")
        return

    lines = []

    # Agregar la estructura de directorios al inicio del Markdown
    lines.append("# Estructura de la Codebase")
    lines.append("")
    lines.append("```")
    lines.append("app/")
    tree_lines = build_tree(app_dir)
    lines.extend(tree_lines)
    lines.append("```")
    lines.append("")

    # Agregar el contenido de la codebase en Markdown
    lines.append("# Codebase: `app`")
    lines.append("")

    # Recorrer solo la carpeta app
    for path in sorted(app_dir.rglob("*")):
        # Ignorar directorios excluidos
        if any(part in EXCLUDED_DIRS for part in path.parts):
            continue

        if path.is_file():
            rel_path = path.relative_to(base)
            lines.append(f"## File: `{rel_path}`")
            try:
                content = path.read_text(encoding='utf-8')
            except UnicodeDecodeError:
                lines.append("_[Skipped: binary or non-UTF8 file]_")
                continue
            except Exception as e:
                lines.append(f"_[Error al leer el archivo: {e}]_")
                continue
            ext = path.suffix.lstrip('.')
            lang = ext if ext else ""
            lines.append(f"```{lang}")
            lines.append(content)
            lines.append("```")
            lines.append("")

    # Agregar pyproject.toml si existe en la raíz
    toml_path = base / "pyproject.toml"
    if toml_path.exists():
        lines.append("## File: `pyproject.toml`")
        try:
            content = toml_path.read_text(encoding='utf-8')
        except UnicodeDecodeError:
            lines.append("_[Skipped: binary or non-UTF8 file]_")
        except Exception as e:
            lines.append(f"_[Error al leer el archivo: {e}]_")
        else:
            lines.append("```toml")
            lines.append(content)
            lines.append("```")
            lines.append("")

    output_path = base / output_file
    try:
        output_path.write_text("\n".join(lines), encoding='utf-8')
        print(f"[OK] Código exportado a Markdown en: {output_path}")
    except Exception as e:
        print(f"[ERROR] Error al escribir el archivo de salida: {e}")

# Si se corre el script directamente
if __name__ == "__main__":
    generate_codebase_markdown()

```

## File: `full_codebase.md`
```md
# Estructura de los manifiestos de Kubernetes

```
manifests-nyro/
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
├── full_codebase.md
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
└── sparse-search-service
    ├── configmap.yaml
    ├── cronjob.yaml
    ├── deployment.yaml
    └── service.yaml
```

# Codebase: `app`

## File: `api-gateway\configmap.yaml`
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
  GATEWAY_DEFAULT_COMPANY_ID: "d7387dc8-0312-4b1c-97a5-f96f7995f36c"
  GATEWAY_HTTP_CLIENT_TIMEOUT: "30"
  GATEWAY_HTTP_CLIENT_MAX_KEEPALIVE_CONNECTIONS: "100" # Corregido
  GATEWAY_HTTP_CLIENT_MAX_CONNECTIONS: "200"
  GATEWAY_JWT_ALGORITHM: "HS256"
```

## File: `api-gateway\deployment.yaml`
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
          image: ghcr.io/dev-nyro/api-gateway:develop-9bcb370
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
          # --- Secciones livenessProbe y readinessProbe eliminadas ---

      # imagePullSecrets: # Descomentar si GHCR es privado y necesitas un secreto para autenticar
      # - name: ghcr-secret # Nombre del secreto tipo 'kubernetes.io/dockerconfigjson'


```

## File: `api-gateway\service.yaml`
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

## File: `docproc-service\configmap.yaml`
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
  DOCPROC_SUPPORTED_CONTENT_TYPES: '["application/pdf", "application/vnd.openxmlformats-officedocument.wordprocessingml.document", "application/msword", "text/plain", "text/markdown", "text/html"]'
  # DOCPROC_GUNICORN_WORKERS: "4" # Gunicorn workers se configuran en el CMD del Dockerfile, pero podría ser una variable aquí si se desea
```

## File: `docproc-service\deployment.yaml`
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
          image: ghcr.io/dev-nyro/docproc-service:develop-1c92a51
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

## File: `docproc-service\service.yaml`
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

## File: `embedding-service\configmap.yaml`
```yaml
# embedding-service/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: embedding-service-config
  namespace: atenex
data:
  EMBEDDING_LOG_LEVEL: "INFO"
  EMBEDDING_PORT: "8003" # Gunicorn/Uvicorn will listen on this port inside the container

  # FastEmbed Model Configuration
  EMBEDDING_FASTEMBED_MODEL_NAME: "sentence-transformers/all-MiniLM-L6-v2"
  EMBEDDING_EMBEDDING_DIMENSION: "384" # Must match the model

  # Optional: Cache directory for FastEmbed models inside the container
  # If set, consider if this path needs a persistent volume or if ephemeral is fine.
  # For small models, ephemeral might be okay as download is quick.
  # EMBEDDING_FASTEMBED_CACHE_DIR: "/app/.cache/fastembed"
  EMBEDDING_FASTEMBED_CACHE_DIR: "/dev/shm/fastembed_cache" # Use in-memory /dev/shm for faster startup if models are small

  # Optional: Number of threads for FastEmbed tokenization
  # EMBEDDING_FASTEMBED_THREADS: "2" # Example: limit to 2 threads

  # Optional: Max sequence length for the model
  EMBEDDING_FASTEMBED_MAX_LENGTH: "512"
```

## File: `embedding-service\deployment.yaml`
```yaml
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
      volumes:
        - name: dshm # Nombre del volumen
          emptyDir:
            medium: Memory # Usar memoria para este emptyDir
            sizeLimit: 256Mi # Límite de tamaño para el tmpfs
      containers:
        - name: embedding-service
          image: ghcr.io/dev-nyro/embedding-service:develop-b2e97c8
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8003
              protocol: TCP
          envFrom:
            - configMapRef:
                name: embedding-service-config
          volumeMounts:
            - name: dshm # Referencia al volumen definido arriba
              mountPath: /dev/shm # CORREGIDO: Montar en /dev/shm
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "1500m"
              memory: "3Gi"
          readinessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 30 # Dar tiempo suficiente para la descarga y carga del modelo
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

## File: `embedding-service\service.yaml`
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

## File: `export_codebase.py`
```py
from pathlib import Path

# Carpetas que queremos excluir dentro de /app
EXCLUDED_DIRS = {'.git', '__pycache__', '.venv', '.idea', '.mypy_cache', '.vscode', '.github', 'node_modules'}

def build_tree(directory: Path, prefix: str = "") -> list:
    """
    Genera una representación en árbol de la estructura de directorios y archivos,
    excluyendo las carpetas especificadas en EXCLUDED_DIRS.
    """
    # Filtrar y ordenar los elementos del directorio
    entries = sorted(
        [entry for entry in directory.iterdir() if entry.name not in EXCLUDED_DIRS],
        key=lambda e: e.name
    )
    tree_lines = []
    for index, entry in enumerate(entries):
        connector = "└── " if index == len(entries) - 1 else "├── "
        tree_lines.append(prefix + connector + entry.name)
        if entry.is_dir():
            extension = "    " if index == len(entries) - 1 else "│   "
            tree_lines.extend(build_tree(entry, prefix + extension))
    return tree_lines

def generate_codebase_markdown(base_path: str = ".", output_file: str = "full_codebase.md"):
    base = Path(base_path).resolve()
    app_dir = base / "."

    if not app_dir.exists():
        print(f"[ERROR] La carpeta 'app' no existe en {base}")
        return

    lines = []

    # Agregar la estructura de directorios al inicio del Markdown
    lines.append("# Estructura de la Codebase")
    lines.append("")
    lines.append("```")
    lines.append("app/")
    tree_lines = build_tree(app_dir)
    lines.extend(tree_lines)
    lines.append("```")
    lines.append("")

    # Agregar el contenido de la codebase en Markdown
    lines.append("# Codebase: `app`")
    lines.append("")

    # Recorrer solo la carpeta app
    for path in sorted(app_dir.rglob("*")):
        # Ignorar directorios excluidos
        if any(part in EXCLUDED_DIRS for part in path.parts):
            continue

        if path.is_file():
            rel_path = path.relative_to(base)
            lines.append(f"## File: `{rel_path}`")
            try:
                content = path.read_text(encoding='utf-8')
            except UnicodeDecodeError:
                lines.append("_[Skipped: binary or non-UTF8 file]_")
                continue
            except Exception as e:
                lines.append(f"_[Error al leer el archivo: {e}]_")
                continue
            ext = path.suffix.lstrip('.')
            lang = ext if ext else ""
            lines.append(f"```{lang}")
            lines.append(content)
            lines.append("```")
            lines.append("")

    # Agregar pyproject.toml si existe en la raíz
    toml_path = base / "pyproject.toml"
    if toml_path.exists():
        lines.append("## File: `pyproject.toml`")
        try:
            content = toml_path.read_text(encoding='utf-8')
        except UnicodeDecodeError:
            lines.append("_[Skipped: binary or non-UTF8 file]_")
        except Exception as e:
            lines.append(f"_[Error al leer el archivo: {e}]_")
        else:
            lines.append("```toml")
            lines.append(content)
            lines.append("```")
            lines.append("")

    output_path = base / output_file
    try:
        output_path.write_text("\n".join(lines), encoding='utf-8')
        print(f"[OK] Código exportado a Markdown en: {output_path}")
    except Exception as e:
        print(f"[ERROR] Error al escribir el archivo de salida: {e}")

# Si se corre el script directamente
if __name__ == "__main__":
    generate_codebase_markdown()

```

## File: `full_codebase.md`
```md
# Estructura de los manifiestos de Kubernetes

```
manifests-nyro/
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
├── full_codebase.md
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
└── sparse-search-service
    ├── configmap.yaml
    ├── cronjob.yaml
    ├── deployment.yaml
    └── service.yaml
```

# Codebase: `app`

## File: `api-gateway\configmap.yaml`
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
  GATEWAY_DEFAULT_COMPANY_ID: "d7387dc8-0312-4b1c-97a5-f96f7995f36c"
  GATEWAY_HTTP_CLIENT_TIMEOUT: "30"
  GATEWAY_HTTP_CLIENT_MAX_KEEPALIVE_CONNECTIONS: "100" # Corregido
  GATEWAY_HTTP_CLIENT_MAX_CONNECTIONS: "200"
  GATEWAY_JWT_ALGORITHM: "HS256"
```

## File: `api-gateway\deployment.yaml`
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
          image: ghcr.io/dev-nyro/api-gateway:develop-9bcb370
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
          # --- Secciones livenessProbe y readinessProbe eliminadas ---

      # imagePullSecrets: # Descomentar si GHCR es privado y necesitas un secreto para autenticar
      # - name: ghcr-secret # Nombre del secreto tipo 'kubernetes.io/dockerconfigjson'


```

## File: `api-gateway\service.yaml`
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

## File: `docproc-service\configmap.yaml`
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
  DOCPROC_SUPPORTED_CONTENT_TYPES: '["application/pdf", "application/vnd.openxmlformats-officedocument.wordprocessingml.document", "application/msword", "text/plain", "text/markdown", "text/html"]'
  # DOCPROC_GUNICORN_WORKERS: "4" # Gunicorn workers se configuran en el CMD del Dockerfile, pero podría ser una variable aquí si se desea
```

## File: `docproc-service\deployment.yaml`
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
          image: ghcr.io/dev-nyro/docproc-service:develop-1c92a51
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

## File: `docproc-service\service.yaml`
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

## File: `embedding-service\configmap.yaml`
```yaml
# embedding-service/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: embedding-service-config
  namespace: atenex
data:
  EMBEDDING_LOG_LEVEL: "INFO"
  EMBEDDING_PORT: "8003" # Gunicorn/Uvicorn will listen on this port inside the container

  # FastEmbed Model Configuration
  EMBEDDING_FASTEMBED_MODEL_NAME: "sentence-transformers/all-MiniLM-L6-v2"
  EMBEDDING_EMBEDDING_DIMENSION: "384" # Must match the model

  # Optional: Cache directory for FastEmbed models inside the container
  # If set, consider if this path needs a persistent volume or if ephemeral is fine.
  # For small models, ephemeral might be okay as download is quick.
  # EMBEDDING_FASTEMBED_CACHE_DIR: "/app/.cache/fastembed"
  EMBEDDING_FASTEMBED_CACHE_DIR: "/dev/shm/fastembed_cache" # Use in-memory /dev/shm for faster startup if models are small

  # Optional: Number of threads for FastEmbed tokenization
  # EMBEDDING_FASTEMBED_THREADS: "2" # Example: limit to 2 threads

  # Optional: Max sequence length for the model
  EMBEDDING_FASTEMBED_MAX_LENGTH: "512"
```

## File: `embedding-service\deployment.yaml`
```yaml
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
      volumes:
        - name: dshm # Nombre del volumen
          emptyDir:
            medium: Memory # Usar memoria para este emptyDir
            sizeLimit: 256Mi # Límite de tamaño para el tmpfs
      containers:
        - name: embedding-service
          image: ghcr.io/dev-nyro/embedding-service:develop-b2e97c8
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8003
              protocol: TCP
          envFrom:
            - configMapRef:
                name: embedding-service-config
          volumeMounts:
            - name: dshm # Referencia al volumen definido arriba
              mountPath: /dev/shm # CORREGIDO: Montar en /dev/shm
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "1500m"
              memory: "3Gi"
          readinessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 30 # Dar tiempo suficiente para la descarga y carga del modelo
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

## File: `embedding-service\service.yaml`
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

## File: `export_codebase.py`
```py
from pathlib import Path

# Carpetas que queremos excluir dentro de /app
EXCLUDED_DIRS = {'.git', '__pycache__', '.venv', '.idea', '.mypy_cache', '.vscode', '.github', 'node_modules'}

def build_tree(directory: Path, prefix: str = "") -> list:
    """
    Genera una representación en árbol de la estructura de directorios y archivos,
    excluyendo las carpetas especificadas en EXCLUDED_DIRS.
    """
    # Filtrar y ordenar los elementos del directorio
    entries = sorted(
        [entry for entry in directory.iterdir() if entry.name not in EXCLUDED_DIRS],
        key=lambda e: e.name
    )
    tree_lines = []
    for index, entry in enumerate(entries):
        connector = "└── " if index == len(entries) - 1 else "├── "
        tree_lines.append(prefix + connector + entry.name)
        if entry.is_dir():
            extension = "    " if index == len(entries) - 1 else "│   "
            tree_lines.extend(build_tree(entry, prefix + extension))
    return tree_lines

def generate_codebase_markdown(base_path: str = ".", output_file: str = "full_codebase.md"):
    base = Path(base_path).resolve()
    app_dir = base / "."

    if not app_dir.exists():
        print(f"[ERROR] La carpeta 'app' no existe en {base}")
        return

    lines = []

    # Agregar la estructura de directorios al inicio del Markdown
    lines.append("# Estructura de la Codebase")
    lines.append("")
    lines.append("```")
    lines.append("app/")
    tree_lines = build_tree(app_dir)
    lines.extend(tree_lines)
    lines.append("```")
    lines.append("")

    # Agregar el contenido de la codebase en Markdown
    lines.append("# Codebase: `app`")
    lines.append("")

    # Recorrer solo la carpeta app
    for path in sorted(app_dir.rglob("*")):
        # Ignorar directorios excluidos
        if any(part in EXCLUDED_DIRS for part in path.parts):
            continue

        if path.is_file():
            rel_path = path.relative_to(base)
            lines.append(f"## File: `{rel_path}`")
            try:
                content = path.read_text(encoding='utf-8')
            except UnicodeDecodeError:
                lines.append("_[Skipped: binary or non-UTF8 file]_")
                continue
            except Exception as e:
                lines.append(f"_[Error al leer el archivo: {e}]_")
                continue
            ext = path.suffix.lstrip('.')
            lang = ext if ext else ""
            lines.append(f"```{lang}")
            lines.append(content)
            lines.append("```")
            lines.append("")

    # Agregar pyproject.toml si existe en la raíz
    toml_path = base / "pyproject.toml"
    if toml_path.exists():
        lines.append("## File: `pyproject.toml`")
        try:
            content = toml_path.read_text(encoding='utf-8')
        except UnicodeDecodeError:
            lines.append("_[Skipped: binary or non-UTF8 file]_")
        except Exception as e:
            lines.append(f"_[Error al leer el archivo: {e}]_")
        else:
            lines.append("```toml")
            lines.append(content)
            lines.append("```")
            lines.append("")

    output_path = base / output_file
    try:
        output_path.write_text("\n".join(lines), encoding='utf-8')
        print(f"[OK] Código exportado a Markdown en: {output_path}")
    except Exception as e:
        print(f"[ERROR] Error al escribir el archivo de salida: {e}")

# Si se corre el script directamente
if __name__ == "__main__":
    generate_codebase_markdown()

```

## File: `full_codebase.md`
```md
# Estructura de manifiestos de Kubernetes

```
manifest-nyro/
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
├── full_codebase.md
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
└── reranker-service
    ├── configmap.yaml
    ├── deployment.yaml
    └── service.yaml
```

# Codebase: `app`

## File: `api-gateway\configmap.yaml`
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
  GATEWAY_DEFAULT_COMPANY_ID: "d7387dc8-0312-4b1c-97a5-f96f7995f36c"
  GATEWAY_HTTP_CLIENT_TIMEOUT: "30"
  GATEWAY_HTTP_CLIENT_MAX_KEEPALIVE_CONNECTIONS: "100" # Corregido
  GATEWAY_HTTP_CLIENT_MAX_CONNECTIONS: "200"
  GATEWAY_JWT_ALGORITHM: "HS256"
```

## File: `api-gateway\deployment.yaml`
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
          image: ghcr.io/dev-nyro/api-gateway:develop-9bcb370
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
          # --- Secciones livenessProbe y readinessProbe eliminadas ---

      # imagePullSecrets: # Descomentar si GHCR es privado y necesitas un secreto para autenticar
      # - name: ghcr-secret # Nombre del secreto tipo 'kubernetes.io/dockerconfigjson'


```

## File: `api-gateway\service.yaml`
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

## File: `docproc-service\configmap.yaml`
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
  DOCPROC_SUPPORTED_CONTENT_TYPES: '["application/pdf", "application/vnd.openxmlformats-officedocument.wordprocessingml.document", "application/msword", "text/plain", "text/markdown", "text/html"]'
  # DOCPROC_GUNICORN_WORKERS: "4" # Gunicorn workers se configuran en el CMD del Dockerfile, pero podría ser una variable aquí si se desea
```

## File: `docproc-service\deployment.yaml`
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
          image: ghcr.io/dev-nyro/docproc-service:develop-1c92a51
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

## File: `docproc-service\service.yaml`
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

## File: `embedding-service\configmap.yaml`
```yaml
# embedding-service/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: embedding-service-config
  namespace: atenex
data:
  EMBEDDING_LOG_LEVEL: "INFO"
  EMBEDDING_PORT: "8003" # Gunicorn/Uvicorn will listen on this port inside the container

  # FastEmbed Model Configuration
  EMBEDDING_FASTEMBED_MODEL_NAME: "sentence-transformers/all-MiniLM-L6-v2"
  EMBEDDING_EMBEDDING_DIMENSION: "384" # Must match the model

  # Optional: Cache directory for FastEmbed models inside the container
  # If set, consider if this path needs a persistent volume or if ephemeral is fine.
  # For small models, ephemeral might be okay as download is quick.
  # EMBEDDING_FASTEMBED_CACHE_DIR: "/app/.cache/fastembed"
  EMBEDDING_FASTEMBED_CACHE_DIR: "/dev/shm/fastembed_cache" # Use in-memory /dev/shm for faster startup if models are small

  # Optional: Number of threads for FastEmbed tokenization
  # EMBEDDING_FASTEMBED_THREADS: "2" # Example: limit to 2 threads

  # Optional: Max sequence length for the model
  EMBEDDING_FASTEMBED_MAX_LENGTH: "512"
```

## File: `embedding-service\deployment.yaml`
```yaml
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
      volumes:
        - name: dshm # Nombre del volumen
          emptyDir:
            medium: Memory # Usar memoria para este emptyDir
            sizeLimit: 256Mi # Límite de tamaño para el tmpfs
      containers:
        - name: embedding-service
          image: ghcr.io/dev-nyro/embedding-service:develop-1c92a51 # Usa el tag más reciente después de reconstruir con correcciones de código si las hubo
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8003
              protocol: TCP
          envFrom:
            - configMapRef:
                name: embedding-service-config
          volumeMounts:
            - name: dshm # Referencia al volumen definido arriba
              mountPath: /dev/shm # CORREGIDO: Montar en /dev/shm
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "1500m"
              memory: "3Gi"
          readinessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 30 # Dar tiempo suficiente para la descarga y carga del modelo
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

## File: `embedding-service\service.yaml`
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

## File: `export_codebase.py`
```py
from pathlib import Path

# Carpetas que queremos excluir dentro de /app
EXCLUDED_DIRS = {'.git', '__pycache__', '.venv', '.idea', '.mypy_cache', '.vscode', '.github', 'node_modules'}

def build_tree(directory: Path, prefix: str = "") -> list:
    """
    Genera una representación en árbol de la estructura de directorios y archivos,
    excluyendo las carpetas especificadas en EXCLUDED_DIRS.
    """
    # Filtrar y ordenar los elementos del directorio
    entries = sorted(
        [entry for entry in directory.iterdir() if entry.name not in EXCLUDED_DIRS],
        key=lambda e: e.name
    )
    tree_lines = []
    for index, entry in enumerate(entries):
        connector = "└── " if index == len(entries) - 1 else "├── "
        tree_lines.append(prefix + connector + entry.name)
        if entry.is_dir():
            extension = "    " if index == len(entries) - 1 else "│   "
            tree_lines.extend(build_tree(entry, prefix + extension))
    return tree_lines

def generate_codebase_markdown(base_path: str = ".", output_file: str = "full_codebase.md"):
    base = Path(base_path).resolve()
    app_dir = base / "."

    if not app_dir.exists():
        print(f"[ERROR] La carpeta 'app' no existe en {base}")
        return

    lines = []

    # Agregar la estructura de directorios al inicio del Markdown
    lines.append("# Estructura de la Codebase")
    lines.append("")
    lines.append("```")
    lines.append("app/")
    tree_lines = build_tree(app_dir)
    lines.extend(tree_lines)
    lines.append("```")
    lines.append("")

    # Agregar el contenido de la codebase en Markdown
    lines.append("# Codebase: `app`")
    lines.append("")

    # Recorrer solo la carpeta app
    for path in sorted(app_dir.rglob("*")):
        # Ignorar directorios excluidos
        if any(part in EXCLUDED_DIRS for part in path.parts):
            continue

        if path.is_file():
            rel_path = path.relative_to(base)
            lines.append(f"## File: `{rel_path}`")
            try:
                content = path.read_text(encoding='utf-8')
            except UnicodeDecodeError:
                lines.append("_[Skipped: binary or non-UTF8 file]_")
                continue
            except Exception as e:
                lines.append(f"_[Error al leer el archivo: {e}]_")
                continue
            ext = path.suffix.lstrip('.')
            lang = ext if ext else ""
            lines.append(f"```{lang}")
            lines.append(content)
            lines.append("```")
            lines.append("")

    # Agregar pyproject.toml si existe en la raíz
    toml_path = base / "pyproject.toml"
    if toml_path.exists():
        lines.append("## File: `pyproject.toml`")
        try:
            content = toml_path.read_text(encoding='utf-8')
        except UnicodeDecodeError:
            lines.append("_[Skipped: binary or non-UTF8 file]_")
        except Exception as e:
            lines.append(f"_[Error al leer el archivo: {e}]_")
        else:
            lines.append("```toml")
            lines.append(content)
            lines.append("```")
            lines.append("")

    output_path = base / output_file
    try:
        output_path.write_text("\n".join(lines), encoding='utf-8')
        print(f"[OK] Código exportado a Markdown en: {output_path}")
    except Exception as e:
        print(f"[ERROR] Error al escribir el archivo de salida: {e}")

# Si se corre el script directamente
if __name__ == "__main__":
    generate_codebase_markdown()

```

## File: `full_codebase.md`
```md
# Estructura de los manifiestos de Kubernetes

```
MAIFEST/
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
├── full_codebase.md
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
└── reranker-service
    ├── configmap.yaml
    ├── deployment.yaml
    └── service.yaml
```

# Codebase: `app`

## File: `api-gateway\configmap.yaml`
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
  GATEWAY_DEFAULT_COMPANY_ID: "d7387dc8-0312-4b1c-97a5-f96f7995f36c"
  GATEWAY_HTTP_CLIENT_TIMEOUT: "30"
  GATEWAY_HTTP_CLIENT_MAX_KEEPALIVE_CONNECTIONS: "100" # Corregido
  GATEWAY_HTTP_CLIENT_MAX_CONNECTIONS: "200"
  GATEWAY_JWT_ALGORITHM: "HS256"
```

## File: `api-gateway\deployment.yaml`
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
          image: ghcr.io/dev-nyro/api-gateway:develop-9bcb370
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
          # --- Secciones livenessProbe y readinessProbe eliminadas ---

      # imagePullSecrets: # Descomentar si GHCR es privado y necesitas un secreto para autenticar
      # - name: ghcr-secret # Nombre del secreto tipo 'kubernetes.io/dockerconfigjson'


```

## File: `api-gateway\service.yaml`
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

## File: `docproc-service\configmap.yaml`
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
  DOCPROC_SUPPORTED_CONTENT_TYPES: '["application/pdf", "application/vnd.openxmlformats-officedocument.wordprocessingml.document", "application/msword", "text/plain", "text/markdown", "text/html"]'
  # DOCPROC_GUNICORN_WORKERS: "4" # Gunicorn workers se configuran en el CMD del Dockerfile, pero podría ser una variable aquí si se desea
```

## File: `docproc-service\deployment.yaml`
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
          image: ghcr.io/dev-nyro/docproc-service:develop-6be259e
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

## File: `docproc-service\service.yaml`
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

## File: `embedding-service\configmap.yaml`
```yaml
# embedding-service/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: embedding-service-config
  namespace: atenex
data:
  EMBEDDING_LOG_LEVEL: "INFO"
  EMBEDDING_PORT: "8003" # Gunicorn/Uvicorn will listen on this port inside the container

  # FastEmbed Model Configuration
  EMBEDDING_FASTEMBED_MODEL_NAME: "sentence-transformers/all-MiniLM-L6-v2"
  EMBEDDING_EMBEDDING_DIMENSION: "384" # Must match the model

  # Optional: Cache directory for FastEmbed models inside the container
  # If set, consider if this path needs a persistent volume or if ephemeral is fine.
  # For small models, ephemeral might be okay as download is quick.
  # EMBEDDING_FASTEMBED_CACHE_DIR: "/app/.cache/fastembed"
  EMBEDDING_FASTEMBED_CACHE_DIR: "/dev/shm/fastembed_cache" # Use in-memory /dev/shm for faster startup if models are small

  # Optional: Number of threads for FastEmbed tokenization
  # EMBEDDING_FASTEMBED_THREADS: "2" # Example: limit to 2 threads

  # Optional: Max sequence length for the model
  EMBEDDING_FASTEMBED_MAX_LENGTH: "512"
```

## File: `embedding-service\deployment.yaml`
```yaml
# embedding-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: embedding-service-deployment
  namespace: atenex
  labels:
    app: embedding-service
spec:
  replicas: 0
  selector:
    matchLabels:
      app: embedding-service
  template:
    metadata:
      labels:
        app: embedding-service
    spec:
      containers:
        - name: embedding-service
          image: ghcr.io/dev-nyro/embedding-service:develop-d318cb1
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8003 # Must match EMBEDDING_PORT and Gunicorn bind port
              protocol: TCP
          envFrom:
            - configMapRef:
                name: embedding-service-config
          # No secrets needed for this service currently
          #   - secretRef:
          #       name: embedding-service-secrets # If secrets were needed
          resources:
            requests:
              cpu: "500m"   # Request 0.5 vCPU
              memory: "1Gi"  # Request 1 GB RAM (FastEmbed models can be memory intensive)
            limits:
              cpu: "1500m"  # Limit to 1.5 vCPU
              memory: "3Gi" # Limit to 3 GB RAM
          readinessProbe:
            httpGet:
              path: /health
              port: http # Refers to the containerPort named 'http' (8003)
            initialDelaySeconds: 20 # Time for model to load
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 45 # Longer delay for liveness
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 3
      # imagePullSecrets: # Uncomment if using a private Docker registry
      # - name: your-registry-secret

```

## File: `embedding-service\service.yaml`
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

## File: `export_codebase.py`
```py
from pathlib import Path

# Carpetas que queremos excluir dentro de /app
EXCLUDED_DIRS = {'.git', '__pycache__', '.venv', '.idea', '.mypy_cache', '.vscode', '.github', 'node_modules'}

def build_tree(directory: Path, prefix: str = "") -> list:
    """
    Genera una representación en árbol de la estructura de directorios y archivos,
    excluyendo las carpetas especificadas en EXCLUDED_DIRS.
    """
    # Filtrar y ordenar los elementos del directorio
    entries = sorted(
        [entry for entry in directory.iterdir() if entry.name not in EXCLUDED_DIRS],
        key=lambda e: e.name
    )
    tree_lines = []
    for index, entry in enumerate(entries):
        connector = "└── " if index == len(entries) - 1 else "├── "
        tree_lines.append(prefix + connector + entry.name)
        if entry.is_dir():
            extension = "    " if index == len(entries) - 1 else "│   "
            tree_lines.extend(build_tree(entry, prefix + extension))
    return tree_lines

def generate_codebase_markdown(base_path: str = ".", output_file: str = "full_codebase.md"):
    base = Path(base_path).resolve()
    app_dir = base / "."

    if not app_dir.exists():
        print(f"[ERROR] La carpeta 'app' no existe en {base}")
        return

    lines = []

    # Agregar la estructura de directorios al inicio del Markdown
    lines.append("# Estructura de la Codebase")
    lines.append("")
    lines.append("```")
    lines.append("app/")
    tree_lines = build_tree(app_dir)
    lines.extend(tree_lines)
    lines.append("```")
    lines.append("")

    # Agregar el contenido de la codebase en Markdown
    lines.append("# Codebase: `app`")
    lines.append("")

    # Recorrer solo la carpeta app
    for path in sorted(app_dir.rglob("*")):
        # Ignorar directorios excluidos
        if any(part in EXCLUDED_DIRS for part in path.parts):
            continue

        if path.is_file():
            rel_path = path.relative_to(base)
            lines.append(f"## File: `{rel_path}`")
            try:
                content = path.read_text(encoding='utf-8')
            except UnicodeDecodeError:
                lines.append("_[Skipped: binary or non-UTF8 file]_")
                continue
            except Exception as e:
                lines.append(f"_[Error al leer el archivo: {e}]_")
                continue
            ext = path.suffix.lstrip('.')
            lang = ext if ext else ""
            lines.append(f"```{lang}")
            lines.append(content)
            lines.append("```")
            lines.append("")

    # Agregar pyproject.toml si existe en la raíz
    toml_path = base / "pyproject.toml"
    if toml_path.exists():
        lines.append("## File: `pyproject.toml`")
        try:
            content = toml_path.read_text(encoding='utf-8')
        except UnicodeDecodeError:
            lines.append("_[Skipped: binary or non-UTF8 file]_")
        except Exception as e:
            lines.append(f"_[Error al leer el archivo: {e}]_")
        else:
            lines.append("```toml")
            lines.append(content)
            lines.append("```")
            lines.append("")

    output_path = base / output_file
    try:
        output_path.write_text("\n".join(lines), encoding='utf-8')
        print(f"[OK] Código exportado a Markdown en: {output_path}")
    except Exception as e:
        print(f"[ERROR] Error al escribir el archivo de salida: {e}")

# Si se corre el script directamente
if __name__ == "__main__":
    generate_codebase_markdown()

```

## File: `full_codebase.md`
```md
# Estructura de los manifiestos de Kubernetes

```
manifest-nyro/
├── MinIO
│   ├── deployment.yaml
│   ├── persistent-volume-claim.yaml
│   ├── secret.yaml
│   └── service.yaml
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
└── reranker-service
    ├── configmap.yaml
    ├── deployment.yaml
    └── service.yaml
```

# Codebase: `app`

## File: `api-gateway\configmap.yaml`
```yaml
# api-gateway/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-gateway-config
  namespace: atenex # Asegúrate que el namespace sea correcto
data:
  # General
  GATEWAY_LOG_LEVEL: "INFO" # O "DEBUG" para más detalle

  # URLs de los servicios downstream (usando nombres de Service de K8s)
  # Sin cambios aquí, ya apuntan a servicios internos
  GATEWAY_INGEST_SERVICE_URL: "http://ingest-api-service.atenex.svc.cluster.local:80"
  GATEWAY_QUERY_SERVICE_URL: "http://query-service.atenex.svc.cluster.local:80"
  # GATEWAY_AUTH_SERVICE_URL: "http://auth-service.atenex.svc.cluster.local:80" # Descomentar si se usa

  # --- Configuración PostgreSQL INTERNA ---
  # Apunta al Service de Kubernetes para PostgreSQL dentro del mismo namespace
  GATEWAY_POSTGRES_SERVER: "postgresql-service.atenex.svc.cluster.local"
  # Puerto estándar de PostgreSQL
  GATEWAY_POSTGRES_PORT: "5432"
  # Usuario de la base de datos interna (asumiendo 'postgres', cambia si es diferente)
  GATEWAY_POSTGRES_USER: "postgres"
  # Nombre de la base de datos interna
  GATEWAY_POSTGRES_DB: "atenex"
  # GATEWAY_POSTGRES_PASSWORD se define en el Secret 'api-gateway-secrets'
  # ----------------------------------------

  # --- Eliminada variable específica de Supabase ---
  # GATEWAY_SUPABASE_URL: "..." # Ya no es necesaria

  # --- Company ID por defecto ---
  # ¡REEMPLAZA con un UUID válido de una compañía existente en tu DB interna!
  GATEWAY_DEFAULT_COMPANY_ID: "d7387dc8-0312-4b1c-97a5-f96f7995f36c" # Mantén tu valor real

  # HTTP Client settings (Sin cambios respecto a tu versión anterior)
  GATEWAY_HTTP_CLIENT_TIMEOUT: "30"
  # Nota: Asegúrate que tu código usa 'GATEWAY_HTTP_CLIENT_MAX_KEEPALIVE_CONNECTIONS' si esa es la variable correcta
  GATEWAY_HTTP_CLIENT_MAX_KEEPALIAS_CONNECTIONS: "100" # O renombra a KEEPALIVE si es un typo
  GATEWAY_HTTP_CLIENT_MAX_CONNECTIONS: "200"

  # JWT Algorithm (El Secret va en un objeto Secret)
  GATEWAY_JWT_ALGORITHM: "HS256"

  # URLs externas opcionales (Manténlas si tu aplicación las usa)
  # GATEWAY_VERCEL_FRONTEND_URL: "https://atenex-frontend.vercel.app"
  # NGROK_URL: "https://tu-id.ngrok-free.app"
```

## File: `api-gateway\deployment.yaml`
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
          image: ghcr.io/dev-nyro/api-gateway:develop-9bcb370
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
          # --- Secciones livenessProbe y readinessProbe eliminadas ---

      # imagePullSecrets: # Descomentar si GHCR es privado y necesitas un secreto para autenticar
      # - name: ghcr-secret # Nombre del secreto tipo 'kubernetes.io/dockerconfigjson'


```

## File: `api-gateway\service.yaml`
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

## File: `docproc-service\configmap.yaml`
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
  DOCPROC_SUPPORTED_CONTENT_TYPES: '["application/pdf", "application/vnd.openxmlformats-officedocument.wordprocessingml.document", "application/msword", "text/plain", "text/markdown", "text/html"]'
  # DOCPROC_GUNICORN_WORKERS: "4" # Gunicorn workers se configuran en el CMD del Dockerfile, pero podría ser una variable aquí si se desea
```

## File: `docproc-service\deployment.yaml`
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
  replicas: 2 # Ajustar según la carga esperada
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
          image: atenex/docproc-service:latest # Reemplazar con la URL/tag de tu imagen
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

## File: `docproc-service\service.yaml`
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

## File: `embedding-service\configmap.yaml`
```yaml
# embedding-service/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: embedding-service-config
  namespace: atenex
data:
  EMBEDDING_LOG_LEVEL: "INFO"
  EMBEDDING_PORT: "8003" # Gunicorn/Uvicorn will listen on this port inside the container

  # FastEmbed Model Configuration
  EMBEDDING_FASTEMBED_MODEL_NAME: "sentence-transformers/all-MiniLM-L6-v2"
  EMBEDDING_EMBEDDING_DIMENSION: "384" # Must match the model

  # Optional: Cache directory for FastEmbed models inside the container
  # If set, consider if this path needs a persistent volume or if ephemeral is fine.
  # For small models, ephemeral might be okay as download is quick.
  # EMBEDDING_FASTEMBED_CACHE_DIR: "/app/.cache/fastembed"
  EMBEDDING_FASTEMBED_CACHE_DIR: "/dev/shm/fastembed_cache" # Use in-memory /dev/shm for faster startup if models are small

  # Optional: Number of threads for FastEmbed tokenization
  # EMBEDDING_FASTEMBED_THREADS: "2" # Example: limit to 2 threads

  # Optional: Max sequence length for the model
  EMBEDDING_FASTEMBED_MAX_LENGTH: "512"
```

## File: `embedding-service\deployment.yaml`
```yaml
# embedding-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: embedding-service-deployment
  namespace: atenex
  labels:
    app: embedding-service
spec:
  replicas: 1 # Start with 1, scale as needed
  selector:
    matchLabels:
      app: embedding-service
  template:
    metadata:
      labels:
        app: embedding-service
    spec:
      containers:
        - name: embedding-service
          image: ghcr.io/dev-nyro/embedding-service:latest # Placeholder, CI/CD will update this
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8003 # Must match EMBEDDING_PORT and Gunicorn bind port
              protocol: TCP
          envFrom:
            - configMapRef:
                name: embedding-service-config
          # No secrets needed for this service currently
          #   - secretRef:
          #       name: embedding-service-secrets # If secrets were needed
          resources:
            requests:
              cpu: "500m"   # Request 0.5 vCPU
              memory: "1Gi"  # Request 1 GB RAM (FastEmbed models can be memory intensive)
            limits:
              cpu: "1500m"  # Limit to 1.5 vCPU
              memory: "3Gi" # Limit to 3 GB RAM
          readinessProbe:
            httpGet:
              path: /health
              port: http # Refers to the containerPort named 'http' (8003)
            initialDelaySeconds: 20 # Time for model to load
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 45 # Longer delay for liveness
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 3
      # imagePullSecrets: # Uncomment if using a private Docker registry
      # - name: your-registry-secret
```

## File: `embedding-service\service.yaml`
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

## File: `export_codebase.py`
```py
from pathlib import Path

# Carpetas que queremos excluir dentro de /app
EXCLUDED_DIRS = {'.git', '__pycache__', '.venv', '.idea', '.mypy_cache', '.vscode', '.github', 'node_modules'}

def build_tree(directory: Path, prefix: str = "") -> list:
    """
    Genera una representación en árbol de la estructura de directorios y archivos,
    excluyendo las carpetas especificadas en EXCLUDED_DIRS.
    """
    # Filtrar y ordenar los elementos del directorio
    entries = sorted(
        [entry for entry in directory.iterdir() if entry.name not in EXCLUDED_DIRS],
        key=lambda e: e.name
    )
    tree_lines = []
    for index, entry in enumerate(entries):
        connector = "└── " if index == len(entries) - 1 else "├── "
        tree_lines.append(prefix + connector + entry.name)
        if entry.is_dir():
            extension = "    " if index == len(entries) - 1 else "│   "
            tree_lines.extend(build_tree(entry, prefix + extension))
    return tree_lines

def generate_codebase_markdown(base_path: str = ".", output_file: str = "full_codebase.md"):
    base = Path(base_path).resolve()
    app_dir = base / "."

    if not app_dir.exists():
        print(f"[ERROR] La carpeta 'app' no existe en {base}")
        return

    lines = []

    # Agregar la estructura de directorios al inicio del Markdown
    lines.append("# Estructura de la Codebase")
    lines.append("")
    lines.append("```")
    lines.append("app/")
    tree_lines = build_tree(app_dir)
    lines.extend(tree_lines)
    lines.append("```")
    lines.append("")

    # Agregar el contenido de la codebase en Markdown
    lines.append("# Codebase: `app`")
    lines.append("")

    # Recorrer solo la carpeta app
    for path in sorted(app_dir.rglob("*")):
        # Ignorar directorios excluidos
        if any(part in EXCLUDED_DIRS for part in path.parts):
            continue

        if path.is_file():
            rel_path = path.relative_to(base)
            lines.append(f"## File: `{rel_path}`")
            try:
                content = path.read_text(encoding='utf-8')
            except UnicodeDecodeError:
                lines.append("_[Skipped: binary or non-UTF8 file]_")
                continue
            except Exception as e:
                lines.append(f"_[Error al leer el archivo: {e}]_")
                continue
            ext = path.suffix.lstrip('.')
            lang = ext if ext else ""
            lines.append(f"```{lang}")
            lines.append(content)
            lines.append("```")
            lines.append("")

    # Agregar pyproject.toml si existe en la raíz
    toml_path = base / "pyproject.toml"
    if toml_path.exists():
        lines.append("## File: `pyproject.toml`")
        try:
            content = toml_path.read_text(encoding='utf-8')
        except UnicodeDecodeError:
            lines.append("_[Skipped: binary or non-UTF8 file]_")
        except Exception as e:
            lines.append(f"_[Error al leer el archivo: {e}]_")
        else:
            lines.append("```toml")
            lines.append(content)
            lines.append("```")
            lines.append("")

    output_path = base / output_file
    try:
        output_path.write_text("\n".join(lines), encoding='utf-8')
        print(f"[OK] Código exportado a Markdown en: {output_path}")
    except Exception as e:
        print(f"[ERROR] Error al escribir el archivo de salida: {e}")

# Si se corre el script directamente
if __name__ == "__main__":
    generate_codebase_markdown()

```

## File: `ingest-service\configmap.yaml`
```yaml
#ingest-service/configmap.yaml
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
  # INGEST_POSTGRES_PASSWORD -> Secret

  # Milvus
  INGEST_MILVUS_URI: "http://milvus-standalone.atenex.svc.cluster.local:19530" # Nombre de servicio actualizado
  INGEST_MILVUS_COLLECTION_NAME: "document_chunks_minilm"
  INGEST_MILVUS_GRPC_TIMEOUT: "10"

  # Google Cloud Storage
  INGEST_GCS_BUCKET_NAME: "atenex"

  # Embedding Settings
  # INGEST_EMBEDDING_MODEL_ID: "sentence-transformers/all-MiniLM-L6-v2" # Eliminado
  INGEST_EMBEDDING_DIMENSION: "384" # Mantenido para el esquema de Milvus

  # Tokenizer Settings
  INGEST_TIKTOKEN_ENCODING_NAME: "cl100k_base"

  # URLs for dependent services
  INGEST_EMBEDDING_SERVICE_URL: "http://embedding-service.atenex.svc.cluster.local:8003/api/v1/embed"
  INGEST_DOCPROC_SERVICE_URL: "http://docproc-service.atenex.svc.cluster.local:8005/api/v1/process"

  # Splitter Settings - Eliminados
  # INGEST_SPLITTER_CHUNK_SIZE: "1000"
  # INGEST_SPLITTER_CHUNK_OVERLAP: "250"
```

## File: `ingest-service\deployment-api.yaml`
```yaml
# ingest-service/deployment-api.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingest-api-deployment
  namespace: atenex
  labels:
    app: ingest-api
spec:
  replicas: 1 # Puedes aumentar si la API recibe muchas peticiones simultáneas
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
          # Asegúrate que la imagen es la correcta y está actualizada
          image: ghcr.io/dev-nyro/ingest-service:develop-fe1af29
          imagePullPolicy: Always # O IfNotPresent
          ports:
            - name: http
              containerPort: 8000 # Puerto que expone Gunicorn/Uvicorn
              protocol: TCP
          command: ["gunicorn"]
          args: [
              "-k", "uvicorn.workers.UvicornWorker",
              "-w", "4",                             # Número de workers (ajustar según CPU/memoria disponible)
              "-b", "0.0.0.0:8000",
              "-t", "120",                           # Timeout extendido
              "--log-level", "info",                 # Nivel de log para Gunicorn/Uvicorn
              "app.main:app"
            ]
          envFrom:
            - configMapRef:
                name: ingest-service-config
            - secretRef:
                name: ingest-service-secrets
          env:
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-secrets/key.json" # Ruta donde se montará la clave
          volumeMounts:
            - name: gcp-key-volume
              mountPath: "/etc/gcp-secrets" # Directorio donde montar
              readOnly: true
          # ***** Recursos - Mantener como estaban (sin cambios específicos para GCS) *****
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1500m"
              memory: "2Gi"
          # Probes eliminados según solicitud previa
      # --- GCS Authentication: Not directly configured here ---
      # Assumes Workload Identity is configured for the service account running this pod, OR
      # GOOGLE_APPLICATION_CREDENTIALS env var is set via secrets/volumeMounts if using key files.
      # serviceAccountName: your-ksa-with-workload-identity # Example if using WI

      # imagePullSecrets: # Descomentar si usas registro privado
      # - name: your-registry-secret
      volumes:
        - name: gcp-key-volume
          secret:
            secretName: gcs-worker-sa-key # Nombre del secreto creado
            items:
              - key: key.json # Nombre del archivo dentro del secreto
                path: key.json # Nombre que tendrá el archivo en mountPath
```

## File: `ingest-service\deployment-worker.yaml`
```yaml
# ingest-service/deployment-worker.yaml
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
      # No se necesita serviceAccountName si usas claves
      containers:
        - name: ingest-worker
          # Ensure this image includes CPU ONNX runtime and correct dependencies
          # ACTUALIZA ESTA ETIQUETA con la imagen construida DESPUÉS de aplicar los cambios de código
          image: ghcr.io/dev-nyro/ingest-service:develop-fe1af29
          imagePullPolicy: Always
          command: ["celery"]
          args:
            [
              "-A",
              "app.tasks.celery_app",
              "worker",
              "--loglevel=INFO",
              "-P",
              "prefork",
              "-c",
              "4",
            ]
          envFrom:
            - configMapRef:
                name: ingest-service-config
            - secretRef:
                name: ingest-service-secrets
          env:
            # --- Establecer variable de entorno para la clave ---
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-secrets/key.json" # Ruta donde se montará la clave
            # -------------------------------------------------
          volumeMounts:
            # --- Montar el secreto en el contenedor ---
            - name: gcp-key-volume
              mountPath: "/etc/gcp-secrets" # Directorio donde montar
              readOnly: true
            # ------------------------------------------
          resources:
            requests:
              cpu: "1000m"
              memory: "3Gi"
            limits:
              cpu: "4000m"
              memory: "8Gi"
      volumes:
        # --- Definir el volumen desde el secreto ---
        - name: gcp-key-volume
          secret:
            secretName: gcs-worker-sa-key # Nombre del secreto creado
            items:
            - key: key.json # Nombre del archivo DENTRO del secreto
              path: key.json # Nombre que tendrá el archivo en mountPath
        # ---------------------------------------
      # imagePullSecrets:
      # - name: your-registry-secret

```

## File: `ingest-service\service-api.yaml`
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

## File: `MinIO\deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: atenex
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
        - name: minio
          image: minio/minio:RELEASE.2023-03-20T20-16-18Z
          args:
            - server
            - /data
            - --console-address
            - ":9001"
          envFrom:
            - secretRef:
                name: minio-secrets
          ports:
            - containerPort: 9000
              name: api
            - containerPort: 9001
              name: console
          resources:
            limits:
              cpu: "1"
              memory: "2Gi"
            requests:
              cpu: "500m"
              memory: "1Gi"
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: minio-pvc
```

## File: `MinIO\persistent-volume-claim.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-pvc
  namespace: atenex
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## File: `MinIO\secret.yaml`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: minio-secrets
  namespace: atenex
stringData:
  MINIO_ROOT_USER: minioadmin
  MINIO_ROOT_PASSWORD: minioadmin
```

## File: `MinIO\service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: minio-service
  namespace: atenex
spec:
  type: NodePort
  ports:
    - name: api
      port: 9000
      targetPort: 9000
      nodePort: 30900
    - name: console
      port: 9001
      targetPort: 9001
      nodePort: 30901
  selector:
    app: minio
```

## File: `postgresql\persistent-volume-claim.yaml`
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

## File: `postgresql\service.yaml`
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

## File: `postgresql\statefulset.yaml`
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

## File: `query-service\configmap.yaml`
```yaml
# query-service/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: query-service-config
  namespace: atenex
data:
  # General
  QUERY_LOG_LEVEL: "INFO" # O "DEBUG" para más detalle

  # PostgreSQL (Sin cambios)
  QUERY_POSTGRES_SERVER: "postgresql-service.atenex.svc.cluster.local"
  QUERY_POSTGRES_PORT: "5432"
  QUERY_POSTGRES_USER: "postgres"
  QUERY_POSTGRES_DB: "atenex"
  # QUERY_POSTGRES_PASSWORD -> Proveniente de Secret

  # Milvus (Sin cambios)
  QUERY_MILVUS_URI: "http://milvus.atenex.svc.cluster.local:19530" # Asumiendo que Milvus está en el mismo namespace
  QUERY_MILVUS_COLLECTION_NAME: "document_chunks_minilm"
  QUERY_MILVUS_EMBEDDING_FIELD: "embedding"
  QUERY_MILVUS_CONTENT_FIELD: "content"
  QUERY_MILVUS_COMPANY_ID_FIELD: "company_id"
  QUERY_MILVUS_DOCUMENT_ID_FIELD: "document_id"
  QUERY_MILVUS_FILENAME_FIELD: "file_name"
  QUERY_MILVUS_GRPC_TIMEOUT: "15"
  # QUERY_MILVUS_METADATA_FIELDS: (...)

  # Embedding Settings
  QUERY_EMBEDDING_DIMENSION: "384" # Dimensión esperada/configurada, para Milvus y validación
  # --- NUEVA CONFIGURACIÓN ---
  QUERY_EMBEDDING_SERVICE_URL: "http://embedding-service.atenex.svc.cluster.local:8003" # URL del servicio de embedding
  # --- REMOVED FastEmbed local settings ---
  # QUERY_FASTEMBED_MODEL_NAME: "sentence-transformers/all-MiniLM-L6-v2"
  # QUERY_FASTEMBED_QUERY_PREFIX: "query: "


  # Gemini LLM Settings
  QUERY_GEMINI_MODEL_NAME: "gemini-1.5-flash-latest" # Corregido, no "gemini-2.5..."
  # QUERY_GEMINI_API_KEY -> Proveniente de Secret

  # RAG Pipeline Control & Parameters
  QUERY_RETRIEVER_TOP_K: "100"
  QUERY_MAX_CONTEXT_CHUNKS: "75"
  QUERY_MAX_PROMPT_TOKENS: "500000"

  QUERY_BM25_ENABLED: "true"
  QUERY_RERANKER_ENABLED: "true"
  QUERY_RERANKER_MODEL_NAME: "BAAI/bge-reranker-base"
  QUERY_DIVERSITY_FILTER_ENABLED: "true" # Habilitar MMR por defecto
  QUERY_DIVERSITY_LAMBDA: "0.5" # Lambda para MMR
  # QUERY_HYBRID_FUSION_ALPHA: "0.5" # (Actualmente no usado por RRF)

  # Prompt Templates (Sin cambios aquí, usan los defaults de config.py)
  # QUERY_RAG_PROMPT_TEMPLATE: |
  #   ...
  # QUERY_GENERAL_PROMPT_TEMPLATE: |
  #   ...
```

## File: `query-service\deployment.yaml`
```yaml
# query-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: query-service-deployment
  namespace: atenex # Asegurar namespace correcto
  labels:
    app: query-service
spec:
  replicas: 1 # Empezar con 1, escalar según necesidad
  selector:
    matchLabels:
      app: query-service
  template:
    metadata:
      labels:
        app: query-service
    spec:
      containers:
        - name: query-service # Nombre del contenedor
          # La imagen será actualizada por el pipeline CI/CD (este es un placeholder)
          image: ghcr.io/dev-nyro/query-service:develop-e122a62
          imagePullPolicy: Always # O IfNotPresent si prefieres usar caché local en dev
          ports:
            - name: http
              containerPort: 8001 # Puerto interno definido en Dockerfile/Config
              protocol: TCP
          # Comando Gunicorn ajustado
          command: ["gunicorn", "-k", "uvicorn.workers.UvicornWorker"]
          args: [
              "-w", "2", # Número de workers (ajustar según CPU/Memoria)
              "-b", "0.0.0.0:8001", # Vincular al puerto interno
              "-t", "120", # Worker timeout
              # "--log-level", "$(QUERY_LOG_LEVEL)", # Heredar nivel de log (opcional)
              "app.main:app"
            ]
          envFrom:
            - configMapRef:
                name: query-service-config
            - secretRef:
                name: query-service-secrets # Asegurar que el Secret existe
          resources: # Aumentado para mejorar velocidad de arranque y performance
            requests:
              cpu: "1500m" # 1.5 vCPU
              memory: "3Gi"
            limits:
              cpu: "3000m" # 3 vCPU
              memory: "8Gi"

          # --- Sondas eliminadas según solicitud ---
          # livenessProbe: ...
          # readinessProbe: ...

      # Descomentar si se usa registro privado
      # imagePullSecrets:
      # - name: your-registry-secret
```

## File: `query-service\service.yaml`
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

## File: `README.md`
```md
# manifests-nyro
Repositorio de los manifiestos de Nyro B2B

```

## File: `reranker-service\configmap.yaml`
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
  RERANKER_BATCH_SIZE: "32" # Batch size for reranker predictions
  RERANKER_MAX_SEQ_LENGTH: "512" # Max sequence length for the model
  # RERANKER_PORT is typically set in the Deployment's container spec or defaults in Dockerfile
  # RERANKER_WORKERS is typically set in the Deployment's container spec or defaults in Dockerfile
```

## File: `reranker-service\deployment.yaml`
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
  replicas: 1 # Start with 1, scale based on load
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
          image: ghcr.io/dev-nyro/reranker-service:latest # Placeholder: replace with actual image path and tag
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

## File: `reranker-service\service.yaml`
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

```

## File: `ingest-service\configmap.yaml`
```yaml
#ingest-service/configmap.yaml
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
  # INGEST_POSTGRES_PASSWORD -> Secret

  # Milvus
  INGEST_MILVUS_URI: "http://milvus-standalone.atenex.svc.cluster.local:19530" # Nombre de servicio actualizado
  INGEST_MILVUS_COLLECTION_NAME: "document_chunks_minilm"
  INGEST_MILVUS_GRPC_TIMEOUT: "10"

  # Google Cloud Storage
  INGEST_GCS_BUCKET_NAME: "atenex"

  # Embedding Settings
  # INGEST_EMBEDDING_MODEL_ID: "sentence-transformers/all-MiniLM-L6-v2" # Eliminado
  INGEST_EMBEDDING_DIMENSION: "384" # Mantenido para el esquema de Milvus

  # Tokenizer Settings
  INGEST_TIKTOKEN_ENCODING_NAME: "cl100k_base"

  # URLs for dependent services
  INGEST_EMBEDDING_SERVICE_URL: "http://embedding-service.atenex.svc.cluster.local:8003/api/v1/embed"
  INGEST_DOCPROC_SERVICE_URL: "http://docproc-service.atenex.svc.cluster.local:8005/api/v1/process"

  # Splitter Settings - Eliminados
  # INGEST_SPLITTER_CHUNK_SIZE: "1000"
  # INGEST_SPLITTER_CHUNK_OVERLAP: "250"
```

## File: `ingest-service\deployment-api.yaml`
```yaml
# ingest-service/deployment-api.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingest-api-deployment
  namespace: atenex
  labels:
    app: ingest-api
spec:
  replicas: 1 # Puedes aumentar si la API recibe muchas peticiones simultáneas
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
          # Asegúrate que la imagen es la correcta y está actualizada
          image: ghcr.io/dev-nyro/ingest-service:develop-d318cb1
          imagePullPolicy: Always # O IfNotPresent
          ports:
            - name: http
              containerPort: 8000 # Puerto que expone Gunicorn/Uvicorn
              protocol: TCP
          command: ["gunicorn"]
          args: [
              "-k", "uvicorn.workers.UvicornWorker",
              "-w", "4",                             # Número de workers (ajustar según CPU/memoria disponible)
              "-b", "0.0.0.0:8000",
              "-t", "120",                           # Timeout extendido
              "--log-level", "info",                 # Nivel de log para Gunicorn/Uvicorn
              "app.main:app"
            ]
          envFrom:
            - configMapRef:
                name: ingest-service-config
            - secretRef:
                name: ingest-service-secrets
          env:
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-secrets/key.json" # Ruta donde se montará la clave
          volumeMounts:
            - name: gcp-key-volume
              mountPath: "/etc/gcp-secrets" # Directorio donde montar
              readOnly: true
          # ***** Recursos - Mantener como estaban (sin cambios específicos para GCS) *****
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1500m"
              memory: "2Gi"
          # Probes eliminados según solicitud previa
      # --- GCS Authentication: Not directly configured here ---
      # Assumes Workload Identity is configured for the service account running this pod, OR
      # GOOGLE_APPLICATION_CREDENTIALS env var is set via secrets/volumeMounts if using key files.
      # serviceAccountName: your-ksa-with-workload-identity # Example if using WI

      # imagePullSecrets: # Descomentar si usas registro privado
      # - name: your-registry-secret
      volumes:
        - name: gcp-key-volume
          secret:
            secretName: gcs-worker-sa-key # Nombre del secreto creado
            items:
              - key: key.json # Nombre del archivo dentro del secreto
                path: key.json # Nombre que tendrá el archivo en mountPath
```

## File: `ingest-service\deployment-worker.yaml`
```yaml
# ingest-service/deployment-worker.yaml
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
      # No se necesita serviceAccountName si usas claves
      containers:
        - name: ingest-worker
          # Ensure this image includes CPU ONNX runtime and correct dependencies
          # ACTUALIZA ESTA ETIQUETA con la imagen construida DESPUÉS de aplicar los cambios de código
          image: ghcr.io/dev-nyro/ingest-service:develop-d318cb1
          imagePullPolicy: Always
          command: ["celery"]
          args:
            [
              "-A",
              "app.tasks.celery_app",
              "worker",
              "--loglevel=INFO",
              "-P",
              "prefork",
              "-c",
              "4",
            ]
          envFrom:
            - configMapRef:
                name: ingest-service-config
            - secretRef:
                name: ingest-service-secrets
          env:
            # --- Establecer variable de entorno para la clave ---
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-secrets/key.json" # Ruta donde se montará la clave
            # -------------------------------------------------
          volumeMounts:
            # --- Montar el secreto en el contenedor ---
            - name: gcp-key-volume
              mountPath: "/etc/gcp-secrets" # Directorio donde montar
              readOnly: true
            # ------------------------------------------
          resources:
            requests:
              cpu: "1000m"
              memory: "3Gi"
            limits:
              cpu: "4000m"
              memory: "8Gi"
      volumes:
        # --- Definir el volumen desde el secreto ---
        - name: gcp-key-volume
          secret:
            secretName: gcs-worker-sa-key # Nombre del secreto creado
            items:
            - key: key.json # Nombre del archivo DENTRO del secreto
              path: key.json # Nombre que tendrá el archivo en mountPath
        # ---------------------------------------
      # imagePullSecrets:
      # - name: your-registry-secret

```

## File: `ingest-service\service-api.yaml`
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

## File: `postgresql\persistent-volume-claim.yaml`
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

## File: `postgresql\service.yaml`
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

## File: `postgresql\statefulset.yaml`
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

## File: `query-service\configmap.yaml`
```yaml
# query-service/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: query-service-config
  namespace: atenex
data:
  # General
  QUERY_LOG_LEVEL: "INFO" # O "DEBUG" para más detalle

  # PostgreSQL (Sin cambios)
  QUERY_POSTGRES_SERVER: "postgresql-service.atenex.svc.cluster.local"
  QUERY_POSTGRES_PORT: "5432"
  QUERY_POSTGRES_USER: "postgres"
  QUERY_POSTGRES_DB: "atenex"
  # QUERY_POSTGRES_PASSWORD -> Proveniente de Secret

  # Milvus (Sin cambios)
  QUERY_MILVUS_URI: "http://milvus.atenex.svc.cluster.local:19530" # Asumiendo que Milvus está en el mismo namespace
  QUERY_MILVUS_COLLECTION_NAME: "document_chunks_minilm"
  QUERY_MILVUS_EMBEDDING_FIELD: "embedding"
  QUERY_MILVUS_CONTENT_FIELD: "content"
  QUERY_MILVUS_COMPANY_ID_FIELD: "company_id"
  QUERY_MILVUS_DOCUMENT_ID_FIELD: "document_id"
  QUERY_MILVUS_FILENAME_FIELD: "file_name"
  QUERY_MILVUS_GRPC_TIMEOUT: "15"
  # QUERY_MILVUS_METADATA_FIELDS: (...)

  # Embedding Settings
  QUERY_EMBEDDING_DIMENSION: "384" # Dimensión esperada/configurada, para Milvus y validación
  # --- NUEVA CONFIGURACIÓN ---
  QUERY_EMBEDDING_SERVICE_URL: "http://embedding-service.atenex.svc.cluster.local:8003" # URL del servicio de embedding
  # --- REMOVED FastEmbed local settings ---
  # QUERY_FASTEMBED_MODEL_NAME: "sentence-transformers/all-MiniLM-L6-v2"
  # QUERY_FASTEMBED_QUERY_PREFIX: "query: "


  # Gemini LLM Settings
  QUERY_GEMINI_MODEL_NAME: "gemini-1.5-flash-latest" # Corregido, no "gemini-2.5..."
  # QUERY_GEMINI_API_KEY -> Proveniente de Secret

  # RAG Pipeline Control & Parameters
  QUERY_RETRIEVER_TOP_K: "100"
  QUERY_MAX_CONTEXT_CHUNKS: "75"
  QUERY_MAX_PROMPT_TOKENS: "500000"

  QUERY_BM25_ENABLED: "true"
  QUERY_RERANKER_ENABLED: "true"
  QUERY_RERANKER_MODEL_NAME: "BAAI/bge-reranker-base"
  QUERY_DIVERSITY_FILTER_ENABLED: "true" # Habilitar MMR por defecto
  QUERY_DIVERSITY_LAMBDA: "0.5" # Lambda para MMR
  # QUERY_HYBRID_FUSION_ALPHA: "0.5" # (Actualmente no usado por RRF)

  # Prompt Templates (Sin cambios aquí, usan los defaults de config.py)
  # QUERY_RAG_PROMPT_TEMPLATE: |
  #   ...
  # QUERY_GENERAL_PROMPT_TEMPLATE: |
  #   ...
```

## File: `query-service\deployment.yaml`
```yaml
# query-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: query-service-deployment
  namespace: atenex # Asegurar namespace correcto
  labels:
    app: query-service
spec:
  replicas: 1 # Empezar con 1, escalar según necesidad
  selector:
    matchLabels:
      app: query-service
  template:
    metadata:
      labels:
        app: query-service
    spec:
      containers:
        - name: query-service # Nombre del contenedor
          # La imagen será actualizada por el pipeline CI/CD (este es un placeholder)
          image: ghcr.io/dev-nyro/query-service:develop-d318cb1
          imagePullPolicy: Always # O IfNotPresent si prefieres usar caché local en dev
          ports:
            - name: http
              containerPort: 8001 # Puerto interno definido en Dockerfile/Config
              protocol: TCP
          # Comando Gunicorn ajustado
          command: ["gunicorn", "-k", "uvicorn.workers.UvicornWorker"]
          args: [
              "-w", "2", # Número de workers (ajustar según CPU/Memoria)
              "-b", "0.0.0.0:8001", # Vincular al puerto interno
              "-t", "120", # Worker timeout
              # "--log-level", "$(QUERY_LOG_LEVEL)", # Heredar nivel de log (opcional)
              "app.main:app"
            ]
          envFrom:
            - configMapRef:
                name: query-service-config
            - secretRef:
                name: query-service-secrets # Asegurar que el Secret existe
          resources: # Aumentado para mejorar velocidad de arranque y performance
            requests:
              cpu: "1500m" # 1.5 vCPU
              memory: "3Gi"
            limits:
              cpu: "3000m" # 3 vCPU
              memory: "8Gi"

          # --- Sondas eliminadas según solicitud ---
          # livenessProbe: ...
          # readinessProbe: ...

      # Descomentar si se usa registro privado
      # imagePullSecrets:
      # - name: your-registry-secret
```

## File: `query-service\service.yaml`
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

## File: `README.md`
```md
# manifests-nyro
Repositorio de los manifiestos de Nyro B2B

```

## File: `reranker-service\configmap.yaml`
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
  RERANKER_BATCH_SIZE: "32" # Batch size for reranker predictions
  RERANKER_MAX_SEQ_LENGTH: "512" # Max sequence length for the model
  # RERANKER_PORT is typically set in the Deployment's container spec or defaults in Dockerfile
  # RERANKER_WORKERS is typically set in the Deployment's container spec or defaults in Dockerfile
```

## File: `reranker-service\deployment.yaml`
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
          image: ghcr.io/dev-nyro/reranker-service:latest # Placeholder: replace with actual image path and tag
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

## File: `reranker-service\service.yaml`
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

```

## File: `ingest-service\configmap.yaml`
```yaml
#ingest-service/configmap.yaml
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
  # INGEST_POSTGRES_PASSWORD -> Secret

  # Milvus
  INGEST_MILVUS_URI: "http://milvus-standalone.atenex.svc.cluster.local:19530" # Nombre de servicio actualizado
  INGEST_MILVUS_COLLECTION_NAME: "document_chunks_minilm"
  INGEST_MILVUS_GRPC_TIMEOUT: "10"

  # Google Cloud Storage
  INGEST_GCS_BUCKET_NAME: "atenex"

  # Embedding Settings
  # INGEST_EMBEDDING_MODEL_ID: "sentence-transformers/all-MiniLM-L6-v2" # Eliminado
  INGEST_EMBEDDING_DIMENSION: "384" # Mantenido para el esquema de Milvus

  # Tokenizer Settings
  INGEST_TIKTOKEN_ENCODING_NAME: "cl100k_base"

  # URLs for dependent services
  INGEST_EMBEDDING_SERVICE_URL: "http://embedding-service.atenex.svc.cluster.local:8003/api/v1/embed"
  INGEST_DOCPROC_SERVICE_URL: "http://docproc-service.atenex.svc.cluster.local:8005/api/v1/process"

  # Splitter Settings - Eliminados
  # INGEST_SPLITTER_CHUNK_SIZE: "1000"
  # INGEST_SPLITTER_CHUNK_OVERLAP: "250"
```

## File: `ingest-service\deployment-api.yaml`
```yaml
# ingest-service/deployment-api.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingest-api-deployment
  namespace: atenex
  labels:
    app: ingest-api
spec:
  replicas: 1 # Puedes aumentar si la API recibe muchas peticiones simultáneas
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
          # Asegúrate que la imagen es la correcta y está actualizada
          image: ghcr.io/dev-nyro/ingest-service:develop-23382da
          imagePullPolicy: Always # O IfNotPresent
          ports:
            - name: http
              containerPort: 8000 # Puerto que expone Gunicorn/Uvicorn
              protocol: TCP
          command: ["gunicorn"]
          args: [
              "-k", "uvicorn.workers.UvicornWorker",
              "-w", "4",                             # Número de workers (ajustar según CPU/memoria disponible)
              "-b", "0.0.0.0:8000",
              "-t", "120",                           # Timeout extendido
              "--log-level", "info",                 # Nivel de log para Gunicorn/Uvicorn
              "app.main:app"
            ]
          envFrom:
            - configMapRef:
                name: ingest-service-config
            - secretRef:
                name: ingest-service-secrets
          env:
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-secrets/key.json" # Ruta donde se montará la clave
          volumeMounts:
            - name: gcp-key-volume
              mountPath: "/etc/gcp-secrets" # Directorio donde montar
              readOnly: true
          # ***** Recursos - Mantener como estaban (sin cambios específicos para GCS) *****
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1500m"
              memory: "2Gi"
          # Probes eliminados según solicitud previa
      # --- GCS Authentication: Not directly configured here ---
      # Assumes Workload Identity is configured for the service account running this pod, OR
      # GOOGLE_APPLICATION_CREDENTIALS env var is set via secrets/volumeMounts if using key files.
      # serviceAccountName: your-ksa-with-workload-identity # Example if using WI

      # imagePullSecrets: # Descomentar si usas registro privado
      # - name: your-registry-secret
      volumes:
        - name: gcp-key-volume
          secret:
            secretName: gcs-worker-sa-key # Nombre del secreto creado
            items:
              - key: key.json # Nombre del archivo dentro del secreto
                path: key.json # Nombre que tendrá el archivo en mountPath
```

## File: `ingest-service\deployment-worker.yaml`
```yaml
# ingest-service/deployment-worker.yaml
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
      # No se necesita serviceAccountName si usas claves
      containers:
        - name: ingest-worker
          # Ensure this image includes CPU ONNX runtime and correct dependencies
          # ACTUALIZA ESTA ETIQUETA con la imagen construida DESPUÉS de aplicar los cambios de código
          image: ghcr.io/dev-nyro/ingest-service:develop-23382da
          imagePullPolicy: Always
          command: ["celery"]
          args:
            [
              "-A",
              "app.tasks.celery_app",
              "worker",
              "--loglevel=INFO",
              "-P",
              "prefork",
              "-c",
              "4",
            ]
          envFrom:
            - configMapRef:
                name: ingest-service-config
            - secretRef:
                name: ingest-service-secrets
          env:
            # --- Establecer variable de entorno para la clave ---
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-secrets/key.json" # Ruta donde se montará la clave
            # -------------------------------------------------
          volumeMounts:
            # --- Montar el secreto en el contenedor ---
            - name: gcp-key-volume
              mountPath: "/etc/gcp-secrets" # Directorio donde montar
              readOnly: true
            # ------------------------------------------
          resources:
            requests:
              cpu: "1000m"
              memory: "3Gi"
            limits:
              cpu: "4000m"
              memory: "8Gi"
      volumes:
        # --- Definir el volumen desde el secreto ---
        - name: gcp-key-volume
          secret:
            secretName: gcs-worker-sa-key # Nombre del secreto creado
            items:
            - key: key.json # Nombre del archivo DENTRO del secreto
              path: key.json # Nombre que tendrá el archivo en mountPath
        # ---------------------------------------
      # imagePullSecrets:
      # - name: your-registry-secret

```

## File: `ingest-service\service-api.yaml`
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

## File: `postgresql\persistent-volume-claim.yaml`
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

## File: `postgresql\service.yaml`
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

## File: `postgresql\statefulset.yaml`
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

## File: `query-service\configmap.yaml`
```yaml
# query-service/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: query-service-config
  namespace: atenex
data:
  # General
  QUERY_LOG_LEVEL: "INFO" # O "DEBUG" para más detalle

  # PostgreSQL (Sin cambios)
  QUERY_POSTGRES_SERVER: "postgresql-service.atenex.svc.cluster.local"
  QUERY_POSTGRES_PORT: "5432"
  QUERY_POSTGRES_USER: "postgres"
  QUERY_POSTGRES_DB: "atenex"
  # QUERY_POSTGRES_PASSWORD -> Proveniente de Secret

  # Milvus (Sin cambios)
  QUERY_MILVUS_URI: "http://milvus.atenex.svc.cluster.local:19530" # Asumiendo que Milvus está en el mismo namespace
  QUERY_MILVUS_COLLECTION_NAME: "document_chunks_minilm"
  QUERY_MILVUS_EMBEDDING_FIELD: "embedding"
  QUERY_MILVUS_CONTENT_FIELD: "content"
  QUERY_MILVUS_COMPANY_ID_FIELD: "company_id"
  QUERY_MILVUS_DOCUMENT_ID_FIELD: "document_id"
  QUERY_MILVUS_FILENAME_FIELD: "file_name"
  QUERY_MILVUS_GRPC_TIMEOUT: "15"
  # QUERY_MILVUS_METADATA_FIELDS: (...)

  # Embedding Settings
  QUERY_EMBEDDING_DIMENSION: "384" # Dimensión esperada/configurada, para Milvus y validación
  # --- NUEVA CONFIGURACIÓN ---
  QUERY_EMBEDDING_SERVICE_URL: "http://embedding-service.atenex.svc.cluster.local:8003" # URL del servicio de embedding
  # --- REMOVED FastEmbed local settings ---
  # QUERY_FASTEMBED_MODEL_NAME: "sentence-transformers/all-MiniLM-L6-v2"
  # QUERY_FASTEMBED_QUERY_PREFIX: "query: "


  # Gemini LLM Settings
  QUERY_GEMINI_MODEL_NAME: "gemini-1.5-flash-latest" # Corregido, no "gemini-2.5..."
  # QUERY_GEMINI_API_KEY -> Proveniente de Secret

  # RAG Pipeline Control & Parameters
  QUERY_RETRIEVER_TOP_K: "100"
  QUERY_MAX_CONTEXT_CHUNKS: "75"
  QUERY_MAX_PROMPT_TOKENS: "500000"

  QUERY_BM25_ENABLED: "true"
  QUERY_RERANKER_ENABLED: "true"
  QUERY_RERANKER_MODEL_NAME: "BAAI/bge-reranker-base"
  QUERY_DIVERSITY_FILTER_ENABLED: "true" # Habilitar MMR por defecto
  QUERY_DIVERSITY_LAMBDA: "0.5" # Lambda para MMR
  # QUERY_HYBRID_FUSION_ALPHA: "0.5" # (Actualmente no usado por RRF)

  # Prompt Templates (Sin cambios aquí, usan los defaults de config.py)
  # QUERY_RAG_PROMPT_TEMPLATE: |
  #   ...
  # QUERY_GENERAL_PROMPT_TEMPLATE: |
  #   ...
```

## File: `query-service\deployment.yaml`
```yaml
# query-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: query-service-deployment
  namespace: atenex # Asegurar namespace correcto
  labels:
    app: query-service
spec:
  replicas: 1 # Empezar con 1, escalar según necesidad
  selector:
    matchLabels:
      app: query-service
  template:
    metadata:
      labels:
        app: query-service
    spec:
      containers:
        - name: query-service # Nombre del contenedor
          # La imagen será actualizada por el pipeline CI/CD (este es un placeholder)
          image: ghcr.io/dev-nyro/query-service:develop-23382da
          imagePullPolicy: Always # O IfNotPresent si prefieres usar caché local en dev
          ports:
            - name: http
              containerPort: 8001 # Puerto interno definido en Dockerfile/Config
              protocol: TCP
          # Comando Gunicorn ajustado
          command: ["gunicorn", "-k", "uvicorn.workers.UvicornWorker"]
          args: [
              "-w", "2", # Número de workers (ajustar según CPU/Memoria)
              "-b", "0.0.0.0:8001", # Vincular al puerto interno
              "-t", "120", # Worker timeout
              # "--log-level", "$(QUERY_LOG_LEVEL)", # Heredar nivel de log (opcional)
              "app.main:app"
            ]
          envFrom:
            - configMapRef:
                name: query-service-config
            - secretRef:
                name: query-service-secrets # Asegurar que el Secret existe
          resources: # Aumentado para mejorar velocidad de arranque y performance
            requests:
              cpu: "1500m" # 1.5 vCPU
              memory: "3Gi"
            limits:
              cpu: "3000m" # 3 vCPU
              memory: "8Gi"

          # --- Sondas eliminadas según solicitud ---
          # livenessProbe: ...
          # readinessProbe: ...

      # Descomentar si se usa registro privado
      # imagePullSecrets:
      # - name: your-registry-secret
```

## File: `query-service\service.yaml`
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

## File: `README.md`
```md
# manifests-nyro
Repositorio de los manifiestos de Nyro B2B

```

## File: `reranker-service\configmap.yaml`
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
  RERANKER_BATCH_SIZE: "32" # Batch size for reranker predictions
  RERANKER_MAX_SEQ_LENGTH: "512" # Max sequence length for the model
  # RERANKER_PORT is typically set in the Deployment's container spec or defaults in Dockerfile
  # RERANKER_WORKERS is typically set in the Deployment's container spec or defaults in Dockerfile
```

## File: `reranker-service\deployment.yaml`
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
          image: ghcr.io/dev-nyro/reranker-service:latest # Placeholder: replace with actual image path and tag
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

## File: `reranker-service\service.yaml`
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

```

## File: `ingest-service\configmap.yaml`
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
  # INGEST_POSTGRES_PASSWORD -> Secret

  # Milvus/Zilliz
  INGEST_MILVUS_URI: "https://in03-0afab716eb46d7f.serverless.gcp-us-west1.cloud.zilliz.com" # MODIFIED
  INGEST_MILVUS_COLLECTION_NAME: "document_chunks_minilm"
  INGEST_MILVUS_GRPC_TIMEOUT: "20" # Increased timeout as per consideration for Zilliz Cloud

  # Google Cloud Storage
  INGEST_GCS_BUCKET_NAME: "atenex"

  # Embedding Settings
  INGEST_EMBEDDING_DIMENSION: "384" 

  # Tokenizer Settings
  INGEST_TIKTOKEN_ENCODING_NAME: "cl100k_base"

  # URLs for dependent services
  INGEST_EMBEDDING_SERVICE_URL: "http://embedding-service.atenex.svc.cluster.local:80/api/v1/embed"
  INGEST_DOCPROC_SERVICE_URL: "http://docproc-service.atenex.svc.cluster.local:80/api/v1/process"
```

## File: `ingest-service\deployment-api.yaml`
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
          image: ghcr.io/dev-nyro/ingest-service:develop-0c221d2 # Placeholder, CI/CD to update
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
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-secrets/key.json"
            - name: INGEST_ZILLIZ_API_KEY # MODIFIED: Added Zilliz API Key from Secret
              valueFrom:
                secretKeyRef:
                  name: ingest-service-secrets # Name of the k8s Secret
                  key: ZILLIZ_API_KEY # Key within the Secret
          volumeMounts:
            - name: gcp-key-volume
              mountPath: "/etc/gcp-secrets"
              readOnly: true
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1500m"
              memory: "2Gi"
      volumes:
        - name: gcp-key-volume
          secret:
            secretName: gcs-worker-sa-key
            items:
              - key: key.json
                path: key.json

```

## File: `ingest-service\deployment-worker.yaml`
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
          image: ghcr.io/dev-nyro/ingest-service:develop-0c221d2
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
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-secrets/key.json"
            - name: INGEST_ZILLIZ_API_KEY # MODIFIED: Added Zilliz API Key from Secret
              valueFrom:
                secretKeyRef:
                  name: ingest-service-secrets # Name of the k8s Secret
                  key: ZILLIZ_API_KEY # Key within the Secret
          volumeMounts:
            - name: gcp-key-volume
              mountPath: "/etc/gcp-secrets"
              readOnly: true
          resources:
            requests:
              cpu: "1000m" # 1 vCPU
              memory: "3Gi"
            limits:
              cpu: "4000m" # 4 vCPU
              memory: "8Gi"
      volumes:
        - name: gcp-key-volume
          secret:
            secretName: gcs-worker-sa-key
            items:
              - key: key.json
                path: key.json

```

## File: `ingest-service\service-api.yaml`
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

## File: `postgresql\persistent-volume-claim.yaml`
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

## File: `postgresql\service.yaml`
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

## File: `postgresql\statefulset.yaml`
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

## File: `query-service\configmap.yaml`
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
  QUERY_MILVUS_COLLECTION_NAME: "document_chunks_minilm"
  QUERY_MILVUS_EMBEDDING_FIELD: "embedding"
  QUERY_MILVUS_CONTENT_FIELD: "content"
  QUERY_MILVUS_COMPANY_ID_FIELD: "company_id"
  QUERY_MILVUS_DOCUMENT_ID_FIELD: "document_id"
  QUERY_MILVUS_FILENAME_FIELD: "file_name"
  QUERY_MILVUS_GRPC_TIMEOUT: "20" 
  # QUERY_MILVUS_METADATA_FIELDS: (...) # Uses default from config.py

  # Embedding Service (Remote)
  QUERY_EMBEDDING_DIMENSION: "384" # Debe coincidir con el modelo del embedding-service
  QUERY_EMBEDDING_SERVICE_URL: "http://embedding-service.atenex.svc.cluster.local:80"
  QUERY_EMBEDDING_CLIENT_TIMEOUT: "30"

  # Sparse Search Service (Remote BM25)
  QUERY_BM25_ENABLED: "true" # Controla si se usa el paso de búsqueda dispersa (llamada al servicio remoto)
  QUERY_SPARSE_SEARCH_SERVICE_URL: "http://sparse-search-service.atenex.svc.cluster.local:80"
  QUERY_SPARSE_SEARCH_CLIENT_TIMEOUT: "30"

  # Reranker Service (Remote)
  QUERY_RERANKER_ENABLED: "true"
  QUERY_RERANKER_SERVICE_URL: "http://reranker-service.atenex.svc.cluster.local:80"
  QUERY_RERANKER_CLIENT_TIMEOUT: "30"

  # Gemini LLM Settings
  QUERY_GEMINI_MODEL_NAME: "gemini-1.5-flash-latest"
  # QUERY_GEMINI_API_KEY -> Proveniente de Secret

  # RAG Pipeline Control & Parameters
  QUERY_RETRIEVER_TOP_K: "100" # Nº de chunks a pedir a dense y sparse retrievers inicialmente
  QUERY_MAX_CONTEXT_CHUNKS: "75" # Nº máx de chunks para el LLM después de reranking/filtering
  QUERY_MAX_PROMPT_TOKENS: "500000" 
  QUERY_DIVERSITY_FILTER_ENABLED: "true"
  QUERY_DIVERSITY_LAMBDA: "0.5" # Para MMR si está habilitado

  # HTTP Client Settings (Globales, usados por http_client en UseCase para Reranker y otros futuros)
  QUERY_HTTP_CLIENT_TIMEOUT: "60" 
  QUERY_HTTP_CLIENT_MAX_RETRIES: "2"
  QUERY_HTTP_CLIENT_BACKOFF_FACTOR: "1.0"

  # Chat history settings
  QUERY_MAX_CHAT_HISTORY_MESSAGES: "10"
  QUERY_NUM_SOURCES_TO_SHOW: "7"
  
  # MapReduce (Opcional, actualmente gestionado en config.py pero podría moverse aquí)
  QUERY_MAPREDUCE_ENABLED: "true"
  QUERY_MAPREDUCE_CHUNK_BATCH_SIZE: "5"
  QUERY_MAPREDUCE_ACTIVATION_THRESHOLD: "25"
```

## File: `query-service\deployment.yaml`
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
          image: ghcr.io/dev-nyro/query-service:develop-69dfa0f # Placeholder, CI/CD to update
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

## File: `query-service\service.yaml`
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

## File: `README.md`
```md
# manifests-nyro
Repositorio de los manifiestos de Nyro B2B

```

## File: `reranker-service\configmap.yaml`
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
  RERANKER_BATCH_SIZE: "32" # Batch size for reranker predictions
  RERANKER_MAX_SEQ_LENGTH: "512" # Max sequence length for the model
  # RERANKER_PORT is typically set in the Deployment's container spec or defaults in Dockerfile
  # RERANKER_WORKERS is typically set in the Deployment's container spec or defaults in Dockerfile
```

## File: `reranker-service\deployment.yaml`
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
  replicas: 1
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
          image: ghcr.io/dev-nyro/reranker-service:develop-67d3507 # Placeholder: replace with actual image path and tag
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

## File: `reranker-service\service.yaml`
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

## File: `sparse-search-service\configmap.yaml`
```yaml
# sparse-search-service/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sparse-search-service-config
  namespace: atenex # Asegúrate que el namespace sea el correcto
  labels:
    app.kubernetes.io/name: sparse-search-service
    app.kubernetes.io/part-of: atenex-platform
data:
  SPARSE_LOG_LEVEL: "INFO"
  # PORT for Gunicorn to listen on inside the container.
  # This should match containerPort in the Deployment.
  # The K8s Service will target this port.
  PORT: "8004" # El puerto que Gunicorn usará (definido en settings.PORT)

  SPARSE_POSTGRES_SERVER: "postgresql-service.atenex.svc.cluster.local"
  SPARSE_POSTGRES_PORT: "5432"
  SPARSE_POSTGRES_DB: "atenex"
  SPARSE_POSTGRES_USER: "postgres" 
  SPARSE_DB_POOL_MIN_SIZE: "2"
  SPARSE_DB_POOL_MAX_SIZE: "10"
  SPARSE_DB_CONNECT_TIMEOUT: "30"
  SPARSE_DB_COMMAND_TIMEOUT: "60"

  # Gunicorn specific settings (can also be in gunicorn_conf.py or Dockerfile CMD)
  # These are examples if you want to override gunicorn_conf.py defaults via env vars
  # GUNICORN_PROCESSES: "2" 
  # GUNICORN_THREADS: "1"
  # GUNICORN_TIMEOUT: "120"
  # GUNICORN_LOG_LEVEL: "info"
```

## File: `sparse-search-service\cronjob.yaml`
```yaml
# sparse-search-service/k8s/sparse-search-service-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: sparse-search-index-builder
  namespace: atenex # Asegúrate que el namespace es correcto
  labels:
    app: sparse-search-service
    component: index-builder
spec:
  schedule: "0 */6 * * *" # Cada 6 horas (a las 00:00, 06:00, 12:00, 18:00 UTC)
  jobTemplate:
    spec:
      ttlSecondsAfterFinished: 3600 # Limpia jobs finalizados después de 1 hora
      backoffLimit: 2 # Número de reintentos para el job si falla
      template:
        metadata:
          labels:
            app: sparse-search-service
            component: index-builder-pod
        spec:
          # Especificar el ServiceAccount que tiene permisos para GCS y, si es necesario, PostgreSQL
          # serviceAccountName: sparse-search-builder-sa # (Crear este SA con los permisos necesarios)
          # Si usas Workload Identity, el SA de K8s debe estar vinculado al SA de GCP
          serviceAccountName: default # O el SA que uses que tenga permisos para PG
                                      # Para GCS, si usas Workload Identity, este SA K8s debe estar mapeado
                                      # a un SA de GCP con permisos de escritura en el bucket de índices.
          containers:
          - name: index-builder-container
            image: TU_REGISTRO_DOCKER/sparse-search-service:latest # Reemplazar con tu imagen y tag
            imagePullPolicy: Always # O IfNotPresent si el tag es estable
            command: ["python", "-m", "app.jobs.index_builder_cronjob"]
            args:
            - "--company-id"
            - "ALL" # Procesa todas las compañías activas
            envFrom:
            - configMapRef:
                name: sparse-search-service-config # Nombre de tu ConfigMap
            - secretRef:
                name: sparse-search-service-secrets # Nombre de tu Secret para SPARSE_POSTGRES_PASSWORD
            env:
            # Si necesitas GOOGLE_APPLICATION_CREDENTIALS montado como un secret:
            # - name: GOOGLE_APPLICATION_CREDENTIALS
            #   value: "/etc/gcp-sa-keys/sparse-builder-key.json"
            # Si usas variables de entorno específicas para el builder, puedes añadirlas aquí:
            # - name: JOB_SPECIFIC_SETTING
            #   value: "some_value"
            resources:
              requests:
                memory: "512Mi"
                cpu: "250m"
              limits:
                memory: "2Gi" # BM25 puede consumir memoria dependiendo del tamaño del corpus
                cpu: "1"
            # volumeMounts:
            # - name: gcp-sa-key-volume
            #   mountPath: /etc/gcp-sa-keys
            #   readOnly: true
          restartPolicy: OnFailure # O Never, dependiendo de la idempotencia y manejo de errores
          # volumes:
          # - name: gcp-sa-key-volume
          #   secret:
          #     secretName: sparse-builder-gcp-sa-key # K8s secret que contiene la key del SA de GCP
```

## File: `sparse-search-service\deployment.yaml`
```yaml
# sparse-search-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sparse-search-service
  namespace: atenex
  labels:
    app: sparse-search-service # Selector label
    app.kubernetes.io/name: sparse-search-service
    app.kubernetes.io/version: "0.1.0" # Corresponde a la versión de la app
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: atenex-platform
spec:
  replicas: 1 # Ajustar según necesidad y pruebas de carga
  selector:
    matchLabels:
      app: sparse-search-service # Debe coincidir con template.metadata.labels.app
  template:
    metadata:
      labels:
        app: sparse-search-service # Etiqueta para que el Service lo seleccione
        app.kubernetes.io/name: sparse-search-service
        app.kubernetes.io/version: "0.1.0"
    spec:
      # serviceAccountName: sparse-search-service-sa # Si se requiere un SA específico
      containers:
      - name: sparse-search-service-container # Nombre descriptivo del contenedor
        image: your-container-registry/sparse-search-service:latest # ¡CAMBIAR ESTO por tu imagen real!
        imagePullPolicy: Always # O IfNotPresent para desarrollo
        
        ports:
        - name: http # Nombre del puerto
          containerPort: 8004 # Puerto en el que Gunicorn escucha DENTRO del contenedor.
                              # Debe coincidir con settings.PORT y el 'bind' de gunicorn_conf.py.
          protocol: TCP
        
        envFrom:
        - configMapRef:
            name: sparse-search-service-config # Carga todas las claves del ConfigMap como env vars
        # - secretRef: # Descomentar y usar si tienes un secret específico para este servicio
        #     name: sparse-search-service-secrets 

        env:
        # Mapeo de la contraseña de PostgreSQL desde un secret existente.
        # Si 'atenex-db-secrets' tiene la clave 'POSTGRES_PASSWORD_GLOBAL'
        # esta se mapeará a la variable de entorno SPARSE_POSTGRES_PASSWORD en el pod.
        - name: SPARSE_POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: atenex-db-secrets # Asume un secret común, o usa 'sparse-search-service-secrets'
              key: POSTGRES_PASSWORD_GLOBAL # La clave en el secret que tiene la pass
                                          # Cambiar a 'SPARSE_POSTGRES_PASSWORD' si el secret es específico.
        # La variable PORT se pasa a Gunicorn a través de gunicorn_conf.py que lee
        # os.environ.get('PORT', settings.PORT). El ConfigMap ya define PORT=8004
        # que se carga vía envFrom.
        # - name: PORT # Opcional si gunicorn_conf.py lo maneja bien con settings.PORT
        #   valueFrom:
        #     configMapKeyRef:
        #       name: sparse-search-service-config
        #       key: PORT
        
        resources:
          requests:
            memory: "512Mi" # BM2S puede consumir memoria con corpus grandes
            cpu: "500m"    
          limits:
            memory: "2Gi" 
            cpu: "1500m"

        readinessProbe:
          httpGet:
            path: /health
            port: http # Referencia al nombre del puerto definido arriba (o el número de puerto directamente)
          initialDelaySeconds: 20 # Dar tiempo para que la app y DB se inicien
          periodSeconds: 15
          timeoutSeconds: 5
          failureThreshold: 3
          successThreshold: 1
        livenessProbe:
          httpGet:
            path: /health
            port: http 
          initialDelaySeconds: 45 # Más tiempo para el liveness
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
      # imagePullSecrets: # Si la imagen está en un registro privado
      # - name: your-registry-secret-name
```

## File: `sparse-search-service\service.yaml`
```yaml
# sparse-search-service/k8s/sparse-search-service-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: sparse-search-service # Nombre DNS interno del servicio
  namespace: atenex
  labels:
    app: sparse-search-service # Label para que el Ingress/otros servicios lo encuentren
    app.kubernetes.io/name: sparse-search-service
    app.kubernetes.io/part-of: atenex-platform
spec:
  type: ClusterIP # Servicio interno, no expuesto fuera del clúster directamente
  selector:
    app: sparse-search-service # Selecciona los Pods con esta etiqueta (del Deployment)
  ports:
  - name: http
    port: 80 # Puerto en el que el servicio K8s escucha (DNS: http://sparse-search-service:80)
    targetPort: http # Nombre del puerto del Pod al que redirigir (definido en el Deployment como 8004)
                     # O puedes usar el número de puerto directamente: targetPort: 8004
    protocol: TCP
```

```

## File: `ingest-service\configmap.yaml`
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
  # INGEST_POSTGRES_PASSWORD -> Secret

  # Milvus/Zilliz
  INGEST_MILVUS_URI: "https://in03-0afab716eb46d7f.serverless.gcp-us-west1.cloud.zilliz.com" # MODIFIED
  INGEST_MILVUS_COLLECTION_NAME: "document_chunks_minilm"
  INGEST_MILVUS_GRPC_TIMEOUT: "20" # Increased timeout as per consideration for Zilliz Cloud

  # Google Cloud Storage
  INGEST_GCS_BUCKET_NAME: "atenex"

  # Embedding Settings
  INGEST_EMBEDDING_DIMENSION: "384" 

  # Tokenizer Settings
  INGEST_TIKTOKEN_ENCODING_NAME: "cl100k_base"

  # URLs for dependent services
  INGEST_EMBEDDING_SERVICE_URL: "http://embedding-service.atenex.svc.cluster.local:80/api/v1/embed"
  INGEST_DOCPROC_SERVICE_URL: "http://docproc-service.atenex.svc.cluster.local:80/api/v1/process"
```

## File: `ingest-service\deployment-api.yaml`
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
          image: ghcr.io/dev-nyro/ingest-service:develop-0c221d2 # Placeholder, CI/CD to update
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
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-secrets/key.json"
            - name: INGEST_ZILLIZ_API_KEY # MODIFIED: Added Zilliz API Key from Secret
              valueFrom:
                secretKeyRef:
                  name: ingest-service-secrets # Name of the k8s Secret
                  key: ZILLIZ_API_KEY # Key within the Secret
          volumeMounts:
            - name: gcp-key-volume
              mountPath: "/etc/gcp-secrets"
              readOnly: true
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1500m"
              memory: "2Gi"
      volumes:
        - name: gcp-key-volume
          secret:
            secretName: gcs-worker-sa-key
            items:
              - key: key.json
                path: key.json

```

## File: `ingest-service\deployment-worker.yaml`
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
          image: ghcr.io/dev-nyro/ingest-service:develop-0c221d2
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
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-secrets/key.json"
            - name: INGEST_ZILLIZ_API_KEY # MODIFIED: Added Zilliz API Key from Secret
              valueFrom:
                secretKeyRef:
                  name: ingest-service-secrets # Name of the k8s Secret
                  key: ZILLIZ_API_KEY # Key within the Secret
          volumeMounts:
            - name: gcp-key-volume
              mountPath: "/etc/gcp-secrets"
              readOnly: true
          resources:
            requests:
              cpu: "1000m" # 1 vCPU
              memory: "3Gi"
            limits:
              cpu: "4000m" # 4 vCPU
              memory: "8Gi"
      volumes:
        - name: gcp-key-volume
          secret:
            secretName: gcs-worker-sa-key
            items:
              - key: key.json
                path: key.json

```

## File: `ingest-service\service-api.yaml`
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

## File: `postgresql\persistent-volume-claim.yaml`
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

## File: `postgresql\service.yaml`
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

## File: `postgresql\statefulset.yaml`
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

## File: `query-service\configmap.yaml`
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
  QUERY_MILVUS_COLLECTION_NAME: "document_chunks_minilm"
  QUERY_MILVUS_EMBEDDING_FIELD: "embedding"
  QUERY_MILVUS_CONTENT_FIELD: "content"
  QUERY_MILVUS_COMPANY_ID_FIELD: "company_id"
  QUERY_MILVUS_DOCUMENT_ID_FIELD: "document_id"
  QUERY_MILVUS_FILENAME_FIELD: "file_name"
  QUERY_MILVUS_GRPC_TIMEOUT: "20" 
  # QUERY_MILVUS_METADATA_FIELDS: (...) # Uses default from config.py

  # Embedding Service (Remote)
  QUERY_EMBEDDING_DIMENSION: "384" # Debe coincidir con el modelo del embedding-service
  QUERY_EMBEDDING_SERVICE_URL: "http://embedding-service.atenex.svc.cluster.local:80"
  QUERY_EMBEDDING_CLIENT_TIMEOUT: "30"

  # Sparse Search Service (Remote BM25)
  QUERY_BM25_ENABLED: "true" # Controla si se usa el paso de búsqueda dispersa (llamada al servicio remoto)
  QUERY_SPARSE_SEARCH_SERVICE_URL: "http://sparse-search-service.atenex.svc.cluster.local:80"
  QUERY_SPARSE_SEARCH_CLIENT_TIMEOUT: "30"

  # Reranker Service (Remote)
  QUERY_RERANKER_ENABLED: "true"
  QUERY_RERANKER_SERVICE_URL: "http://reranker-service.atenex.svc.cluster.local:80"
  QUERY_RERANKER_CLIENT_TIMEOUT: "30"

  # Gemini LLM Settings
  QUERY_GEMINI_MODEL_NAME: "gemini-1.5-flash-latest"
  # QUERY_GEMINI_API_KEY -> Proveniente de Secret

  # RAG Pipeline Control & Parameters
  QUERY_RETRIEVER_TOP_K: "100" # Nº de chunks a pedir a dense y sparse retrievers inicialmente
  QUERY_MAX_CONTEXT_CHUNKS: "75" # Nº máx de chunks para el LLM después de reranking/filtering
  QUERY_MAX_PROMPT_TOKENS: "500000" 
  QUERY_DIVERSITY_FILTER_ENABLED: "true"
  QUERY_DIVERSITY_LAMBDA: "0.5" # Para MMR si está habilitado

  # HTTP Client Settings (Globales, usados por http_client en UseCase para Reranker y otros futuros)
  QUERY_HTTP_CLIENT_TIMEOUT: "60" 
  QUERY_HTTP_CLIENT_MAX_RETRIES: "2"
  QUERY_HTTP_CLIENT_BACKOFF_FACTOR: "1.0"

  # Chat history settings
  QUERY_MAX_CHAT_HISTORY_MESSAGES: "10"
  QUERY_NUM_SOURCES_TO_SHOW: "7"
  
  # MapReduce (Opcional, actualmente gestionado en config.py pero podría moverse aquí)
  QUERY_MAPREDUCE_ENABLED: "true"
  QUERY_MAPREDUCE_CHUNK_BATCH_SIZE: "5"
  QUERY_MAPREDUCE_ACTIVATION_THRESHOLD: "25"
```

## File: `query-service\deployment.yaml`
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
          image: ghcr.io/dev-nyro/query-service:develop-69dfa0f # Placeholder, CI/CD to update
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

## File: `query-service\service.yaml`
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

## File: `README.md`
```md
# manifests-nyro
Repositorio de los manifiestos de Nyro B2B

```

## File: `reranker-service\configmap.yaml`
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
  RERANKER_BATCH_SIZE: "32" # Batch size for reranker predictions
  RERANKER_MAX_SEQ_LENGTH: "512" # Max sequence length for the model
  # RERANKER_PORT is typically set in the Deployment's container spec or defaults in Dockerfile
  # RERANKER_WORKERS is typically set in the Deployment's container spec or defaults in Dockerfile
```

## File: `reranker-service\deployment.yaml`
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
  replicas: 1
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
          image: ghcr.io/dev-nyro/reranker-service:develop-67d3507 # Placeholder: replace with actual image path and tag
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

## File: `reranker-service\service.yaml`
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

## File: `sparse-search-service\configmap.yaml`
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

  SPARSE_INDEX_GCS_BUCKET_NAME: "atenex-sparse-indices" # Asegúrate que este bucket exista
  SPARSE_INDEX_CACHE_MAX_ITEMS: "50"
  SPARSE_INDEX_CACHE_TTL_SECONDS: "3600" # 1 hora
```

## File: `sparse-search-service\cronjob.yaml`
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
  schedule: "0 */6 * * *" 
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
          serviceAccountName: sparse-search-builder-sa # SA con permisos a GCS (escritura) y PG (lectura)
          containers:
          - name: index-builder-container
            image: your-container-registry/sparse-search-service:v1.0.0 # Usa la misma imagen que el servicio
            imagePullPolicy: Always 
            command: ["python", "-m", "app.jobs.index_builder_cronjob"]
            args:
            - "--company-id"
            - "ALL" 
            envFrom:
            - configMapRef:
                name: sparse-search-service-config 
            - secretRef:
                name: sparse-search-service-secrets # Para SPARSE_POSTGRES_PASSWORD
            env:
            - name: SPARSE_POSTGRES_PASSWORD # Asegurar que la variable está disponible para el script del job
              valueFrom:
                secretKeyRef:
                  name: sparse-search-service-secrets 
                  key: SPARSE_POSTGRES_PASSWORD
            # Configuración para GCS, si no usas Workload Identity para el SA 'sparse-search-builder-sa'
            # - name: GOOGLE_APPLICATION_CREDENTIALS
            #   value: "/etc/gcp-builder-sa-keys/key.json"
            # volumeMounts:
            # - name: gcp-builder-sa-key-volume
            #   mountPath: "/etc/gcp-builder-sa-keys"
            #   readOnly: true
            resources:
              requests:
                memory: "1Gi" # Puede necesitar más memoria para construir índices grandes
                cpu: "500m"
              limits:
                memory: "4Gi" 
                cpu: "2" # Más CPU puede acelerar la indexación
          restartPolicy: OnFailure 
          # volumes: # Si montas la clave JSON para el builder SA
          # - name: gcp-builder-sa-key-volume
          #   secret:
          #     secretName: sparse-search-gcs-sa-key # El secreto con la clave del SA del builder
```

## File: `sparse-search-service\deployment.yaml`
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
      # serviceAccountName: sparse-search-runtime-sa # SA para el runtime, con permisos de lectura en GCS
      containers:
      - name: sparse-search-service-container 
        image: your-container-registry/sparse-search-service:v1.0.0 # Reemplaza con tu imagen
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
        # Para GCS: Asumiendo Workload Identity o un SA montado para el runtime
        # Si usas clave JSON montada para el runtime (menos ideal que Workload Identity):
        # - name: GOOGLE_APPLICATION_CREDENTIALS
        #   value: "/etc/gcp-runtime-sa-keys/key.json"
        # volumeMounts:
        # - name: gcp-runtime-sa-key-volume
        #   mountPath: "/etc/gcp-runtime-sa-keys"
        #   readOnly: true
        
        resources:
          requests:
            memory: "1Gi" # Aumentado para el caché LRU y objetos BM25
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
      # volumes: # Si montas clave JSON para el runtime
      # - name: gcp-runtime-sa-key-volume
      #   secret:
      #     secretName: sparse-search-gcs-sa-key # O un secreto específico para el runtime
```

## File: `sparse-search-service\service.yaml`
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

```

## File: `ingest-service\configmap.yaml`
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
  # INGEST_POSTGRES_PASSWORD -> Secret

  # Milvus/Zilliz
  INGEST_MILVUS_URI: "https://in03-0afab716eb46d7f.serverless.gcp-us-west1.cloud.zilliz.com" # MODIFIED
  INGEST_MILVUS_COLLECTION_NAME: "document_chunks_minilm"
  INGEST_MILVUS_GRPC_TIMEOUT: "20" # Increased timeout as per consideration for Zilliz Cloud

  # Google Cloud Storage
  INGEST_GCS_BUCKET_NAME: "atenex"

  # Embedding Settings
  INGEST_EMBEDDING_DIMENSION: "384" 

  # Tokenizer Settings
  INGEST_TIKTOKEN_ENCODING_NAME: "cl100k_base"

  # URLs for dependent services
  INGEST_EMBEDDING_SERVICE_URL: "http://embedding-service.atenex.svc.cluster.local:80/api/v1/embed"
  INGEST_DOCPROC_SERVICE_URL: "http://docproc-service.atenex.svc.cluster.local:80/api/v1/process"
```

## File: `ingest-service\deployment-api.yaml`
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
          image: ghcr.io/dev-nyro/ingest-service:develop-d60bccd # Placeholder, CI/CD to update
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
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-secrets/key.json"
            - name: INGEST_ZILLIZ_API_KEY # MODIFIED: Added Zilliz API Key from Secret
              valueFrom:
                secretKeyRef:
                  name: ingest-service-secrets # Name of the k8s Secret
                  key: ZILLIZ_API_KEY # Key within the Secret
          volumeMounts:
            - name: gcp-key-volume
              mountPath: "/etc/gcp-secrets"
              readOnly: true
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1500m"
              memory: "2Gi"
      volumes:
        - name: gcp-key-volume
          secret:
            secretName: gcs-worker-sa-key
            items:
              - key: key.json
                path: key.json

```

## File: `ingest-service\deployment-worker.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingest-worker-deployment
  namespace: atenex
  labels:
    app: ingest-worker
spec:
  replicas: 1
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
          image: ghcr.io/dev-nyro/ingest-service:develop-d60bccd
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
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-secrets/key.json"
            - name: INGEST_ZILLIZ_API_KEY # MODIFIED: Added Zilliz API Key from Secret
              valueFrom:
                secretKeyRef:
                  name: ingest-service-secrets # Name of the k8s Secret
                  key: ZILLIZ_API_KEY # Key within the Secret
          volumeMounts:
            - name: gcp-key-volume
              mountPath: "/etc/gcp-secrets"
              readOnly: true
          resources:
            requests:
              cpu: "1000m" # 1 vCPU
              memory: "3Gi"
            limits:
              cpu: "4000m" # 4 vCPU
              memory: "8Gi"
      volumes:
        - name: gcp-key-volume
          secret:
            secretName: gcs-worker-sa-key
            items:
              - key: key.json
                path: key.json

```

## File: `ingest-service\service-api.yaml`
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

## File: `postgresql\persistent-volume-claim.yaml`
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

## File: `postgresql\service.yaml`
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

## File: `postgresql\statefulset.yaml`
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

## File: `query-service\configmap.yaml`
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
  QUERY_MILVUS_COLLECTION_NAME: "document_chunks_minilm"
  QUERY_MILVUS_EMBEDDING_FIELD: "embedding"
  QUERY_MILVUS_CONTENT_FIELD: "content"
  QUERY_MILVUS_COMPANY_ID_FIELD: "company_id"
  QUERY_MILVUS_DOCUMENT_ID_FIELD: "document_id"
  QUERY_MILVUS_FILENAME_FIELD: "file_name"
  QUERY_MILVUS_GRPC_TIMEOUT: "20" 
  # QUERY_MILVUS_METADATA_FIELDS: (...) # Uses default from config.py

  # Embedding Service (Remote)
  QUERY_EMBEDDING_DIMENSION: "384" # Debe coincidir con el modelo del embedding-service
  QUERY_EMBEDDING_SERVICE_URL: "http://embedding-service.atenex.svc.cluster.local:80"
  QUERY_EMBEDDING_CLIENT_TIMEOUT: "30"

  # Sparse Search Service (Remote BM25)
  QUERY_BM25_ENABLED: "true" # Controla si se usa el paso de búsqueda dispersa (llamada al servicio remoto)
  QUERY_SPARSE_SEARCH_SERVICE_URL: "http://sparse-search-service.atenex.svc.cluster.local:80"
  QUERY_SPARSE_SEARCH_CLIENT_TIMEOUT: "30"

  # Reranker Service (Remote)
  QUERY_RERANKER_ENABLED: "true"
  QUERY_RERANKER_SERVICE_URL: "http://reranker-service.atenex.svc.cluster.local:80"
  QUERY_RERANKER_CLIENT_TIMEOUT: "220"

  # Gemini LLM Settings
  QUERY_GEMINI_MODEL_NAME: "gemini-1.5-flash-latest"
  # QUERY_GEMINI_API_KEY -> Proveniente de Secret

  # RAG Pipeline Control & Parameters
  QUERY_RETRIEVER_TOP_K: "100" # Nº de chunks a pedir a dense y sparse retrievers inicialmente
  QUERY_MAX_CONTEXT_CHUNKS: "75" # Nº máx de chunks para el LLM después de reranking/filtering
  QUERY_MAX_PROMPT_TOKENS: "500000" 
  QUERY_DIVERSITY_FILTER_ENABLED: "true"
  QUERY_DIVERSITY_LAMBDA: "0.5" # Para MMR si está habilitado

  # HTTP Client Settings (Globales, usados por http_client en UseCase para Reranker y otros futuros)
  QUERY_HTTP_CLIENT_TIMEOUT: "60" 
  QUERY_HTTP_CLIENT_MAX_RETRIES: "2"
  QUERY_HTTP_CLIENT_BACKOFF_FACTOR: "1.0"

  # Chat history settings
  QUERY_MAX_CHAT_HISTORY_MESSAGES: "10"
  QUERY_NUM_SOURCES_TO_SHOW: "7"
  
  # MapReduce (Opcional, actualmente gestionado en config.py pero podría moverse aquí)
  QUERY_MAPREDUCE_ENABLED: "true"
  QUERY_MAPREDUCE_CHUNK_BATCH_SIZE: "5"
  QUERY_MAPREDUCE_ACTIVATION_THRESHOLD: "25"
```

## File: `query-service\deployment.yaml`
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
          image: ghcr.io/dev-nyro/query-service:develop-a78f997 # Placeholder, CI/CD to update
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

## File: `query-service\service.yaml`
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

## File: `README.md`
```md
# manifests-nyro
Repositorio de los manifiestos de Nyro B2B

```

## File: `reranker-service\configmap.yaml`
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

## File: `reranker-service\deployment.yaml`
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
  replicas: 1
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
          image: ghcr.io/dev-nyro/reranker-service:develop-32cc8c5 # Placeholder: replace with actual image path and tag
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

## File: `reranker-service\service.yaml`
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

## File: `sparse-search-service\configmap.yaml`
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

  SPARSE_INDEX_GCS_BUCKET_NAME: "atenex-sparse-indices" # Asegúrate que este bucket exista
  SPARSE_INDEX_CACHE_MAX_ITEMS: "50"
  SPARSE_INDEX_CACHE_TTL_SECONDS: "3600" # 1 hora
```

## File: `sparse-search-service\cronjob.yaml`
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
          volumes:
            - name: gcs-key-volume
              secret:
                secretName: sparse-search-gcs-key
          containers:
            - name: index-builder-container
              image: ghcr.io/dev-nyro/sparse-search-service:develop-d60bccd # Asegúrate que CI/CD actualice esto
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
                - name: GOOGLE_APPLICATION_CREDENTIALS
                  value: "/etc/gcp-keys/key.json"
              volumeMounts:
                - name: gcs-key-volume
                  mountPath: "/etc/gcp-keys"
                  readOnly: true
              resources:
                requests:
                  memory: "1Gi"
                  cpu: "500m"
                limits:
                  memory: "2Gi"
                  cpu: "1"
          restartPolicy: OnFailure

```

## File: `sparse-search-service\deployment.yaml`
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
      volumes: # LLM: ADDED - Definir el volumen desde el Secret
        - name: gcs-key-volume
          secret:
            secretName: sparse-search-gcs-key # Nombre del Secret creado en K8s
      containers:
        - name: sparse-search-service-container
          image: ghcr.io/dev-nyro/sparse-search-service:develop-d60bccd # Reemplaza con tu imagen
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
            # LLM: ADDED - Configurar GOOGLE_APPLICATION_CREDENTIALS
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-keys/key.json" # Ruta donde se montará la clave dentro del pod
          volumeMounts: # LLM: ADDED - Montar el volumen en el contenedor
            - name: gcs-key-volume
              mountPath: "/etc/gcp-keys" # Directorio donde se montará el contenido del Secret
              readOnly: true
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

## File: `sparse-search-service\service.yaml`
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

## File: `sparse-search-service\serviceaccount.yaml`
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

```

## File: `ingest-service\configmap.yaml`
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
  # INGEST_POSTGRES_PASSWORD -> Secret

  # Milvus/Zilliz
  INGEST_MILVUS_URI: "https://in03-0afab716eb46d7f.serverless.gcp-us-west1.cloud.zilliz.com" # MODIFIED
  INGEST_MILVUS_COLLECTION_NAME: "document_chunks_minilm"
  INGEST_MILVUS_GRPC_TIMEOUT: "20" # Increased timeout as per consideration for Zilliz Cloud

  # Google Cloud Storage
  INGEST_GCS_BUCKET_NAME: "atenex"

  # Embedding Settings
  INGEST_EMBEDDING_DIMENSION: "384" 

  # Tokenizer Settings
  INGEST_TIKTOKEN_ENCODING_NAME: "cl100k_base"

  # URLs for dependent services
  INGEST_EMBEDDING_SERVICE_URL: "http://embedding-service.atenex.svc.cluster.local:80/api/v1/embed"
  INGEST_DOCPROC_SERVICE_URL: "http://docproc-service.atenex.svc.cluster.local:80/api/v1/process"
```

## File: `ingest-service\deployment-api.yaml`
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
          image: ghcr.io/dev-nyro/ingest-service:develop-d60bccd # Placeholder, CI/CD to update
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
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-secrets/key.json"
            - name: INGEST_ZILLIZ_API_KEY # MODIFIED: Added Zilliz API Key from Secret
              valueFrom:
                secretKeyRef:
                  name: ingest-service-secrets # Name of the k8s Secret
                  key: ZILLIZ_API_KEY # Key within the Secret
          volumeMounts:
            - name: gcp-key-volume
              mountPath: "/etc/gcp-secrets"
              readOnly: true
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1500m"
              memory: "2Gi"
      volumes:
        - name: gcp-key-volume
          secret:
            secretName: gcs-worker-sa-key
            items:
              - key: key.json
                path: key.json

```

## File: `ingest-service\deployment-worker.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingest-worker-deployment
  namespace: atenex
  labels:
    app: ingest-worker
spec:
  replicas: 1
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
          image: ghcr.io/dev-nyro/ingest-service:develop-d60bccd
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
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-secrets/key.json"
            - name: INGEST_ZILLIZ_API_KEY # MODIFIED: Added Zilliz API Key from Secret
              valueFrom:
                secretKeyRef:
                  name: ingest-service-secrets # Name of the k8s Secret
                  key: ZILLIZ_API_KEY # Key within the Secret
          volumeMounts:
            - name: gcp-key-volume
              mountPath: "/etc/gcp-secrets"
              readOnly: true
          resources:
            requests:
              cpu: "1000m" # 1 vCPU
              memory: "3Gi"
            limits:
              cpu: "4000m" # 4 vCPU
              memory: "8Gi"
      volumes:
        - name: gcp-key-volume
          secret:
            secretName: gcs-worker-sa-key
            items:
              - key: key.json
                path: key.json

```

## File: `ingest-service\service-api.yaml`
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

## File: `postgresql\persistent-volume-claim.yaml`
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

## File: `postgresql\service.yaml`
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

## File: `postgresql\statefulset.yaml`
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

## File: `query-service\configmap.yaml`
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
  QUERY_MILVUS_COLLECTION_NAME: "document_chunks_minilm"
  QUERY_MILVUS_EMBEDDING_FIELD: "embedding"
  QUERY_MILVUS_CONTENT_FIELD: "content"
  QUERY_MILVUS_COMPANY_ID_FIELD: "company_id"
  QUERY_MILVUS_DOCUMENT_ID_FIELD: "document_id"
  QUERY_MILVUS_FILENAME_FIELD: "file_name"
  QUERY_MILVUS_GRPC_TIMEOUT: "20" 
  # QUERY_MILVUS_METADATA_FIELDS: (...) # Uses default from config.py

  # Embedding Service (Remote)
  QUERY_EMBEDDING_DIMENSION: "384" # Debe coincidir con el modelo del embedding-service
  QUERY_EMBEDDING_SERVICE_URL: "http://embedding-service.atenex.svc.cluster.local:80"
  QUERY_EMBEDDING_CLIENT_TIMEOUT: "30"

  # Sparse Search Service (Remote BM25)
  QUERY_BM25_ENABLED: "true" # Controla si se usa el paso de búsqueda dispersa (llamada al servicio remoto)
  QUERY_SPARSE_SEARCH_SERVICE_URL: "http://sparse-search-service.atenex.svc.cluster.local:80"
  QUERY_SPARSE_SEARCH_CLIENT_TIMEOUT: "30"

  # Reranker Service (Remote)
  QUERY_RERANKER_ENABLED: "true"
  QUERY_RERANKER_SERVICE_URL: "http://reranker-gpu.atenex.svc.cluster.local" # "http://reranker-service.atenex.svc.cluster.local:80" normal
  QUERY_RERANKER_CLIENT_TIMEOUT: "220"

  # Gemini LLM Settings
  QUERY_GEMINI_MODEL_NAME: "gemini-1.5-flash-latest"
  # QUERY_GEMINI_API_KEY -> Proveniente de Secret

  # RAG Pipeline Control & Parameters
  QUERY_RETRIEVER_TOP_K: "100" # Nº de chunks a pedir a dense y sparse retrievers inicialmente
  QUERY_MAX_CONTEXT_CHUNKS: "75" # Nº máx de chunks para el LLM después de reranking/filtering
  QUERY_MAX_PROMPT_TOKENS: "500000" 
  QUERY_DIVERSITY_FILTER_ENABLED: "true"
  QUERY_DIVERSITY_LAMBDA: "0.5" # Para MMR si está habilitado

  # HTTP Client Settings (Globales, usados por http_client en UseCase para Reranker y otros futuros)
  QUERY_HTTP_CLIENT_TIMEOUT: "60" 
  QUERY_HTTP_CLIENT_MAX_RETRIES: "2"
  QUERY_HTTP_CLIENT_BACKOFF_FACTOR: "1.0"

  # Chat history settings
  QUERY_MAX_CHAT_HISTORY_MESSAGES: "10"
  QUERY_NUM_SOURCES_TO_SHOW: "7"
  
  # MapReduce (Opcional, actualmente gestionado en config.py pero podría moverse aquí)
  QUERY_MAPREDUCE_ENABLED: "true"
  QUERY_MAPREDUCE_CHUNK_BATCH_SIZE: "5"
  QUERY_MAPREDUCE_ACTIVATION_THRESHOLD: "25"
```

## File: `query-service\deployment.yaml`
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
          image: ghcr.io/dev-nyro/query-service:develop-df4c9b9 # Placeholder, CI/CD to update
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

## File: `query-service\service.yaml`
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

## File: `README.md`
```md
# manifests-nyro
Repositorio de los manifiestos de Nyro B2B

```

## File: `reranker-service\configmap.yaml`
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

## File: `reranker-service\deployment.yaml`
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
  replicas: 1
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
          image: ghcr.io/dev-nyro/reranker-service:develop-32cc8c5 # Placeholder: replace with actual image path and tag
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

## File: `reranker-service\service.yaml`
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

## File: `reranker_gpu-service\endpoints.yaml`
```yaml
# reranker_gpu-service/endpoints.yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: reranker-gpu
  namespace: atenex
subsets:
- addresses:
  - ip: 192.168.65.3
    nodeName: external-reranker
  ports:
  - name: http
    port: 8004
```

## File: `reranker_gpu-service\service.yaml`
```yaml
# reranker_gpu-service/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: reranker-gpu
  namespace: atenex
spec:
  ports:
  - name: http
    port: 80          # Cluster-internal port
    targetPort: 8004  # Puerto real del contenedor en WSL2
  type: ClusterIP
```

## File: `sparse-search-service\configmap.yaml`
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

  SPARSE_INDEX_GCS_BUCKET_NAME: "atenex-sparse-indices" # Asegúrate que este bucket exista
  SPARSE_INDEX_CACHE_MAX_ITEMS: "50"
  SPARSE_INDEX_CACHE_TTL_SECONDS: "3600" # 1 hora
```

## File: `sparse-search-service\cronjob.yaml`
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
          volumes:
            - name: gcs-key-volume
              secret:
                secretName: sparse-search-gcs-key
          containers:
            - name: index-builder-container
              image: ghcr.io/dev-nyro/sparse-search-service:develop-df4c9b9 # Asegúrate que CI/CD actualice esto
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
                - name: GOOGLE_APPLICATION_CREDENTIALS
                  value: "/etc/gcp-keys/key.json"
              volumeMounts:
                - name: gcs-key-volume
                  mountPath: "/etc/gcp-keys"
                  readOnly: true
              resources:
                requests:
                  memory: "1Gi"
                  cpu: "500m"
                limits:
                  memory: "2Gi"
                  cpu: "1"
          restartPolicy: OnFailure

```

## File: `sparse-search-service\deployment.yaml`
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
      volumes: # LLM: ADDED - Definir el volumen desde el Secret
        - name: gcs-key-volume
          secret:
            secretName: sparse-search-gcs-key # Nombre del Secret creado en K8s
      containers:
        - name: sparse-search-service-container
          image: ghcr.io/dev-nyro/sparse-search-service:develop-df4c9b9 # Reemplaza con tu imagen
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
            # LLM: ADDED - Configurar GOOGLE_APPLICATION_CREDENTIALS
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-keys/key.json" # Ruta donde se montará la clave dentro del pod
          volumeMounts: # LLM: ADDED - Montar el volumen en el contenedor
            - name: gcs-key-volume
              mountPath: "/etc/gcp-keys" # Directorio donde se montará el contenido del Secret
              readOnly: true
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

## File: `sparse-search-service\service.yaml`
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

## File: `sparse-search-service\serviceaccount.yaml`
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

```

## File: `ingest-service\configmap.yaml`
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
  # INGEST_POSTGRES_PASSWORD -> Secret

  # Milvus/Zilliz
  INGEST_MILVUS_URI: "https://in03-0afab716eb46d7f.serverless.gcp-us-west1.cloud.zilliz.com" # MODIFIED
  INGEST_MILVUS_COLLECTION_NAME: "document_chunks_minilm"
  INGEST_MILVUS_GRPC_TIMEOUT: "20" # Increased timeout as per consideration for Zilliz Cloud

  # Google Cloud Storage
  INGEST_GCS_BUCKET_NAME: "atenex"

  # Embedding Settings
  INGEST_EMBEDDING_DIMENSION: "384" 

  # Tokenizer Settings
  INGEST_TIKTOKEN_ENCODING_NAME: "cl100k_base"

  # URLs for dependent services
  INGEST_EMBEDDING_SERVICE_URL: "http://embedding-service.atenex.svc.cluster.local:80/api/v1/embed"
  INGEST_DOCPROC_SERVICE_URL: "http://docproc-service.atenex.svc.cluster.local:80/api/v1/process"
```

## File: `ingest-service\deployment-api.yaml`
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
          image: ghcr.io/dev-nyro/ingest-service:develop-d60bccd # Placeholder, CI/CD to update
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
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-secrets/key.json"
            - name: INGEST_ZILLIZ_API_KEY # MODIFIED: Added Zilliz API Key from Secret
              valueFrom:
                secretKeyRef:
                  name: ingest-service-secrets # Name of the k8s Secret
                  key: ZILLIZ_API_KEY # Key within the Secret
          volumeMounts:
            - name: gcp-key-volume
              mountPath: "/etc/gcp-secrets"
              readOnly: true
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1500m"
              memory: "2Gi"
      volumes:
        - name: gcp-key-volume
          secret:
            secretName: gcs-worker-sa-key
            items:
              - key: key.json
                path: key.json

```

## File: `ingest-service\deployment-worker.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingest-worker-deployment
  namespace: atenex
  labels:
    app: ingest-worker
spec:
  replicas: 1
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
          image: ghcr.io/dev-nyro/ingest-service:develop-d60bccd
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
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-secrets/key.json"
            - name: INGEST_ZILLIZ_API_KEY # MODIFIED: Added Zilliz API Key from Secret
              valueFrom:
                secretKeyRef:
                  name: ingest-service-secrets # Name of the k8s Secret
                  key: ZILLIZ_API_KEY # Key within the Secret
          volumeMounts:
            - name: gcp-key-volume
              mountPath: "/etc/gcp-secrets"
              readOnly: true
          resources:
            requests:
              cpu: "1000m" # 1 vCPU
              memory: "3Gi"
            limits:
              cpu: "4000m" # 4 vCPU
              memory: "8Gi"
      volumes:
        - name: gcp-key-volume
          secret:
            secretName: gcs-worker-sa-key
            items:
              - key: key.json
                path: key.json

```

## File: `ingest-service\service-api.yaml`
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

## File: `postgresql\persistent-volume-claim.yaml`
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

## File: `postgresql\service.yaml`
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

## File: `postgresql\statefulset.yaml`
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

## File: `query-service\configmap.yaml`
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
  QUERY_MILVUS_COLLECTION_NAME: "document_chunks_minilm"
  QUERY_MILVUS_EMBEDDING_FIELD: "embedding"
  QUERY_MILVUS_CONTENT_FIELD: "content"
  QUERY_MILVUS_COMPANY_ID_FIELD: "company_id"
  QUERY_MILVUS_DOCUMENT_ID_FIELD: "document_id"
  QUERY_MILVUS_FILENAME_FIELD: "file_name"
  QUERY_MILVUS_GRPC_TIMEOUT: "20" 
  # QUERY_MILVUS_METADATA_FIELDS: (...) # Uses default from config.py

  # Embedding Service (Remote)
  QUERY_EMBEDDING_DIMENSION: "384" # Debe coincidir con el modelo del embedding-service
  QUERY_EMBEDDING_SERVICE_URL: "http://embedding-service.atenex.svc.cluster.local:80"
  QUERY_EMBEDDING_CLIENT_TIMEOUT: "30"

  # Sparse Search Service (Remote BM25)
  QUERY_BM25_ENABLED: "true" # Controla si se usa el paso de búsqueda dispersa (llamada al servicio remoto)
  QUERY_SPARSE_SEARCH_SERVICE_URL: "http://sparse-search-service.atenex.svc.cluster.local:80"
  QUERY_SPARSE_SEARCH_CLIENT_TIMEOUT: "30"

  # Reranker Service (Remote)
  QUERY_RERANKER_ENABLED: "true"
  QUERY_RERANKER_SERVICE_URL: "http://reranker-gpu-service.atenex.svc.cluster.local:8004/" # "http://reranker-service.atenex.svc.cluster.local:80" normal
  QUERY_RERANKER_CLIENT_TIMEOUT: "220"

  # Gemini LLM Settings
  QUERY_GEMINI_MODEL_NAME: "gemini-1.5-flash-latest"
  # QUERY_GEMINI_API_KEY -> Proveniente de Secret

  # RAG Pipeline Control & Parameters
  QUERY_RETRIEVER_TOP_K: "100" # Nº de chunks a pedir a dense y sparse retrievers inicialmente
  QUERY_MAX_CONTEXT_CHUNKS: "75" # Nº máx de chunks para el LLM después de reranking/filtering
  QUERY_MAX_PROMPT_TOKENS: "500000" 
  QUERY_DIVERSITY_FILTER_ENABLED: "true"
  QUERY_DIVERSITY_LAMBDA: "0.5" # Para MMR si está habilitado

  # HTTP Client Settings (Globales, usados por http_client en UseCase para Reranker y otros futuros)
  QUERY_HTTP_CLIENT_TIMEOUT: "60" 
  QUERY_HTTP_CLIENT_MAX_RETRIES: "2"
  QUERY_HTTP_CLIENT_BACKOFF_FACTOR: "1.0"

  # Chat history settings
  QUERY_MAX_CHAT_HISTORY_MESSAGES: "10"
  QUERY_NUM_SOURCES_TO_SHOW: "7"
  
  # MapReduce (Opcional, actualmente gestionado en config.py pero podría moverse aquí)
  QUERY_MAPREDUCE_ENABLED: "true"
  QUERY_MAPREDUCE_CHUNK_BATCH_SIZE: "5"
  QUERY_MAPREDUCE_ACTIVATION_THRESHOLD: "25"
```

## File: `query-service\deployment.yaml`
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
          image: ghcr.io/dev-nyro/query-service:develop-857ff5d # Placeholder, CI/CD to update
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

## File: `query-service\service.yaml`
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

## File: `README.md`
```md
# manifests-nyro
Repositorio de los manifiestos de Nyro B2B

```

## File: `reranker-service\configmap.yaml`
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

## File: `reranker-service\deployment.yaml`
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
          image: ghcr.io/dev-nyro/reranker-service:develop-48c4f60 # Placeholder: replace with actual image path and tag
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

## File: `reranker-service\service.yaml`
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

## File: `reranker_gpu-service\service.yaml`
```yaml
# reranker_gpu-service/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: reranker-gpu-service
  namespace: atenex
spec:
  type: ExternalName
  externalName: host.docker.internal
  ports:
    - port: 80
      targetPort: 8004

```

## File: `sparse-search-service\configmap.yaml`
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

  SPARSE_INDEX_GCS_BUCKET_NAME: "atenex-sparse-indices" # Asegúrate que este bucket exista
  SPARSE_INDEX_CACHE_MAX_ITEMS: "50"
  SPARSE_INDEX_CACHE_TTL_SECONDS: "3600" # 1 hora
```

## File: `sparse-search-service\cronjob.yaml`
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
          volumes:
            - name: gcs-key-volume
              secret:
                secretName: sparse-search-gcs-key
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
                - name: GOOGLE_APPLICATION_CREDENTIALS
                  value: "/etc/gcp-keys/key.json"
              volumeMounts:
                - name: gcs-key-volume
                  mountPath: "/etc/gcp-keys"
                  readOnly: true
              resources:
                requests:
                  memory: "1Gi"
                  cpu: "500m"
                limits:
                  memory: "2Gi"
                  cpu: "1"
          restartPolicy: OnFailure

```

## File: `sparse-search-service\deployment.yaml`
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
      volumes: # LLM: ADDED - Definir el volumen desde el Secret
        - name: gcs-key-volume
          secret:
            secretName: sparse-search-gcs-key # Nombre del Secret creado en K8s
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
            # LLM: ADDED - Configurar GOOGLE_APPLICATION_CREDENTIALS
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-keys/key.json" # Ruta donde se montará la clave dentro del pod
          volumeMounts: # LLM: ADDED - Montar el volumen en el contenedor
            - name: gcs-key-volume
              mountPath: "/etc/gcp-keys" # Directorio donde se montará el contenido del Secret
              readOnly: true
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

## File: `sparse-search-service\service.yaml`
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

## File: `sparse-search-service\serviceaccount.yaml`
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

```

## File: `ingest-service\configmap.yaml`
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
  # INGEST_POSTGRES_PASSWORD -> Secret

  # Milvus/Zilliz
  INGEST_MILVUS_URI: "https://in03-0afab716eb46d7f.serverless.gcp-us-west1.cloud.zilliz.com" # MODIFIED
  INGEST_MILVUS_COLLECTION_NAME: "document_chunks_minilm"
  INGEST_MILVUS_GRPC_TIMEOUT: "20" # Increased timeout as per consideration for Zilliz Cloud

  # Google Cloud Storage
  INGEST_GCS_BUCKET_NAME: "atenex"

  # Embedding Settings
  INGEST_EMBEDDING_DIMENSION: "384" 

  # Tokenizer Settings
  INGEST_TIKTOKEN_ENCODING_NAME: "cl100k_base"

  # URLs for dependent services
  INGEST_EMBEDDING_SERVICE_URL: "http://embedding-service.atenex.svc.cluster.local:80/api/v1/embed"
  INGEST_DOCPROC_SERVICE_URL: "http://docproc-service.atenex.svc.cluster.local:80/api/v1/process"
```

## File: `ingest-service\deployment-api.yaml`
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
          image: ghcr.io/dev-nyro/ingest-service:develop-5be78d9 # Placeholder, CI/CD to update
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
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-secrets/key.json"
            - name: INGEST_ZILLIZ_API_KEY # MODIFIED: Added Zilliz API Key from Secret
              valueFrom:
                secretKeyRef:
                  name: ingest-service-secrets # Name of the k8s Secret
                  key: ZILLIZ_API_KEY # Key within the Secret
          volumeMounts:
            - name: gcp-key-volume
              mountPath: "/etc/gcp-secrets"
              readOnly: true
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1500m"
              memory: "2Gi"
      volumes:
        - name: gcp-key-volume
          secret:
            secretName: gcs-worker-sa-key
            items:
              - key: key.json
                path: key.json

```

## File: `ingest-service\deployment-worker.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingest-worker-deployment
  namespace: atenex
  labels:
    app: ingest-worker
spec:
  replicas: 1
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
          image: ghcr.io/dev-nyro/ingest-service:develop-5be78d9
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
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-secrets/key.json"
            - name: INGEST_ZILLIZ_API_KEY # MODIFIED: Added Zilliz API Key from Secret
              valueFrom:
                secretKeyRef:
                  name: ingest-service-secrets # Name of the k8s Secret
                  key: ZILLIZ_API_KEY # Key within the Secret
          volumeMounts:
            - name: gcp-key-volume
              mountPath: "/etc/gcp-secrets"
              readOnly: true
          resources:
            requests:
              cpu: "1000m" # 1 vCPU
              memory: "3Gi"
            limits:
              cpu: "4000m" # 4 vCPU
              memory: "8Gi"
      volumes:
        - name: gcp-key-volume
          secret:
            secretName: gcs-worker-sa-key
            items:
              - key: key.json
                path: key.json

```

## File: `ingest-service\service-api.yaml`
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

## File: `postgresql\persistent-volume-claim.yaml`
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

## File: `postgresql\service.yaml`
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

## File: `postgresql\statefulset.yaml`
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

## File: `query-service\configmap.yaml`
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
  QUERY_MILVUS_COLLECTION_NAME: "document_chunks_minilm"
  QUERY_MILVUS_EMBEDDING_FIELD: "embedding"
  QUERY_MILVUS_CONTENT_FIELD: "content"
  QUERY_MILVUS_COMPANY_ID_FIELD: "company_id"
  QUERY_MILVUS_DOCUMENT_ID_FIELD: "document_id"
  QUERY_MILVUS_FILENAME_FIELD: "file_name"
  QUERY_MILVUS_GRPC_TIMEOUT: "20" 
  # QUERY_MILVUS_METADATA_FIELDS: (...) # Uses default from config.py

  # Embedding Service (Remote)
  QUERY_EMBEDDING_DIMENSION: "384" # Debe coincidir con el modelo del embedding-service
  QUERY_EMBEDDING_SERVICE_URL: "http://embedding-service.atenex.svc.cluster.local:80"
  QUERY_EMBEDDING_CLIENT_TIMEOUT: "30"

  # Sparse Search Service (Remote BM25)
  QUERY_BM25_ENABLED: "true" # Controla si se usa el paso de búsqueda dispersa (llamada al servicio remoto)
  QUERY_SPARSE_SEARCH_SERVICE_URL: "http://sparse-search-service.atenex.svc.cluster.local:80"
  QUERY_SPARSE_SEARCH_CLIENT_TIMEOUT: "30"

  # Reranker Service (Remote)
  QUERY_RERANKER_ENABLED: "true"
  QUERY_RERANKER_SERVICE_URL: "http://reranker-gpu-service.atenex.svc.cluster.local:8004/" # "http://reranker-service.atenex.svc.cluster.local:80" normal
  QUERY_RERANKER_CLIENT_TIMEOUT: "220"

  # Gemini LLM Settings
  QUERY_GEMINI_MODEL_NAME: "gemini-1.5-flash-latest"
  # QUERY_GEMINI_API_KEY -> Proveniente de Secret

  # RAG Pipeline Control & Parameters
  QUERY_RETRIEVER_TOP_K: "100" # Nº de chunks a pedir a dense y sparse retrievers inicialmente
  QUERY_MAX_CONTEXT_CHUNKS: "75" # Nº máx de chunks para el LLM después de reranking/filtering
  QUERY_MAX_PROMPT_TOKENS: "500000" 
  QUERY_DIVERSITY_FILTER_ENABLED: "true"
  QUERY_DIVERSITY_LAMBDA: "0.5" # Para MMR si está habilitado

  # HTTP Client Settings (Globales, usados por http_client en UseCase para Reranker y otros futuros)
  QUERY_HTTP_CLIENT_TIMEOUT: "60" 
  QUERY_HTTP_CLIENT_MAX_RETRIES: "2"
  QUERY_HTTP_CLIENT_BACKOFF_FACTOR: "1.0"

  # Chat history settings
  QUERY_MAX_CHAT_HISTORY_MESSAGES: "10"
  QUERY_NUM_SOURCES_TO_SHOW: "7"
  
  # MapReduce (Opcional, actualmente gestionado en config.py pero podría moverse aquí)
  QUERY_MAPREDUCE_ENABLED: "true"
  QUERY_MAPREDUCE_CHUNK_BATCH_SIZE: "5"
  QUERY_MAPREDUCE_ACTIVATION_THRESHOLD: "25"
```

## File: `query-service\deployment.yaml`
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
          image: ghcr.io/dev-nyro/query-service:develop-5be78d9 # Placeholder, CI/CD to update
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

## File: `query-service\service.yaml`
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

## File: `README.md`
```md
# manifests-nyro
Repositorio de los manifiestos de Nyro B2B

```

## File: `reranker-service\configmap.yaml`
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

## File: `reranker-service\deployment.yaml`
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
          image: ghcr.io/dev-nyro/reranker-service:develop-9a58400 # Placeholder: replace with actual image path and tag
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

## File: `reranker-service\service.yaml`
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

## File: `reranker_gpu-service\service.yaml`
```yaml
# reranker_gpu-service/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: reranker-gpu-service
  namespace: atenex
spec:
  type: ExternalName
  externalName: host.docker.internal
  ports:
    - port: 80
      targetPort: 8004

```

## File: `sparse-search-service\configmap.yaml`
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

  SPARSE_INDEX_GCS_BUCKET_NAME: "atenex-sparse-indices" # Asegúrate que este bucket exista
  SPARSE_INDEX_CACHE_MAX_ITEMS: "50"
  SPARSE_INDEX_CACHE_TTL_SECONDS: "3600" # 1 hora
```

## File: `sparse-search-service\cronjob.yaml`
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
          volumes:
            - name: gcs-key-volume
              secret:
                secretName: sparse-search-gcs-key
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
                - name: GOOGLE_APPLICATION_CREDENTIALS
                  value: "/etc/gcp-keys/key.json"
              volumeMounts:
                - name: gcs-key-volume
                  mountPath: "/etc/gcp-keys"
                  readOnly: true
              resources:
                requests:
                  memory: "1Gi"
                  cpu: "500m"
                limits:
                  memory: "2Gi"
                  cpu: "1"
          restartPolicy: OnFailure

```

## File: `sparse-search-service\deployment.yaml`
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
      volumes: # LLM: ADDED - Definir el volumen desde el Secret
        - name: gcs-key-volume
          secret:
            secretName: sparse-search-gcs-key # Nombre del Secret creado en K8s
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
            # LLM: ADDED - Configurar GOOGLE_APPLICATION_CREDENTIALS
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-keys/key.json" # Ruta donde se montará la clave dentro del pod
          volumeMounts: # LLM: ADDED - Montar el volumen en el contenedor
            - name: gcs-key-volume
              mountPath: "/etc/gcp-keys" # Directorio donde se montará el contenido del Secret
              readOnly: true
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

## File: `sparse-search-service\service.yaml`
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

## File: `sparse-search-service\serviceaccount.yaml`
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

```

## File: `ingest-service\configmap.yaml`
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
  # INGEST_POSTGRES_PASSWORD -> Secret

  # Milvus/Zilliz
  INGEST_MILVUS_URI: "https://in03-0afab716eb46d7f.serverless.gcp-us-west1.cloud.zilliz.com" # MODIFIED
  INGEST_MILVUS_COLLECTION_NAME: "atenex_collection"
  INGEST_MILVUS_GRPC_TIMEOUT: "20" # Increased timeout as per consideration for Zilliz Cloud

  # Google Cloud Storage
  INGEST_GCS_BUCKET_NAME: "atenex"

  # Embedding Settings
  INGEST_EMBEDDING_DIMENSION: "1536" 

  # Tokenizer Settings
  INGEST_TIKTOKEN_ENCODING_NAME: "cl100k_base"

  # URLs for dependent services
  INGEST_EMBEDDING_SERVICE_URL: "http://embedding-service.atenex.svc.cluster.local:80/api/v1/embed"
  INGEST_DOCPROC_SERVICE_URL: "http://docproc-service.atenex.svc.cluster.local:80/api/v1/process"
```

## File: `ingest-service\deployment-api.yaml`
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
          image: ghcr.io/dev-nyro/ingest-service:develop-1e75db5 # Placeholder, CI/CD to update
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
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-secrets/key.json"
            - name: INGEST_ZILLIZ_API_KEY # MODIFIED: Added Zilliz API Key from Secret
              valueFrom:
                secretKeyRef:
                  name: ingest-service-secrets # Name of the k8s Secret
                  key: ZILLIZ_API_KEY # Key within the Secret
          volumeMounts:
            - name: gcp-key-volume
              mountPath: "/etc/gcp-secrets"
              readOnly: true
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1500m"
              memory: "2Gi"
      volumes:
        - name: gcp-key-volume
          secret:
            secretName: gcs-worker-sa-key
            items:
              - key: key.json
                path: key.json

```

## File: `ingest-service\deployment-worker.yaml`
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
          image: ghcr.io/dev-nyro/ingest-service:develop-1e75db5
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
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-secrets/key.json"
            - name: INGEST_ZILLIZ_API_KEY # MODIFIED: Added Zilliz API Key from Secret
              valueFrom:
                secretKeyRef:
                  name: ingest-service-secrets # Name of the k8s Secret
                  key: ZILLIZ_API_KEY # Key within the Secret
          volumeMounts:
            - name: gcp-key-volume
              mountPath: "/etc/gcp-secrets"
              readOnly: true
          resources:
            requests:
              cpu: "1000m" # 1 vCPU
              memory: "3Gi"
            limits:
              cpu: "4000m" # 4 vCPU
              memory: "8Gi"
      volumes:
        - name: gcp-key-volume
          secret:
            secretName: gcs-worker-sa-key
            items:
              - key: key.json
                path: key.json

```

## File: `ingest-service\service-api.yaml`
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

## File: `postgresql\persistent-volume-claim.yaml`
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

## File: `postgresql\service.yaml`
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

## File: `postgresql\statefulset.yaml`
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

## File: `query-service\configmap.yaml`
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
  QUERY_MILVUS_COLLECTION_NAME: "atenex_collection"
  QUERY_MILVUS_EMBEDDING_FIELD: "embedding"
  QUERY_MILVUS_CONTENT_FIELD: "content"
  QUERY_MILVUS_COMPANY_ID_FIELD: "company_id"
  QUERY_MILVUS_DOCUMENT_ID_FIELD: "document_id"
  QUERY_MILVUS_FILENAME_FIELD: "file_name"
  QUERY_MILVUS_GRPC_TIMEOUT: "20" 
  # QUERY_MILVUS_METADATA_FIELDS: (...) # Uses default from config.py

  # Embedding Service (Remote)
  QUERY_EMBEDDING_DIMENSION: "1536" # Debe coincidir con el modelo del embedding-service
  QUERY_EMBEDDING_SERVICE_URL: "http://embedding-service.atenex.svc.cluster.local:80"
  QUERY_EMBEDDING_CLIENT_TIMEOUT: "30"

  # Sparse Search Service (Remote BM25)
  QUERY_BM25_ENABLED: "true" # Controla si se usa el paso de búsqueda dispersa (llamada al servicio remoto)
  QUERY_SPARSE_SEARCH_SERVICE_URL: "http://sparse-search-service.atenex.svc.cluster.local:80"
  QUERY_SPARSE_SEARCH_CLIENT_TIMEOUT: "30"

  # Reranker Service (Remote)
  QUERY_RERANKER_ENABLED: "true"
  QUERY_RERANKER_SERVICE_URL: "http://host.docker.internal:8004" # "http://reranker-service.atenex.svc.cluster.local:80" normal
  QUERY_RERANKER_CLIENT_TIMEOUT: "220"

  # Gemini LLM Settings
  QUERY_GEMINI_MODEL_NAME: "gemini-1.5-flash-latest"
  # QUERY_GEMINI_API_KEY -> Proveniente de Secret

  # RAG Pipeline Control & Parameters
  QUERY_RETRIEVER_TOP_K: "100" # Nº de chunks a pedir a dense y sparse retrievers inicialmente
  QUERY_MAX_CONTEXT_CHUNKS: "75" # Nº máx de chunks para el LLM después de reranking/filtering
  QUERY_MAX_PROMPT_TOKENS: "500000" 
  QUERY_DIVERSITY_FILTER_ENABLED: "true"
  QUERY_DIVERSITY_LAMBDA: "0.5" # Para MMR si está habilitado

  # HTTP Client Settings (Globales, usados por http_client en UseCase para Reranker y otros futuros)
  QUERY_HTTP_CLIENT_TIMEOUT: "60" 
  QUERY_HTTP_CLIENT_MAX_RETRIES: "2"
  QUERY_HTTP_CLIENT_BACKOFF_FACTOR: "1.0"

  # Chat history settings
  QUERY_MAX_CHAT_HISTORY_MESSAGES: "10"
  QUERY_NUM_SOURCES_TO_SHOW: "7"
  
  # MapReduce (Opcional, actualmente gestionado en config.py pero podría moverse aquí)
  QUERY_MAPREDUCE_ENABLED: "true"
  QUERY_MAPREDUCE_CHUNK_BATCH_SIZE: "5"
  QUERY_MAPREDUCE_ACTIVATION_THRESHOLD: "25"
```

## File: `query-service\deployment.yaml`
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
          image: ghcr.io/dev-nyro/query-service:develop-52e88f7 # Placeholder, CI/CD to update
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

## File: `query-service\service.yaml`
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

## File: `README.md`
```md
# manifests-nyro
Repositorio de los manifiestos de Nyro B2B

```

## File: `reranker-service\configmap.yaml`
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

## File: `reranker-service\deployment.yaml`
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

## File: `reranker-service\service.yaml`
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

## File: `reranker_gpu-service\reranker-gpu.yaml`
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

## File: `sparse-search-service\configmap.yaml`
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

  SPARSE_INDEX_GCS_BUCKET_NAME: "atenex-sparse-indices" # Asegúrate que este bucket exista
  SPARSE_INDEX_CACHE_MAX_ITEMS: "50"
  SPARSE_INDEX_CACHE_TTL_SECONDS: "3600" # 1 hora
```

## File: `sparse-search-service\cronjob.yaml`
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
          volumes:
            - name: gcs-key-volume
              secret:
                secretName: sparse-search-gcs-key
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
                - name: GOOGLE_APPLICATION_CREDENTIALS
                  value: "/etc/gcp-keys/key.json"
              volumeMounts:
                - name: gcs-key-volume
                  mountPath: "/etc/gcp-keys"
                  readOnly: true
              resources:
                requests:
                  memory: "1Gi"
                  cpu: "500m"
                limits:
                  memory: "2Gi"
                  cpu: "1"
          restartPolicy: OnFailure

```

## File: `sparse-search-service\deployment.yaml`
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
      volumes: # LLM: ADDED - Definir el volumen desde el Secret
        - name: gcs-key-volume
          secret:
            secretName: sparse-search-gcs-key # Nombre del Secret creado en K8s
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
            # LLM: ADDED - Configurar GOOGLE_APPLICATION_CREDENTIALS
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/gcp-keys/key.json" # Ruta donde se montará la clave dentro del pod
          volumeMounts: # LLM: ADDED - Montar el volumen en el contenedor
            - name: gcs-key-volume
              mountPath: "/etc/gcp-keys" # Directorio donde se montará el contenido del Secret
              readOnly: true
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

## File: `sparse-search-service\service.yaml`
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

## File: `sparse-search-service\serviceaccount.yaml`
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
