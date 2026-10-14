# Arquitectura — Manager Library (Laravel/PHP)

Arquitectura **hexagonal** (puertos y adaptadores). Las reglas de dependencia son los artículos I a III de [`constitution.md`](constitution.md).

## 1. Capas y dependencias

```mermaid
flowchart LR
    subgraph Entrada
      HTTP["app/Adapters/Inbound/Http<br/>(Laravel)"]
    end
    subgraph Núcleo
      APP["app/Application<br/>casos de uso + puertos"]
      DOM["app/Domain<br/>agregados y objetos de valor"]
    end
    subgraph Salida
      DB["app/Adapters/Outbound/Persistence<br/>(Eloquent + migraciones de Laravel)"]
      STO["app/Adapters/Outbound/Storage<br/>portadas"]
      SEC["app/Adapters/Outbound/Security<br/>hash y token"]
    end
    HTTP --> APP --> DOM
    DB -. implementa puertos de .-> APP
    STO -. implementa .-> APP
    SEC -. implementa .-> APP
```

Las flechas de dependencia apuntan **hacia adentro**. `Bootstrap` es el único que ve las dos mitades.

## 2. Estructura de carpetas

```text
app/
├── Domain/              # Book, Loan, LoanItem, Member, User; Isbn, Quantity, DateRange
├── Application/
│   ├── Ports/Outbound/  # BookRepository, LoanRepository, MemberRepository, UserRepository,
│   │                    # ReportReader, Clock, PasswordHasher, TokenIssuer, CoverStorage
│   └── Services/        # LendBooksService, ReturnItemService, CreateBookService, SearchBooksService,
│                        # OverdueService, MostBorrowedReportService, LoginService, CreateLibrarianService
├── Adapters/
│   ├── Inbound/Http/    # controladores, requests, resources y manejo de errores
│   └── Outbound/        # Persistence/, Storage/, Security/
└── Bootstrap/Composition/   # app/Bootstrap/Composition/PortBindings.php — único punto de composición
```

## 3. Agregados

| Raíz | Contiene | Invariantes que defiende |
|---|---|---|
| `Book` | `Isbn`, copias totales y disponibles, baja lógica | disponibles entre 0 y total; total ≥ 1; título no vacío |
| `Loan` | colección de `LoanItem` | 1 a 3 líneas; sin libro repetido; vencimiento a 14 días; inmutable salvo devolución |
| `Member` | documento | documento obligatorio y único |
| `User` | rol, hash | nombre normalizado; rol en (`admin`, `librarian`) |

`Loan.AddItem` llama a `Book.Withdraw` antes de añadir la línea: descontar el ejemplar y crear la línea son **una sola operación**.
`Loan.ReturnItem` llama a `Book.Restock` una sola vez por línea.

## 4. Flujo: registrar un préstamo

```mermaid
sequenceDiagram
    participant C as Cliente
    participant H as HTTP
    participant S as LendBooksService
    participant M as MemberRepository
    participant B as BookRepository
    participant L as LoanRepository
    C->>H: POST /api/v1/loans
    H->>S: Lend(memberId, bookIds)
    S->>M: obtener socio + líneas sin devolver
    S->>S: ¿tiene vencidos? ¿supera 3?
    S->>B: obtener libros por lote (Q3)
    S->>S: Loan.AddItem → Book.Withdraw (por cada libro)
    S->>L: guardar préstamo + libros (una transacción)
    L-->>S: ok o conflicto de versión
    S-->>H: préstamo o BOOK_NOT_AVAILABLE
    H-->>C: 201 o 409
```

## 5. Errores

El dominio y la aplicación lanzan errores propios (`BookNotAvailable`, `MemberHasOverdue`, `MemberLimitExceeded`, …). **Solo** el adaptador HTTP los
traduce al cuerpo único y al código de estado de [`api-contract.md`](api-contract.md). Un error de framework nunca llega al cliente.

## 6. Qué no hay

Sin eventos, sin colas, sin caché, sin microservicios, sin multi-tenencia. Se añaden si un requisito lo pide, no antes.
