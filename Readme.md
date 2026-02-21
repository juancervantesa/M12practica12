## Enfoque de la práctica
Esta práctica debe ser implementada, no solo diseñada.
El objetivo es aplicar DevSecOps de manera práctica, integrando:
        - - Front-end
        - - Back-end
        - - Inicio de sesión seguro
        - - Arquitectura de microservicios
        - - Automatización CI/CD con seguridad embebida
     

# 1. Adición del Front-end

[ Front-end ]
     |
     | Login / JWT
     v
[ users-service ]
     |
     | JWT
     v
[ api-gateway ]
     |
     v
[ academic-service ]


## Integración DevSecOps (obligatoria)
El Front-end y el inicio de sesión deben estar cubiertos por el pipeline DevSecOps existente:

- SAST: análisis del código de autenticación y manejo de inputs.
- SCA: análisis de dependencias relacionadas con seguridad.
- DAST: pruebas de acceso no autorizado a endpoints protegidos.

**El login no se asume seguro, se valida automáticamente

## Propósito de esta extensión
Consolidar una visión end-to-end DevSecOps, donde:
 - El diseño,
 - La seguridad,
 - La automatización,
  Y la experiencia de usuario,
se integran desde las primeras etapas del desarrollo.

## Pipeline 
Commit / Pull Request
   ↓
Tests automatizados
   ↓
SAST (Semgrep)
   ↓
Build (Docker)
   ↓
SCA (dependencias)
   ↓
Deploy automático
   ↓
DAST (aplicación en ejecución)

## Docker Compose
docker-compose down
docker-compose up --build

## Estructura del Pipeline
Push / Pull Request
   ↓
Install dependencies
   ↓
Tests (backend + frontend)
   ↓
SAST (Semgrep)
   ↓
Build Docker images
   ↓
SCA (Trivy)
   ↓
docker-compose up
   ↓
Smoke tests

## Kubernetes
kubectl apply -f k8s/users-service/
kubectl apply -f k8s/academic-service/
kubectl apply -f k8s/api-gateway/

kubectl get pods
kubectl get services

# Correr api-gateway
minikube service api-gateway
minikube start
# Trabajar con Docker
eval $(minikube docker-env -u)
# Trabajar Docker dentro Kubernetes
1. minikube start --driver=docker
   eval $(minikube docker-env)
2. minikube status
3. kubectl config current-context
4. kubectl get nodes
## Construir las imágenes
docker build -t frontend:latest ../frontend
kubectl get pods -n backend
docker build -t users-service:latest ../backend/users-service

## Desplegar en Kubernetes
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/users-service/
kubectl apply -f k8s/academic-service/
kubectl apply -f k8s/api-gateway/
kubectl apply -f k8s/frontend/

# eliminar cluster
minikube delete
minikube start
eval $(minikube docker-env)

docker build -t users-service backend/users-service
docker build -t academic-service backend/academic-service
docker build -t api-gateway backend/api-gateway
docker build -t frontend frontend

kubectl apply -f k8s/



## 📋 Justificación técnica del Pipeline DevSecOps
A continuación se explica cada etapa del workflow `.github/workflows/devsecops.yml` y las razones detrás de las decisiones técnicas.

1. **Checkout repository**
   - **Herramienta:** `actions/checkout@v4`
   - **Fase DevSecOps:** CI inicial
   - **Riesgo mitigado:** falta de acceso al código fuente en la máquina de runner.
   - **Justificación:** necesario para operar sobre el repositorio; aunque el sistema funcione localmente, el pipeline no puede comenzar sin el código.

2. **Setup Node.js**
   - **Herramienta:** `actions/setup-node@v4` con `node-version: 20`
   - **Fase:** Build/CI
   - **Riesgo mitigado:** diferencias de versión de Node que causen fallos en dependencias o tests.
   - **Por qué:** asegura un entorno consistente entre desarrolladores y CI.

3. **Creación de archivos `.env`**
   - **Herramienta:** comandos shell en el propio job
   - **Fase:** Preparación de entorno
   - **Riesgo:** exposición de variables sensibles o falta de configuración durante pruebas.
   - **Razón:** el pipeline necesita valores de configuración seguros (desde `secrets`) aunque en local se usen ejemplos.

4. **Instalar y ejecutar tests (backend y frontend)**
   - **Herramienta:** `npm ci` y `npm test` para cada microservicio y el frontend
   - **Fase:** Verificación/CI
   - **Riesgo:** compilación rota, regresiones funcionales.
   - **Necesidad:** garantiza que los cambios no rompan la funcionalidad antes de avanzar a etapas de seguridad.

5. **SAST – Análisis estático con Semgrep**
   - **Herramienta:** `semgrep` (instalado vía `pip`)
   - **Fase:** Seguridad temprana (shift-left)
   - **Riesgo mitigado:** código vulnerable (inyección, secretos, malas prácticas).
   - **Por qué:** identificador de problemas de código antes de construir imágenes; incluso con el sistema funcional, el código puede tener vulnerabilidades que pasan desapercibidas en tests.

6. **Validación de env files**
   - **Herramienta:** listado de directorios (`ls -la`)
   - **Fase:** Control de calidad
   - **Riesgo:** .env mal generados o faltantes que podrían hacer fallar aplicaciones.
   - **Por qué:** sanity check sencillo en CI.

7. **Docker Build**
   - **Herramienta:** `docker/setup-buildx-action` y `docker compose build`
   - **Fase:** Build/Packaging
   - **Riesgo:** imágenes rotas, dependencias no incluidas.
   - **Necesidad:** empaquetar el código en contenedores reproducibles para despliegue y escaneo.

8. **SCA – Análisis de composición de software con Trivy**
   - **Herramienta:** `aquasecurity/trivy-action`
   - **Fase:** Seguridad en contenedores
   - **Riesgo mitigado:** bibliotecas con vulnerabilidades conocidas en las imágenes.
   - **Motivo:** el sistema puede funcionar, pero las dependencias pueden contener CVE; Trivy impide la promoción de imágenes inseguras.

9. **Smoke test**
   - **Herramienta:** `docker compose up` + `curl`
   - **Fase:** Validación de despliegue
   - **Riesgo:** fallos de integración entre servicios o problemas de red.
   - **Por qué:** comprobar que los microservicios arrancan y la puerta de enlace responde antes de considerar el build como válido.

10. **Cleanup / Shutdown**
    - **Herramienta:** `docker compose down`
    - **Fase:** Limpieza
    - **Riesgo:** recursos residuales en runner.
    - **Razón:** mantener el runner limpio para posteriores jobs.

> 🔐 _Por qué estas etapas son necesarias incluso con un sistema funcional_:
> Un proyecto puede cumplir sus requisitos funcionales localmente, pero aún presenta riesgos de seguridad, inconsistencias de entorno o dependencias vulnerables. El pipeline DevSecOps introduce defensas automáticas, consistencia y visibilidad a lo largo de todo el ciclo de vida, reduciendo costos de corrección y exposición en producción.

---




