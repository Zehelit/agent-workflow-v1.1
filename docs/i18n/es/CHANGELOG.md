<a name="changelog"></a>

# Historial de cambios

Este es el historial existente. La copia de trabajo de origen y el recibo de compilación aceptado identifican el paquete exacto.

<a name="11--2026-11-10"></a>

## 1.1, 10 de noviembre de 2026

<a name="added"></a>

### Novedades

- Se guarda una prueba duradera del primer intento de activate o bind-active-goal. El reintento seguro usa el SHA-256 de fallo conservado fuera del paquete.
- El ciclo de vida del candidate tiene un bloqueo de escritor único con propiedad verificable.
- tooling/run-lifecycle-check.py permite preparar, arrancar, renovar observaciones, reanudar tras halt y cerrar normalmente.
- El modo explícito timed-test registra distintas horas de SAVE, punto de control y halt y clasifica eventos tardíos. No acredita una pausa nativa.
- El estado muestra hora de medición, plazo, intervalo y una sugerencia para renovar antes.
- Se añadieron propósito, arquitectura, diagramas de flujo y orquestación, instalación manual o mediante agente y guía del ciclo de vida.

<a name="fixed"></a>

### Correcciones

- Recuperación controlada tras caducidad de la medición en el mismo objetivo activo autorizado. La cuota agotada y la parada del usuario requieren otra vía.
- El reintento del primer arranque conserva alcance, SAVE anterior, vínculo de paquete y ausencia verificada del proceso.
- Las excepciones semanales acotadas se aplican igual en núcleo y monitor sin reescribir valores observados.
- Se reintentan de forma limitada los fallos temporales identificados de uso compartido o bloqueo de lectura en Windows. Los persistentes o desconocidos siguen visibles.
- Se aclararon la actualización de medidas, la renovación del plazo y la vigencia de los datos posteriores a halt.
- Se ajustaron firmas de funciones de pruebas y esperas de archivos y procesos.

<a name="compatibility-and-evidence"></a>

### Compatibilidad y alcance de las pruebas

- La invocación, carpeta y espacio de ejecución pasan juntos a 1.1: `$agent-workflow-v1-1`, `agent-workflow-v1-1/` y `.agent-workflow-v1-1/runs/`.
- Los manifiestos nuevos rechazan la identidad anterior. Sus ejecuciones quedan como historial y no se vinculan al paquete nuevo. Los espacios anteriores siguen reservados para propiedad y restauración.
- Una actualización no modifica el vínculo inmutable. Usa una ejecución nueva después de aceptar otro paquete.
- Python 3.11+ y biblioteca estándar. El monitor no accede por sí solo al uso nativo ni inicia otro servidor Codex.
- Las pruebas aisladas usan observaciones nativas sintéticas y procesos Python reales. Remote, Continue y la sincronización con la interfaz necesitan pruebas independientes.
- La versión quedó registrada en commits fuente. Un GitHub Release o etiqueta no formó parte de la publicación histórica.

<a name="10--previous-source-version"></a>

## 1.0, versión anterior del código fuente

- Candidate fijado por manifiesto, flujo tipado, perfiles, propiedad de archivos y controles de capacidad.
- Monitor integrado sin modelo, puntos de control y restauración, diario duradero y empaquetado determinista.
- El historial no afirma que exista un GitHub Release o etiqueta correspondientes.
