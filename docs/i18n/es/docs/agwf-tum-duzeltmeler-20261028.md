<a name="agwf-monitoring-follow-up-on-28-october-2026"></a>

# Seguimiento de la monitorización de AGWF del 28 de octubre de 2026

> Registro histórico. Continúa el [primer mantenimiento](agwf-duzeltmeler-20261028.md) y no concede autorización para activar una ejecución actual.

| Problema | Cambio y pruebas registrados entonces |
| --- | --- |
| Seis roles podían seleccionarse automáticamente | Cada uno recibió `allow_implicit_invocation: false` y quedó limitado a asignaciones preparadas. Las pruebas inspeccionaron las siete skills. |
| El código, el repositorio remoto y la instalación diferían | Se incluyeron las herramientas en el paquete. La instalación se cotejó con el manifiesto y ZIP generados desde un commit limpio. |
| Conflictos de lectura y escritura en Windows | Se acordó un lector compartido y una forma de sustituir archivos. Las pruebas usaron archivos temporales reales para comprobar bloqueos, reintentos y conservación del contenido previo. |
| Cada medida iniciaba otro servidor Codex | Se eliminó ese código. El agente principal consulta la herramienta actual y Python lee el recibo. Las pruebas bloquean el arranque de procesos en la medición. |
| PATH podía elegir otra CLI de Codex | Se retiró la selección de ejecutables de esa ruta. |
| Se perdía la salida de errores | Al eliminar el servidor hijo, el diagnóstico retiene hasta 20 entradas de código, clase y fase. Los fallos Python inesperados guardan stderr local. |
| Dos bucles recorrían el paquete cada segundo | La señal solo valida ejecución y época. La revisión completa se hace al medir o al ejecutar un comando. |
| Las pruebas solo usaban un servidor simulado | Las pruebas se adaptaron a los recibos de uso. El registro indica que un resultado real de Codex también pasó por el archivo temporal. |
| La regla del rol usaba un umbral equivocado | El `%10` se corrigió al `%20` real. Si el inicio se rechaza por falta de datos, se borra la marca de actividad. |

El flujo usa `--usage-input` con hora original e identidad de origen. El monitor vincula el recibo a ejecución y época. Releerlo no crea una observación nueva. Se rechazan datos ausentes, futuros, caducados, atrasados o ajenos. Publicar otra vez el mismo dato no adelanta su hora. Sin datos nuevos se reintenta la comprobación y se registra SAVE_AND_STOP.

El agente principal consulta uso antes del vencimiento; Python no puede llamar a la herramienta del entorno. `ingest` solo publica. Activar, iniciar procesos, conceder autorización, renovar la observación del objetivo y limpiar la parada son operaciones independientes.

Se registraron 111 pruebas dirigidas en modo normal y `-O`. Las pruebas bloqueaban inicios reales de AGWF y Codex y permitían los subprocesos necesarios del generador de paquete y las herramientas de solo lectura. Las pruebas de código, los hashes instalados, la configuración deshabilitada y la reproducción desde fuente limpia se registraron aparte.

El mantenimiento también pasó un resultado real de uso por el flujo de recibos sin iniciar otro proceso ni cambiar la hora. No añadió cuentas o credenciales. La causa exacta de la incidencia de Remote de entonces no quedó demostrada. La convivencia de Remote y AGWF real quedó fuera de las pruebas. [Contrato de uso nativo](../skills/agent-workflow-v1-1/references/native-usage.md).
