# Registro de deuda técnica — Manager Library (Laravel/PHP)

Este registro no amplía el producto ni cambia sus reglas. Documenta decisiones pendientes que deben resolverse antes de implementar la tarea relacionada.

| ID | Deuda | Afecta | Estado | Criterio para cerrarla |
|---|---|---|---|---|
| DT-001 | ADR-002 exige un testigo de versión por libro, pero `data-model.md` no define una columna ni el mecanismo persistente. | T-13 / RNF-2 | Pendiente | Definir el mecanismo Laravel/PostgreSQL y documentarlo sin permitir que dos préstamos consuman el último ejemplar. |
| DT-002 | El contrato exige JWT firmado y hash seguro, pero falta fijar la librería y versión concreta para Laravel/PHP. | T-14 | Pendiente | Elegir y documentar las dependencias sin exponer secretos ni modificar el contrato. |
| DT-003 | `api-contract.md` describe `GET /books/{id}/cover`, pero no aparece como fila en la tabla de endpoints. | T-16 | Pendiente | Completar el contrato con rol, respuesta binaria, tipo MIME y códigos de error, sin crear una operación nueva. |
| DT-004 | La prueba de arquitectura PHP está definida por comportamiento, pero no por una herramienta concreta. | T-19 | Pendiente | Elegir la herramienta y verificar los artículos I, II y III. |
| DT-005 | Las dependencias declaradas de T-16, T-20 y T-22 no expresan todas las precondiciones de persistencia, frontend y admin inicial. | T-16 / T-17 / T-20 / T-22 | Pendiente | Ajustar únicamente la matriz de dependencias para que la evidencia sea ejecutable, sin cambiar el alcance. |
| DT-006 | HU-05 define crear, listar, buscar y consultar socios, pero la matriz de tareas no tiene un caso de uso ni una evidencia específica para esas operaciones. | HU-05 / T-06 / T-16 | Pendiente | Incorporar la evidencia o dependencia necesaria en `tasks.md` sin añadir funcionalidades al producto. |

Una deuda no se considera resuelta por estar programada: debe tener una decisión documentada y evidencia ejecutable.
