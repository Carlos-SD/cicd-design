# CICD-DEMO

Proyecto de demostración de integración y entrega continua (CI/CD) con Jenkins, SonarQube y Trivy.  
Aplicación Spring Boot con pipeline declarativo completo: build, análisis estático, escaneo de seguridad y despliegue local.

---

## Arquitectura del Pipeline

```
Git Push
   │
   ▼
┌─────────────┐
│  Checkout   │  Clona el repositorio desde SCM
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│  Build & Test   │  mvn clean package  (unit tests incluidos)
└──────┬──────────┘
       │
       ▼
┌──────────────┐
│ Docker Build │  docker build -t mi-app:latest .
└──────┬───────┘
       │
       ▼
┌──────────────────────────┐
│ Static Analysis          │  mvn sonar:sonar → SonarQube
│ (SonarQube)              │
└──────┬───────────────────┘
       │
       ▼
┌──────────────────────────┐
│ Quality Gate             │  Falla si:
│                          │  • QG status != OK
│                          │  • Security Hotspots sin revisar > 0
└──────┬───────────────────┘
       │
       ▼
┌──────────────────────────┐
│ Container Security Scan  │  trivy image --severity CRITICAL
│ (Trivy)                  │  Falla si hay vulnerabilidades CRITICAL
└──────┬───────────────────┘
       │
       ▼
┌─────────────┐   (solo en rama master)
│   Deploy    │  docker run -d -p 80:8080 mi-app:latest
└─────────────┘
```

---

## Prerrequisitos

| Herramienta | Versión mínima | Notas |
|-------------|----------------|-------|
| Docker      | 20+            | Requerido para todos los pasos |
| Jenkins     | LTS            | Con plugins: Git, Pipeline, SonarQube Scanner, Docker |
| Trivy       | 0.40+          | Instalado en el agente Jenkins |
| Java / Maven| JDK 12 / 3.6   | O usar el contenedor `builder` de docker-compose |

---

## Levantar la infraestructura local

### 1. Jenkins

```bash
docker run -d \
  --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts
```

> Monta el socket de Docker para que Jenkins pueda ejecutar `docker build` y `docker run`.

Desbloquea Jenkins en `http://localhost:8080` e instala los plugins sugeridos más:
- **SonarQube Scanner**
- **Docker Pipeline**

### 2. SonarQube

```bash
docker-compose up -d sonarqube-db sonarqube
```

Accede a `http://localhost:9000` (admin / admin).  
Crea un proyecto con key `cicd-demo` y genera un token de autenticación.

### 3. Trivy (en el agente Jenkins)

```bash
# macOS
brew install aquasecurity/trivy/trivy

# Linux
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
```

---

## Configuración en Jenkins

### Credenciales necesarias

| ID en Jenkins       | Tipo   | Descripción                         |
|---------------------|--------|-------------------------------------|
| `SONAR_AUTH_TOKEN`  | Secret | Token de autenticación de SonarQube |

### SonarQube Server

En **Manage Jenkins → Configure System → SonarQube servers**:
- Name: `SonarQube`
- URL: `http://sonarqube:9000`
- Token: la credencial `SONAR_AUTH_TOKEN`

> Si Jenkins y SonarQube están en la misma red Docker, usa el nombre de servicio `sonarqube`. Si corren por separado, usa `http://localhost:9000`.

### Configurar el Pipeline

1. Crea un nuevo job tipo **Pipeline**.
2. En *Pipeline Definition* selecciona **Pipeline script from SCM**.
3. SCM: Git → URL del repositorio.
4. Script Path: `Jenkinsfile`.

---

## Puertas de Calidad (Quality Gates)

El pipeline falla automáticamente en dos escenarios:

| Herramienta | Condición de fallo |
|-------------|-------------------|
| SonarQube   | Quality Gate con estado distinto a `OK` **o** al menos 1 Security Hotspot sin revisar |
| Trivy       | Al menos 1 vulnerabilidad de severidad `CRITICAL` en la imagen Docker |

---

## Prueba del Pipeline

1. Haz un cambio en el código (p. ej. agrega código con deuda técnica en `ApiController.java`).
2. Haz commit y push:

```bash
git add .
git commit -m "test: add code change to trigger pipeline"
git push origin master
```

3. Jenkins detecta el cambio, ejecuta el pipeline y despliega si pasa todas las validaciones.
4. Verifica la aplicación en `http://localhost:80/api`.

---

## Bloque `post` y limpieza

El pipeline limpia el workspace al finalizar (éxito o falla) con `cleanWs()`.  
En caso de falla, el bloque `post { failure { ... } }` registra el error en consola.  
Para notificaciones por correo, descomenta la línea `mail to:` en el Jenkinsfile y configura el servidor SMTP en Jenkins.

---

## Estructura del proyecto

```
cicd-design/
├── Jenkinsfile            # Pipeline declarativo completo
├── Dockerfile             # Imagen Docker de la aplicación
├── docker-compose.yml     # Jenkins, SonarQube, builder, selenium
├── Makefile               # Tareas de build y despliegue
├── pom.xml                # Dependencias Maven (Spring Boot, JaCoCo, Sonar)
├── src/
│   ├── main/              # Código fuente Spring Boot
│   └── test/              # Tests unitarios e integración
└── k8s-config/            # Manifiestos Kubernetes (despliegue en cluster)
```
