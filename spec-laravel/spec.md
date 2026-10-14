# Especificación — Manager Library

**Qué se construye y por qué.** Un sistema para que una biblioteca pequeña lleve su catálogo de libros,
sus socios y sus préstamos, y vea qué libros se prestan más.

Rige bajo [`constitution.md`](constitution.md). El modelo de datos vive solo en [`data-model.md`](data-model.md)
y el contrato HTTP solo en [`api-contract.md`](api-contract.md).

---

## 1. Actores

| Actor | Qué hace | Cómo se autentica |
|---|---|---|
| **Administrador** (`admin`) | Mantiene el catálogo de libros y los socios; da de alta bibliotecarios; ve todos los reportes | Usuario y contraseña → token |
| **Bibliotecario** (`librarian`) | Registra préstamos y devoluciones; consulta catálogo, socios y vencidos | Usuario y contraseña → token |
| **Socio** | Persona que pide libros prestados. **No entra al sistema**: es un dato, no un usuario | — |

El rol `admin` lo provisiona el despliegue desde variables de entorno. **Nadie lo otorga en ejecución.**

## 2. Alcance

**Dentro:** catálogo de libros con portada opcional, cinco categorías fijas, socios, préstamos con varias
líneas, devolución por línea, libros vencidos, reporte de más prestados, inicio de sesión con roles.

**Fuera, y no se reabre:** multas y pagos, reservas, renovaciones, correo o SMS, importación masiva,
préstamos entre sedes, atributos de libro más allá de los de §3 (sin editorial, sin año, sin idioma).

## 3. Historias de usuario y criterios de aceptación

### HU-01 · Iniciar sesión
Como usuario quiero autenticarme para operar según mi rol.
- **CA-01.1** Con credenciales válidas se recibe un token con vigencia de 8 horas.
- **CA-01.2** Con credenciales inválidas responde 401 con un mensaje que **no** dice si falló el usuario o la clave.
- **CA-01.3** Todo endpoint salvo el inicio de sesión y `/health` exige token. Sin token: 401. Rol insuficiente: 403.

**Precondiciones**

- El usuario ya existe y tiene uno de los roles permitidos: `admin` o `librarian`.
- El endpoint de inicio de sesión es público; no requiere un token previo.

**Flujo principal**

1. El usuario envía `username` y `password` a `POST /api/v1/auth/login`.
2. El sistema busca el usuario y verifica la contraseña contra el hash almacenado.
3. El sistema emite un JWT con vigencia de 8 horas.
4. La respuesta incluye el token, el tipo `Bearer`, `expires_in` igual a `28800` y el rol del usuario.

**Flujos alternos y errores**

- Si las credenciales no son válidas, responde `401` con `INVALID_CREDENTIALS` sin revelar si falló el usuario o la contraseña.
- Si se consulta un endpoint protegido sin token, responde `401`.
- Si el token es inválido o está vencido, responde `401`.
- Si el token es válido pero el rol no tiene permiso para la operación, responde `403`.

**Reglas de seguridad**

- El JWT contiene `sub` y `role`, pero no contiene datos personales innecesarios.
- La contraseña nunca se devuelve, se registra en logs ni se almacena en texto claro.
- La clave usada para firmar el token proviene del entorno.

### HU-02 · Dar de alta bibliotecarios (admin)
- **CA-02.1** El admin crea un usuario con nombre, clave y rol `librarian`.
- **CA-02.2** El nombre de usuario se guarda en minúsculas y recortado; es único. Repetido: 409.
- **CA-02.3** Pedir el rol `admin` por la API responde 422. La clave nunca se devuelve ni se registra en logs.

**Precondiciones**

- El solicitante tiene una sesión válida con rol `admin`.
- El endpoint no está disponible para usuarios `librarian` ni para solicitudes sin token.

**Flujo principal**

1. El admin envía `username`, `password` y `role` a `POST /api/v1/users`.
2. El sistema recorta el nombre y lo normaliza a minúsculas.
3. El sistema verifica que `role` sea `librarian` y que los campos sean válidos.
4. El sistema guarda únicamente el hash de la contraseña.
5. La respuesta `201` devuelve `id`, `username` normalizado y `role`, sin contraseña ni hash.

**Flujos alternos y errores**

- Sin token o con token inválido: `401`.
- Con token válido pero sin rol `admin`: `403`.
- Con `role` igual a `admin` o un valor no permitido: `422 VALIDATION_ERROR`.
- Con campos inválidos: `422 VALIDATION_ERROR`.
- Con un nombre de usuario ya existente después de normalizarlo: `409 USERNAME_TAKEN`.

**Reglas de seguridad**

- El rol `admin` no se puede otorgar mediante esta operación.
- La contraseña nunca aparece en respuestas ni logs.
- El hash de la contraseña no se devuelve ni se expone mediante la API.

### HU-03 · Gestionar libros (admin)
- **CA-03.1** Crear libro con título, autor, ISBN, categoría y total de ejemplares. Los disponibles arrancan iguales al total.
- **CA-03.2** El ISBN tiene 13 dígitos y es único entre libros activos. Repetido: 409.
- **CA-03.3** Total de ejemplares ≥ 1. Cambiar el total ajusta los disponibles en la misma diferencia y **nunca** deja disponibles < 0 (422).
- **CA-03.4** Subir portada (JPG/PNG, máx. 2 MB) guarda una **clave opaca**; sin portada el campo es `null`, nunca cadena vacía.
- **CA-03.5** Eliminar un libro es **baja lógica**: deja de aparecer en el catálogo pero sigue en los préstamos pasados. Un libro con préstamos abiertos no se da de baja (409).

**Precondiciones**

- El solicitante tiene una sesión válida con rol `admin`.
- La categoría del libro existe en el catálogo de cinco categorías fijas.
- Las operaciones de creación, edición, portada y baja se realizan sobre el recurso identificado por UUID.

**Flujo principal**

1. El admin crea un libro con título, autor, ISBN, categoría y total de ejemplares.
2. El sistema valida el ISBN, la categoría y el total; inicia `available_copies` igual a `total_copies`.
3. El admin puede reemplazar título, autor, categoría y total de un libro activo.
4. El sistema ajusta los disponibles por la diferencia del total y rechaza cualquier resultado menor que cero.
5. El admin puede subir una portada válida; el sistema almacena una clave opaca y no expone la ruta ni los bytes en el registro del libro.
6. El admin puede quitar la portada; el registro queda con `cover_key` nulo.
7. El admin puede solicitar la baja lógica de un libro sin préstamos abiertos.

**Flujos alternos y errores**

- Sin token o con token inválido: `401`.
- Con token válido pero sin rol `admin`: `403`.
- ISBN inválido o total menor que uno: `422 VALIDATION_ERROR`.
- ISBN repetido entre libros activos: `409 ISBN_TAKEN`.
- Libro inexistente o dado de baja: `404 NOT_FOUND`.
- Ajustar el total dejaría `available_copies` menor que cero: `422 VALIDATION_ERROR`.
- Portada que no sea JPG/PNG válido o supere 2 MB: `422 VALIDATION_ERROR`.
- Intentar dar de baja un libro con préstamos abiertos: `409 BOOK_HAS_OPEN_LOANS`.

**Reglas de conservación**

- La baja es lógica mediante `deleted_at`; no se borra el libro ni su historial.
- Los libros dados de baja no aparecen en el catálogo activo.
- El ISBN puede reutilizarse únicamente cuando el registro anterior está dado de baja.
- Las contraseñas, rutas físicas y bytes de portada nunca forman parte de las respuestas del libro.

### HU-04 · Buscar libros
- **CA-04.1** Búsqueda por texto parcial en título o autor, filtro por categoría, solo libros activos, orden por título, paginada (`page`, `size`, máx. 50).
- **CA-04.2** La respuesta incluye el total de resultados.
- **CA-04.3** Consultar un libro por id inexistente o dado de baja: 404.

**Precondiciones**

- El solicitante tiene una sesión válida con rol `admin` o `librarian`.
- Las consultas se realizan sobre el catálogo activo y sus categorías existentes.

**Flujo principal: listado**

1. El usuario autenticado consulta `GET /api/v1/books`.
2. Puede enviar `q` para buscar texto parcial en título o autor.
3. Puede enviar `category_id` para filtrar por categoría.
4. El sistema excluye los libros con baja lógica, ordena por título y aplica `page` y `size`.
5. El sistema devuelve los elementos de la página y el total de coincidencias antes de paginar.

**Flujo principal: detalle**

1. El usuario autenticado consulta `GET /api/v1/books/{id}`.
2. El sistema devuelve el detalle del libro si existe y está activo.
3. Un libro dado de baja se trata igual que un libro inexistente para esta consulta.

**Reglas de paginación y respuesta**

- `page` comienza en `1` y `size` no puede superar `50`.
- La respuesta de listado usa `{ "items": [...], "page": 1, "size": 20, "total": 134 }`.
- El orden principal es ascendente por `title`.
- La respuesta de detalle conserva `available_copies`, `total_copies`, categoría y `cover_key` sin exponer rutas ni bytes de portada.

**Flujos alternos y errores**

- Sin token o con token inválido: `401`.
- Con un filtro o parámetro de paginación inválido: `422 VALIDATION_ERROR`.
- Consultar un libro inexistente o dado de baja: `404 NOT_FOUND`.
- Una búsqueda sin coincidencias responde correctamente con `items: []` y `total: 0`.

### HU-05 · Gestionar socios
- **CA-05.1** El admin y el bibliotecario crean socios con nombre y documento. El documento es único: repetido 409.
- **CA-05.2** Se lista y se busca por nombre o documento, paginado.
- **CA-05.3** Los socios no se eliminan.

**Precondiciones**

- El solicitante tiene una sesión válida con rol `admin` o `librarian`.
- El socio es un dato de la biblioteca; no tiene credenciales ni inicia sesión.

**Flujo principal: crear socio**

1. El usuario autorizado envía `full_name` y `document_number` a `POST /api/v1/members`.
2. El sistema valida los campos y comprueba la unicidad del documento.
3. El sistema crea el socio y devuelve sus datos públicos de operación, sin credenciales porque el socio no es un usuario.

**Flujo principal: consultar socios**

1. El usuario autorizado consulta `GET /api/v1/members`.
2. Puede enviar `q` para buscar por nombre o número de documento y usar `page` y `size` para paginar.
3. El sistema devuelve los elementos de la página y el total de coincidencias.
4. Al consultar `GET /api/v1/members/{id}`, el sistema devuelve el detalle y `active_loans`, que cuenta los libros sin devolver.

**Flujos alternos y errores**

- Sin token o con token inválido: `401`.
- Con un rol no autorizado: `403`.
- Documento repetido: `409 DOCUMENT_TAKEN`.
- Campos o parámetros de paginación inválidos: `422 VALIDATION_ERROR`.
- Socio inexistente: `404 NOT_FOUND`.

**Reglas de conservación y privacidad**

- Los socios no se eliminan y no existe una ruta para hacerlo.
- `full_name` y `document_number` son datos personales disponibles solo para `admin` y `librarian`.
- El conteo `active_loans` se calcula a partir de líneas de préstamo sin devolver; no se almacena como atributo del socio.

### HU-06 · Registrar un préstamo
- **CA-06.1** Un préstamo es de **un** socio, lo registra el usuario autenticado y trae de 1 a 3 líneas.
- **CA-06.2** No se repite el mismo libro en un préstamo (422).
- **CA-06.3** Cada línea descuenta **1** ejemplar disponible en la misma operación. Sin ejemplares: 409 y **no se registra nada**.
- **CA-06.4** Un socio no puede tener más de **3 libros sin devolver** en total (422 con el conteo actual).
- **CA-06.5** Un socio con algún libro **vencido** no puede pedir otro (409).
- **CA-06.6** La fecha de vencimiento es `loaned_at + 14 días`. La línea congela título, ISBN y nombre de categoría **en ese instante**.
- **CA-06.7** El préstamo registrado no se edita ni se borra.

**Precondiciones**

- El solicitante tiene una sesión válida con rol `admin` o `librarian`; su identidad es quien registra el préstamo.
- El socio existe y cada libro solicitado existe y está activo.
- El mecanismo concreto para evitar que dos solicitudes consuman el último ejemplar continúa registrado en DT-001; el resultado observable es el de RNF-2.

**Flujo principal: registrar préstamo**

1. El usuario autorizado envía `member_id` y de 1 a 3 elementos `{ "book_id" }` a `POST /api/v1/loans`.
2. El sistema valida que no haya libros repetidos, consulta las líneas sin devolver del socio y verifica que no tenga vencidos ni supere tres libros sin devolver al sumar la solicitud.
3. En una única operación, comprueba la disponibilidad de cada libro, descuenta exactamente un ejemplar por línea y crea el préstamo asociado al usuario autenticado.
4. Para cada línea, fija `due_at` a `loaned_at + 14 días` y congela `book_title`, `isbn` y `category_name` del catálogo en ese instante.
5. Devuelve `201` con el préstamo, sus líneas y el estado calculado `open`.

**Flujos alternos y errores**

- Sin token o con token inválido: `401`; con un rol no autorizado: `403`.
- Socio o libro inexistente, o libro dado de baja: `404 NOT_FOUND`.
- Cero, más de tres líneas o campos inválidos: `422 VALIDATION_ERROR`.
- Un mismo `book_id` más de una vez: `422 DUPLICATE_BOOK_IN_LOAN`.
- Si el socio ya tiene libros abiertos y la solicitud haría que excediera tres, responde `422 MEMBER_LIMIT_EXCEEDED` e informa su conteo actual.
- Si el socio tiene al menos una línea vencida sin devolver: `409 MEMBER_HAS_OVERDUE`.
- Si algún libro no tiene ejemplares disponibles, incluso por conflicto concurrente, responde `409 BOOK_NOT_AVAILABLE`; no se crea el préstamo ni se descuenta ningún ejemplar de los demás libros.

**Reglas de conservación**

- `loaned_by_user_id` procede exclusivamente de la sesión autenticada; el cliente no puede elegir al bibliotecario que registra el préstamo.
- El estado (`open`, `overdue`, `closed`) se calcula a partir de las líneas y el reloj, y no se almacena.
- El préstamo y sus líneas no se editan ni se eliminan. La única modificación posterior permitida por el alcance es registrar una devolución en su propia HU, llenando `returned_at`.
- Los valores congelados de la línea no cambian si después se modifica el libro o su categoría.

### HU-07 · Registrar una devolución
- **CA-07.1** Se devuelve **una línea** a la vez. Pone `returned_at` y suma 1 a los disponibles.
- **CA-07.2** Devolver una línea ya devuelta: 409. **Nunca** suma dos veces.
- **CA-07.3** Los disponibles jamás superan el total (la base lo garantiza).
- **CA-07.4** Cuando todas las líneas están devueltas, el préstamo figura como **cerrado**.

**Precondiciones**

- El solicitante tiene una sesión válida con rol `admin` o `librarian`.
- El préstamo, la línea y su relación entre sí existen; la línea todavía no tiene `returned_at`.

**Flujo principal: devolver una línea**

1. El usuario autorizado envía `POST /api/v1/loans/{loan_id}/items/{item_id}/return`.
2. El sistema localiza la línea que pertenece al préstamo indicado y comprueba que aún no fue devuelta.
3. En una misma operación, establece `returned_at` con el instante UTC actual y suma exactamente un ejemplar a `available_copies` del libro.
4. Devuelve `200` con la línea actualizada.
5. El estado del préstamo se sigue calculando a partir de sus líneas: pasa a `closed` cuando todas tienen `returned_at`; en otro caso conserva el estado que corresponda al reloj y las líneas pendientes.

**Flujos alternos y errores**

- Sin token o con token inválido: `401`; con un rol no autorizado: `403`.
- Préstamo, línea o relación préstamo-línea inexistente: `404 NOT_FOUND`.
- Identificadores con formato inválido: `422 VALIDATION_ERROR`.
- Línea que ya tiene `returned_at`: `409 ITEM_ALREADY_RETURNED`; no se modifica la disponibilidad.

**Reglas de conservación**

- Una devolución solo puede afectar una línea por solicitud y solo puede incrementar una vez la disponibilidad del libro asociado.
- La disponibilidad nunca puede superar `total_copies`; `ck_book_available_range` es la barrera final del motor.
- El préstamo y los valores congelados de sus líneas permanecen inmutables; esta operación únicamente completa `returned_at` en la línea devuelta.
- No existen rutas para editar ni eliminar préstamos, líneas o devoluciones.

### HU-08 · Ver un préstamo y listar préstamos
- **CA-08.1** Consultar un préstamo trae sus líneas y su estado calculado (`open`, `overdue`, `closed`).
- **CA-08.2** Listar por rango de fechas, estado y socio, orden por fecha descendente, paginado.
- **CA-08.3** El estado **se calcula**, no se almacena.

### HU-09 · Libros vencidos
- **CA-09.1** Lista las líneas sin devolver con `due_at` anterior a ahora: libro, socio, días de retraso.
- **CA-09.2** Ordenadas por más días de retraso primero. Solo `admin` y `librarian`.

### HU-10 · Reporte de libros más prestados
- **CA-10.1** Dado un rango de fechas, una fila por **libro y nombre de categoría congelado**, con la cantidad de veces prestado, ordenado de mayor a menor.
- **CA-10.2** Fin anterior al inicio: 422. Rango sin préstamos: lista vacía, no error.
- **CA-10.3** El reporte **no** se desglosa por bibliotecario ni por socio.
- **CA-10.4** Un reporte de un periodo cerrado **no cambia** aunque luego se renombre el libro o su categoría.
- **CA-10.5** Solo `admin`.

## 4. Reglas de negocio

| # | Regla |
|---|---|
| RN-1 | Disponibles entre 0 y total, siempre. |
| RN-2 | Plazo de préstamo: 14 días. Máximo 3 libros sin devolver por socio. |
| RN-3 | Un socio con vencidos no pide más hasta devolverlos. |
| RN-4 | Lo congelado en la línea (título, ISBN, categoría) no sigue al catálogo. |
| RN-5 | Préstamos y devoluciones no se borran. Los libros solo tienen baja lógica. |
| RN-6 | Las fechas son instantes UTC. Mensajes al usuario en español; código y columnas en inglés. |

## 5. Requisitos no funcionales

| # | Requisito | Cómo se mide |
|---|---|---|
| RNF-1 | La búsqueda de libros responde en menos de 300 ms con 5 000 libros | Prueba de carga con el sembrador de `tool` |
| RNF-2 | Dos bibliotecarios prestando el **último ejemplar** a la vez: uno gana, el otro recibe 409 | Prueba concurrente |
| RNF-3 | La contraseña nunca viaja ni se guarda en claro; nunca aparece en logs | Revisión de logs y de respuestas |
| RNF-4 | Todo error HTTP usa el mismo cuerpo (ver contrato) | Prueba de contrato |
| RNF-5 | El sistema levanta con un solo comando y la base arranca vacía | `README` de `infra` |
