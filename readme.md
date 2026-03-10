# DOCUMENTO DE ARQUITECTURA MAESTRA: IA SOBERANA PARA EPM (END-TO-END)

## FASE 1: CAPA 0 - INFRAESTRUCTURA Y HARDENING (OCI)
*Configuración de seguridad a nivel de nube antes de tocar Kubernetes.*

### A. IAM & Resource Principals (Seguridad de Identidad)
Para eliminar el uso de llaves `.pem` o archivos de configuración, se debe crear un Dynamic Group y asignarle políticas.

1. **Definición del Dynamic Group:**
   * **Nombre:** `EPM_OKE_Nodes`
   * **Matching Rule (Regla de coincidencia):** Esto agrupa automáticamente a los nodos de OKE.
     `ALL {instance.compartment.id = '<OCID_COMPARTMENT_EPM>'}`

2. **Políticas de Acceso Soberano a la IA:**
   Aplica estas políticas en el compartimento raíz de EPM:
```hcl  
Allow dynamic-group EPM_OKE_Nodes to use generative-ai-family in compartment id <OCID_COMPARTMENT_EPM>  
Allow dynamic-group EPM_OKE_Nodes to use keys in compartment id <OCID_COMPARTMENT_EPM>  
Allow dynamic-group EPM_OKE_Nodes to read secret-bundles in compartment id <OCID_COMPARTMENT_EPM>  
```

### B. Cifrado de Almacenamiento en Reposo (KMS)
Antes de crear el Node Pool del clúster OKE, se debe garantizar que EPM es dueño de los datos físicos:
1. **Vault:** Crea un OCI Vault llamado `EPM-Security-Vault`.
2. **Master Key:** Crea una llave AES-256 llamada `EPM-Disk-Encryption`.
3. **Boot Volumes (Nodos OKE):** Al aprovisionar los nodos worker, en la sección de "Boot Volume", selecciona **"Use Customer-Managed Keys"** y elige la llave `EPM-Disk-Encryption`.

### C. Seguridad de Imágenes (Scanning & Verification)
1. **Vulnerability Scanning (OCIR):** Activa la opción de "Escaneo automático" en el repositorio de Oracle Cloud Infrastructure Registry. El pipeline de CI/CD debe estar configurado para abortar el pase a producción si el reporte no muestra "0 Vulnerabilities" (bloqueando vulnerabilidades Críticas/Altas).
2. **Image Verification (OCI Image Trust):** Registra tu llave pública (GPG) en el clúster OKE. Configura el Admission Controller de Kubernetes para que **solo las imágenes firmadas digitalmente** por el equipo de seguridad de EPM puedan ejecutarse en los nodos.

---

## FASE 2: CAPA 1 - CONSTRUCCIÓN (DOCKERFILES & APP)
*Código endurecido (Hardened) para Front y Back.*

### A. Backend IA (app.py + Dockerfile)
**Archivo:** `app.py` - Conexión Soberana
```python  
import oci, os  
from fastapi import FastAPI  
from oci.generative_ai_inference import GenerativeAiInferenceClient  

app = FastAPI()  

# Sin credenciales: usa el hardware del nodo (Resource Principal)  
signer = oci.auth.signers.get_resource_principals_signer()  
genai_client = GenerativeAiInferenceClient(config={}, signer=signer)  

@app.post("/api/ask")  
async def ask(prompt: str):  
    return {"status": "Encrypted", "response": "IA Soberana respondiendo..."}  
```

**Archivo:** `Dockerfile.backend`
```dockerfile  
FROM python:3.11-slim  
WORKDIR /app  
RUN groupadd -g 1000 python && useradd -u 1000 -g python python  
RUN pip install --no-cache-dir fastapi uvicorn oci  
COPY app.py .  
USER 1000  
EXPOSE 8000  
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]  
```

### B. Frontend UI (Dockerfile.frontend)
**Archivo:** `Dockerfile.frontend`
```dockerfile  
FROM nginx:alpine  
RUN rm /etc/nginx/conf.d/default.conf  
RUN echo 'server { listen 80; server_tokens off; location / { root /usr/share/nginx/html; index index.html; } }' > /etc/nginx/conf.d/security.conf  
COPY ./dist /usr/share/nginx/html  
RUN touch /var/run/nginx.pid && chown -R nginx:nginx /var/run/nginx.pid /var/cache/nginx  
USER nginx  
EXPOSE 80  
```

---

## FASE 3: CAPA 2 - EL MANIFIESTO MAESTRO DE KUBERNETES (`epm-full-stack.yaml`)
*Este es el cerebro de la operación. Copia esto en un solo archivo.*

```yaml  
# 1. NAMESPACE E ISTIO  
apiVersion: v1  
kind: Namespace  
metadata:  
  name: epm-intelligence  
  labels: { istio-injection: enabled }  
---  
# 2. SEGURIDAD DE RED (Zero-Trust K8s NetworkPolicy)  
apiVersion: networking.k8s.io/v1  
kind: NetworkPolicy  
metadata:  
  name: default-deny-all  
  namespace: epm-intelligence  
spec:  
  podSelector: {}  
  policyTypes: ["Ingress", "Egress"]  
---  
# 3. CIFRADO mTLS (Strict Mode Istio)  
apiVersion: security.istio.io/v1beta1  
kind: PeerAuthentication  
metadata:  
  name: default  
  namespace: epm-intelligence  
spec:  
  mtls: { mode: STRICT }  
---  
# 4. IDENTIDAD Y PERMISOS DE APLICACIÓN (ServiceAccounts y Istio AuthPolicy)  
apiVersion: v1  
kind: ServiceAccount  
metadata:  
  name: epm-frontend-sa  
  namespace: epm-intelligence  
---  
apiVersion: v1  
kind: ServiceAccount  
metadata:  
  name: epm-ai-sa  
  namespace: epm-intelligence  
---  
apiVersion: security.istio.io/v1beta1  
kind: AuthorizationPolicy  
metadata:  
  name: backend-access-policy  
  namespace: epm-intelligence  
spec:  
  selector: { matchLabels: { app: backend-ai } }  
  action: ALLOW  
  rules:  
  - from:  
    - source:  
        principals: ["cluster.local/ns/epm-intelligence/sa/epm-frontend-sa"]  
---  
# 5. GATEWAY DE ENTRADA E INGRESS (Istio Gateway)  
apiVersion: networking.istio.io/v1alpha3  
kind: Gateway  
metadata:  
  name: epm-gateway  
  namespace: epm-intelligence  
spec:  
  selector: { istio: ingressgateway }  
  servers:  
  - port: { number: 80, name: http, protocol: HTTP }  
    hosts: ["*"]  
---  
# 6. ENRUTAMIENTO (VirtualService)  
apiVersion: networking.istio.io/v1alpha3  
kind: VirtualService  
metadata:  
  name: epm-routing  
  namespace: epm-intelligence  
spec:  
  hosts: ["*"]  
  gateways: ["epm-gateway"]  
  http:  
  - match: [{ uri: { prefix: "/api" } }]  
    route: [{ destination: { host: backend-ai-svc, port: { number: 8000 } } }]  
  - route: [{ destination: { host: frontend-ai-svc, port: { number: 80 } } }]  
---  
# 7. DESPLIEGUE BACKEND E INTELIGENCIA  
apiVersion: apps/v1  
kind: Deployment  
metadata:  
  name: backend-ai  
  namespace: epm-intelligence  
spec:  
  replicas: 2  
  selector: { matchLabels: { app: backend-ai } }  
  template:  
    metadata: { labels: { app: backend-ai } }  
    spec:  
      serviceAccountName: epm-ai-sa  
      securityContext: { runAsNonRoot: true, runAsUser: 1000 }  
      containers:  
      - name: backend  
        image: bog.ocir.io/tenancy/repo/backend:v5-secure  
        env: [{ name: OCI_RESOURCE_PRINCIPAL, value: "True" }]  
        resources: { limits: { cpu: "500m", memory: "512Mi" } }  
---  
apiVersion: v1  
kind: Service  
metadata:  
  name: backend-ai-svc  
  namespace: epm-intelligence  
spec:  
  ports: [{ port: 8000, name: http }]  
  selector: { app: backend-ai }  
---  
# 8. DESPLIEGUE FRONTEND UI  
apiVersion: apps/v1  
kind: Deployment  
metadata:  
  name: frontend-ai  
  namespace: epm-intelligence  
spec:  
  replicas: 2  
  selector: { matchLabels: { app: frontend-ai } }  
  template:  
    metadata: { labels: { app: frontend-ai } }  
    spec:  
      serviceAccountName: epm-frontend-sa  
      securityContext: { runAsNonRoot: true }  
      containers:  
      - name: web-interface  
        image: bog.ocir.io/tenancy/repo/frontend:v5-secure  
---  
apiVersion: v1  
kind: Service  
metadata:  
  name: frontend-ai-svc  
  namespace: epm-intelligence  
spec:  
  ports: [{ port: 80, name: http }]  
  selector: { app: frontend-ai }  
```
