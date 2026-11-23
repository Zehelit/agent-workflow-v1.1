---
name: agent-workflow-v1-1
description: Coordinate substantial project work across coding, research, writing and analysis; delegate bounded tasks when useful.
---


# agent-workflow-v1.1

Usa esta skill solo cuando el usuario seleccione explícitamente agent-workflow-v1.1 para esta ejecución. La selección implícita está deshabilitada. Una invocación actual exige al agente principal iniciar AGWF en el mismo turno dentro del ámbito autorizado, incluso para preparación y tareas pequeñas. Puede volver a habilitar ese ámbito tras una desactivación. Una elección anterior o en otra sesión no cuenta. Discutir, citar, inspeccionar o mantener la skill no reanuda el trabajo detenido. El chat principal se encarga de la entrega, las escrituras de estado y todas las asignaciones reales. El usuario elige el modelo y el nivel de razonamiento en el cliente. Tras el arranque, una tarea pequeña puede quedar en manos del agente principal. El equipo es opcional. El arranque y la supervisión son obligatorios.

<a name="required-startup-on-explicit-invocation"></a>

## Inicio requerido al invocar la skill

Antes de investigar, modificar, probar o delegar, obtén un recibo vigente de uso nativo e inicia el flujo con el candidate fijado. Usa `workflow activate` o `bind-active-goal` con autorización y sus pruebas. Si solo se permite preparar, se prohíbe crear un objetivo nativo o no hay objetivo activo, inicia el monitor manual según [operaciones](references/operations.md) y respeta el ámbito autorizado. No crees un objetivo sin permiso explícito. Leer la skill, habilitar el selector o comprobar el uso una vez no inicia el monitor.

Verifica una muestra reciente y sana y un proceso propio vivo para la ejecución y época correctas; en un flujo completo verifica también su estado activo. Informa del resultado antes del trabajo sustancial. Reutiliza una ejecución sana que coincida con el alcance autorizado. Una invocación explícita posterior a la parada puede autorizar la vía existente de reanudación o nueva época; conserva el evento real del usuario y todas las comprobaciones de recuperación.

Si se rechaza el inicio, faltan las pruebas o el monitor pide `SAVE_AND_STOP`, indica enseguida la causa concreta y realiza solo la recuperación necesaria. No sigas sin supervisión en el agente principal. Una orden de parada posterior tiene prioridad. Inspeccionar o actualizar la skill mientras está detenida no reanuda el encargo anterior.

Usa `scripts/candidate.py` fijado para el flujo, la coordinación y el monitor integrado. También puedes invocarlo desde una terminal con el hash de manifiesto aprobado, sin antes invocar la skill o el monitor. Las CLI directas del núcleo se rechazan. Consulta la [vinculación](references/coexistence.md) para el anclaje de confianza y el aislamiento, y [operaciones](references/operations.md) para arrancar. Las opciones de modelo proceden de los TOML de rol y `private/limit-watcher/role.json`. El motor, las instrucciones y el rol del monitor están en el paquete y se verifican juntos. No hay otra skill ni lanzador de monitorización.

<a name="role-skills-and-memory"></a>

## Skills y memoria de rol

Cada subagente tiene `SKILL.md` y `MEMORY.md`. Los perfiles de coordinación, arquitectura, implementación, investigación y verificación viven en `roles/<role>/`, y limit_watcher en `private/limit-watcher/`. Cada definición referencia sus documentos. El encargo solo incluye la skill y la memoria de ese rol; rechaza documentos ausentes o vacíos. Los especialistas temporales heredan los documentos del rol base y las instrucciones particulares del encargo.

Las skills de rol se usan solo en asignaciones preparadas y con invocación expresa. Si falta la asignación o las pruebas del modelo y nivel real no coinciden, explica el problema y detente. No activan AGWF ni el monitor. Para deshabilitar el flujo, deshabilita las siete entradas y detén sus procesos propios. Una actualización no los habilita.

La memoria de rol sirve para consultar conocimientos duraderos. No concede permisos ni sustituye pruebas actuales del proyecto. No inventes observaciones pasadas. Devuelve propuestas de memoria al agente principal para que las revise e incorpore entre ejecuciones. Como los documentos pertenecen al manifiesto, cualquier cambio de memoria sigue el proceso de actualización y nueva vinculación. No edites la skill instalada durante un trabajo en curso.

<a name="integrated-limit-watcher"></a>

## Monitor de límites integrado

Un `workflow activate` o `bind-active-goal` autorizado arranca por el candidate el monitor sin modelo. Obtén un resultado real de `mcp__codex_app__get_usage_limits` desde la conexión actual y entrégalo mediante `--usage-input`. Sigue el [contrato de uso nativo](references/native-usage.md). El monitor no crea otro app-server, elige una CLI en PATH ni usa otro transporte. El candidate exige un primer estado sano y vigente antes de escribir el estado activo. La ausencia o caducidad del recibo bloquea el arranque. `resume` necesita observación reciente y nueva época. Para una detención registrada como `usage_receipt_expired` se usa la recuperación técnica descrita abajo. Los otros motivos exigen una continuación directa o transición nativa aceptada.

El agente principal toma muestras nuevas dentro del plazo adaptativo de 300/120/60/30 segundos. El proceso Python verifica las lecturas y bloquea trabajo nuevo cuando no llegan datos vigentes. No llama a la herramienta del entorno por su cuenta. Releer el recibo no cambia su fecha. Los comandos activos renuevan la vigencia de 600 segundos. Un monitor no sano bloquea nuevas tareas y conserva la recuperación. `complete` y `halt` detienen el proceso propio y comprueban la salida. La señal de actividad solo comprueba la propiedad de ejecución y época; la integridad completa del paquete se revisa al medir y al ejecutar comandos. En las transiciones del trabajo usa `workflow status`. Un chat inactivo no renueva ni medición ni vigencia. El cliente carece de arranque automático. El agente principal inicia y verifica el camino elegido. Los comandos manuales `watcher start/ingest/status/renew/stop` respetan el mismo contrato de recibos y autorización.

Con un 25% restante, prepara un punto de control verificado. Con un 20% o señal anticipada, detén trabajo nuevo, pide una confirmación breve a quienes escriben, verifica los archivos y detén solo procesos propios. Si falta la medición, el valor es desconocido. Reintenta una vez y después guarda y detente. Con `SAVE_AND_STOP`, el agente principal actúa de inmediato sin esperar a otro agente. Una lectura caducada indica una interrupción técnica, no demuestra agotamiento de cuota ni revocación de permiso. Una vez guardado y detenido, `usage_receipt_expired` puede recuperarse en el mismo objetivo nativo autorizado y activo con `resume` y `usage_recovery: true`. Necesitas uso real reciente, nueva observación e inventario, punto de control intacto y procesos propios detenidos. El candidate registra el motivo real al ejecutar `halt`. Este camino rechaza cuota, reserva anticipada, otros errores, paradas explícitas y objetivos inactivos. Si hay pruebas suficientes, realiza esa recuperación. No exijas otro mensaje o una pausa observada solo por caducidad. No pongas el objetivo en `blocked` únicamente por ese motivo; aplica las condiciones nativas. Se recuperan monitor y ayudante, no el estado del objetivo, y no se inventa una cita o un clic. Otras paradas conservan sus condiciones. Nunca canjees un reinicio ni cambies de forma automática el estado nativo. La excepción semanal exige una cita real y una caducidad acotada. Se conservan ambas mediciones. Para dudas de recuperación, usa solo el perfil integrado `limit_watcher`, exactamente `gpt-6.1-terra`/`medium`, e informa si no está disponible. Véase [operaciones](references/operations.md) y [el asesor](private/limit-watcher/limit_watcher.md).

Aplica el [contrato de contexto y aceptación](references/context.md) para traspasos limitados al estado de hoy. Conserva el tiempo de la autorización original, las referencias fijadas y las objeciones del revisor. Un resumen no autoriza un envío ni demuestra ahorro de caché. Fin de ejecución nativa y aceptación independiente son observaciones distintas y versionadas. Antes de una revisión independiente importante, ejecuta el verificador del productor que no usa modelos.

Da a cada subagente el objetivo, entradas necesarias, rutas autorizadas y criterios de aceptación. Empieza con un contexto nuevo, sin copiar el chat principal. Nombra las sesiones por su tarea, por ejemplo frontend, backend, editor o data_analysis. Usa las skills pertinentes del proyecto. Para una disciplina concreta, crea un especialista temporal con un rol base y nombre, instrucciones, fuentes, herramientas y criterios propios. No necesita un rol permanente ni una invocación separada.

Investigación reúne pruebas, implementación construye y corrige, verificación revisa por separado los resultados relevantes. Arquitectura resuelve decisiones costosas o desacuerdos técnicos. Usa scripts existentes para el trabajo repetitivo, o crea y valida uno local si lo justifica la repetición o la fiabilidad. Esto depende de la tarea, no de un planificador ocioso. No hay capas permanentes Admin o Executor.

Asigna un solo responsable de escritura a cada archivo. Acordad las interfaces compartidas antes de modificarlas. Verifica de manera independiente la seguridad, autenticación, posibles pérdidas de datos, API compartidas y cambios amplios. El agente principal puede verificar cambios estrechos y reversibles. Devuelve conclusiones, rutas y pruebas pertinentes en lugar de registros en bruto.

Respeta autorizaciones y restricciones vigentes. Si el usuario detiene, no envíes asignaciones nuevas hasta que autorice continuar. Mantén el trabajo vigente y las decisiones duraderas. Elimina tus archivos temporales solo después de cerrar las dependencias de recuperación. No impongas límites arbitrarios de extensión ni reuniones obligatorias.

El ayudante verifica lo que pasa por sus comandos. No intercepta herramientas nativas ni detiene procesos automáticamente. El agente principal realiza las llamadas, anota resultados e inspecciona el inventario. Sin metadatos observados, no presentes un modelo solicitado como el efectivamente utilizado.

<a name="coordination"></a>

## Coordinación

Para trabajo sustancial con especialidades dependientes, pide antes al rol `coordinator` una división, dependencias, elección de especialistas, rutas asignadas y criterios de aceptación. Vuelve a consultarlo ante un cambio material de alcance, un conflicto sin resolver o un fallo repetido que necesite replantear el trabajo. Las tareas pequeñas o independientes y el seguimiento rutinario corresponden al agente principal.

Envía objetivo, restricciones, hallazgos y fuentes relevantes con un contexto nuevo. Coordinación devuelve un plan de solo lectura, sin iniciar agentes ni escribir estado. El agente principal revisa alcance y viabilidad, registra tareas aceptadas y aplica el plan sin duplicarlo. Vincula tareas dependientes a la coordinación aceptada mediante `depends_on`. Ajusta capacidad y rutas menores desde el agente principal. Solo los cambios importantes vuelven a coordinación. Arquitectura decide el diseño técnico, coordinación organiza el trabajo. No esperes a sesiones terminadas; asigna un nuevo trabajo delimitado si hace falta. Si no está disponible el modelo o el nivel exacto, informa y detén el trabajo que dependa del plan. Puede continuar el trabajo independiente autorizado. La recuperación nunca espera a un agente de planificación.

<a name="user-model-policy-27-october-2026"></a>

## Política de modelos del usuario, 27 de octubre de 2026

Los perfiles del flujo solo pueden usar `gpt-6.1-sol`, `gpt-6-astra` o `gpt-6.1-terra`. El nivel mínimo es `medium`; `none`, `minimal` y `low` se excluyen. Los documentos fijan modelo y nivel por defecto sin sustitución automática. No se restaura `astra_coordinator`. El perfil integrado `limit_watcher` solo usa `gpt-6.1-terra`/`medium`. El usuario controla el modelo y nivel del cliente principal.
