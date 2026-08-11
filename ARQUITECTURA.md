# Arquitectura de la Aplicación — bdget

## Descripción General

**bdget** es un microservicio REST desarrollado con **Spring Boot 3** que expone una API CRUD para gestionar estudiantes. La base de datos es **Oracle Autonomous Database** (nube), y el despliegue es completamente automatizado mediante **GitHub Actions → Docker → AWS EC2**.

---

## Stack Tecnológico

| Capa | Tecnología |
|---|---|
| Lenguaje | Java 17 |
| Framework | Spring Boot 3.3.7 |
| Persistencia | Spring Data JPA + Hibernate |
| Base de datos | Oracle Autonomous DB (ojdbc11 + mTLS) |
| Configuración externa | Spring Cloud Config |
| Contenedor | Docker / Docker Compose |
| CI/CD | GitHub Actions |
| Infraestructura | AWS EC2 |
| Cobertura de tests | JaCoCo |

---

## Diagrama de Capas (Arquitectura en N-Capas)

```
┌──────────────────────────────────────────────────────┐
│                  CLIENTE (HTTP)                      │
│         Navegador / Postman / Frontend               │
└───────────────────────┬──────────────────────────────┘
                        │  HTTP Request (JSON)
                        ▼
┌──────────────────────────────────────────────────────┐
│               CAPA DE PRESENTACIÓN                   │
│           StudentController  (@RestController)       │
│   GET /students   POST /students   PUT /students/{id}│
│   GET /students/{id}              DELETE /students/{id}│
│                                                      │
│   ┌──────────────────────────────────────────────┐   │
│   │  GlobalExceptionHandler (@ControllerAdvice)  │   │
│   │  Captura RuntimeException  → HTTP 400         │   │
│   │  Captura ValidationException → HTTP 400       │   │
│   │  Captura Exception          → HTTP 500         │   │
│   └──────────────────────────────────────────────┘   │
└───────────────────────┬──────────────────────────────┘
                        │  Llamada a interfaz
                        ▼
┌──────────────────────────────────────────────────────┐
│                 CAPA DE NEGOCIO                      │
│   StudentService (interfaz)                          │
│   StudentServiceImpl (@Service)                      │
│                                                      │
│   getAllStudents()   getStudentById(id)               │
│   createStudent()   updateStudent(id, student)       │
│   deleteStudent(id)                                  │
└───────────────────────┬──────────────────────────────┘
                        │  Llamada a repositorio
                        ▼
┌──────────────────────────────────────────────────────┐
│                 CAPA DE DATOS                        │
│   StudentRepository (JpaRepository<Student, Long>)   │
│   Spring Data genera las queries automáticamente     │
└───────────────────────┬──────────────────────────────┘
                        │  JDBC + mTLS (Oracle Wallet)
                        ▼
┌──────────────────────────────────────────────────────┐
│           ORACLE AUTONOMOUS DATABASE (Nube)          │
│   Tabla: student  (id BIGINT PK, name VARCHAR)       │
│   Conexión segura con certificados en:               │
│   Wallet_N72BZHZWYZGTE7OH/                           │
└──────────────────────────────────────────────────────┘
```

---

## Modelo de Datos

### Entidad `Student`

```
┌─────────────────────────┐
│         STUDENT         │
├──────────┬──────────────┤
│ id       │ BIGINT (PK)  │
│ name     │ VARCHAR(50)  │
└──────────┴──────────────┘
```

**Validaciones sobre `name`:**
- No puede estar vacío (`@NotBlank`)
- Entre 2 y 50 caracteres (`@Size`)
- Solo letras (`@Pattern: ^[a-zA-Z]+$`)

---

## API REST — Endpoints

| Método | Endpoint | Descripción | Respuesta OK |
|--------|----------|-------------|--------------|
| `GET` | `/students` | Lista todos los estudiantes | `200 OK` + array JSON |
| `GET` | `/students/{id}` | Obtiene un estudiante por ID | `200 OK` + objeto JSON |
| `POST` | `/students` | Crea un nuevo estudiante | `200 OK` + objeto creado |
| `PUT` | `/students/{id}` | Actualiza un estudiante | `200 OK` + objeto actualizado |
| `DELETE` | `/students/{id}` | Elimina un estudiante | `200 OK` |

**Formato de error (400 / 500):**
```json
{
  "timestamp": "2024-01-15T10:30:00",
  "status": 400,
  "message": "Descripción del error"
}
```

---

## Estructura del Proyecto

```
bdget/
├── src/
│   ├── main/
│   │   ├── java/com/example/bdget/
│   │   │   ├── BdgetApplication.java          ← Punto de entrada (@SpringBootApplication)
│   │   │   ├── controller/
│   │   │   │   └── StudentController.java     ← Endpoints REST
│   │   │   ├── service/
│   │   │   │   ├── StudentService.java        ← Interfaz de negocio
│   │   │   │   └── StudentServiceImpl.java    ← Implementación
│   │   │   ├── repository/
│   │   │   │   └── StudentRepository.java     ← Acceso a datos (JPA)
│   │   │   ├── model/
│   │   │   │   └── Student.java               ← Entidad JPA + validaciones
│   │   │   └── exception/
│   │   │       └── GlobalExceptionHandler.java← Manejo centralizado de errores
│   │   └── resources/
│   │       └── application.properties         ← Configuración (DB, Cloud Config, JPA)
│   └── test/
│       └── java/com/example/bdget/
│           ├── BdgetApplicationTests.java     ← Test de contexto Spring
│           ├── controller/StudentControllerTest.java  ← Tests con MockMvc
│           ├── service/StudentServiceImplTest.java    ← Tests unitarios puros
│           └── model/StudentModelTest.java            ← Tests del modelo
├── Wallet_N72BZHZWYZGTE7OH/                   ← Certificados Oracle mTLS (NO mover)
├── Dockerfile                                 ← Imagen multi-stage (build + runtime)
├── docker-compose.yml                         ← Orquestación local
└── pom.xml                                    ← Dependencias Maven
```

---

## Flujo de Configuración con Spring Cloud Config

```
                  ┌─────────────────────┐
                  │   Config Server     │
                  │  (opcional, :8888)  │
                  └────────┬────────────┘
                           │ intenta conectar al iniciar
                           │ (fail-fast=false → no bloquea)
                  ┌────────▼────────────┐
                  │  bdget application  │
                  │  application.       │
                  │  properties (local) │ ← valores en uso real
                  └─────────────────────┘
```

> Si el Config Server no está disponible, la aplicación inicia igual usando `application.properties`.

---

## Flujo de CI/CD

```
  Developer
     │
     │  git push → branch main
     ▼
┌────────────────────────────────────────────────────────────┐
│                   GitHub Actions                           │
│                                                            │
│  1. Checkout código                                        │
│  2. Inyectar secretos en application.properties (sed)     │
│     DB_URL, DB_USERNAME, DB_PASSWORD                       │
│  3. docker build → imagen con wallet + JAR                 │
│  4. docker push → DockerHub (:latest)                      │
│  5. SSH → EC2                                              │
│     ├── docker pull :latest                                │
│     ├── docker stop my-app && docker rm my-app             │
│     └── docker run -d -p 8080:8080 my-app:latest           │
└────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                          ┌─────────────────┐
                          │   AWS EC2       │
                          │  :8080 expuesto │
                          └─────────────────┘
                                    │
                                    │ JDBC mTLS
                                    ▼
                          ┌─────────────────────────┐
                          │  Oracle Autonomous DB   │
                          │       (Oracle Cloud)    │
                          └─────────────────────────┘
```

---

## Estrategia de Testing

```
┌─────────────────────────────────────────────────────────┐
│                    Pirámide de Tests                    │
│                                                         │
│              ┌──────────────────┐                       │
│              │ BdgetApplication │  @SpringBootTest      │
│              │     Tests        │  (contexto completo)  │
│              └────────┬─────────┘                       │
│           ┌───────────┴──────────────┐                  │
│           │  StudentControllerTest   │  @WebMvcTest      │
│           │  (MockMvc + @MockBean)   │  (slice test)     │
│           └───────────┬──────────────┘                  │
│      ┌────────────────┴───────────────────┐             │
│      │      StudentServiceImplTest        │  Mockito     │
│      │  (@ExtendWith MockitoExtension)    │  (sin Spring)│
│      └────────────────┬───────────────────┘             │
│   ┌───────────────────┴────────────────────────┐        │
│   │          StudentModelTest                   │  JUnit5│
│   │    (plain, sin anotaciones Spring)          │  puro  │
│   └─────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────┘

Cobertura: JaCoCo → target/site/jacoco/index.html
```

---

## Seguridad y Conexión a la Base de Datos

- La conexión a Oracle usa **mTLS** (mutual TLS): tanto el cliente como el servidor se autentican con certificados.
- Los certificados viven en `Wallet_N72BZHZWYZGTE7OH/` y deben estar disponibles en la ruta que apunte `TNS_ADMIN`.
- En producción (Docker/EC2): `TNS_ADMIN=/app/wallet`
- En local: exportar `export TNS_ADMIN=./Wallet_N72BZHZWYZGTE7OH`
- El pool de conexiones es **HikariCP** con máximo 10 conexiones y timeout de 30 segundos.
- El esquema de BD **nunca se modifica automáticamente** (`ddl-auto=none`): cualquier cambio estructural requiere DDL manual.

---

## Puntos Clave para la Clase

| # | Concepto | Dónde se aplica |
|---|----------|-----------------|
| 1 | **Arquitectura en capas** | Controller → Service → Repository → DB |
| 2 | **Inyección de dependencias** | `@Autowired` en Controller y ServiceImpl |
| 3 | **Interfaz + implementación** | `StudentService` / `StudentServiceImpl` |
| 4 | **Spring Data JPA** | `JpaRepository` genera queries automáticamente |
| 5 | **Bean Validation** | `@NotBlank`, `@Size`, `@Pattern` en la entidad |
| 6 | **Manejo global de errores** | `@ControllerAdvice` + `@ExceptionHandler` |
| 7 | **Testing por capas** | MockMvc / Mockito / JUnit5 puro |
| 8 | **CI/CD completo** | GitHub Actions → Docker → DockerHub → EC2 |
| 9 | **Seguridad de BD** | mTLS con Oracle Wallet |
| 10 | **Configuración externalizada** | Spring Cloud Config + secrets en CI |
