## Descripcion

<!-- Que hace este PR y por que. -->

## Modulo / rama

- **Modulo:** <!-- auth | academic | document | assessment | attempt | community | notification | security | common -->
- **Rama:** `feature/...` / `fix/...` / `chore/...` / `docs/...`
- **Issue relacionado:** <!-- #123 -->

## Tipo de cambio

- [ ] Feature nueva
- [ ] Fix
- [ ] Refactor
- [ ] Chore / configuracion
- [ ] Documentacion
- [ ] Tests

## Checklist

- [ ] `./mvnw -q compile` pasa sin errores
- [ ] `./mvnw test` pasa sin errores
- [ ] Controllers delgados (Controller -> Service -> Repository), sin logica de negocio en el controller
- [ ] DTOs (records) + MapStruct, ninguna entidad expuesta en la API
- [ ] Excepciones especificas (no genericas) y manejadas por `GlobalExceptionHandler`
- [ ] Endpoints agregados/actualizados en `postman_collection.json`
- [ ] Sin secretos ni `.env` commiteados
- [ ] Rama creada desde `develop`, no se commitea directo a `main`/`develop`

## Como probar

<!-- Pasos o requests de Postman para validar el cambio. -->

## Revisor sugerido

<!-- Rotacion: A revisa a B, B revisa a C, C revisa a A. -->
