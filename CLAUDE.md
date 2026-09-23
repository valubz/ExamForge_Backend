# CLAUDE.md — ExamForge Backend

Instrucciones para Claude Code. Léelas completas antes de cualquier tarea.
La especificación funcional completa está en `docs/PROJECT_SPEC.md`: léela al inicio de cada sesión.

## Contexto
- Proyecto del curso **CS 2031 Desarrollo Basado en Plataforma (UTEC)**, entrega backend Semana 7.
- Equipo de 3 personas. Cada una trabaja en su propia rama y sus propios módulos.
- Se evalúa con una rúbrica de 20 puntos (ver `docs/PROJECT_SPEC.md` §9). Cada decisión debe sumar puntos de la rúbrica.
- Responde y explica en **español**. El código (clases, métodos, variables, endpoints, commits) va en **inglés**.

## Stack
- Java 21, Spring Boot 3.5.x, Maven (`./mvnw`)
- PostgreSQL 16 + pgvector (Docker Compose: `docker compose up -d`)
- Spring Data JPA, Spring Security + JWT (jjwt 0.12.x), Bean Validation
- MapStruct (componentModel = spring) + Lombok
- JavaMailSender + Thymeleaf (plantillas HTML en `src/main/resources/templates/email/`)
- springdoc-openapi (Swagger en `/swagger-ui.html`)
- Configuración por variables de entorno leídas desde `.env` (ver `.env.example`)

## Arquitectura (OBLIGATORIO)
Paquete raíz: `com.examforge`. Organización **por feature**, con capas dentro de cada feature:

```
com.examforge
├── config/        # JpaConfig, AsyncConfig, OpenApiConfig, CorsConfig
├── common/        # entity/BaseEntity, dto/ErrorResponse, dto/PageResponse, exception/
├── security/      # SecurityConfig, jwt/, CustomUserDetailsService
├── auth/ user/ academic/ document/ assessment/ attempt/ community/ notification/
└── <feature>/{controller, service, repository, entity, dto, mapper, event}
```

Reglas:
1. **Controller → Service → Repository.** Los controllers NUNCA acceden a repositorios ni tienen lógica de negocio: validan con `@Valid`, llaman a un servicio y devuelven `ResponseEntity`.
2. **Inyección por constructor** con `@RequiredArgsConstructor` y campos `private final`. Nunca `@Autowired` en campos. Nunca `new` para componentes de Spring.
3. **SRP:** métodos de máximo ~25 líneas. Si un servicio crece, divídelo (ej. `DocumentIngestionService`, `ChunkingService`, `EmbeddingService`).
4. Un servicio solo modifica entidades de su dominio. Si necesita otro dominio, usa el **servicio** de ese dominio, no su repositorio.
5. **Nunca exponer entidades** en la API. Siempre DTOs.

## Entidades
- Todas extienden `BaseEntity` (`id`, `createdAt`, `updatedAt` con `@CreatedDate` / `@LastModifiedDate`).
- Anotaciones: `@Entity`, `@Table(name = "snake_case_plural")`, `@Column(nullable, length, unique)` explícitos.
- Relaciones: `fetch = FetchType.LAZY` por defecto en `@ManyToOne` y `@OneToOne`. `cascade = CascadeType.ALL` + `orphanRemoval = true` solo para composición real (Assessment→Question→Option, Attempt→Answer, Document→DocumentChunk).
- Relaciones N:M con **entidad intermedia explícita** (`CareerCourse`, `UserCourse`, `QuestionSource`, `Favorite`).
- Enums con `@Enumerated(EnumType.STRING)`.
- Constraints de BD (`@Column(nullable=false)`, `@UniqueConstraint`, `@Index`) + validaciones en DTOs.
- Lombok en entidades: `@Getter @Setter @NoArgsConstructor`. NO usar `@Data` ni `@ToString`/`@EqualsAndHashCode` que recorran relaciones.

## DTOs y mappers
- DTOs como **Java records** en `<feature>/dto/`.
- Nomenclatura: `XxxCreateRequest`, `XxxUpdateRequest`, `XxxResponse`, `XxxSummaryResponse`, `XxxDetailResponse`.
- Validaciones en los requests: `@NotBlank`, `@NotNull`, `@Size`, `@Email`, `@Min`, `@Max`, `@Pattern`.
- Mapeo con interfaces MapStruct en `<feature>/mapper/`. Nunca incluir `password` ni datos sensibles en responses.

## Excepciones
- En `common/exception/`. Jerarquía: `ApiException extends RuntimeException` (con `HttpStatus`) → excepciones específicas.
- `GlobalExceptionHandler` con `@RestControllerAdvice` devuelve siempre `ErrorResponse(timestamp, status, error, message, path)`.
- Lanza la excepción más específica posible. Nunca devuelvas `null` para "no encontrado".

## API REST
- Prefijo `/api/v1/`, recursos en plural y kebab-case, sustantivos (no verbos) salvo acciones claras (`/assessments/{id}/publish`).
- Códigos: 200 GET/PUT/PATCH, 201 POST de creación (con `Location`), 202 para procesos asíncronos iniciados, 204 DELETE, y 400/401/403/404/409/500 vía el handler global.
- Listados con paginación (`Pageable`) y `PageResponse`.
- Autorización con `@PreAuthorize` en servicios o controllers. Verificar propiedad del recurso (el usuario autenticado es el dueño) en el servicio.

## Asincronía y eventos
- `@EnableAsync` + `ThreadPoolTaskExecutor` en `config/AsyncConfig`.
- Eventos de dominio como records en `<feature>/event/`, publicados con `ApplicationEventPublisher`.
- Listeners con `@TransactionalEventListener(phase = AFTER_COMMIT)` + `@Async`.
- Procesos lentos (ingesta de PDF, embeddings, generación con IA, correos) SIEMPRE asíncronos, con estado en la entidad (`PENDING`, `PROCESSING`, `READY`, `FAILED`).

## Seguridad
- Secretos SOLO en variables de entorno. **Nunca** escribir claves reales en el código ni commitear `.env`.
- Contraseñas con `BCryptPasswordEncoder`. JWT con claims `userId`, `email`, `roles`. Refresh tokens persistidos.

## Forma de trabajar
1. Antes de programar un módulo, **explica el plan en español** (qué archivos crearás y por qué) y espera confirmación.
2. Trabaja **solo en los paquetes del módulo pedido**. No modifiques módulos de otras integrantes sin avisar.
3. Después de cada cambio corre `./mvnw -q compile`; al terminar un módulo, `./mvnw test`. Corrige los errores antes de seguir.
4. Commits pequeños con **Conventional Commits** en inglés: `feat(auth): add refresh token endpoint`, `fix(attempt): ...`, `chore: ...`, `docs: ...`, `test(...): ...`.
5. **Nunca** hacer push ni commits en `main` o `develop`. Trabajar en `feature/*`, `fix/*`, `chore/*` o `docs/*` creadas desde `develop`.
6. Al terminar un módulo: agrega sus requests a `postman_collection.json` (Collection v2.1, variables `{{baseUrl}}` y `{{accessToken}}`, auth heredada de la colección) y lista los endpoints nuevos con ejemplos de body para probar.
7. Cuando tomes una decisión de diseño no obvia, explícala en 1–2 frases: la integrante debe poder defenderla ante el profesor.
