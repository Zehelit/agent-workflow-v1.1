<a name="install-verify-and-maintain-agwf"></a>

# Instalar, verificar y mantener AGWF

Necesitas Python 3.11 o posterior y Git. La compilación y el programa candidate solo usan la biblioteca estándar de Python. Instala el paquete en un directorio de skills en el que la cuenta prevista pueda escribir. Los comandos usan PowerShell. En Linux o macOS, usa el directorio equivalente bajo la raíz de Codex y adapta la sintaxis. Las comprobaciones de ACL descritas se aplican a Windows.

<a name="build-from-the-selected-checkout"></a>

## Generar el paquete desde la copia de trabajo

Ejecuta estos comandos desde la raíz de AGWF. Anota el commit de origen y revisa los cambios locales antes de aceptar el paquete. El directorio de salida debe ser nuevo.

```powershell
python --version
git rev-parse HEAD
git status --short
$build = Join-Path $env:TEMP ('agwf-build-' + [guid]::NewGuid().ToString('N'))
python -X utf8 -B tooling/build-candidate.py --output $build
if ($LASTEXITCODE -ne 0) { throw 'AGWF build failed.' }
$receipt = Get-Content -LiteralPath (Join-Path $build 'build-receipt.json') -Raw | ConvertFrom-Json
$pin = $receipt.manifest_sha256
$package = Join-Path $build 'agent-workflow-v1-1'
$entry = Join-Path $package 'scripts/candidate.py'
$manifest = Join-Path $package 'manifest.json'
```

El empaquetador genera un directorio, un ZIP determinista y un recibo de compilación. Conserva el recibo aceptado fuera del paquete modificable, junto al commit de origen. El instalador espera el valor aceptado de `manifest_sha256`. Recalcular el hash después de una modificación no prevista solo produce una observación nueva. [Vinculación e instalación](../skills/agent-workflow-v1-1/references/coexistence.md).

<a name="choose-a-human-or-agent-path"></a>

## Instalación manual o a cargo de un agente

Las siguientes instrucciones apuntan a la raíz de skills del usuario actual de Codex. En una prueba desechable, indica un directorio aislado que termine en `agent-workflow-v1-1`. Utiliza el mismo comando para una instalación nueva o para verificar un paquete idéntico sin cambiarlo.

```powershell
$codexRoot = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { Join-Path $env:USERPROFILE '.codex' }
$destination = Join-Path $codexRoot 'skills/agent-workflow-v1-1'
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination $destination
if ($LASTEXITCODE -ne 0) { throw 'AGWF installation failed.' }
```

`activated:false` indica que la operación solo instaló o verificó los archivos. En Windows, el directorio de preparación hereda permisos del padre del destino. Comprueba que la cuenta ordinaria que usará Codex pueda leer y ejecutar los archivos instalados.

Para que lo instale un agente:

1. Verifica el destino y si se trata de una instalación nueva, una actualización o una prueba aislada. Antes de cambiar una instalación existente, inventaría el paquete, las ediciones del usuario y los procesos que le pertenecen.
2. Comprueba la identidad de la copia de trabajo y la versión de Python. Compila en una carpeta nueva y conserva el hash externo del manifiesto aceptado.
3. Usa el instalador candidate fijado con el destino acordado. Antes de actualizar, detén solo los procesos anteriores que pertenezcan a esa instalación y verifica que salieron. Si el instalador rechaza la operación, conserva el destino para inspeccionarlo.
4. Verifica el manifiesto y el contenido instalado. Ejecuta la comprobación pasiva de abajo desde el punto de entrada instalado. Comunica el commit, el hash del manifiesto, el destino, el resultado y si se realizó una comprobación pasiva o si se inició explícitamente un flujo real.

<a name="verify-the-installed-entrypoint"></a>

## Verificar el punto de entrada instalado

Usa un proyecto desechable y un identificador de ejecución nuevo. El comando escribe el vínculo de ejecución en el directorio desechable, sin arrancar un monitor ni activar un objetivo.

```powershell
$probe = Join-Path $env:TEMP ('agwf-probe-' + [guid]::NewGuid().ToString('N'))
New-Item -ItemType Directory -Path $probe -ErrorAction Stop | Out-Null
$installedEntry = Join-Path $destination 'scripts/candidate.py'
$installedManifest = Join-Path $destination 'manifest.json'
python -X utf8 -B $installedEntry --manifest $installedManifest --manifest-sha256 $pin --version 1.1 --project $probe --run-id install-check workflow validate
if ($LASTEXITCODE -ne 0) { throw 'AGWF binding validation failed.' }
```

Comprueba en la respuesta `identity`, `version`, `run_id`, `manifest_sha256` y `missing_dependencies`. En la prueba aislada para esta documentación, el instalador devolvió `activated:false`. Se creó `run-binding.json` sin archivo de estado ni carpeta del monitor. El manifiesto con hash incorrecto se rechazó con código 2.

La invocación real en Codex es un paso aparte. Una sesión abierta puede requerir que se actualice el catálogo de skills. Selecciona `$agent-workflow-v1-1` de forma explícita e indica el alcance. La [guía de operaciones](../skills/agent-workflow-v1-1/references/operations.md) explica cómo verificar el arranque autorizado y la salud del monitor.

<a name="update-recover-or-remove"></a>

## Actualizar, recuperar o desinstalar

- Antes de actualizar una instalación administrada, detén el proceso propio con su punto de entrada fijado y verifica su salida. Genera y acepta el paquete nuevo por separado. Añade `--update` solo a una instalación de propiedad verificada que permanezca intacta. Revisa primero las ediciones del usuario.
- El instalador valida el paquete preparado antes de sustituir la carpeta e intenta restaurar la anterior si falla la sustitución. Si también falla la restauración, conserva el directorio de la transacción. Una actualización correcta no guarda un respaldo permanente. [Detalles del instalador](../skills/agent-workflow-v1-1/references/coexistence.md).
- Al cambiar la identidad técnica, realiza una instalación desde cero en la nueva carpeta. Conserva las vinculaciones antiguas con sus manifiestos originales y gestiona por separado el descubrimiento de skills. Usa un identificador de ejecución nuevo después de aceptar una actualización. [Migración y límites](../skills/agent-workflow-v1-1/references/migration.md).
- No existe comando de desinstalación. Antes de retirar el paquete, confirma el directorio exacto, detén sus procesos y guarda una copia verificada fuera del alcance de descubrimiento de skills. Conserva los proyectos, el estado, los puntos de control y los recibos externos. Comprueba el descubrimiento después.
