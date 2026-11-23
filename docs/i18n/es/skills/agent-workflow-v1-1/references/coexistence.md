<a name="pinned-runtime-binding-and-installation"></a>

# Vinculación del entorno de ejecución e instalación

La identidad técnica es `agent-workflow-v1-1` y el nombre visible `agent-workflow-v1.1`. La invocación implícita está desactivada. Solo una selección expresa adopta el candidate. El modelo principal lo elige el usuario. Los perfiles del flujo solo usan `gpt-6.1-sol`, `gpt-6-astra` o `gpt-6.1-terra` con nivel de razonamiento mínimo `medium`. El asesor privado está fijado a Terra/medium. No hay modelo alternativo.

La selección expresa vigente exige al agente principal arrancar y verificar en el mismo turno la ejecución del ámbito elegido. Puede volver a habilitar ese ámbito tras una desactivación; una selección antigua o una actualización no lo habilita. Si falta la prueba de activación completa o se prohíbe crear un objetivo nativo, usa la supervisión manual prevista en [operaciones](operations.md). Confirma el inicio antes de comenzar trabajo sustancial. Respeta cualquier orden de parada posterior.

Genera el paquete con Python 3.11+ y la biblioteca estándar:

```powershell
python -X utf8 -B tooling/build-candidate.py --output C:/staging/build-one
```

La salida contiene `agent-workflow-v1-1/`, un ZIP determinista y `build-receipt.json`. Conserva `manifest_sha256` del recibo en las pruebas aceptadas de coordinación o traspaso, fuera del paquete modificable. El valor externo aportado por quien llama es el anclaje de confianza. El manifiesto completo y las dependencias se revisan antes de vincular la ejecución, escribir transiciones importantes, coordinar o evaluar una medición. La señal periódica solo comprueba el control de ejecución y época. Recalcular el hash de un manifiesto modificado no demuestra aprobación externa. Python comprueba la identidad y la integridad de los bytes, sin autenticar la aprobación humana, los metadatos del proveedor o un sistema de archivos controlado por quien llama. Si el ejecutable o el intérprete están comprometidos o quien llama controla tanto el vínculo como el hash externo aceptado, eso queda fuera del contrato.

Pon las opciones globales antes del modo. Para el código fuente usa `skills/agent-workflow-v1-1/scripts/candidate.py` y el manifiesto generado. Para la instalación usa su `scripts/candidate.py` y manifiesto, con el mismo hash externo conservado. Da un ID separado a cada mapeo de fuente o instalación. El vínculo inmutable conserva las rutas resueltas de dependencias y manifiesto, proyecto, ejecución, versión y hashes. Rechaza mover la ejecución a otro ayudante, cambiar su paquete o sus dependencias, quitar archivos o reutilizarla con otro hash. Tras una actualización aprobada, inicia otra ejecución.

```powershell
$pin = '<preserved manifest_sha256 from accepted build receipt>'
$entry = 'C:/staging/build-one/agent-workflow-v1-1/scripts/candidate.py'
$manifest = 'C:/staging/build-one/agent-workflow-v1-1/manifest.json'
$common = @('-X','utf8','-B',$entry,'--manifest',$manifest,'--manifest-sha256',$pin,'--version','1.1','--project','C:/project','--run-id','candidate-run-1')
python @common workflow validate
python @common workflow status
python @common coordination --input C:/project/batch.json
```

`workflow activate` o `bind-active-goal` autorizado inicia el monitor y comprueba su primera muestra vigente pasada por `--usage-input`. `resume` abre otra época tras una continuación válida o la recuperación controlada de una medida caducada bajo el mismo objetivo activo autorizado. Requiere un nuevo recibo nativo. Consulta [operaciones](operations.md) y [uso nativo](native-usage.md). La medición no lanza otro servidor ni otra CLI de Codex. Los comandos activos renuevan la vigencia y `complete`/`halt` detienen el proceso Python propio y comprueban su salida. Los comandos `watcher` respetan el mismo contrato.

`workflow` acepta los comandos tipados del núcleo y `validate`. Leen JSON mediante `--input` o stdin. Las antiguas CLI del núcleo `native/install/export/migrate` no están disponibles. La autorización de `activate` y la prueba de `bind-active-goal` incluyen el ID elegido. `validate` verifica roles y dependencias y vincula el espacio sin activar el objetivo. Un lote tiene la forma `{ "batch_id": "inspect", "steps": [{ "key": "status", "command": "status", "payload": {} }] }`, hasta 64 pasos, un escritor principal, resultados pendientes duraderos y sin reintentos ni envíos nativos automáticos. Los comandos importantes mantienen sus comprobaciones de autoridad y aceptación. Un inventario terminal no acepta archivos. El trabajo obsoleto sin escritores activos se puede descartar con `finish accepted:false`. La compactación exige un punto de control verificado y archiva tareas e invalidaciones.

La raíz aislada es `<project>/.agent-workflow-v1-1/runs/<run-id>/`. Las tareas no pueden reclamar los directorios operativos. El monitor usa allí `watcher/`, un planificador fijado, una propiedad inmutable y parámetros de reinicio del candidate. Controles ajenos a la ejecución o una CLI privada directa no pueden renovar ni detener otro proceso. Un chat inactivo no renueva el plazo. Ejecuta un comando activo en los límites del trabajo. Una actualización no migra procesos vivos. No hay otro lanzador ni otra skill de monitor y las dependencias privadas proceden del paquete aprobado.

Instalación nueva, verificación sin cambios y actualización administrada exigen un destino explícito terminado en `agent-workflow-v1-1`:

```powershell
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination C:/staging/skills/agent-workflow-v1-1
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination C:/staging/skills/agent-workflow-v1-1 --update
```

Para actualizar, la instalación debe pertenecer al instalador y conservar sus bytes. Se valida el paquete en una carpeta temporal antes de reemplazarla. Si falla el reemplazo, se restaura el directorio anterior. Si la operación tiene éxito, se quitan los archivos de la transacción. No hay archivo de reversión permanente ni comando de reversión. Si también falla la restauración, conserva el directorio de transacción, que guarda la única copia anterior, hasta verificar la recuperación. Los comandos solo preparan archivos y devuelven `activated:false`. Descubrimiento en el catálogo, selección explícita, modelo observado en el proveedor y aceptación real son comprobaciones posteriores separadas.

En Windows, el instalador cambia la ACL del área temporal para heredar permisos del directorio padre del destino antes de copiar. No usa el nombre de una cuenta particular. La cuenta nueva o renombrada recibe los permisos de su propio directorio padre `skills`, que debe conceder lectura y ejecución. Comprueba descubrimiento y acceso al candidate de manera directa desde un Codex normal, sin elevación. El éxito con una cuenta elevada no basta.
