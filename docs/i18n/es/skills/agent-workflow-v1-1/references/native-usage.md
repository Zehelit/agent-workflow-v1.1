<a name="native-usage-receipts"></a>

# Recibos de uso nativos

Usa este proceso solo en una ejecución de AGWF habilitada y autorizada de forma expresa. Invocar la skill de forma explícita exige iniciar la supervisión en el ámbito aprobado. Si no está autorizado crear un objetivo nativo, inicia solo el monitor según la [guía de operaciones](operations.md). Actualizar el paquete, hablar de la skill o publicar datos de uso no reactiva un flujo deshabilitado.

El agente principal llama a `mcp__codex_app__get_usage_limits` mediante la conexión existente de Codex. Python no arranca otro app-server, elige otra CLI, lee archivos de autenticación, accede a transportes privados ni solicita modelos. El proceso en segundo plano lee los recibos y controla su vigencia. El agente principal sigue obteniendo las lecturas en los límites entre fases del trabajo.

Crea el JSON privado a partir del resultado real de la herramienta.

```json
{
  "source": "mcp__codex_app__get_usage_limits",
  "sourceId": "actual-tool-call-receipt-id",
  "observedAt": 1800000000,
  "result": {
    "rateLimits": {
      "limitId": "codex",
      "primary": {"usedPercent": 25, "windowDurationMins": 300, "resetsAt": 1800010000},
      "secondary": {"usedPercent": 30, "windowDurationMins": 10080, "resetsAt": 1800600000}
    }
  }
}
```

Los números solo ilustran la forma. Conserva la hora de observación real, la identidad del recibo y los valores. No los inventes ni uses la fecha de escritura del archivo. Si el resultado tiene varios grupos, da prioridad a `rateLimitsByLimitId.codex`. Un mapa de varios grupos sin el de Codex se rechaza. Se necesitan los dos intervalos con sus duraciones y fechas de reinicio. Copia solo los campos de uso necesarios. No hacen falta los de cuenta, perfil o crédito. `watcher/usage-source.json` conserva exclusivamente los campos permitidos, la identidad y hora de la observación y el vínculo a la ejecución y época.

`workflow activate --input AUTH.json --usage-input USAGE.json` y `bind-active-goal` validan la autoridad antes de arrancar. `resume` exige otro recibo mediante `--usage-input`. No se puede reciclar el de una época previa. `watcher start --usage-input USAGE.json` aplica la misma comprobación. Un recibo ausente o ya caducado impide crear el proceso.

Para registrar una lectura nueva usa `watcher ingest --usage-input USAGE.json` o la opción `--usage-input` de un comando del flujo o la coordinación. `ingest` publica datos, pero no activa el flujo, crea procesos, renueva la autorización ni limpia un bloqueo SAVE. El archivo nuevo dispara la evaluación sin esperar el plazo anterior. Mientras el monitor lo procesa, el estado puede mostrar brevemente la evaluación previa. Antes de despachar trabajo importante, espera un estado reciente y sano.

El agente principal renueva la lectura real antes del plazo adaptativo, de hasta 300 segundos y más corto cerca de un umbral o si crece el consumo. Releer el mismo archivo no refresca la hora. Cuando no llegan datos nuevos se reintenta la validación una vez y se activa SAVE_AND_STOP. Si se registró `usage_receipt_expired`, el mismo objetivo activo y autorizado puede recuperarse con `workflow resume` y `usage_recovery: true`, según la [recuperación técnica](operations.md#expired-telemetry-recovery-under-existing-authority). Datos recientes por sí solos no borran el bloqueo. Si la recuperación técnica cumple sus condiciones, no requiere otro mensaje del usuario. La cuota real y la parada expresa conservan sus condiciones de continuación. Renueva con tiempo suficiente para la siguiente llamada. El plazo no es un tiempo de espera recomendado. Una llamada larga que no se puede interrumpir limita la renovación; el proceso de fondo no puede llamar por su cuenta a la herramienta nativa.

Se rechazan formato incorrecto, ejecución o época ajena, observación futura o vencida, tiempo que retrocede y una repetición con contenido diferente. Publicar exactamente el mismo recibo no cambia el estado. La hora y fuente las aporta quien llama y la forma del JSON no autentica al proveedor. Guarda en privado las pruebas originales, y exige más que una etiqueta arbitraria de procedencia.

Cuando falla la medición, la muestra y el estado devuelven un `diagnosticCode` permitido y la clase de error. `watcher/diagnostics.json` guarda hasta 20 registros estructurados. Se excluyen el texto de la excepción, los datos de autenticación y los cuerpos de las herramientas. Los fallos imprevistos de Python conservan el archivo local `worker-stderr.log`. Como no hay otro servidor, tampoco hay un stderr de ese servidor que descartar.

La integridad completa se verifica por comando de candidate y por evaluación del monitor, incluidos los datos nuevos. La señal periódica solo verifica el control de ejecución y época y publica el estado de salud. No vuelve a recorrer y calcular los hashes de todo el paquete. Una dependencia cambiada se rechaza en el siguiente comando o evaluación, sin prometer detectarla en cada señal.

Las pruebas unitarias cubren la escritura y lectura real de los recibos, el JSON estricto, caducidad, repetición, identidad, umbrales, diagnósticos acotados y ausencia de procesos nuevos en la medición. No verifican Remote real. Mantén el flujo instalado desactivado hasta que el usuario lo habilite explícitamente. Las tareas de mantenimiento no arrancan un AGWF real para simular esa aceptación.
