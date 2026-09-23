# ExamForge — Especificación del proyecto (Backend)

> Documento de referencia para el equipo y para Claude Code. No es el README final del curso.

## 0. Datos generales
- **Curso:** CS 2031 Desarrollo Basado en Plataforma — UTEC
- **Entrega:** Semana 7, Backend final — viernes 25 de septiembre, 11:59 p. m.
- **Equipo:** Lucia Rodriguez (202310459), Samira Rincon (202220436), Valeria Briceño (202310513)
- **Repositorio:** `valubz/ExamForge_Backend`

## 1. Problema y solución
**Problema:** estudiantes y docentes trabajan con PDFs, diapositivas y lecturas dispersas y necesitan convertir ese material en práctica evaluativa confiable. Las herramientas generativas actuales crean preguntas, pero no organizan el contenido por curso, no conservan trazabilidad de las fuentes y no ofrecen un espacio para editar, resolver, publicar y reutilizar evaluaciones.

**Solución:** plataforma que permite cargar material académico, generar evaluaciones con IA mediante **RAG**, editar o regenerar preguntas, **verificar la fuente** de cada pregunta y publicar exámenes dentro de cursos para que otros usuarios los resuelvan o creen nuevas versiones (**fork**).

## 2. Módulos del MVP (prioridad)
1. **Auth + estructura académica + carga de PDFs:** registro/login JWT, universidad/carrera/curso, colecciones y documentos, ingesta asíncrona.
2. **Generación RAG con fuentes:** configurar cantidad, dificultad, tipos de pregunta y documentos; recuperar fragmentos relevantes con pgvector; generar preguntas con el LLM; guardar qué fragmentos sustentan cada pregunta.
3. **Editor de evaluaciones:** CRUD manual de preguntas/opciones y regeneración puntual de UNA pregunta vía IA (reformular, cambiar dificultad) sin regenerar el examen completo.
4. **Motor de resolución:** iniciar intento, responder, enviar, calificación automática, feedback y desempeño por tema.
5. *(Extra si hay tiempo)* **Comunidad:** publicar, buscar, favoritos, comentarios/valoración y fork con referencia al original.

## 3. Modelo de datos

### Enums
- `Role`: `STUDENT`, `TEACHER`, `ADMIN`
- `DocumentStatus`: `PENDING`, `PROCESSING`, `READY`, `FAILED`
- `AssessmentStatus`: `DRAFT`, `GENERATING`, `READY`, `PUBLISHED`, `FAILED`
- `Visibility`: `PRIVATE`, `COURSE`, `PUBLIC`
- `QuestionType`: `MULTIPLE_CHOICE`, `TRUE_FALSE`, `SHORT_ANSWER`
- `Difficulty`: `EASY`, `MEDIUM`, `HARD`
- `AttemptStatus`: `IN_PROGRESS`, `SUBMITTED`

### Entidades y atributos principales
| Entidad | Atributos clave | Relaciones |
|---|---|---|
| **User** | email (unique), passwordHash, firstName, lastName, role, enabled | N:1 University, N:1 Career; 1:N UserCourse, Collection, Assessment, Attempt |
| **RefreshToken** | token (unique), expiresAt, revoked | N:1 User |
| **University** | name (unique), acronym | 1:N Career |
| **Career** | name | N:1 University; 1:N CareerCourse |
| **Course** | code, name | 1:N CareerCourse, UserCourse |
| **CareerCourse** | (unique career+course) | N:1 Career, N:1 Course |
| **UserCourse** | (unique user+course), enrolledAt | N:1 User, N:1 Course |
| **Collection** | name, description | N:1 User (owner), N:1 Course; 1:N Document |
| **Document** | originalFilename, storagePath, contentType, sizeBytes, pageCount, status, errorMessage | N:1 Collection; 1:N DocumentChunk |
| **DocumentChunk** | chunkIndex, content (TEXT), pageNumber, tokenCount, embedding `vector(1536)` | N:1 Document; 1:N QuestionSource |
| **Assessment** | title, description, status, visibility, difficulty, questionCount, timeLimitMinutes, instructions | N:1 User (author), N:1 Course; 1:N Question; N:1 Assessment `forkedFrom` (nullable); N:M Document usado como contexto |
| **Question** | statement (TEXT), type, difficulty, topic, explanation, position, points | N:1 Assessment; 1:N Option; 1:N QuestionSource |
| **Option** | text, correct (boolean), position | N:1 Question |
| **QuestionSource** | relevanceScore, excerpt | N:1 Question, N:1 DocumentChunk |
| **Attempt** | status, startedAt, submittedAt, score, maxScore | N:1 User, N:1 Assessment; 1:N Answer |
| **Answer** | textAnswer, correct, pointsAwarded | N:1 Attempt, N:1 Question, N:1 Option (selected, nullable) |
| **Favorite** | (unique user+assessment) | N:1 User, N:1 Assessment |
| **Review** | rating (1–5), comment | N:1 User, N:1 Assessment |

> **Fork:** se modela como auto-relación `Assessment.forkedFrom` (no requiere tabla aparte). Al hacer fork se copian preguntas y opciones al nuevo Assessment del usuario actual.
> **Nota:** en la propuesta las relaciones N:M se describieron como "1:M mediante X". En el código son N:M implementadas con entidad intermedia explícita.

### Vectores (pgvector)
- Extensión `vector` creada por `docker/init.sql`.
- `DocumentChunk.embedding` mapeado con el módulo `hibernate-vector` (`@JdbcTypeCode(SqlTypes.VECTOR)` + `@Array(length = 1536)`). La dimensión depende del modelo de embeddings elegido.
- Búsqueda por similitud con query nativa en `DocumentChunkRepository`, ordenada por distancia coseno (`<=>`) y filtrada por los `documentIds` seleccionados.
- Mantener los chunks en **nuestra propia tabla** (no en un vector store externo) para conservar la FK `QuestionSource → DocumentChunk`: esa es la trazabilidad pregunta-fuente.

## 4. Pipeline RAG
1. **Upload** (`POST /documents`): guardar archivo, crear `Document(PENDING)`, publicar `DocumentUploadedEvent`, responder **202**.
2. **Ingesta async:** `PROCESSING` → extraer texto (Apache PDFBox) → dividir en chunks (~800 tokens, solape ~100) → embeddings por lotes → guardar chunks → `READY` (o `FAILED` + errorMessage).
3. **Generación** (`POST /assessments/generate`): validar que los documentos estén `READY`, crear `Assessment(GENERATING)`, publicar `AssessmentGenerationRequestedEvent`, responder **202**.
4. **Generación async:** por tema/lote, recuperar top-k chunks → prompt al LLM pidiendo **JSON estricto** (enunciado, opciones, correcta, explicación, topic, ids de chunks usados) → validar/parsear (reintentar ante JSON inválido) → guardar Question/Option/QuestionSource → `READY`.
5. **Regeneración puntual** (`POST /questions/{id}/regenerate`): reutiliza las fuentes de esa pregunta y reemplaza solo esa pregunta.
6. Proveedor de IA: OpenAI o Gemini (decidir en el módulo 2). Encapsular detrás de interfaces `EmbeddingClient` y `LlmClient` para desacoplar el proveedor.

## 5. Endpoints (borrador, `/api/v1`)
**Auth:** `POST /auth/register`, `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`
**Users:** `GET /users/me`, `PATCH /users/me`, `GET /users` (ADMIN), `PATCH /users/{id}/role` (ADMIN)
**Academic:** `GET|POST /universities`, `GET|POST /universities/{id}/careers`, `GET|POST /courses`, `POST /careers/{id}/courses/{courseId}`, `POST /courses/{id}/enrollments`, `GET /users/me/courses`
**Collections:** `GET|POST /collections`, `GET|PUT|DELETE /collections/{id}`
**Documents:** `POST /collections/{id}/documents` (multipart, 202), `GET /collections/{id}/documents`, `GET /documents/{id}` (incluye status), `DELETE /documents/{id}`
**Assessments:** `POST /assessments/generate` (202), `POST /assessments` (manual), `GET /assessments` (búsqueda + filtros + paginación), `GET /assessments/{id}`, `PUT /assessments/{id}`, `DELETE /assessments/{id}`, `POST /assessments/{id}/publish`, `POST /assessments/{id}/forks`
**Questions:** `POST /assessments/{id}/questions`, `PUT /questions/{id}`, `DELETE /questions/{id}`, `POST /questions/{id}/regenerate`, `GET /questions/{id}/sources`
**Attempts:** `POST /assessments/{id}/attempts`, `PUT /attempts/{id}/answers`, `POST /attempts/{id}/submit`, `GET /attempts/{id}/result`, `GET /users/me/attempts`
**Community:** `POST|DELETE /assessments/{id}/favorite`, `GET /users/me/favorites`, `GET|POST /assessments/{id}/reviews`

## 6. Roles y permisos
- **STUDENT:** gestiona sus colecciones, documentos y evaluaciones; resuelve evaluaciones publicadas; favoritos, reviews y forks.
- **TEACHER:** lo mismo que STUDENT, más crear cursos y publicar con visibilidad `COURSE`.
- **ADMIN:** CRUD de universidades y carreras, gestión de usuarios y roles, moderación de reviews.
- Solo el autor puede editar, borrar o publicar su evaluación (verificado en el servicio con el usuario del `SecurityContext`).

## 7. Eventos (mínimo 3, rúbrica pide >2)
| Evento | Listener (async, AFTER_COMMIT) |
|---|---|
| `UserRegisteredEvent` | Email HTML de bienvenida |
| `DocumentUploadedEvent` | Pipeline de ingesta (extraer, chunk, embeddings) |
| `AssessmentGenerationRequestedEvent` | Generación RAG de preguntas |
| `AssessmentGeneratedEvent` | Email "tu evaluación está lista" |
| `AttemptSubmittedEvent` | Email con resultado y desempeño por tema |
| `AssessmentForkedEvent` | Notificar al autor original por email |

## 8. Excepciones personalizadas (mínimo 8)
`ApiException` (base), `ResourceNotFoundException` (404), `DuplicateResourceException` (409), `InvalidOperationException` (400/409), `UnauthorizedException` (401), `ForbiddenOperationException` (403), `InvalidTokenException` (401), `DocumentProcessingException` (500/422), `AiGenerationException` (502), `FileStorageException` (500), `AttemptAlreadySubmittedException` (409).
Manejar también: `MethodArgumentNotValidException`, `HttpMessageNotReadableException`, `MaxUploadSizeExceededException`, `AccessDeniedException`, `AuthenticationException`, `Exception` (500).

## 9. Rúbrica (checklist, 20 pts)
- [ ] **Entidades (1.5):** >6 entidades con @Entity/@Table/@Column correctos
- [ ] **Relaciones (1.0):** todas las relaciones con cascade adecuado y fetch LAZY optimizado
- [ ] **Constraints (0.5):** @NotNull/@Size/@Email/@Pattern/@Min/@Max + unique + índices
- [ ] **DTOs (1.2):** >10 DTOs Request/Response/Create/Update/Detail
- [ ] **Mapeo (0.8):** MapStruct en todos los endpoints, sin datos sensibles
- [ ] **Capas (0.8):** Controller → Service → Repository estricto
- [ ] **SRP (0.6):** métodos ≤ 20–30 líneas, servicios enfocados
- [ ] **DI (0.6):** inyección por constructor, sin `new` de componentes
- [ ] **Excepciones (0.8):** >7 custom con jerarquía
- [ ] **Handler global (1.2):** ErrorResponse(timestamp, status, error, message, path) + 400/401/403/404/409/500
- [ ] **Spring Security (1.0):** rutas + CORS + SecurityContext usado en servicios
- [ ] **JWT (1.5):** login, filtro, UserDetailsService, claims, expiración, **refresh tokens**, secret en env
- [ ] **Roles (1.0):** varios roles, @PreAuthorize, roles en BD y en el token
- [ ] **Registro/Login (0.5):** email único, password strength, BCrypt, tokens en la respuesta
- [ ] **REST (0.8):** /api/v1, plural, verbos correctos
- [ ] **HTTP codes (0.7):** 200, 201, 204, 400, 401, 403, 404, 409, 500
- [ ] **Controllers (0.5):** delgados, @Valid, ResponseEntity
- [ ] **Eventos (1.0):** >2 casos de uso con @TransactionalEventListener
- [ ] **@Async (0.5):** @EnableAsync + ThreadPoolTaskExecutor, varios servicios
- [ ] **Email (0.5):** HTML con Thymeleaf, async, manejo de errores
- [ ] **Deployment (2.0):** AWS EC2/ECS + RDS (1.0 si Railway/Render)
- [ ] **README (0.4):** informe con las 12 secciones, 1000–2000 palabras
- [ ] **Git (0.4):** commits descriptivos, ramas, PRs con review, sin secretos
- [ ] **Gestión (0.2):** GitHub Issues/Projects, labels, milestones
- [ ] **Postman:** `postman_collection.json` en la raíz, todos los endpoints, variables y auth
- **Bonus:** Swagger, logging SLF4J, paginación, filtros, S3, tests >80%, Docker Compose, GitHub Actions

## 10. Flujo de Git del equipo
- Ramas: `main` (release), `develop` (integración, rama por defecto), `feature/*`, `fix/*`, `chore/*`, `docs/*`.
- Ruleset `protect-main-develop`: PR obligatorio, **1 aprobación**, dismiss stale approvals, conversation resolution, sin force push, merge methods Squash + Merge.
- `feature/* → develop` con **Squash and merge**. `develop → main` (release final) con **Merge commit**.
- Cada integrante commitea desde su cuenta. Reviews en rotación: A revisa a B, B revisa a C, C revisa a A.

## 11. Reparto sugerido
| Integrante | Módulos |
|---|---|
| A | security, auth, user, notification (emails) |
| B | academic, document (ingesta RAG) |
| C | assessment (generación, editor), attempt, community |
Compartido primero: `common/` (BaseEntity, ErrorResponse, excepciones, GlobalExceptionHandler) y `config/AsyncConfig`.
