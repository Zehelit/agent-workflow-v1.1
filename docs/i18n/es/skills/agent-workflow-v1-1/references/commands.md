<a name="command-contract-a02"></a>

# Contrato de comandos (A02)

El ayudante de biblioteca estándar valida los comandos que pasan por `Workflow.execute` y `Workflow.apply`. El agente principal invoca herramientas nativas, obtiene pruebas de sus respuestas y de los mensajes, detiene procesos y observa el modelo efectivo. El ayudante no realiza esas operaciones. Siguen vigentes las reglas A26 de autorización y los controles de puntos de control, propiedad y capacidad.

El estado usa el entero `schema_version: 2`. Se rechazan estados sin versión, anteriores, futuros o malformados antes de escribir. No se ofrece migración ni importación de estado antiguo. Usa una ejecución nueva y autorización tipada vigente. Se admiten puntos de control y restauración. La evidencia histórica no autoriza trabajo. Los fallos inyectados se prueban en copias desechables.

<a name="json-and-error-policy"></a>

## JSON y errores

El estado, `--input`, stdin, metadatos del punto de control y propiedad del instalador usan el mismo análisis estricto. No se permiten claves repetidas a ninguna profundidad, aunque tengan el mismo valor. Se rechazan `NaN`, `Infinity`, `-Infinity` y números que se desbordan, como `1e999`. Los archivos usan UTF-8 con BOM opcional. La raíz del comando debe ser un objeto. Stdin vacío da `{}`, que solo pasa si se permite. `write_json` comprueba y serializa con `allow_nan=False` antes de crear carpetas o archivos temporales.

Las claves son cadenas y las listas siguen siendo listas. Un indicador presente debe ser de tipo Python `bool` exacto, `true` o `false` en JSON. Cadenas, 0/1, null, listas y objetos no sustituyen booleanos. Si falta un indicador obligatorio se rechaza. Uno opcional vale false solo si se omite; null no vale. Todo número es finito. Los conteos son enteros ≥1, los tiempos son números entre 0 y sys.float_info.max y el porcentaje está entre 0 y 100. El máximo del tiempo evita desbordamiento y se aplican los plazos de vigencia. Se conservan los límites de hijos y el conteo de plazas que incluye al agente principal.

Se valida antes de crear carpetas o bloqueos y antes de ejecutar. `apply` opera sobre una copia privada y solo la confirma si pasan los controles. Los rechazos conservan el estado y su SHA-256, salvo el recibo persistente INPUT_INVALIDATED y el indicador contract_invalid previstos abajo. Un primer comando rechazado no crea estado. Un snapshot fallido puede dejar una carpeta propia sin confirmar. Uso bien formado pero no disponible sigue su transición especial.

Los errores del ayudante salen con código 1 y emiten este objeto por stderr:

```json
{"error":"exact boolean required","code":"FIELD_TYPE","field":"$.accepted"}
```

| Código | Significado | Campo |
| --- | --- | --- |
| `COMMAND_UNKNOWN` | Comando de núcleo desconocido | `$.command` |
| `FIELD_REQUIRED` | Campo obligatorio u opción necesaria ausente | Campo exacto |
| `FIELD_UNKNOWN` | Campo desconocido u opción CLI inaplicable | Campo exacto |
| `FIELD_TYPE` | Tipo incorrecto | Campo o índice |
| `FIELD_VALUE` | ID, elemento obligatorio, rango o estado de inventario inválido | Campo o índice |
| `JSON_DUPLICATE_KEY` | Clave repetida | Miembro exacto, incluso anidado |
| `JSON_NONFINITE` | Valor no finito, incluso en metadatos brutos | Campo o índice |
| `JSON_SYNTAX` | Sintaxis o codificación JSON inválida | `$` |
| `JSON_TYPE` | Valor incompatible con JSON o clave no textual | Objeto o valor |
| `COMMAND_REFERENCE` | No existe la tarea o campo guardado | `$.id` para la búsqueda; el texto indica la clave |
| `COMMAND_CONSTRAINT` | Límite de autorización, estado, propiedad, revisión o vigencia | Campo si consta, o `$.command` para el estado general |
| `STATE_VERSION` / `STATE_TYPE` | Versión no admitida o raíz que no es objeto | `$.schema_version` / `$` |
| `IO_ERROR` / `INVALID_VALUE` | Fallo del sistema de archivos u otro valor inválido | `$` |

Se usa punto para claves simples, corchetes en las demás e `[index]` para listas. Una opción o comando desconocidos o un argumento de opción ausente mantienen el stderr habitual de argparse y el código 2. Una opción reconocida pero no aplicable genera `FIELD_UNKNOWN`. Los controles no usan `assert`, así que también funcionan con `python -O`.

<a name="payload-types"></a>

## Tipos de entrada

`text` es texto no vacío después de recortar para comprobar, sin cambiar el original. `string` puede estar vacío. `task_id` cumple `[a-z][a-z0-9_]{0,63}`. `strings` es una lista, incluso vacía, de textos no vacíos. `criteria` requiere al menos uno. `task_ids` enumera IDs. `timestamp`, `count` y `percent` respetan las reglas numéricas. Solo `?` acepta null explícito.

Cada comando y cada objeto anidado rechazan campos no declarados. El contenido del recibo y los metadatos no relacionados con medición de `rateLimits` y de las ventanas se conservan como JSON opaco. Aun así se rechazan claves repetidas y valores no finitos o incompatibles. Un campo desconocido no cuenta como indicador o medición válida.

| Tipo | Obligatorio | Opciones y condiciones |
| --- | --- | --- |
| `authorization` | `event_id`, `exact_quote`, `kind`, `source_kind`, `source_id`, `goal_id`, `thread_id`, `run_id` texto; `scope` ámbito; `observed_at`, `source_created_at` tiempos; `original_turn_id`, `observed_turn_id` texto | Nada más. A26 controla clase, origen directo, coincidencia, edad y repetición. |
| `scope` | `kind`, `id`, `objective` texto | `kind` objective o report según A26. |
| `source` | `source_kind`, `source_id`, `exact_quote`, `original_turn_id` texto; `source_created_at` tiempo | Igual a la proyección de la autorización. |
| `native_binding` | `binding_id`, `goal_id`, `thread_id`, `run_id`, `objective` texto; `scope` ámbito; `goal_created_at`, `observed_at` tiempos | Solo N02 inicial. Ámbito objective y texto exacto. Creación anterior a la observación. Sin campos de origen humano. |
| `app_goal` | `goal_id`, `thread_id`, `status`, `source_kind`, `source_id` texto; `observed_at` tiempo | `objective` y `goal_created_at` opcionales, pero obligatorios e iguales en cada renovación de un vínculo nativo. A26 exige origen get_goal, identidad, edad ≤60 y actividad apropiada. |
| `inventory` | Objetos con `id` y `status` de texto | IDs únicos, sin campos extra. running, interrupting, unknown, idle, stopped y done, edad 0 a 60 segundos. |
| `models` | Nombres no vacíos asociados a listas de niveles `strings` | Se admiten objetos y listas vacíos. Prepare requiere los ajustes exactos observados. |
| `arguments` | Sin campos estructurales obligatorios | Solo `task_name` ID, `fork_turns` texto, `message` cadena, `model` y `reasoning_effort` texto. Record requiere igualdad completa con lo preparado, así que no se puede asignar con datos faltantes. |
| `receipt` | Objeto, lista o cadena no vacía | JSON opaco del resultado real, sin prueba de autenticidad por forma. |
| `rate_limits?` | Objeto o null | Ventanas opcionales primary y secondary y metadatos opacos. |
| `window?` | Objeto o null | `usedPercent` porcentaje, `windowDurationMins` conteo y `resetsAt` tiempo opcionales, otros datos opacos. |

<a name="engine-commands"></a>

## Comandos del núcleo

La lista de campos es exhaustiva. Si no hay opcionales, solo se aceptan los obligatorios. Un comando vacío recibe `{}`.

| Comando | Entrada obligatoria | Opciones y condiciones |
| --- | --- | --- |
| `activate` | `objective` texto, `authorization` autorización, `interpretation` cadena, `source_message` fuente, `app_goal_observation` observación, `models` modelos, `slot_limit` conteo, `inventory` inventario, `observed_at` tiempo | `session_limit` conteo opcional. Ausente o null no acota sesiones abiertas. |
| `bind-active-goal` | `objective` texto, `native_binding` vínculo, `interpretation` y `procedural_check` texto, `app_goal_observation` observación, `models` modelos, `slot_limit` conteo, `inventory` inventario, `observed_at` tiempo | `session_limit` conteo opcional. Solo estado schema2 inicial e inactivo, sin tareas ni historial. |
| `observe-authorization` | `authorization`, `interpretation` y `source_message` | Acepta `app_goal_observation` sin renovar la observación almacenada. |
| `app-goal` | `app_goal_observation` | Ninguna. |
| `usage` | Ninguna. | `observed_at` tiempo y `rateLimits` uso; la omisión indica medición no disponible. |
| `usage-due` | Ninguna. | `substantial_work` booleano, false. |
| `inventory` | `inventory` y `observed_at` | Ninguna. |
| `task` | `id` ID, `role` y `objective` texto, `criteria` criterios | `file_inputs` [], `context_refs` [], `review_targets` [], `revision` texto, UUID si falta; `writes` [], `depends_on` [], `independent_review` booleano false. |
| `prepare` | `id` ID | `effort` texto, valor del rol si se omite; `current_context` context_pin N06. |
| `record` | `id`, `submitted_arguments`, `session_id`, `receipt` | `effective_model` y `effective_effort` texto opcional; omitido o null significa no verificado. |
| `dispatch_failed` | `id` y `evidence` texto | `certain_not_started` booleano false; omitirlo o false mantiene incertidumbre. |
| `reconcile` | `id`, `evidence`, `no_owned_work_running` booleano | Debe ser true y existir inventario reciente sin hijos activos. |
| `finish` | `id`, `accepted` booleano, `evidence` texto | `review_task` ID, obligatorio al aceptar lo que requiere revisión independiente. |
| `cancel` | `id`, `reason` texto | Solo tarea lista y sin enviar. |
| `stop` | Ninguna. | Ninguna. |
| `snapshot` | `writers_quiescent` booleano, `next_action` texto | `files` textos []; el indicador debe ser true. |
| `restore` | `destination` texto | Destino nuevo fuera del proyecto. |
| `halt` | `inventory`, `observed_at`, `owned_processes_stopped` booleano y `stop_evidence` texto | `degraded` booleano false, si es true requiere `reason` y `next_action`. Los procesos deben haber salido. |
| `resume` | `authorization`, `interpretation`, `source_message`, `app_goal_observation` | Continuación posterior a halt coincidente, uso y lista recientes y punto de control verificado. |
| `compact` | Ninguna. | Tareas terminales aceptadas o rechazadas, sin hijos activos y punto de control vigente. |
| `complete` | Ninguna. | Condiciones de compactación y ejecución activa. |
| `status` | Ninguna. | Requiere estado activado. |

Las asignaciones son archivos exactos relativos al proyecto y mantienen las limitaciones de alias Windows, límites y enlaces. Los roles y niveles se comprueban con el catálogo y disponibilidad. Sin escrituras se permite revisión y coordinación de solo lectura. No se crean tareas por referencia. `accepted: false` rechaza trabajo que ya no escribe. `accepted: true` requiere que pasen los controles de revisión.

<a name="raw-usage-lifecycle"></a>

## Ciclo de vida del uso

El `context_pin` N06 tiene exactamente `{path, sha256, manifest_path, manifest_sha256}`, con rutas relativas no vacías y hashes en minúsculas. El manifiesto es una entrada inmutable existente. `prepare` coteja un resumen version1 reciente con el estado y el manifiesto fijado aparte. La preparación con resumen vincula ese protocolo. `dispatch-start` puede recibir `current_context`, con resumen generado de nuevo después de preparar y el manifiesto original. Las comprobaciones nativas siguen activas. [Contexto](context.md) explica el generador, impedimentos, control del productor y límites.

N07 añade `task_observations: {contract_version: 1, tasks: ...}` a `status`. Done queda pendiente de aprobación de artefactos. El cierre independiente guarda el enlace opcional `review` y exige dos identidades de sesión distintas y no vacías y hashes de revisión asignados y anotados iguales. Schema2 no cambia. Una revisión no depende de la aceptación de su propio productor. La aceptación histórica sin identidades o hashes completos se muestra `historical_unrecorded` y no habilita otro prepare, record o dispatch-start. Se usa la misma regla de independencia en el resumen y el núcleo, sin migrar datos históricos.

Tipos o rangos incorrectos se rechazan sin cambio, por ejemplo `usedPercent: "10"`, booleanos como conteos o tiempos, 101%, reinicio negativo, una lista como ventana o metadatos no finitos. No hay uso disponible si faltan `rateLimits`, ventanas o valores, si falta una ventana de 300 o 10080 minutos o si la lectura válida es futura o caducada. El `usage` bien formado en ese caso borra la lectura anterior y aumenta `usage_errors`. Primero `RETRY_USAGE_ONCE` y luego `SAVE_AND_STOP`, con estado de parada salvo si ya está halted o complete. Nunca revive un flujo detenido. 0, 100 y porcentajes fraccionarios finitos son válidos. 100 activa el umbral. Un tipo inválido no aumenta el contador de medición desconocida. Los metadatos se conservan, sin sustituir ventanas válidas.

<a name="supported-pinned-entry"></a>

## Entrada fijada compatible

Usa `scripts/candidate.py --manifest <manifest> --manifest-sha256 <accepted-pin> --version 1.1`. Los modos de ejecución también requieren `--project <absolute-project>` y `--run-id <fresh-run>`.

| Modo | Entrada | Comportamiento |
| --- | --- | --- |
| `workflow validate` | Ninguna | Verifica el paquete y las políticas de rol y vincula un espacio inactivo. |
| `workflow <typed-command>` | `--input <json-file>` o stdin | Ejecuta el contrato tipado. |
| `coordination` | `--input <json-file>` | Ejecuta un lote acotado sin modelo y sin envío nativo automático. |
| `watcher` | `start`, `renew`, `stop`, `status`, `once` | Usa el monitor privado fijado y su estado propio. |
| `install` | `--destination .../agent-workflow-v1-1` y `--update` opcional | Comprueba contenido y condiciones de propiedad. |

No se admite llamar directamente a `workflow.py`. Carece de instalador, exportador y generación de roles nativos. Genera distribución con `tooling/build-candidate.py` y usa el instalador fijado. No hay conversión de esquemas para reactivar una instalación retirada.

<a name="input-and-state-boundary"></a>

## Límites de entrada y estado

Los booleanos correctos conservan su comportamiento en repeticiones desechables. Se admite false opcional y se rechaza interpretar como verdadero un valor de tipo incorrecto. La lectura de estado exige JSON estricto y valores finitos. Se rechazan cadenas de autorización y estado antiguo. Las pruebas usan proyectos desechables.

<a name="typed-assignment-and-source-chronology-a04--gi01"></a>

## Asignación tipada y cronología de origen (A04 / GI01)

N02 `bind-active-goal` vincula el objetivo nativo activo tras preparaciones largas sin fabricar o cambiar fecha humana. Solo acepta estado ausente o `{schema_version: 2, status: inactive, tasks: {}}` exacto con cadenas JSON entre comillas. Un ayudante migrado, detenido, terminado o que ya ha operado no se vincula. Se exige `get_goal` real de 0 a 60 segundos, con objetivo, hilo, texto, creación y observación coincidentes. La creación puede ser antigua, nunca posterior a la observación. Comando y ámbito tienen el mismo objetivo, sin ampliar ni usar report. Interpretación y comprobación de procedimiento, distintas y no vacías, recogen que el principal revisó el contexto y restricciones actuales. No autentican ese acto.

El estado guarda `native_binding` y los registros `native_binding_events` y `native_binding_ids` aparte del origen humano. Cada evento guarda `binding`, `interpretation`, `procedural_check` y `app_goal_observation` original. El ID se reserva frente a eventos humanos posteriores y ejecución y ámbito también respetan reservas históricas. Se necesitan nuevas observaciones con texto y creación nativos coincidentes. La autorización humana mantiene su esquema y 300 segundos desde el origen. Una continuación humana sustituye la base nativa actual y conserva su historial, también en la siguiente activación. Se rechazan registros ausentes, repetidos, malformados o contradictorios después de continuar. Stop, snapshot y halt siguen vigentes, y un nuevo estado activo no reanuda el ayudante detenido por sí solo.

Schema 2 rechaza `inputs` antiguo. Puede conservar texto histórico en `context_refs`, sin tratarlo como rutas ni autoridad. `file_inputs` enumera objetos exactos `{path, revision}`. Normaliza rutas relativas, rechaza duplicados sin distinguir mayúsculas, escapes, rutas reservadas y enlaces, y fija `sha256` y `present` (null/false para ausencia). No solapa entradas inmutables con escrituras de la misma tarea. Las escrituras mantienen su hash base anterior al envío. La revisión y hash de asignación abarcan todo el contrato. Recalcular tras sustituir entradas o revisión no acepta trabajos anteriores.

`review_targets` es una lista de `{producer_id, revision}` solo para verificadores. El productor debe estar en esa revisión y tener una sesión idle, done o stopped observada recientemente, tanto al asignar como al revisar. Se fija hash y presencia de salidas exactas. Las referencias no autorizan revisión. Los parámetros preparados incluyen los hashes tipados. Se repiten controles de entradas, contrato y salidas en prepare, record, finish del verificador y aceptación del autor, incluyendo dependencias aceptadas. A05 conserva estados de ejecución y aceptación; el productor puede figurar running si se ha observado que dejó de escribir mientras espera revisión.

Un contrato vencido genera `INPUT_INVALIDATED` con `invalidation`, ID, revisión, motivo, ruta exacta si procede, hash/revisión esperado y real, fecha y lista ordenada de afectados. Los otros consumidores de la ruta entran solo si su contrato, presencia o hash contradice lo observado. Una revisión nueva fijada a bytes actuales o a su ausencia sobrevive. La propagación sigue dependencias y revisiones reales. Ruta o texto de contexto por sí solos no conectan tareas. Los estados operativos siguen iguales. `contract_invalid: true` y la prueba persistente bloquean aceptar resultados o revisiones antiguos. Restaurar bytes no revive la revisión o autor; repetir la comprobación no altera la prueba ni el estado.

Antes del hash se rechazan symlinks, puntos de unión, enlaces duros, reemplazo por carpeta y escapes en entradas y salidas de dependencia y revisión. Al asignar, preparar, registrar, terminar o aceptar, la salida insegura invalida al productor y revisiones y dependientes pertinentes antes de devolver el error. El recibo conserva hash esperado o null, ruta relativa y `actual` que comienza con `unsafe: ` y el motivo controlado. Un archivo que falta da null. Errores de E/S ajenos como EIO se propagan sin fingir que cambió el contrato. La observación de una ruta no es una instantánea atómica de todo el sistema.

Se descartan cambios parciales ajenos. `apply` directo deja solo la invalidación en el estado proporcionado. Execute la guarda de forma atómica antes de fallar. No hay reset ni rehash. Revisa el recibo y asigna ID, revisión y hashes nuevos. Otras condiciones fallidas conservan sus reglas de no alteración del estado.

GI01 exige source_created_at y original_turn_id en authorization y en source_message exacto. observed_at y observed_turn_id describen la observación. El origen debe preceder a lo observado y tener hasta 300 segundos al aceptarse. Para continuar, la creación de la fuente es posterior a stop, halt o cierre, sin usar en su lugar la hora de observación. La marca de origen se conserva tras continuar o activar. No retrocede y un empate exige el mismo turno original. Los IDs de evento y origen se reservan sin depender de tipo, hora, ejecución, hilo o continuación. La observación coincide con objetivo, ejecución e hilo, y una parada expresa con su ámbito. Intención futura, cambio de política e informe solo registran evidencia. Una parada mecánica por uso no cambia la observación nativa ni concede autorización para pausar.

Los campos son evidencia aportada. El ayudante no intercepta herramientas nativas ni llama a update_goal. La prueba de parada antigua, continuación nueva y repetición de contexto del mismo turno acredita que el ayudante rechaza la repetición y conserva la observación activa. N07 añade la vista de ejecución y aceptación antes descrita. Contexto de un hijo nuevo, medición de caché y pausa nativa A06/GI02 necesitan pruebas propias.
