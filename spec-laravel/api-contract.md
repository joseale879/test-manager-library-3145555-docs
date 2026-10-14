# Contrato de la API — Manager Library

El contrato HTTP es **independiente de la implementación interna**: la aplicación React y el backend Laravel/PHP deben respetarlo.
Qué hace cada operación está en [`spec.md`](spec.md); qué se guarda, en [`data-model.md`](data-model.md).

## 1. Convenciones

| Tema | Regla |
|---|---|
| Base | `/api/v1` |
| Formato | JSON, UTF-8. Nombres de campo en `snake_case` |
| Identificadores | UUID v4 como texto |
| Fechas | ISO 8601 en UTC, con `Z` (`2026-10-14T14:15:00Z`) |
| Dinero | No hay. El sistema no maneja importes |
| Autenticación | `Authorization: Bearer <token>` en todo salvo `POST /auth/login` y `GET /health` |
| Paginación | `?page=1&size=20`. `size` máximo 50. La respuesta es `{ "items": [...], "page": 1, "size": 20, "total": 134 }` |
| Idioma | Mensajes de error en español; campos y códigos en inglés |

### Cuerpo de error (único para todos los errores)

```json
{
  "code": "BOOK_NOT_AVAILABLE",
  "message": "No quedan ejemplares disponibles de «Cien años de soledad».",
  "details": [ { "field": "items[0].book_id", "issue": "sin ejemplares" } ]
}
```

`details` puede ir vacío. **Nunca** incluye trazas, SQL, rutas del servidor ni la contraseña.

### Códigos de estado

| Estado | Cuándo |
|---|---|
| 200 / 201 / 204 | Éxito / creado / sin contenido |
| 401 | Sin token, token inválido o vencido, credenciales inválidas |
| 403 | Rol insuficiente |
| 404 | No existe (o está dado de baja) |
| 409 | Conflicto de estado: duplicado, sin ejemplares, ya devuelto, socio con vencidos, libro con préstamos abiertos |
| 422 | Validación: campo inválido, regla de entrada rota, rango de fechas invertido |

## 2. Códigos de error de negocio

| `code` | Estado | Origen |
|---|---|---|
| `INVALID_CREDENTIALS` | 401 | CA-01.2 |
| `FORBIDDEN_ROLE` | 403 | CA-01.3 |
| `NOT_FOUND` | 404 | Cualquier recurso |
| `USERNAME_TAKEN` | 409 | CA-02.2 |
| `ISBN_TAKEN` | 409 | CA-03.2 |
| `DOCUMENT_TAKEN` | 409 | CA-05.1 |
| `BOOK_HAS_OPEN_LOANS` | 409 | CA-03.5 |
| `BOOK_NOT_AVAILABLE` | 409 | CA-06.3 |
| `MEMBER_HAS_OVERDUE` | 409 | CA-06.5 |
| `ITEM_ALREADY_RETURNED` | 409 | CA-07.2 |
| `MEMBER_LIMIT_EXCEEDED` | 422 | CA-06.4 |
| `DUPLICATE_BOOK_IN_LOAN` | 422 | CA-06.2 |
| `VALIDATION_ERROR` | 422 | Cualquier campo inválido |
| `INVALID_DATE_RANGE` | 422 | CA-10.2 |

## 3. Endpoints

### 3.1 Autenticación y salud

| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| POST | `/auth/login` | público | Devuelve token |
| GET | `/health` | público | `{ "status": "ok" }` |

`POST /auth/login`

```json
// petición
{ "username": "ana", "password": "********" }
// 200
{ "access_token": "eyJ...", "token_type": "Bearer", "expires_in": 28800, "role": "librarian" }
```

Credenciales inválidas (`401`):

```json
{
  "code": "INVALID_CREDENTIALS",
  "message": "Usuario o contraseña inválidos.",
  "details": []
}
```

El mensaje es el mismo tanto si el usuario no existe como si la contraseña no coincide.

### 3.2 Usuarios

| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| POST | `/users` | admin | Crea un bibliotecario |

```json
// petición  (role solo admite "librarian")
{ "username": "Ana ", "password": "********", "role": "librarian" }
// 201  (sin password ni password_hash)
{ "id": "…", "username": "ana", "role": "librarian" }
```

La ruta exige un token de un usuario `admin`. El campo `role` solo acepta `librarian`.

Errores específicos:

- `401` si falta el token o no es válido.
- `403` si el token pertenece a un `librarian`.
- `409 USERNAME_TAKEN` si el nombre normalizado ya existe.
- `422 VALIDATION_ERROR` si los campos no son válidos o se solicita el rol `admin`.

La contraseña y su hash nunca forman parte de una respuesta o de un log.

### 3.3 Categorías

| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| GET | `/categories` | autenticado | Las cinco categorías, orden por nombre |

### 3.4 Libros

| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| GET | `/books` | autenticado | Búsqueda: `q`, `category_id`, `page`, `size` |
| GET | `/books/{id}` | autenticado | Detalle |
| POST | `/books` | admin | Crea |
| PUT | `/books/{id}` | admin | Reemplaza título, autor, categoría y total |
| DELETE | `/books/{id}` | admin | Baja lógica |
| PUT | `/books/{id}/cover` | admin | Sube portada (`multipart/form-data`, campo `file`) |
| DELETE | `/books/{id}/cover` | admin | Quita la portada |

`GET /books` acepta `q`, `category_id`, `page` y `size`. `q` busca texto parcial en título o autor; los resultados solo incluyen libros activos, se ordenan por título y se paginan con un `size` máximo de `50`.

Respuesta de listado:

```json
{ "items": [], "page": 1, "size": 20, "total": 0 }
```

`GET /books/{id}` devuelve `404 NOT_FOUND` si el UUID no existe o si el libro está dado de baja. Los filtros y parámetros de paginación inválidos responden `422 VALIDATION_ERROR`.

Libro (respuesta):

```json
{
  "id": "…", "title": "Cien años de soledad", "author": "Gabriel García Márquez",
  "isbn": "9780307474728", "category": { "id": "…", "name": "Novela" },
  "total_copies": 4, "available_copies": 3, "cover_key": null
}
```

`POST /books` (petición): `title`, `author`, `isbn`, `category_id`, `total_copies`. Los disponibles los calcula el servidor.
`cover_key` es una clave opaca: para pintar la portada se usa `GET /books/{id}/cover`, que devuelve el binario o 404.

Reglas de autorización y errores para la gestión de libros:

- `POST /books`, `PUT /books/{id}`, `DELETE /books/{id}` y las operaciones de portada requieren rol `admin`.
- Sin token o con token inválido: `401`.
- Con rol insuficiente: `403`.
- ISBN repetido entre libros activos: `409 ISBN_TAKEN`.
- Libro con préstamos abiertos al solicitar la baja: `409 BOOK_HAS_OPEN_LOANS`.
- Campos inválidos, total inválido, disponibles menores que cero o portada no válida: `422 VALIDATION_ERROR`.
- Libro inexistente o dado de baja: `404 NOT_FOUND`.

### 3.5 Socios

| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| GET | `/members` | admin, librarian | Lista y busca: `q`, `page`, `size` |
| GET | `/members/{id}` | admin, librarian | Detalle con `active_loans` (conteo sin devolver) |
| POST | `/members` | admin, librarian | Crea: `full_name`, `document_number` |

`POST /members`:

```json
// petición
{ "full_name": "Ana Pérez", "document_number": "10001234" }
// 201
{ "id": "…", "full_name": "Ana Pérez", "document_number": "10001234" }
```

`GET /members` acepta `q`, `page` y `size`; busca por nombre o documento y usa la respuesta paginada común. `GET /members/{id}` incluye `active_loans`, el conteo de libros sin devolver.

Errores específicos:

- `409 DOCUMENT_TAKEN` si el documento ya existe.
- `422 VALIDATION_ERROR` si los campos o la paginación no son válidos.
- `404 NOT_FOUND` si el socio no existe.

No existe una ruta para eliminar socios.

### 3.6 Préstamos

| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| POST | `/loans` | admin, librarian | Registra un préstamo |
| GET | `/loans` | admin, librarian | Lista: `from`, `to`, `status`, `member_id`, `page`, `size` |
| GET | `/loans/{id}` | admin, librarian | Detalle con líneas y estado |
| POST | `/loans/{loan_id}/items/{item_id}/return` | admin, librarian | Devuelve una línea |
| GET | `/loans/overdue` | admin, librarian | Líneas vencidas |

`POST /loans`

```json
// petición  (1 a 3 elementos, sin libros repetidos)
{ "member_id": "…", "items": [ { "book_id": "…" }, { "book_id": "…" } ] }
// 201
{
  "id": "…", "member_id": "…", "loaned_at": "2026-10-14T14:15:00Z", "status": "open",
  "items": [
    { "id": "…", "book_id": "…", "book_title": "Cien años de soledad", "isbn": "9780307474728",
      "category_name": "Novela", "due_at": "2026-10-28T14:15:00Z", "returned_at": null }
  ]
}
```

`status` es calculado: `open` (hay líneas sin devolver y ninguna vencida), `overdue` (alguna sin devolver y vencida), `closed` (todas devueltas).

`POST /loans/{loan_id}/items/{item_id}/return` → 200 con la línea (`returned_at` lleno).

`GET /loans/overdue` → `items`: `{ "loan_id", "item_id", "book_title", "member_name", "due_at", "days_late" }`. Aquí sí se muestra el nombre del
socio, porque es un listado operativo; **el reporte del §3.7 no**.

### 3.7 Reporte

| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| GET | `/reports/most-borrowed` | admin | `from` y `to` obligatorios (fecha, ambos inclusive) |

```json
{
  "from": "2026-10-01", "to": "2026-10-31",
  "rows": [
    { "book_id": "…", "book_title": "Cien años de soledad", "category_name": "Novela", "times_borrowed": 12 }
  ]
}
```

Una fila por `book_id + book_title + category_name` congelados. Orden: `times_borrowed` descendente, luego título. Sin socios ni bibliotecarios.

## 4. Seguridad

| Aspecto | Regla |
|---|---|
| Token | JWT firmado, 8 horas, con `sub` y `role`. Sin datos personales dentro |
| Contraseñas | Solo se guarda el hash; el hash nunca sale en ninguna respuesta |
| Autorización | Se evalúa en cada endpoint por rol; un 403 no revela si el recurso existe |
| CORS | Solo el origen del frontend configurado por entorno |
| Subida de portada | Solo `image/jpeg` y `image/png`, máx. 2 MB; el tipo se valida por contenido, no por extensión |
