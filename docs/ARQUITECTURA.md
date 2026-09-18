# Arquitectura de TaskFlow

Bienvenido a TaskFlow. Esta guía rápida orienta a un desarrollador nuevo sobre la estructura del proyecto, el flujo de una petición de creación de tarea, dónde están las reglas de negocio, cómo funciona la seguridad con JWT y cómo están organizados los tests.

## Capas y paquetes

La aplicación sigue una arquitectura en capas clásica (Controller → Service → Repository → Model) bajo el paquete raíz `com.taskflow`:

- `controller` — pide y responde HTTP con DTOs. Ej.: `src/main/java/com/taskflow/controller/ProjectController.java`, `ProjectController`.
- `service` — casos de uso y orquestación. Ej.: `src/main/java/com/taskflow/service/TaskService.java`, `TaskService`.
- `repository` — persistencia con Spring Data JPA. Ej.: `src/main/java/com/taskflow/repository/TaskRepository.java`, `TaskRepository`.
- `model` — entidades con comportamiento/rules. Ej.: `src/main/java/com/taskflow/model/Task.java`, `Task`.
- `dto` — objetos de transporte (records) y validación. Ej.: `src/main/java/com/taskflow/dto/TaskRequest.java`, `TaskRequest`.
- `mapper` — mapeos entre entidades y DTOs. Ej.: `src/main/java/com/taskflow/mapper/TaskMapper.java`, `TaskMapper`.
- `security` / `config` — configuración de seguridad, filtros y beans JWT. Ej.: `src/main/java/com/taskflow/security/JwtAuthenticationFilter.java`, `src/main/java/com/taskflow/security/JwtService.java`, `src/main/java/com/taskflow/config/SecurityConfig.java`.
- `seeder` / `DataSeeder` — carga de datos de ejemplo para el perfil `h2`. Ej.: `src/main/java/com/taskflow/config/DataSeeder.java`, `DataSeeder`.

Convenciones importantes:
- DTOs son `record` con Bean Validation y se validan con `@Valid` en los `@RequestBody`.
- Inyección por constructor, sin Lombok.
- Controladores devuelven DTOs no entidades; mapeos centralizados en `TaskMapper`.
- Rutas públicas: `/auth/**`, `/info`, Swagger y la consola H2; el resto requiere token.

## Recorrido de `POST /projects/{projectId}/tasks`

Resumen paso a paso desde la petición hasta la base de datos:

1. Cliente envía `POST /projects/{projectId}/tasks` con un body validado por `TaskRequest` (`src/main/java/com/taskflow/dto/TaskRequest.java`).
2. Spring MVC despacha a `TaskController.createTask` (`src/main/java/com/taskflow/controller/TaskController.java`), que comprueba que el proyecto existe llamando a `projectService.buscarPorId(projectId)` y lanza 404 si no.
3. El controlador llama a `taskService.crear(request, projectId)` (`src/main/java/com/taskflow/service/TaskService.java`).
4. `TaskService.crear` convierte el DTO a entidad mediante `TaskMapper.aEntidadNueva(request, projectId)` (`src/main/java/com/taskflow/mapper/TaskMapper.java`) y persiste la entidad con `taskRepository.save(...)` (`src/main/java/com/taskflow/repository/TaskRepository.java`).
5. Reglas de negocio principales viven en la entidad `Task` (`src/main/java/com/taskflow/model/Task.java`). Por ejemplo:
   - `Task.crear(...)` encapsula la lógica de creación (status inicial `TODO`, rechazo de fechas pasadas);
   - `Task.estaVencida()` determina vencimiento;
   - transiciones de estado que requieren responsable o validaciones específicas.
   El servicio reutiliza esos métodos en lugar de duplicar reglas.
6. Tras persistir, `TaskController` responde `201 Created` con cabecera `Location` apuntando al nuevo recurso y el body contiene `TaskMapper.aResponse(creada)`.
7. Notas:
   - Validaciones declarativas con Bean Validation devuelven `400` si fallan.
   - Errores de negocio específicos (por ejemplo, proyecto no existe) se lanzan como excepciones mapeadas por `GlobalExceptionHandler` a códigos 404/422/409 según corresponda.

Notas:
- Validaciones declarativas con Bean Validation devuelven `400` si fallan.
- Errores de negocio específicos (por ejemplo, proyecto no existe) se lanzan como excepciones mapeadas por `GlobalExceptionHandler` a códigos 404/422/409 según corresponda.

## Dónde están las reglas de negocio

- Entidades en `src/main/java/com/taskflow/model` contienen la lógica central (por ejemplo `Task.crear(...)`, `Task.estaVencida()`). Reutilizar estas reglas desde servicios.
- `service` contiene orquestación, validaciones de caso de uso y verificaciones de permisos, pero no debe duplicar reglas ya en las entidades.
- `GlobalExceptionHandler` transforma excepciones de dominio en respuestas HTTP estandarizadas.

Este enfoque mantiene la lógica del dominio cerca de los datos que la representan y evita la dispersión de reglas.

## Seguridad con JWT

- Auth sin estado: al autenticar, la API emite un JWT firmado y el cliente lo envía en `Authorization: Bearer <token>`.
- Rutas públicas: `POST /auth/**`, `/info`, Swagger y consola H2.
- Un filtro/interceptor (por ejemplo `src/main/java/com/taskflow/security/JwtAuthenticationFilter.java`) extrae y valida el token antes de alcanzar los controladores.
- `SecurityConfig` (`src/main/java/com/taskflow/config/SecurityConfig.java`) define las rutas públicas, la carga del `UserDetails`/claims y registra el filtro JWT.
- Control de acceso por roles: `@PreAuthorize` y clases como `ProjectSecurity` se usan para decisiones finas (p. ej. borrar sólo si eres owner o ADMIN).

Consecuencias prácticas:
- No hay estado de sesión en el servidor (escalable).
- Revocación de tokens requiere estrategia externa (lista negra, short TTL + refresh tokens, etc.).

## Organización de tests

Tests agrupados por tipo:

- Unit tests: JUnit 5 + Mockito, sin Spring. Prueban servicios y lógica de entidades. Ejecutar con `mvn -q test`. Ej.: `src/test/java/com/taskflow/unit/TaskServiceTest.java`.
- Slice tests (`@WebMvcTest`, `@DataJpaTest`): prueban controladores y/o repositorios con configuración mínima de Spring.
- Integration tests (`@SpringBootTest`) para flujos completos; algunos `*IT.java` usan Testcontainers y no se ejecutan por defecto con `mvn test` a menos que se pase `-Ddocker.tests=true`.
- Convenciones de test:
  - No modificar tests existentes para que pasen. Si fallan, arreglar el código o explicar por qué el test es incorrecto.
  - Datos de ejemplo se cargan con perfil `h2` y `DataSeeder` para pruebas manuales o desarrollo.

## Comandos útiles

- Ejecutar la suite normal: `mvn -q test`
- Correr la app con H2 y datos: `mvn spring-boot:run "-Dspring-boot.run.profiles=h2"`

## Lecturas recomendadas en el código

- `src/main/java/com/taskflow/model/Task.java` — reglas de negocio clave.
- `src/main/java/com/taskflow/service/TaskService.java` — orquestación del caso de uso `crear`.
- `src/main/java/com/taskflow/mapper/TaskMapper.java` — cómo se traducen DTOs ↔ entidades.
- `src/main/java/com/taskflow/controller/ProjectController.java` — ejemplo de endpoint que usa DTOs y devuelve `201 Created`.
- `src/main/java/com/taskflow/config/SecurityConfig.java`, `src/main/java/com/taskflow/security/JwtAuthenticationFilter.java` y `src/main/java/com/taskflow/security/JwtService.java` — flujo de autenticación.

Si se necesita, se puede ampliar este documento con diagramas de secuencia o ejemplos concretos de payloads y respuestas.
