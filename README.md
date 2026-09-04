# Tienda Perritos EKS

Aplicación CRUD de tienda de alimentos para perritos desplegada en Amazon EKS.

## Arquitectura

```
Usuario → ALB (puerto 80)
              ↓
         Frontend (Nginx, 2 réplicas, LoadBalancer)
              ↓ proxy_pass http://tienda-backend:3001
         Backend (Node.js, 2 réplicas, ClusterIP)
              ↓
         MySQL (1 réplica, Headless Service)
```

### Componentes

| Componente | Tecnología | Puerto |
|---|---|---|
| Frontend | Nginx + HTML/JS | 80 |
| Backend | Node.js + Express | 3001 |
| Base de Datos | MySQL 8 | 3306 |

## Infraestructura AWS

### Cluster EKS
- **Nombre:** `tienda-perritos`
- **Versión Kubernetes:** 1.31
- **Nodegroup:** `tienda-nodes` (2x t3.medium, escalable 1-3)
- **Región:** `us-east-1`

### VPC y Redes
- **VPC:** `vpc-0ab3b425dce2f5f74`
- **Subnets:** 5 subnets públicas en `us-east-1a, b, c, d, f`
- **Internet Gateway:** Conectado a la VPC
- **Security Groups:** Administrados por EKS (control plane + nodos)

### Roles IAM
- **Cluster Role:** `LabEksClusterRole` — permisos para que EKS cree recursos (ELB, EC2, etc.)
- **Node Role:** `LabEksNodeRole` — permisos para que los nodos worker se conecten al cluster, ECR, CloudWatch

### Container Registry (ECR)
| Repositorio | URL |
|---|---|
| tienda-frontend | `933499830430.dkr.ecr.us-east-1.amazonaws.com/tienda-frontend` |
| tienda-backend | `933499830430.dkr.ecr.us-east-1.amazonaws.com/tienda-backend` |
| tienda-db | `933499830430.dkr.ecr.us-east-1.amazonaws.com/tienda-db` |

## Despliegue de Servicios

### Frontend
- **Tipo:** LoadBalancer (público)
- **URL:** `http://a3d8a2621fdeb474685bd9e62918ee17-440057208.us-east-1.elb.amazonaws.com`
- **Réplicas:** 2
- **Recursos:** request 50m CPU / 64Mi, limit 300m CPU / 256Mi

### Backend
- **Tipo:** ClusterIP (interno)
- **Puerto:** 3001
- **Réplicas:** 2
- **Variables de entorno:**
  - `DB_HOST=tienda-db` (DNS interno)
  - `DB_USER=root`
  - `DB_PASSWORD` (desde Secret)
  - `DB_NAME=tienda_perritos`
  - `DB_PORT=3306`
- **Recursos:** request 100m CPU / 128Mi, limit 500m CPU / 512Mi

### Comunicación Frontend → Backend
El frontend usa Nginx como proxy reverso. En `default.conf`:

```
location /api/ {
    proxy_pass http://tienda-backend:3001;
}
```

Esto permite que el frontend consuma la API mediante llamadas a `/api/productos` sin exponer el backend directamente.

## Autoscaling (HPA)

| Deployment | Métrica | Target | Réplicas min/máx |
|---|---|---|---|
| Backend | CPU | 70% | 2 / 10 |
| Frontend | CPU | 60% | 2 / 6 |

### Justificación de valores

- **Backend 70%:** Node.js maneja carga transaccional (consultas a MySQL). Al escalar al 70% evitamos escalados prematuros ante picos breves, optimizando costo.
- **Frontend 60%:** Nginx es liviano y maneja conexiones concurrentes. Al escalar más rápido (60%) aseguramos buena experiencia de usuario ante aumento de tráfico.

## Pipeline CI/CD

### Archivo: `.github/workflows/deploy-eks.yml`

**Trigger:** Push a `main` o ejecución manual (`workflow_dispatch`).

**Flujo:**
1. Checkout del código
2. Configura credenciales AWS (GitHub Secrets)
3. Login a ECR
4. Build & push de las 3 imágenes Docker a ECR (tag: SHA del commit + latest)
5. Reemplaza `{{ECR_URL}}` en los manifests por la URL real del ECR
6. Despliega en orden: MySQL → Backend → Frontend
7. Aplica HPA
8. Muestra URL pública del frontend

### Configuración requerida
En GitHub → Settings → Secrets → Actions, agregar:
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_SESSION_TOKEN`

## Pruebas Realizadas

### Acceso público
```
GET http://a3d8a2621fdeb474685bd9e62918ee17-440057208.us-east-1.elb.amazonaws.com/
→ 200 OK

GET http://a3d8a2621fdeb474685bd9e62918ee17-440057208.us-east-1.elb.amazonaws.com/api/productos
→ 200 OK
→ [
    {"id":1,"nombre":"Alimento Cachorro Premium","precio":"19990.00","stock":15},
    {"id":2,"nombre":"Alimento Adulto Light","precio":"17990.00","stock":8},
    ...
  ]
```

### Logs
```bash
kubectl logs -n tienda -l app=tienda-backend --tail=50
kubectl logs -n tienda -l app=tienda-frontend --tail=50
```

### Recuperación ante redeploy
```bash
kubectl rollout restart deployment tienda-frontend -n tienda
kubectl rollout status deployment tienda-frontend -n tienda
# Los pods se reemplazan sin downtime
```

## Problemas Encontrados

1. **Pipeline falló por falta de AWS Secrets** — Solución: configurar `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` y `AWS_SESSION_TOKEN` en GitHub Secrets.
2. **Pod no actualizaba imagen `:latest` tras pipeline** — Solución: ejecutar `kubectl rollout restart` para forzar el uso de la nueva imagen.
3. **Workflow mostraba commits de repositorio original** — Solución: crear rama orphan con historial limpio y forzar push.

---

## Trabajo colaborativo y GitFlow

### Estrategia de ramificación

Usamos **GitFlow** para separar el desarrollo de features de la versión estable.

```
main      ●────────────────────────────────●──── (estable, solo releases)
           \                              /
develop    ●────────────────●────●───────●────── (integración)
            \              /      \     /
             feature/modo-oscuro    feature/busqueda-productos
                                    
hotfix/validacion-stock-precio → main y develop (corrección urgente)
```

### Ramas del proyecto

| Rama | Nace desde | Se fusiona hacia | Propósito |
|---|---|---|---|
| `main` | — | — | Versión estable y desplegable |
| `develop` | `main` | `main` (release) | Integración de features |
| `feature/<nombre>` | `develop` | `develop` | Nueva funcionalidad |
| `hotfix/<nombre>` | `main` | `main` y `develop` | Corrección urgente en producción |

### Naming de ramas

- Features: `feature/<nombre-corto-del-cambio>` (ej: `feature/modo-oscuro`).
- Hotfixes: `hotfix/<falla-a-corregir>` (ej: `hotfix/validacion-stock-precio`).
- Sin espacios, en minúsculas, usando guiones.

### Convenciones de commits

Formato: `<prefijo>: <descripción en español, imperativo>`

| Prefijo | Uso | Ejemplo |
|---|---|---|
| `feat` | Nueva funcionalidad | `feat: agregar modo oscuro` |
| `fix` | Corrección de bug | `fix: rechazar precio negativo` |
| `ci` | Cambios en CI/CD | `ci: agregar workflow en develop` |
| `docs` | Documentación | `docs: agregar convenciones del equipo` |
| `test` | Pruebas | `test: agregar validación básica` |
| `merge` | Fusión de ramas | `merge: resolver conflicto entre features` |

### Flujo de merge mediante pull requests

1. Crear la rama desde `develop`: `git switch develop` → `git switch -c feature/<nombre>`.
2. Desarrollar y commitear con la convención descrita.
3. Abrir un **PR hacia `develop`** describiendo el cambio.
4. Esperar que el **CI quede en verde** (corre en cada push a `develop` y en cada PR hacia `main`).
5. La pareja revisa el PR, hace comentarios si es necesario y aprueba.
6. Fusionar con **merge commit** (`--no-ff`) y eliminar la rama de feature.
7. Releases: PR de `develop` → `main`. Urgencias: `hotfix` desde `main` hacia `main` y `develop`.

### Estrategia de revisión

- Todo cambio entra por **pull request**; nunca se hace push directo a `main` ni a `develop`.
- Mínimo **un revisor** (la pareja) antes de aprobar.
- El CI debe estar **verde** antes del merge.
- Lista de chequeo del revisor: la funcionalidad se probó, no rompe lo existente, el mensaje de commit sigue la convención y no hay secretos ni credenciales en el código.

### Justificación de la estrategia (redacción del equipo)

<!-- IMPORTANTE: Esta sección debe ser redactada por el equipo, sin usar IA. -->
Guía para redactarla (3 a 5 párrafos):
1. ¿Por qué GitFlow y no trunk-based para este proyecto? Consideren: equipo de 2 personas, necesidad de ramas paralelas, urgencias sobre una versión estable, experiencia previa.
2. ¿Qué problemas resuelve tener `main` siempre estable y `develop` como integración?
3. ¿Cuándo usarían una feature y cuándo un hotfix?
4. ¿Qué convención de commits les pareció más útil y por qué?

Reflexión personal de cada integrante (obligatoria, sin IA): qué aprendieron y cuál fue su aporte en este encargo.
