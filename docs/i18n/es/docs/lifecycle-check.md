<a name="repeatable-lifecycle-check"></a>

# Comprobación reproducible del ciclo de vida

`tooling/run-lifecycle-check.py` procesa observaciones reales de objetivos, uso e inventario de subagentes a través del candidate fijado. El agente principal toma las observaciones del entorno. Mantén la sesión y los archivos de entrada fuera de Git, en privado. Designa un solo responsable de escritura por sesión.

<a name="prepare-before-measuring"></a>

## Preparar antes de medir

Genera el paquete y conserva el hash externo aceptado. `prepare` verifica el paquete y escribe una sesión nueva sin iniciar el candidate ni el monitor. Termina de preparar antes de tomar la primera lectura real.

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py prepare --candidate C:/build/agent-workflow-v1-1/scripts/candidate.py --manifest C:/build/agent-workflow-v1-1/manifest.json --manifest-sha256 PRESERVED_SHA256 --version 1.1 --project C:/project --run-id check-1 --session C:/private/check-1.json
```

`PRESERVED_SHA256` procede del recibo de compilación aceptado. Usa un proyecto existente y un archivo de sesión nuevo. Si cambia el paquete, no vuelvas a vincular la sesión antigua a un hash diferente.

<a name="host-input-object"></a>

## Objeto de entrada del entorno

`run`, `observe`, `resume`, `close` y `timed-test` aceptan un objeto JSON por `--input`.

| Campo | Contenido |
| --- | --- |
| `activation` | Solo para el primer `run`. Datos tipados de `bind-active-goal`, con `native_binding`, `app_goal_observation` correspondiente, objetivo, interpretación, comprobación del procedimiento, modelos y grados de razonamiento realmente observados, capacidad, inventario real y hora. |
| `app_goal_observation` | Una observación vigente de `get_goal` con objetivo e hilo, texto, creación y estado, `source_kind: get_goal`, referencia de fuente real y `observed_at`. Incluye `updated_at` real si está disponible. |
| `usage` | `source`, `sourceId`, `observedAt` y `result` de los [recibos de uso nativos](../skills/agent-workflow-v1-1/references/native-usage.md). Conserva solo las ventanas necesarias. |
| `inventory` | Identidades y estados actuales de los subagentes, observados de verdad. Usa `[]` solo si no hay subagentes. |
| `checkpoint_files` | Rutas relativas al proyecto que deben recuperarse, incluidas modificaciones, altas, eliminaciones y archivos del agente principal. El núcleo agrega las rutas asignadas a cada tarea. |
| `writers_quiescent` | `true` solo tras observar que todos han dejado de escribir. El arnés no puede producir esa observación. |
| `next_action` | Siguiente paso útil después de recuperar o cerrar normalmente. |
| `weekly_exception` | Opcional en el primer `run`. `exact_quote` contiene una cita real del usuario y `valid_until` una caducidad Unix acotada. Por defecto no hay excepción. |
| `resume` | Datos completos de la reanudación del candidate. Si se omiten, el arnés solo prueba `usage_recovery: true` para una medida caducada registrada. Las paradas por cuota o por el usuario conservan sus condiciones. |

Si la API nativa no devuelve un ID de objetivo independiente, usa una identidad derivada coherente, como `threadId:createdAt`, y aclara su origen. Consulta los contratos de [operaciones](../skills/agent-workflow-v1-1/references/operations.md) y [comandos](../skills/agent-workflow-v1-1/references/commands.md).

<a name="run-refresh-and-close"></a>

## Ejecutar, renovar y cerrar

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py run --session C:/private/check-1.json --input C:/private/initial-observation.json
python -X utf8 -B tooling/run-lifecycle-check.py observe --session C:/private/check-1.json --input C:/private/new-observation.json
python -X utf8 -B tooling/run-lifecycle-check.py close --session C:/private/check-1.json --input C:/private/final-observation.json
```

Registra la hora real apenas regrese la llamada nativa. El arnés exige `status.at == usage.observedAt`, proceso propio, época, señal de actividad, muestra reciente y plazo vigente. `observe` renueva también ese plazo. `refresh_by` sugiere renovar antes sin modificar el límite real.

Con `CHECKPOINT_SOON`, verifica un punto de control del trabajo acabado y mantén `checkpoint_files` al día. `close` exige monitorización reciente, instantánea comprobada, cierre y salida observada. La operación normal no inyecta SAVE.

Si el primer `run` incluye `weekly_exception`, el arnés configura la política temporal y retorna `NEED_POLICY_OBSERVATION` antes de activar el flujo. Obtén nuevas observaciones de objetivo y uso y repite el comando en la misma sesión. Conserva las horas originales de los recibos previos.

<a name="halt-and-resume-after-expiry"></a>

## Detener y reanudar tras una caducidad

Si el monitor pide SAVE, `observe` aplica stop, snapshot y halt, y devuelve `NEED_POST_HALT_OBSERVATION`. Después de halt, observa de nuevo el objetivo, el uso y el inventario.

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py resume --session C:/private/check-1.json --input C:/private/after-halt-observation.json
```

El núcleo se basa en la causa observada del monitor. La recuperación técnica exige el mismo objetivo activo autorizado, archivos e instantánea intactos, pruebas recientes posteriores a halt y salida de los procesos propios. Cuota, reserva anticipada y parada del usuario mantienen sus vías de continuación. El arnés no pausa, bloquea ni activa el objetivo nativo.

Si falla la primera activación, la sesión conserva el hash del intento. Usa el mismo `run` con nuevas pruebas solo cuando se cumplan las condiciones de reintento. Una sesión modificada no demuestra procedencia. Un intento completo o incierto no se reinicia automáticamente.

<a name="run-a-timed-diagnosis-only-when-selected"></a>

## Diagnóstico de tiempos bajo petición expresa

Prepara un monitor sano antes del evento. Usa `timed-test` solo para el diagnóstico seleccionado. El evento aporta `event_id`, `goal_id`, `thread_id`, `source_id`, `event_at` y `observed_at` de una observación real.

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py timed-test --session C:/private/check-1.json --input C:/private/current-observation.json --event-input C:/private/native-event.json --delay-seconds 10 --tolerance-seconds 1
```

Una observación tardía da `INVALID_TIMING`. `SAVE_REQUEST_ON_TIME` indica que la solicitud de SAVE cumplió el tiempo. La pausa nativa requiere una prueba aparte; el arnés deja `native_pause_verified: false` y `native_paused_at: null`. Una prueba real completa observa dos fases en el mismo objetivo y ejecución: actividad, detención y nueva actividad tras una continuación autorizada. Un reloj simulado solo verifica la lógica.
