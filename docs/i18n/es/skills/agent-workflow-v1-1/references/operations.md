<a name="operation"></a>

# Operaciones

Python 3.11+ y biblioteca estándar. Empieza con `python -X utf8 -B scripts/candidate.py --help`. Consulta la [vinculación](coexistence.md) y conserva el SHA externo del manifiesto aceptado. Si `python` no está disponible, usa el incluido en el entorno. Los comandos leen JSON mediante `--input path` o stdin. El estado reside en `<project>/.agent-workflow-v1-1/runs/<run-id>/state.json`, junto con puntos de control, bloqueos, temporal, propiedad del monitor y diarios, en un espacio aislado. Solo escribe el coordinador principal. No uses archivos, artefactos de prueba ni instantáneas como entrada ordinaria del prompt.

<a name="start-and-dispatch"></a>

## Inicio y asignación

Una invocación expresa requiere arrancar en el mismo turno antes del trabajo sustancial, incluida preparación o labores pequeñas. Leer estas instrucciones no inicia el flujo. Si no se puede crear un objetivo nativo, usa el monitor manual sin inventar objetivo ni evidencia. Conversar sobre la skill o mantenerla no reanuda un encargo detenido.

Fuera del flujo completo, elige un ID nuevo y ejecuta `watcher start --usage-input USAGE.json` con el candidate fijado. Solo activa supervisión. No autoriza agentes ni activa un estado de flujo. `watcher status` confirma muestra sana y vigente y proceso propio vivo para ejecución y época, o informa del rechazo. Usa `watcher ingest` para lecturas reales posteriores, `watcher renew` en límites de trabajo y `watcher stop` al terminar. Confirma la salida. Reutiliza una ejecución sana coincidente. Si está detenida, sigue su continuación o nueva época con controles de recuperación. Otro monitor no permite saltarse la parada. Informa del resultado antes del encargo. Tras un rechazo o `SAVE_AND_STOP`, indica causa y suspende el trabajo sustancial.

Lee los [recibos de uso nativos](native-usage.md). El agente principal obtiene la lectura con `mcp__codex_app__get_usage_limits` y la entrega mediante `--usage-input`. El proceso de fondo lee el recibo sin arrancar otro app-server ni invocar la herramienta del entorno. Obtén un recibo nuevo antes del plazo adaptativo. Reutilizar uno no renueva su fecha. La falta, caducidad, repetición o identidad equivocada detienen el trabajo, también en modo manual. `watcher ingest --usage-input PATH` solo publica datos y no inicia un proceso ni reactiva AGWF.

1. `validate` comprueba definiciones. `activate` toma `objective`, autorización tipada, `models` (modelo y nivel observados), `slot_limit` (incluye al principal), `inventory` (IDs y estados reales), `observed_at` (Unix) y `session_limit` opcional. No inventes disponibilidad. Activar no asigna ni informa ajustes del chat principal.
2. El candidate inicia el monitor privado al autorizar `activate`/`bind-active-goal`, comprueba salud inicial antes de guardar el estado activo y abre nueva época tras `resume` válido. Los comandos activos del flujo y coordinación renuevan el plazo de 600 segundos. Usa `workflow status` entre fases largas para renovar y leer salud. Si vence o falla la medición, se rechaza trabajo nuevo pero se conserva la recuperación. `complete`/`halt` piden detener el proceso propio y verifican su salida. `watcher start/status/renew/stop` sirven para modo manual y diagnóstico con un ID nuevo. No hay gancho de arranque de Codex; el agente principal arranca y verifica. Aplica la política fijada del asesor ante caducidad o recuperación. El muestreador no usa modelos ni controla el chat. Las consultas ordinarias no lanzan subagentes. `usage-due` devuelve `REUSE_MEASUREMENT` o `READ_USAGE`. Pon `substantial_work: true` antes de trabajo importante o paralelo para exigir una lectura nueva. `usage` toma `observed_at` y los `rateLimits` brutos reales. Se necesitan ambas ventanas; ante un valor desconocido se reintenta una vez y se recupera. El plazo es 300, 120, 60 o 30 segundos con >40%, >25%, >20% o ≤20% restante, respectivamente. Se acorta a un cuarto del tiempo estimado hasta 20% según el consumo observado de cada ventana. Una lectura vieja o un reinicio no determina tasa. A ≤25% o ≤10 minutos estimados, `CHECKPOINT_SOON` pide verificar un punto de control de lo ya terminado. Renueva por grupos de escritura y no en cada mensaje. A ≤20% o ≤120 segundos estimados hasta ese umbral, `SAVE_AND_STOP` congela envíos e inicia recuperación anticipada. `prepare` rechaza lecturas fuera del plazo adaptativo. No se consulta en bucle estando inactivo. Las estimaciones no protegen de llamadas ininterrumpibles o cambios repentinos de otro chat.
3. `task` toma `id`, `role`, `objective`, `file_inputs`, `context_refs`, `review_targets`, `criteria`, `writes` (archivos relativos exactos, incluidas eliminaciones), `depends_on` e `independent_review` booleano. Solo lectura usa una lista de escritura vacía. No se reservan carpetas. `inventory` actualiza los estados reales antes de cada envío.
4. `prepare` recibe `id`, `effort` opcional y `current_context` opcional fijado. Comprueba modelo, uso, plazas, dependencias y hashes, reserva capacidad y devuelve los parámetros de `collaboration.spawn_agent` con `fork_turns: none`. Incluye `SKILL.md` y `MEMORY.md` del rol. Si faltan o están vacíos se rechaza; ambos están fijados por el manifiesto. Usa los parámetros sin cambios. Luego `record` toma `id`, `session_id`, `submitted_arguments`, `receipt` real y `effective_model`/`effective_effort` opcionales observados. Para fallos o incertidumbre usa `dispatch_failed` y `certain_not_started`. Un estado incierto bloquea hasta conciliar con el inventario real. No repitas un envío a ciegas.
5. `finish` toma `id`, `accepted`, `evidence` y `review_task` si se exige independencia. Primero observa idle, stopped o done mediante `inventory`. Mantén la propiedad de escritores activos. En `review_targets` se identifica tarea y revisión; su autor ya no escribe, pero sigue pendiente de aceptación. Los hashes no cambian durante la revisión. Un rechazo se conserva; corrige su causa y asigna otra tarea. `cancel` libera una tarea lista no enviada mediante `id` y `reason`. Revisa las ediciones del usuario antes de integrar. Reconcilia resultados externos desconocidos antes de repetir.

Los roles del flujo solo usan `gpt-6.1-sol`, `gpt-6-astra` o `gpt-6.1-terra` y un nivel mínimo de `medium`. Los perfiles fijan modelo y nivel, sin alternativa automática. El chat principal usa la selección del usuario y el programa no cambia un modelo en ejecución. Los parámetros conservan el ajuste exacto del rol. El perfil antiguo `astra_coordinator` no está disponible.

El asesor utiliza exactamente `gpt-6.1-terra`/`medium`. No hay sustitución. Registra la disponibilidad observada al activar. La preparación rechaza un ajuste exacto no disponible.

<a name="recovery-and-stop"></a>

## Parada y recuperación

<a name="initial-activation-retry-v11"></a>

### Reintentar la primera activación (v1.1)

El candidate escribe `activation-attempt.json` antes del primer intento. Solo lo confirma con monitor sano y estado del núcleo escrito. Conserva fuera del directorio modificable el `startup_retry.attempt_sha256` que puede devolver un fallo. Repite el mismo `activate` o `bind-active-goal` con uso real nuevo, observación reciente del mismo objetivo, autoridad original y `--startup-attempt-sha256 PRESERVED_FAILURE_PIN`. Recalcular el hash de un diario editado no concede aprobación.

| Situación | Acción admitida |
| --- | --- |
| Caducidad registrada al iniciar, identidad y estado intactos y proceso ausente verificado | Mismo comando inicial con hash externo del fallo y recibo y observación posteriores. |
| Flujo detenido existente con caducidad registrada | `resume` verificado con `usage_recovery: true` y todos los controles posteriores. |
| Cuota, reserva anticipada, parada del usuario, otro fallo, diario modificado o incompleto, o proceso desconocido | Conserva pruebas y sigue la continuación o conciliación aplicable. |

Inicio, confirmación, reversión, parada y cambios de autoridad comparten el bloqueo propio del candidate. No borres `lifecycle.lock` sin identificar dueño y operación incompleta. No hay transacción atómica con herramientas nativas. Los lotes son secuenciales, con diario y no atómicos como conjunto.

Prepara antes de medir. Si es viable, renueva antes de `sample_at + interval - min(15, interval / 4)`. Es un margen de programación, no extensión del plazo. Confirma por separado fecha de muestra, identidad viva y época, señal periódica, vigencia del dato y plazo del proceso. Ingest por sí solo no renueva el plazo. La excepción semanal usa palabras reales y una fecha límite acotada. Ambos componentes deciden con las mismas ventanas y conservan las dos lecturas brutas.

<a name="initial-native-goal-binding-n02"></a>

### Primera vinculación de objetivo nativo (N02)

Usa `bind-active-goal` si el objetivo ya está activo y preparar tomó más que la vigencia del origen humano. Solo inicia un ayudante schema2 inicial e inactivo, sin tareas, operaciones, parada, halt, cierre, migración o reservas. Las ejecuciones existentes usan autoridad vigente y observación reciente. Una ejecución detenida y vinculada puede observar `paused`/`blocked` después de halt y después `active` para el mismo objetivo con hora nueva. Un cambio de `updated_at` solo no prueba transición del usuario. Para interrupción por caducidad del objetivo activo y autorizado usa la recuperación técnica de abajo.

En `native_binding` indica `binding_id`, IDs de objetivo/hilo/ejecución, ámbito con ID nuevo, `objective` nativo exacto, `goal_created_at` original y `observed_at` actual. La opción `objective` debe ser idéntica al objetivo nativo; incluye modelo/capacidad/inventario. En otro `app_goal_observation` indica los mismos objetivo, hilo, texto, creación y observación, con `status: active`, `source_kind: get_goal` e ID de recibo real. La observación no puede pasar de 60 segundos ni estar en el futuro. Conserva la creación original aunque sea antigua. No prefijes el texto de objetivo.

`interpretation` y `procedural_check` son pruebas distintas y no vacías escritas por el agente principal. Revisa el objetivo autorizado y restricciones humanas más nuevas. No bases el vínculo en una parada, informe o política anterior. No inventes `exact_quote`, `source_message`, `original_turn_id` ni hora de creación. No quedan hijos vivos o desconocidos. El ayudante verifica forma y consistencia; el origen real lo recoge el principal.

La prueba nativa y sus reservas de repetición sobreviven a continuación o activación humana. Con origen nativo, cada `app-goal` nuevo lleva objetivo, hilo, texto y creación original. Siguen los controles de uso, inventario, capacidad, incertidumbre y asignación. Que el objetivo siga activo no borra la parada. Stop, snapshot, halt y recuperación verificada siguen disponibles. Para otro motivo, `resume` requiere `explicit_goal_continuation` tras halt o `native_ui_resume: true` después de ver con `app-goal` que quedó `paused`/`blocked` y reunir un `get_goal` fresco del mismo objetivo `active` con `updated_at` Unix estrictamente posterior a stop y halt. La duda en el mismo segundo se rechaza. Ambos caminos comprueban punto de control, cuota e inventario. Un recibo nativo se usa una vez y no identifica a la persona que lo causó.

<a name="expired-telemetry-recovery-under-existing-authority"></a>

### Recuperar una medición caducada con la autoridad existente

`usage_receipt_expired` indica que se pasó el plazo adaptativo. No prueba agotar la cuota. Suspende trabajo, guarda y ejecuta halt. El candidate registra durante `halt` el motivo real de esa ejecución y época. No envíes `usage_failure` por tu cuenta. Los lotes de recuperación pueden detener y guardar; `halt` y `resume` siguen siendo directos.

Obtén uso nuevo después de halt y `get_goal` real del mismo objetivo activo. Renueva el inventario propio y verifica punto de control, archivos actuales y salida. Registra `workflow usage` e `inventory`. Ejecuta `workflow resume --input RECOVERY.json --usage-input USAGE.json`. `RECOVERY.json` solo contiene `usage_recovery: true` y `app_goal_observation`. Una observación vinculada conserva objetivo y creación originales. No necesita cambiar `updated_at`, observar inactividad, inventar cita o generar clic: usa autoridad vigente. El candidate coteja el motivo con el bloqueo del monitor antes de abrir otra época y espera salud. Conserva excepción semanal y SAVE anterior. Las tareas viejas se rechazan.

Cuota, reserva anticipada, otra falla, falta de causa, objetivo diferente o inactivo, uso obsoleto, hijos propios vivos o desconocidos, punto de control cambiado y parada explícita aún vigente se rechazan. No cambia el objetivo. Si el usuario reactiva desde Continue, observa activo realmente. Con el mismo objetivo activo, los turnos automáticos pueden resolver esta interrupción técnica. No pidas otra continuación escrita ni marques `blocked` solo por caducidad. Aplica las condiciones de `update_goal`. La hora activa no prueba quién pulsó. Solo continuación directa aceptada o transición nativa comprobada de inactivo a activo para una ejecución vinculada sustituye una parada explícita. `usage_recovery` no tiene ese efecto.

Solo monitor sin vínculo de objetivo no usa `usage_recovery` del flujo. Exige su propia continuación y nueva época. Un flujo antiguo sin causa escrita no adopta la vía automáticamente ni se vuelve a etiquetar. Conserva las pruebas fijadas y sigue su contrato de actualización. Declara cualquier procedencia que falte.

<a name="authorization-lifecycle-a26--auth01"></a>

### Ciclo de vida de la autorización (A26 / AUTH01)

`authorization` es un evento explícito. Rechaza incluso un texto histórico no vacío de `authorization`. No se importa estado anterior. Una ejecución nueva usa paquete y autorización actuales. `interpretation` es otra cadena superior. Conserva la cita exacta, incluidos signos. No le añadas la interpretación del agente. La proyección independiente `source_message` coincide con cita y origen.

Ejemplo con IDs y horas sintéticas, que se sustituyen por observaciones reales:

```json
{
  "authorization": {
    "event_id": "user-event-42",
    "kind": "explicit_goal_continuation",
    "exact_quote": "Continue this goal now.",
    "source_kind": "user_goal_continuation",
    "source_id": "user-message-42",
    "goal_id": "goal-7",
    "thread_id": "thread-3",
    "run_id": "run-5",
    "scope": {"kind": "objective", "id": "scope-5", "objective": "Deliver the authorized project change"},
    "observed_at": 1791230401,
    "source_created_at": 1791230401, "original_turn_id": "turn-42", "observed_turn_id": "turn-42"
  },
  "interpretation": "Main classified this source as immediate continuation of the matching objective.",
  "source_message": {
    "source_kind": "user_goal_continuation",
    "source_id": "user-message-42",
    "exact_quote": "Continue this goal now.",
    "source_created_at": 1791230401, "original_turn_id": "turn-42"
  },
  "app_goal_observation": {
    "goal_id": "goal-7", "thread_id": "thread-3", "status": "active",
    "source_kind": "get_goal", "source_id": "actual-goal-tool-receipt",
    "observed_at": 1791230402
  }
}
```

El origen es `user_message` o `user_goal_continuation`. IDs y citas no vacíos. El evento caduca a 300 segundos y no puede estar en el futuro. Evento y mensaje solo se consumen una vez, incluso tras completar una ejecución. El par hilo/ID de origen da la identidad entre clases de fuente, así que cambiar clase o evento no habilita repetición. La observación de objetivo coincide en objetivo e hilo, viene de `get_goal`, tiene recibo no vacío, hora finita de hasta 60 segundos y posterior o igual al evento. Antes de trabajo importante usa `app-goal`. `task` y `prepare` revalidan contra el reloj y la autorización: 60 segundos es válido; 61, futuro o identidad distinta no alteran el estado y se rechazan. Renovar uso e inventario no renueva el objetivo. Estas pruebas vienen del solicitante. Python verifica forma, igualdad, edad y transiciones, no contacta al proveedor ni autentica transcripción o recibo ni clasifica texto natural. El principal recoge y conserva pruebas reales y asigna la clase correcta. Dos citas inventadas que sean iguales no prueban origen.

| Clase | Activación nueva | Objetivo anterior detenido | Efecto en envíos |
| --- | --- | --- | --- |
| `explicit_task_authorization` | Ejecución y ámbito propios, mismo texto y objetivo activo observado | No reanuda | Solo ejecución nueva |
| `explicit_goal_continuation` | Permite ejecución y ámbito propios con prueba activa | Solo después de los controles | Nuevas tareas después de reanudar |
| `future_goal_intent` | Rechazo | Rechazo | No autoriza trabajo viejo |
| `policy_change` | Rechazo | Rechazo | No autoriza trabajo viejo |
| `independent_report_request` | Ámbito independiente `kind: report` con su texto exacto | No reanuda o activa el flujo detenido | Informe separado, el anterior sigue detenido |
| `explicit_stop` | Rechazo | Anotar con `observe-authorization` para identidad y ámbito coincidentes | Congela nuevas tareas y preparación |

`observe-authorization` registra intención futura, política, informe separado o parada acotada sin cambiar autoridad o activar o reanudar. El informe de un objetivo detenido usa otro proyecto y estado; no sobrescribe el detenido. `app-goal` guarda observación nueva separadamente; pausa, bloqueo, parada o final congela actividad. Ver activo después por sí solo no prueba el clic; la vía nativa observa primero inactividad tras halt. Caducidad registrada puede usar `usage_recovery` con autoridad vigente. `status` separa el `status` del ayudante y `app_goal_observation`. La asignación conserva proyecto, ámbito e IDs de objetivo, hilo y ejecución.

Para continuar por un mensaje directo, `explicit_goal_continuation` nuevo y observado estrictamente después de stop y halt, con las mismas identidades y ámbito. Lee uso real e inventario después de halt. Uso reciente, sobre el umbral y a tiempo; inventario reciente sin hijos propios vivos o desconocidos. Repite restauración y control de hashes. Observa el objetivo activo después de halt y no antes de la continuación. Repara el punto de control degradado. La reanudación rechaza tareas antiguas sin terminar y admite nuevas, sin volver a iniciar hijos. Autoridad ausente, vieja, repetida, malformada o ajena a continuar no altera el estado y se rechaza.

Si el usuario pide parar, el principal detiene trabajo y envíos inmediatamente. Solo conserva datos, solicita confirmaciones breves, detiene escritores propios y cierra lo necesario para recuperar. No inicia otro plan ni escribe un traspaso largo. Guarda observaciones de aplicación separadas de transiciones locales. El ayudante no intercepta las herramientas nativas; el principal clasifica procedencia y aplica la parada. Las pruebas reales de continuar/parar de A26 y la auditoría independiente de llamadas no autorizadas van más allá de las pruebas Python desechables.

`stop` congela envíos y devuelve pedidos SAVE_ACK para escritores vivos. Recoge una confirmación breve por escritor. Después de como máximo 30 segundos, si hace falta interrumpe al escritor propio que no responde e indica que puede quedar contenido en memoria. Antes del snapshot, comprueba que no se esté escribiendo. El ayudante no envía mensajes, interrumpe procesos ni consulta permanentemente.

`stop` sigue disponible con monitor no sano o `SAVE_AND_STOP`. No se requiere medición sana para recuperar. La secuencia `stop` → `snapshot` → `halt` y lotes de recuperación usan las mismas acciones permitidas. No conceden tareas, limpian SAVE, renuevan un plazo no sano ni reanudan. `halt` comprueba que no se escribe, integridad del punto y salida del proceso propio.

Tras verificar que los escritores pararon, `snapshot` toma `files` (relativas cambiadas, creadas y borradas), `next_action` y `writers_quiescent: true`. Incluye rutas ya asignadas; añade explícitamente archivos del principal y los no rastreados. Los hashes deben mantenerse durante la copia. El manifiesto lista rutas ausentes o borradas y copia el estado. Ensaya restauración en un directorio nuevo y compara hashes. No se cargan credenciales. Guarda antes las pruebas externas recuperables con sus herramientas.

Usa herramientas del entorno para interrumpir agentes y procesos propios. `halt` recibe `inventory`, `observed_at`, `owned_processes_stopped: true` y `stop_evidence` reales y recientes. Recomprueba punto de control y archivos actuales. Si falla la instantánea o restauración, indica `degraded: true`, `reason`, `next_action` y prueba real de salida sin declarar recuperación verificada. `resume` exige recuperación técnica, continuación directa o transición nativa válida y sus controles. En halt degradado repara primero. No canjees reinicios ni cambies estado nativo automáticamente. Solo recupera un objetivo ya autorizado y activo.

`restore` copia un punto verificado a un destino NUEVO fuera del proyecto. El contenido va a `files/` y los metadatos quedan aparte. El manifiesto incluye rutas que no deben existir. No borra ni sobrescribe archivos del proyecto. Conserva punto actual y anterior bueno; retira exclusivamente carpetas viejas propias, rechaza enlaces y conserva contenido desconocido. `compact` exige ausencia de hijos vivos o tareas pendientes e instantánea verificada y archiva tareas acabadas. `complete` cierra la entrega. Trabajo futuro requiere otra activación.

<a name="mechanical-coordination-and-durable-dispatch-n05"></a>

## Coordinación mecánica y asignación duradera (N05)

`scripts/candidate.py <global pins/options> coordination --input <request.json>` adapta comandos sin modelos y con biblioteca estándar. La solicitud es `{ "batch_id": "unique_id", "steps": [{ "key": "unique_step", "command": "task", "payload": { ... } }], "seen": [] }`. Ejecuta hasta 64 pasos en orden, con verificación, bloqueo y escritura por comando, sin atomicidad de lote. Las lecturas independientes usan `read` con `payload.paths` relativas exactas. El diario guarda texto, bytes y SHA-256, null si falta el archivo. Devuelve una decisión breve con impedimento, sesiones observadas, hash del núcleo, ruta de diario, resultados y token. Solicitud, recibo y resultado brutos permanecen en `.agent-workflow-v1-1/runs/<run-id>/coordination/<batch_id>.json`. Envía tokens ya usados en `seen` para evitar resultados repetidos sin cambios. La salida nativa está fuera del adaptador.

Cada paso se registra como `pending` con hash previo. Se marca después como `applied` con hash y resultado posteriores o `failed` con error y hash posterior. Si el proceso falla entre ambas escrituras, conserva `pending`. Volver a llamar al mismo lote devuelve el impedimento, sin continuar. Un paso aplicado nunca se repite. Un fallo para el lote y conserva anteriores. No reutilices IDs con otro contenido ni borres intención o resultado. Estado inalterado no demuestra que un efecto de restauración o snapshot no haya ocurrido. Inspecciona diario y estado, conserva recibos externos y solo después de resolver lo desconocido élabora un lote nuevo y acotado. Un fallo no se convierte automáticamente en éxito. Cada bloqueo protege su escritura; solo escribe el principal y no hay transacción con otras herramientas. Identifica al propietario antes de retirar un bloqueo viejo.

Camino nativo con diario:

1. Obtén y registra `inventory` real en el control previo. `prepare` devuelve parámetros exactos además de `intent_id` reservado en la tarea.
2. Justo antes de invocar la herramienta ejecuta `dispatch-start` con `id` e `intent_id`. Revalida autoridad, objetivo, uso/SAVE, inventario, tarea y archivos, y registra `uncertain` antes de dar los parámetros `collaboration.spawn_agent`. Un timeout, resultado perdido o reinicio bloquea el duplicado. Un fallo anterior a la llamada también se concilia con inventario. Siguen existiendo `prepare`, llamada y `record` en serie, sin marca previa.
3. Usa directamente `collaboration.spawn_agent`, no desde Python o `functions.exec`. Conserva la salida real. Registra el inventario reciente con `observation: { "source_kind": "collaboration.list_agents", "source_id": "caller-capture-correlation-id", "evidence_path": "operations/raw-native-inventory.json", "sha256": "<actual-file-hash>" }`. Guarda una vez el original en la ruta relativa y solo su referencia y hash en diario y estado. Alternativamente usa `receipt: <actual-native-output>` en línea, que duplica el contenido. Solo vale una opción. Se exige para `dispatch-start` o conciliación de éxito tardío y mantiene 60 segundos, identidad y estados. El núcleo comprueba el hash cada vez. `native_inventory(raw)` normaliza el resultado aceptado `{agents:[{agent_name,agent_status}]}`. `running` e `idle` se conservan. `interrupted` se convierte en `stopped` y `{completed: <final text string>}` en `done`. Rechaza datos desconocidos, ambiguos, inválidos o duplicados. La lista normalizada coincide con el origen al margen del orden y el texto final queda en la evidencia.
4. Concilia mediante `id`, `evidence`, `no_owned_work_running: false`, `submitted_arguments` exactos, `session_id` y `receipt` reales, además del modelo y nivel si se observaron. En el recibo objeto, `id`, `agent_id`, `session_id` o `task_name` coincide con la sesión observada. El estado debe ser conocido en inventario reciente no anterior a la intención. Parámetros y entradas inmutables coinciden y una sesión reusada se rechaza. `Coordination.receipt(...)` escribe el recibo bruto antes de intentar la transición; CLI puede llamar `reconcile`. Una prueba ausente, vieja, de origen incorrecto, inválida o de otra identidad no puede marcar running. Un éxito tardío durante `stopping` solo se guarda para recuperar, sin restablecer permisos. Si cambió la entrada, conserva diario y recibo y detén al hijo real con herramientas nativas.
5. Termina con un inventario terminal real en la aceptación y mantén comprobaciones de independencia y hashes. Vuelve a observar para cerrar o recuperar. Si el inventario demuestra que no hay trabajo propio vivo o dudoso, `reconcile` con `no_owned_work_running: true` rechaza la tarea incierta. No repite ni deduce éxito sin recibo.

El ayudante verifica forma, etiquetas, vigencia y consistencia. No autentica al remitente, recibo o transcripción. El principal guarda la salida nativa real. Si el sistema no emite un ID, `observation.source_id` es un ID de captura aportado por el principal y no debe presentarse como emitido o autenticado por el proveedor. La identidad de asignación real debe coincidir. Coincidencia no autentica metadatos. Prepare, spawn y record no forman una transacción. El adaptador no intercepta herramientas ni oculta sus salidas. `source_pin` calcula el hash del mapa canónico con SHA-256 de workflow.py y coordination.py.

`collaboration.list_agents` admite `path_prefix` de un grupo canónico conocido. Usa una observación por grupo propio en cada control. Si no conoces la raíz, primero consulta inventario. Los prefijos aún pueden devolver textos finales. No existe un filtro nativo `compact` o `exclude_completed`. Los tokens locales solo omiten resultados repetidos del ayudante y no prueban filtrado del proveedor.

Desde la raíz ejecuta `python -X utf8 -B -m unittest discover -s skills/agent-workflow-v1-1/tests -p test_candidate_coexistence.py -v`. Comprueba entradas de fuente y temporal, lotes y ciclo de vida reales del programa. Un fixture histórico de núcleo directo no acredita preparación del candidate. Ambas vías fijan archivos y tareas, preparan, registran evidencia sintética, observan estado terminal simulado, aceptan, guardan y cierran. Se miden comandos hijos, códigos y bytes stdout/stderr. Modelan la decisión del agente principal, no cuentan llamadas reales al modelo. Los recibos sintéticos no prueban asignación nativa. Mide serie y lotes con recibos reales por separado. Tiempo, rutas temporales y serialización cambian el conteo.

<a name="install-and-update"></a>

## Instalación y actualización

Usa `tooling/build-candidate.py` para paquete determinista, ZIP y recibo externo. `install --destination <explicit-directory>/agent-workflow-v1-1` verifica los bytes. Repetir una instalación intacta administrada no cambia archivos. `--update` sustituye solo una instalación propia sin ediciones; rechaza archivos ajenos o modificados. Ante fallo de publicación restaura carpeta anterior. Tras éxito no conserva respaldo de reversión. Consulta [vinculación](coexistence.md) para opciones y confianza externa.

La selección implícita está deshabilitada. El candidate de fuente, temporal o instalado fijado es la identidad de cada ejecución. No la cambies. El monitor, perfil e instrucciones están incluidos bajo `private/limit-watcher` y solo los maneja la entrada fijada. Tras actualizar utiliza una ejecución nueva aprobada; no migres el vínculo anterior. Generar o instalar no activa AGWF ni registra agentes. Si el usuario invoca explícitamente para preparar, aplica la comprobación de inicio en ese ámbito. El usuario conserva la selección del modelo principal.
