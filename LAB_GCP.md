# LAB: Jenkins en una VM de GCP + Artifact Registry + Cloud Run

Objetivo: que el [Jenkinsfile](Jenkinsfile) se autentique en GCP **sin llaves JSON ni OIDC**, usando la Service Account asociada a la VM donde corre Jenkins (credenciales del servidor de metadatos).

```mermaid
sequenceDiagram
    participant J as Jenkins (VM GCP)
    participant M as Metadata server
    participant AR as Artifact Registry
    participant CR as Cloud Run
    J->>M: Pide token de la SA de la VM
    M-->>J: Access token (corta duración)
    J->>AR: docker push
    J->>CR: gcloud run deploy
```

El contenedor `google/cloud-sdk` de las etapas del pipeline hereda esa identidad: `gcloud` consulta el metadata server automáticamente.

## 1. Variables (Cloud Shell)

```bash
export PROJECT_ID="sanbox-aldo-prod"
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
export REGION="us-central1"
export ZONE="us-central1-a"
export AR_REPO="container-repository-gemini-at"
export VM_NAME="jenkins-vm"                 # nombre de tu VM
export SA_NAME="jenkins-deployer"
export SA_EMAIL="$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com"
```

## 2. APIs

```bash
gcloud services enable artifactregistry.googleapis.com run.googleapis.com \
  iam.googleapis.com cloudresourcemanager.googleapis.com --project $PROJECT_ID
```

## 3. Repositorio de Artifact Registry (si no existe)

```bash
gcloud artifacts repositories create $AR_REPO \
  --project=$PROJECT_ID --location=$REGION --repository-format=docker
```

## 4. Service Account y permisos

```bash
gcloud iam service-accounts create $SA_NAME --project=$PROJECT_ID \
  --display-name="Jenkins deployer"

# Push de imágenes
gcloud artifacts repositories add-iam-policy-binding $AR_REPO \
  --project=$PROJECT_ID --location=$REGION \
  --member="serviceAccount:$SA_EMAIL" --role="roles/artifactregistry.writer"

# Deploy en Cloud Run
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SA_EMAIL" --role="roles/run.admin"

# Cloud Run ejecuta con la SA por defecto de Compute: permitir actuar como ella
gcloud iam service-accounts add-iam-policy-binding \
  $PROJECT_NUMBER-compute@developer.gserviceaccount.com \
  --project=$PROJECT_ID \
  --member="serviceAccount:$SA_EMAIL" --role="roles/iam.serviceAccountUser"
```

## 5. Asociar la SA a la VM de Jenkins

Hay que detener la VM para cambiar su service account.

```bash
gcloud compute instances stop $VM_NAME --zone=$ZONE --project=$PROJECT_ID

gcloud compute instances set-service-account $VM_NAME \
  --zone=$ZONE --project=$PROJECT_ID \
  --service-account=$SA_EMAIL \
  --scopes=https://www.googleapis.com/auth/cloud-platform

gcloud compute instances start $VM_NAME --zone=$ZONE --project=$PROJECT_ID
```

El scope `cloud-platform` es necesario: con los scopes por defecto el push o el deploy fallan aunque la SA tenga los roles.

## 6. Preparar la VM

Conectarse (`gcloud compute ssh $VM_NAME --zone=$ZONE`) y verificar:

```bash
# Identidad de la VM
curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/email

# Docker disponible para el usuario de Jenkins
sudo usermod -aG docker jenkins && sudo systemctl restart jenkins
```

Autenticar el Docker **del host** contra Artifact Registry (el `docker push` de la etapa *Build and Push Image* corre en el host, no en el contenedor de `cloud-sdk`). Requiere `gcloud` instalado en la VM:

```bash
sudo -u jenkins gcloud auth configure-docker us-central1-docker.pkg.dev --quiet
```

Esto usa la SA de la VM y se hace una sola vez.

## 7. Configuración en Jenkins

1. Plugin **Docker Pipeline** instalado (para `agent { docker {...} }`).
2. Nodo/agente con acceso a `/var/run/docker.sock` (Jenkins en la misma VM).
3. Crear el job (Pipeline o Multibranch) apuntando a este repo; usa el `Jenkinsfile` de la raíz.
4. No se necesitan credenciales de GCP en Jenkins.

## 8. Cómo lo usa el pipeline

- **GCP & Docker Auth**: `gcloud config set project` y `configure-docker` dentro del contenedor `cloud-sdk`.
- **Build and Push Image**: `docker build/push` en el host con la configuración del paso 6.
- **Deploy to Cloud Run**: `gcloud run deploy` en un contenedor `cloud-sdk`, autenticado por el metadata server.

## 9. Verificación y troubleshooting

```bash
gcloud compute instances describe $VM_NAME --zone=$ZONE --project=$PROJECT_ID \
  --format="value(serviceAccounts[0].email,serviceAccounts[0].scopes)"
```

| Error | Causa |
|---|---|
| `Request had insufficient authentication scopes` | La VM no tiene el scope `cloud-platform` (sección 5) |
| `denied: Permission artifactregistry.repositories.uploadArtifacts` | Falta `artifactregistry.writer` o falta `configure-docker` en el host (sección 6) |
| `unauthenticated: ... docker login` en el push | `configure-docker` no se ejecutó con el usuario `jenkins` |
| `iam.serviceaccounts.actAs` en deploy | Falta `serviceAccountUser` sobre la SA de runtime |
| `Permission 'run.services.create' denied` | Falta `roles/run.admin` |
| `permission denied ... docker.sock` | El usuario `jenkins` no está en el grupo `docker` |
| `gcloud` en el contenedor pide login | El contenedor no alcanza el metadata server; no usar `--network none` |

## Alternativa: OIDC / Workload Identity Federation

Solo es necesaria si Jenkins corre **fuera** de GCP (otra nube u on-premise). Requiere el plugin *OpenID Connect Provider* y un issuer `https://` (real, o ficticio con `--jwk-json-path`). Dentro de GCP la SA asociada a la VM es más simple y no necesita tokens que mantener.
