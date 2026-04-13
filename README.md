# microservices-demo-infra

Repositorio de **operaciones e infraestructura** para el proyecto [microservices-demo](https://github.com/JuanAmor8/microservices-demo).

Este repo contiene todos los Helm charts, configuraciones por ambiente y el pipeline de despliegue (CD). Sigue el patrón **GitOps**: la infraestructura se gestiona como código versionado.

---

## Estructura del Repositorio

```
microservices-demo-infra/
├── .github/
│   └── workflows/
│       └── infra-cd.yml          # Pipeline de CD (Helm lint + deploy)
├── charts/
│   ├── infrastructure/           # Kafka + PostgreSQL
│   ├── vote/                     # Helm chart del servicio vote
│   ├── worker/                   # Helm chart del servicio worker
│   └── result/                   # Helm chart del servicio result
├── environments/
│   ├── production.yaml           # Imágenes y config de producción
│   └── staging.yaml              # Imágenes y config de staging
└── README.md
```

---

## Cómo se Activa el Pipeline

El pipeline `infra-cd.yml` se dispara de **tres formas**:

### 1. Automático — desde el repo de desarrollo
Cuando el repo [microservices-demo](https://github.com/JuanAmor8/microservices-demo) termina de construir y publicar una imagen Docker, envía un `repository_dispatch` a este repo con el tipo:
- `vote-image-updated`
- `worker-image-updated`
- `result-image-updated`

### 2. Cambios directos en los charts
Un push a `infra/staging` o `main` con cambios en `charts/` o `environments/` dispara el pipeline.

### 3. Manual (workflow_dispatch)
Desde la pestaña **Actions** en GitHub → `Infrastructure - CD Pipeline` → `Run workflow`.
Puedes seleccionar el ambiente (`staging` / `production`) y el servicio específico.

---

## Flujo GitOps

```
microservices-demo (dev repo)
  │
  │ 1. Developer hace push a main/develop
  │ 2. CI: test → build Docker → push a GHCR
  │ 3. repository_dispatch (vote-image-updated)
  │
  ▼
microservices-demo-infra (este repo)
  │
  │ 4. infra-cd.yml se activa
  │ 5. Helm lint & validate
  │ 6. Deploy a staging (automático)
  │ 7. Deploy a producción (requiere aprobación manual)
  │
  ▼
Kubernetes / Okteto (cluster)
```

---

## Ambientes

| Ambiente   | Branch          | Deploy       | Aprobación |
|------------|-----------------|--------------|------------|
| Staging    | `infra/staging` | Automático   | No         |
| Production | `main`          | Automático   | Sí (GitHub Environments) |

---

## Estrategia de Branching (Operaciones)

Ver documento completo: [BRANCHING.md](https://github.com/JuanAmor8/microservices-demo/blob/main/docs/BRANCHING.md)

```
infra/feature/* ──► infra/staging ──► main
                         │                │
                   Deploy auto       Deploy con
                   a staging         aprobación manual
```

---

## Secrets Requeridos

| Secret         | Descripción                                              |
|----------------|----------------------------------------------------------|
| `KUBECONFIG`   | Kubeconfig en base64 para conectar al cluster            |
| `OKTETO_TOKEN` | Token de Okteto (alternativa a kubeconfig)               |

> **Nota**: El `GITHUB_TOKEN` se genera automáticamente y no necesita configuración.

---

## Deploy Manual Local

```bash
# Prerrequisitos: helm, kubectl con cluster configurado

# 1. Infraestructura (Kafka + PostgreSQL)
helm upgrade --install infrastructure charts/infrastructure/

# 2. Servicios de aplicación
helm upgrade --install vote    charts/vote/    --set image=ghcr.io/juanamor8/microservices-demo/vote:latest
helm upgrade --install worker  charts/worker/  --set image=ghcr.io/juanamor8/microservices-demo/worker:latest
helm upgrade --install result  charts/result/  --set image=ghcr.io/juanamor8/microservices-demo/result:latest
```

---

## Relación con Repo de Desarrollo

| Aspecto          | microservices-demo (dev)         | microservices-demo-infra (ops) |
|------------------|----------------------------------|--------------------------------|
| Contenido        | Código fuente, Dockerfiles       | Helm charts, values, manifests |
| Pipeline         | CI: test + build + push imagen   | CD: lint + deploy              |
| Acceso           | Todos los desarrolladores        | Solo equipo de operaciones     |
| Branching        | Git Flow (feature/develop/main)  | Environment branches           |
| Trigger de deploy| `repository_dispatch` al infra   | Push a `infra/staging` o `main` |
