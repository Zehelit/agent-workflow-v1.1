<a name="lifecycle-reliability-specification"></a>

# Especificación de fiabilidad del ciclo de vida

> Especificación histórica del 10 de noviembre de 2026. Las pruebas y la autorización correspondían a ese trabajo. Consulta el [contrato vigente](../../../skills/agent-workflow-v1-1/references/operations.md) para el comportamiento actual.

<a name="intended-result"></a>

## Resultado esperado

Hacer fiable la primera activación, renovación de uso, recuperación de una interrupción y final normal. Preparar la prueba antes de que se ejecute el objetivo. Informar separadamente de inicio, recuperación desde un flujo detenido y tiempo.

<a name="evidence-available-at-the-time"></a>

## Evidencia disponible al redactar

- El informe privado de temporización registró una medida caducada antes de activar. La prueba de diez segundos falló en ambos intentos, con 98,74 y 45,05 segundos. Una época nueva y un primer vínculo en la segunda ronda no demostraron que se reanudara un ayudante detenido.
- El informe privado de flujo ordinario registró inicio sano, otra medición real 43,423 segundos después, punto de control verificado, cierre normal y salida. No hubo bloqueo SAVE en esa ejecución.
- `scripts/candidate.py` ya podía abrir una época técnica y restaurar controles tras un `resume` fallido. El primer `activate` o `bind-active-goal` carecía del mismo diario de repetición. Hacía falta una prueba de regresión independiente.
- El mantenimiento previo resolvió conflictos de instrucciones y autoridad y cuatro fallos de espera o auxiliares de pruebas. Había que preservar esas correcciones.

<a name="requirements"></a>

## Requisitos

1. Reintentar el primer arranque bajo la misma autoridad solo con un `usage_receipt_expired` registrado para el mismo objetivo y ejecución, uso real reciente, un objetivo activo realmente observado y ausencia confirmada del proceso. Sin pedir otra cita o ejecución ni reescribir la medición.
2. Rechazar cuota, reserva anticipada, parada de usuario, objetivo inactivo o ajeno, propiedad dudosa y cambio de paquete o evidencia. Python comprueba forma y coherencia, no autentica de forma independiente a la persona o herramienta indicada.
3. Hacer visibles los intentos fallidos. Un arranque a medias no cuenta como activo ni elimina un SAVE real posterior.
4. Preparar paquete y pruebas antes de medir. Conservar el tiempo de observación. Una fecha de publicación nueva no renueva uso antiguo. Mantener `ingest` separado de renovar la vigencia.
5. En ejecución normal, actuar ante `CHECKPOINT_SOON` o `SAVE_AND_STOP` reales. Evitar paradas artificiales.
6. Probar Continue nativo y reanudación real desde actividad y halt de forma separada del primer vínculo.
7. Solo medir tiempos cuando se haya pedido. Preparar el arnés antes del evento real. Llegar tarde no cuenta como éxito. La tolerancia propuesta para pedir SAVE era 10 ± 1 segundos y se informa el valor real.
8. El agente principal sigue operando los objetivos nativos. No se añade acceso no documentado, otro transporte Codex ni lectura de credenciales al monitor. Para medir desde el clic exacto se requiere una fuente de evento real.
9. Comprobar por separado el punto de control, la salida del proceso, el cierre del ayudante y el cierre nativo. Pausar requiere autorización, y `blocked` se rige por las condiciones de la herramienta.
10. Mantener Python 3.11+ y la biblioteca estándar. Un error de uso compartido en Windows no prueba salida ni éxito. Preservar las ediciones locales y las ejecuciones antiguas, sin reescribir sus vínculos.

<a name="original-authorization"></a>

## Autorización original

Primero se autorizó planificar. Después se autorizó implementar ese trabajo, entregar los cambios v1.1 mediante commit y push y renovar la documentación. Un GitHub Release o etiqueta quedó fuera de tal alcance. El caso de parada temporizada real siguió siendo un diagnóstico separado. Se conservaron los vínculos de ejecuciones anteriores.
