<a name="lifecycle-startup-and-recovery-implementation-plan"></a>

# Plan de arranque y recuperación del ciclo de vida

> Plan histórico del 10 de noviembre de 2026. Las tareas, la autorización y las pruebas privadas corresponden a ese trabajo. Para el comportamiento actual, consulta el [contrato de operaciones](../../../skills/agent-workflow-v1-1/references/operations.md).

<a name="goal-and-constraints"></a>

## Objetivos y restricciones

El plan cubría el reintento seguro del primer inicio, la renovación puntual del uso, la recuperación verificada y el cierre normal. Conservaba el candidate fijado, el núcleo tipado y el monitor sin modelos. El arnés separaría preparación, observación real del entorno, ejecución y cierre. El modo temporizado seguiría siendo explícito. Véase la [especificación histórica](../specs/2026-11-10-agwf-lifecycle-reliability.md).

El plan original autorizaba el commit y el push pertinentes para v1.1, sin GitHub Release ni etiqueta. Requería Python 3.11+, biblioteca estándar, unittest, las herramientas actuales para objetivo y uso y el comportamiento de procesos y uso compartido en Windows. Había que conservar el trabajo en curso, las pruebas antiguas y los vínculos de paquete. Paquete, objetivo, medida, cita y proceso debían observarse. Los datos sintéticos se reservaban para pruebas aisladas. Caducidad, cuota, reserva anticipada y parada del usuario mantenían requisitos de autoridad distintos. La operación ordinaria no provocaría SAVE artificial ni tocaría trabajos de otras sesiones o ajustes de modelo.

<a name="review-focus"></a>

## Prioridades de revisión

1. Un SAVE real de caducidad durante el primer arranque debe permitir un reintento elegible, incluso si aún no existe estado activo.
2. Cuota o parada del usuario, objetivo ajeno y proceso desconocido no deben pasar la recuperación técnica.
3. Un inicio parcial o simultáneo sigue siendo incierto hasta disponer de pruebas.
4. Una publicación reciente no refresca la observación. Una muestra reciente puede coexistir con un plazo de proceso vencido.
5. Un evento tardío no cumple la prueba de tiempo. Un bloqueo temporal de Windows no acredita la salida del proceso.

<a name="files-and-responsibilities"></a>

## Archivos y responsabilidades

| Ruta | Responsabilidad prevista |
| --- | --- |
| `skills/agent-workflow-v1-1/scripts/candidate.py` | Intento de arranque, decisión de reintento, propiedad del monitor y resultados del ciclo de vida |
| `skills/agent-workflow-v1-1/private/limit-watcher/scripts/monitor.py` | SAVE, época y recibo con cambios limitados al contrato |
| `skills/agent-workflow-v1-1/scripts/workflow.py` | Estado activo, detenido o completo y reglas de parada del usuario |
| `tooling/run-lifecycle-check.py` | Arnés preparado que recibe observaciones nativas del agente principal |
| `tooling/tests/test_lifecycle_check.py` | Preparación, eventos tardíos y clasificación |
| `skills/agent-workflow-v1-1/tests/test_candidate_coexistence.py` | Entradas de fuente y paquete instalado con procesos Python propios reales |
| `test_native_usage_source.py`, `test_goal_binding.py`, `test_watcher_engine.py`, `test_monitor.py` y `test_watcher_atomic_write.py` | Recibos, objetivos, autoridad, temporización y Windows |
| `SKILL.md`, `references/operations.md`, `references/native-usage.md` y `references/coexistence.md` | Renovación, reintento, instalación y aceptación de entorno separada |

<a name="task-1-retry-a-failed-first-activation"></a>

## Tarea 1. Reintentar un primer inicio fallido

Interfaces previstas: `activation_identity(command: str, payload: dict) -> dict`, `record_activation_attempt(runtime, record: dict) -> None` y `startup_retry_options(runtime, engine, command: str, payload: dict) -> tuple[str, ...]`. Cada ejecución guarda `activation-attempt.json` con origen, `schema_version`, `command`, `pre_state_sha256`, `phase` y detalles observados del monitor y fallo. `prepared`, `failed` y `committed` son fases distintas. Un `prepared` sin cerrar exige conciliación.

- Probar primero el fallo por recibo obsoleto y luego el reintento con muestra nueva del mismo objetivo en `bind-active-goal` y `activate`. Conservar hash de SAVE y autoridad, verificar proceso sano y estado activo.
- Guardar la intención después de `preflight_activation`. Vincular comando, ejecución, objetivo, hilo, alcance, texto y autoridad. Una hora de observación nueva no cambia el alcance.
- Usar un bloqueo de escritor único entre lanzador y núcleo. Rechazar un propietario desconocido. El estado Python y la interfaz nativa son operaciones separadas.
- Abrir la época técnica solo para `usage_receipt_expired` confirmado de la misma ejecución, nuevo objetivo observado activo y recibo real. Una marca de reintento no autoriza. Confirmar salud y escritura del núcleo antes de declarar éxito.
- Cubrir cuota, reserva, otro diagnóstico, SAVE nuevo, parada, objetivo distinto o inactivo, proceso vivo o desconocido, diario modificado o incompleto, paquete erróneo y arranque simultáneo. Probar recuperación después de un recibo inválido. La reversión restaura solo controles verificados dentro de su ámbito y conserva SAVE más recientes.
- Ejecutar las pruebas de coexistencia y recibos. Comprobar estado, identidad y vigencia, historial y salida efectiva.

<a name="task-2-prepare-before-measuring-and-refresh-early"></a>

## Tarea 2. Preparar antes de medir y renovar a tiempo

El arnés ofrecería `prepare`, `run --input PATH`, `observe --input PATH` y `close`. Preparar comprueba paquete, roles, hash y comandos, sin empezar un proceso. `refresh_by(at: float, interval: float) -> float` sugiere `at + interval - min(15, interval / 4)`. Para 60 segundos recomienda renovar a los 45, dejando intacto el plazo real.

- Probar que prepare y help no piden datos reales, que una lectura vieja o ausente no activa el estado y que repetirla no altera su hora.
- Tomar objetivo y uso después de preparar, conservar solo campos permitidos, ambas ventanas y reinicios. Excluir cuenta, identificación y créditos.
- Distinguir `sample_at`, `next_check_at`, `check_interval_seconds` y `refresh_by`. Publicar datos o ver una señal no demuestra salud actual.
- Exigir `status.at == usage.observedAt`, proceso propio, coincidencia de época, `sampleFresh` y `leaseValid`. Comprobar vigencia después de ingest. No garantizar frescura durante una llamada que no pueda interrumpirse.
- Desactivar temporizador e inyección para la operación ordinaria. Verificar instantánea cuando se solicite y comprobar cierre y salida. Comprobar que no se produjo un SAVE artificial.
- Usar horas diferentes de observación y publicación en pruebas del arnés y de recibos nativos.

<a name="task-3-diagnose-timing-without-confusing-late-events-with-success"></a>

## Tarea 3. Medir tiempos sin aceptar eventos tardíos

- Ejecutar `timed-test` solo si se pide, con un proceso sano previamente preparado y un evento genuino. Usar `--event-input PATH --delay-seconds 10 --tolerance-seconds 1`. Plazo `event_at + 10`, con reloj monótono durante la espera.
- Un evento recibido 45 segundos después da `INVALID_TIMING`. Los registros históricos de 98,74 y 45,05 segundos siguen siendo fallos.
- Usar pruebas oficiales del objetivo. Sin un mecanismo nativo soportado, indicar evento tardío o evidencia incompleta. No medir la interfaz con transportes privados o almacenes de credenciales.
- Registrar aparte `native_event_at`, `observed_at`, `worker_healthy_at`, `save_requested_at`, `save_returned_at`, `snapshot_verified_at`, `helper_halted_at` y `native_paused_at`. La ventana de 9 a 11 segundos se refiere al pedido de SAVE con proceso sano de antemano. La pausa nativa exige pruebas propias.
- Usar el mismo objetivo y ejecución en ambos intentos reales. Probar activo, punto de control y halt, luego una continuación real elegible. La primera vinculación no cuenta como reanudación.
- Probar eventos sintéticos tempranos, tardíos, repetidos y modificados, pausa sin prueba, parada de usuario y dos rondas. Mantener la prueba real como opción separada.

<a name="task-4-verify-recovery-and-regressions"></a>

## Tarea 4. Verificar recuperación y regresiones

- Probar inicio, caducidad inicial y reintento, caducidad activa con stop/snapshot/halt/resume, reintento inválido seguido de válido, transición nativa de pausado a activo y cierre normal. Usar entradas de fuente y de paquete aplicables.
- Antes de continuar, observar `halted` y verificar instantánea. Después, confirmar mismo objetivo y ejecución, época nueva, hash SAVE preservado, autorización y salud.
- Rechazar cuota, reserva anticipada, nuevo SAVE de cuota, parada del usuario, objetivo ajeno, recibo obsoleto o repetido, punto de control dañado y proceso aún vivo o desconocido. `blocked` mantiene las condiciones nativas.
- En Windows, copiar solo archivos propios necesarios. Confirmar conjuntamente identidad y salud terminal. Un fallo temporal de uso compartido no prueba salida. Permisos persistentemente incorrectos deben aparecer en tiempo acotado.
- Ejecutar ambas suites después de las pruebas dirigidas. El número se descubre. El ciclo nativo normal requiere objetivo real y arnés listo. Caducidad y tiempo son diagnósticos separados que pueden quedar `not_tested`.

<a name="task-5-document-the-contract-and-verify-distribution"></a>

## Tarea 5. Documentar y verificar distribución

- Separar reintento inicial, reanudación tras halt y paradas por cuota o usuario. Explicar orden de renovación y significado de un diagnóstico tardío.
- Inventariar instalación y trabajo anterior. Conservar ediciones y procesos en curso. Un hash antiguo no prueba actividad y no debe migrarse una ejecución activa.
- Generar dos paquetes de la misma fuente. Comparar manifiesto y ZIP. Probar cualquier archivo cambiado. Validar la instalación en una ejecución pasiva desechable, comprobar hash y acceso Windows, e informar ese alcance.
- Documentar por separado `startup_retry`, `receipt_refresh`, `halted_resume`, `native_continue`, `natural_complete`, `timed_test`, `worker_exit` y `package_match`. Conservar las pruebas negativas anteriores.
- Para la publicación autorizada entonces, revisar de forma independiente permisos, reversión y preservación de SAVE. Excluir informes privados, recibos y estado.

<a name="task-6-align-the-existing-v11-identity-and-presentation"></a>

## Tarea 6. Alinear la identidad y la presentación de v1.1

- Cambiar juntos nombre visible, invocación, ruta de instalación y nuevo espacio de ejecución. Conservar identidades y hashes anteriores.
- Registrar historia real de v1.0 y cambios v1.1 en `CHANGELOG.md`. Explicar propósito, límites, arquitectura, flujo y orquestación en README.
- Documentar instalación manual y mediante agente a partir de una copia limpia, hash externo, verificación e invocación expresa. La instalación no altera objetivos ni modelos.
- En la publicación autorizada en su día, ajustar About y Topics al código. Conservar acceso y protección de rama. Comprobar los comandos y diagramas.
- Publicar solo archivos autorizados. Releer commit, árbol, versión y documentación. Confirmar que no se creó Release ni etiqueta.

<a name="acceptance-boundary"></a>

## Alcance de aceptación

El orden histórico era completar primero las tareas técnicas 1 a 5 y después publicar. Se requerían la matriz de reintento y reanudación desde halt real, regresiones, comprobaciones de paquete y observaciones de alcance correcto. La interfaz nativa necesitaba evidencia del entorno. La afirmación de diez segundos necesitaba un evento real y una medición independiente.
