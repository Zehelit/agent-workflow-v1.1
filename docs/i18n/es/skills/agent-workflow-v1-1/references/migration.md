<a name="state-compatibility-and-recovery-boundaries"></a>

# Compatibilidad de estados y límites de recuperación

El entorno v1.1 solo admite estado schema2 actual en un espacio de ejecución aprobado e inmutable. Rechaza estados desconocidos, dañados o antiguos sin alterar el estado operativo. Los textos de autorización, recibos archivados y observaciones de objetivos caducadas no conceden autoridad.

El punto fijado `scripts/candidate.py` carece de comando de migración. El núcleo no ofrece conversión de estado ni valida autorizaciones antiguas. La importación, el arranque inicial y la reversión automática desde otra instalación no están disponibles. Empieza una ejecución nueva tras aceptar una actualización. No cambies el manifiesto ni la correspondencia con el ayudante de una ejecución existente.

La recuperación actual usa un punto de control verificado de la ejecución aceptada y nueva autorización válida. Conserva las horas de origen, las reservas contra repeticiones, la integridad de archivos y la observación de que han cesado las escrituras. Las órdenes anteriores de detener o continuar se conservan solo como historial. El ayudante no activa por su cuenta el objetivo recuperado.

Los nuevos puntos de control pertenecen a `agent-workflow-v1-1`. El directorio independiente por defecto del núcleo es `.agent-workflow-v1-1`. El punto de entrada soportado usa `.agent-workflow-v1-1/runs/<run-id>`. Los directorios anteriores `.agent-workflow` y `.agent-workflow-v1-0` no se importan. Están reservados y no se pueden indicar como rutas de tarea o destinos de restauración. La limpieza actual no borra sus marcas de propiedad. Se rechaza invocar directamente la CLI del núcleo.

<a name="move-the-user-installation-to-the-11-technical-identity"></a>

## Pasar la instalación a la identidad técnica 1.1

1. Inventaría los procesos y los responsables de escritura. Detén solo el trabajo propio. Antes de mover una instalación anterior, verifica la parada autorizada y la salida real de sus procesos.
2. Verifica el paquete anterior con el registro de propiedad. Comprueba los hashes de un respaldo situado fuera del descubrimiento de skills. Conserva las ediciones antes de seguir.
3. Instala desde cero en `skills/agent-workflow-v1-1` con el manifiesto nuevo y su hash externo. No uses `--update` para una identidad técnica anterior.
4. Antes de cambiar las rutas de descubrimiento, ejecuta `workflow validate` desde la entrada instalada nueva y con un ID de verificación único. Comprueba la identidad, versión, espacio de trabajo y hash de manifiesto devueltos. Esto crea `run-binding.json` sin iniciar objetivo, estado de flujo ni monitor.
5. La nueva invocación es `$agent-workflow-v1-1`. Tras la validación, actualiza las rutas del rol principal y de los demás perfiles en Codex conservando el estado habilitado o deshabilitado y la política de invocación expresa. No pises ediciones simultáneas. Vuelve a revisar procesos propios y conserva instalaciones activas de otras sesiones. Con la transición comprobada, archiva el paquete anterior fuera del descubrimiento y verifica la configuración y el respaldo.
6. Conserva las identidades, rutas y hashes de puntos de control, vínculos y recibos anteriores. Abre otra ejecución con el paquete nuevo. Si interesa recuperar trabajo incompleto, aporta por separado las salidas verificadas y la autoridad actual real.
