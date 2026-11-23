<a name="agwf-maintenance-on-28-october-2026"></a>

# Mantenimiento de AGWF del 28 de octubre de 2026

> Registro histórico. Las observaciones y la autorización correspondían a este mantenimiento. Para la operación actual, consulta el [README](../README.md).

Las seis skills de rol recibieron un `agents/openai.yaml` propio con `allow_implicit_invocation: false`. Las descripciones se limitaron a asignaciones preparadas. Sin una asignación o sin prueba del modelo y nivel realmente utilizados, el rol se detiene. Estas skills no inician el flujo ni el monitor. La skill principal dejó de dar por vigente una selección de proyecto anterior.

El código fuente y la instalación se alinearon en lectura compartida, sustitución que conserva permisos y reintentos acotados para archivos de estado de Windows. `candidate.py` usa el mismo mecanismo para leer estados cambiantes. Si un lector externo bloquea el archivo, se conserva la versión anterior. Un bloqueo persistente sigue siendo un error.

Las herramientas de contexto y uso se incluyeron en `tools/` para que la instalación no dependiera de la copia de trabajo. En esta comprobación histórica coincidían 41 archivos entre fuente, paquete e instalación. Se eliminó la copia de la herramienta de contexto en la raíz y se actualizaron las rutas de pruebas.

El registro da cuenta de 63 pruebas de paquete y herramientas y nueve de acceso a archivos en Windows. Se repitieron las nueve con `-O` y los archivos de la instalación. Se revisaron las siete skills y sus hashes. La prueba para las seis políticas de rol que faltaban pasó después de corregirlas.

AGWF permaneció deshabilitado. Las siete entradas de configuración siguieron desactivadas. Las pruebas cubrieron acceso a archivos temporales en el núcleo. Quedó pendiente probar coexistencia real con Remote.

Seguían abiertos los riesgos de iniciar un servidor Codex cada vez, elegir otra CLI de PATH, descartar stderr de ese servidor y repetir la verificación del paquete completo. El primer mantenimiento no determinó la causa de la incidencia de Remote. Los cambios posteriores constan en el [seguimiento](agwf-tum-duzeltmeler-20261028.md). El monitor actual no inicia un segundo servidor.
