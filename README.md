# Proyecto Final — Control inteligente de compras con IA + RAG

**Coderhouse — Carrera AI Automation**  
**Estado:** prototipo funcional de apoyo a decisiones; no habilitado para emitir órdenes de compra automáticamente.

## 1. Problema de negocio

En el abastecimiento de insumos secos de una bodega, la información de stock, los conteos físicos y las necesidades del cronograma de producción pueden estar distribuidos en distintas planillas. Esto hace más lenta la detección de faltantes y aumenta el riesgo de comprar con datos incompletos, equivalencias equivocadas o sin validar el depósito.

**Objetivo:** integrar las fuentes de información, consolidar existencias por código, contrastar la producción planificada con los insumos identificados y producir recomendaciones fundamentadas y auditables, sujetas a revisión humana.

## 2. Solución técnica

Se implementaron dos workflows en **n8n Cloud**, conectados con **Google Sheets**, **Google Forms** (a través de su hoja de respuestas), un **AI Agent con OpenAI gpt-5-mini**, y **Supabase Vector Store** para recuperación de políticas mediante RAG.

### Workflow A — Control de compras IA + RAG

1. **Inventario:** lee el inventario original en modo consulta; consolida cantidades por código de producto y detecta datos incompletos, lotes repetidos y discrepancias. Actualiza la hoja de trabajo `02_STOCK_SISTEMA`.
2. **Conteos físicos:** lee las respuestas del formulario, valida campos, comprueba la existencia del código y lote y registra el resultado en `03_VALIDACIONES`.
3. **Planificación:** lee la pestaña semanal del programa de llenado, descarta filas de totales y normaliza cantidades, variedades y descripciones de insumos.
4. **Equivalencias:** consulta `04_EQUIVALENCIAS_INSUMOS`, identifica coincidencias y calcula necesidades y faltantes *teóricos*; marca los mapeos no aprobados para revisión.
5. **Análisis IA:** prepara el informe de contexto, lo entrega al agente y consulta mediante una herramienta RAG las políticas documentadas en Supabase.
6. **Control final:** valida la salida del agente con reglas determinísticas y registra la recomendación, explicación y estado en `05_DECISIONES_IA`.

### Workflow B — Carga de políticas RAG

1. Inicia una carga manual.
2. Crea el texto de las **políticas de demostración** de abastecimiento.
3. Lo procesa con **Default Data Loader** y **Recursive Character Text Splitter** (tamaño 500 caracteres; solapamiento 80).
4. Genera embeddings con OpenAI e inserta los fragmentos en la tabla `documents` de **Supabase**.
5. El workflow principal consulta esos fragmentos mediante la función `match_documents`.

**Nota:** la configuración de demostración puede añadir duplicados si se ejecuta reiteradamente la carga sin limpiar o gestionar los documentos existentes. En una versión productiva se requerirían IDs documentales, versionado y actualización controlada.

## 3. ¿Por qué AI Agent y por qué RAG?

El **AI Agent** permite combinar análisis de datos de inventario y producción con la consulta dinámica de una herramienta de conocimiento, y explicar la recomendación obtenida. **RAG** conserva las políticas como documentos externos recuperables, en vez de depender exclusivamente del conocimiento del modelo o incrustar todas las políticas en el prompt. La decisión final, sin embargo, no queda exclusivamente en manos del modelo: se agregaron validaciones en nodos JavaScript.

Se eligió un modelo **en la nube (OpenAI)** para la demostración, priorizando integración con n8n y velocidad de implementación; esto implica dependencia de un proveedor externo, costos variables de consultas y necesidad de gestionar correctamente la privacidad de los datos.

## 4. Políticas y restricciones

Las políticas de prueba establecen que:

- Las equivalencias de insumos deben estar **APROBADAS** antes de proponer una compra.
- El stock del sistema no equivale a disponibilidad física confirmada.
- Los conteos incompletos, los códigos desconocidos y las producciones sin equivalencia deben marcarse para revisión.
- Una propuesta de compra requiere datos completos y autorización humana.
- El sistema **no emite órdenes de compra**, no confirma precios o proveedores y no modifica el inventario original.

**Limitación real del prototipo:** el nodo de validación final mantiene deshabilitada la compra automática y, en ausencia de bloqueos, utiliza `PENDIENTE_EVALUACION`. No se implementó el circuito completo de generación/aprobación de una propuesta ni el envío de órdenes a proveedores.

## 5. Prueba realizada y resultado observado

En la ejecución de demostración se analizaron **3 registros**: dos necesidades identificadas y una producción sin equivalencia:

| Registro | Necesidad prevista | Stock de sistema | Resultado |
|---|---:|---:|---|
| `BOT3011` — botella | 4.800 | 12.760 | `REVISAR_DATOS`: equivalencia pendiente |
| `TP4007` — tapa | 4.800 | 8.500 | `REVISAR_DATOS`: equivalencia pendiente |
| Producción MALBEC | 11.368 botellas | No determinable por falta de equivalencias | `REVISAR_DATOS` |

La herramienta Supabase RAG devolvió contenido de las políticas sobre equivalencias, validación física y aprobación humana. El agente fundamentó su recomendación en esos criterios y el workflow registró la salida en `05_DECISIONES_IA`. El **faltante teórico = 0** para los dos códigos no significa stock disponible verificado.

### Evidencias recomendadas

Adjuntar capturas de: (1) workflow A completo, (2) workflow B completo, (3) ejecución exitosa de inserción en Supabase, (4) respuesta del nodo de consulta RAG, (5) salida del agente, (6) validación determinística y (7) registro en Google Sheets. **Las capturas deben obtenerse de la ejecución real, no generarse artificialmente.**

## 6. Estimación de ROI (escenario hipotético)

Para evaluar el potencial del caso, se propone un escenario **ilustrativo**, que debe validarse con mediciones reales:

| Supuesto mensual | Valor ilustrativo |
|---|---:|
| Análisis manuales de abastecimiento | 20 |
| Tiempo manual por análisis | 45 minutos |
| Tiempo asistido por análisis | 15 minutos |
| Tiempo potencialmente ahorrado | 10 horas / mes |
| Costo de hora de trabajo (supuesto) | ARS 10.000 |
| Beneficio bruto estimado por tiempo | ARS 100.000 / mes |
| Infraestructura, modelos y mantenimiento (supuesto) | ARS 30.000 / mes |
| Beneficio neto operativo estimado | ARS 70.000 / mes |

**Fórmula:** `(análisis mensuales × horas ahorradas por análisis × costo horario) − costos mensuales`. La cifra no incluye costo de desarrollo, amortización, capacitación, ni ahorros por faltantes evitados. No representa un retorno medido de la bodega.

## 7. Riesgos y mitigaciones

| Riesgo | Mitigación implementada o prevista |
|---|---|
| Equivalencias incorrectas | Estado de aprobación obligatorio; bloqueo ante pendientes |
| Stock del sistema desactualizado | Distinguir cobertura teórica de validación física |
| Salida incorrecta o inventada de IA | Prompt restrictivo, RAG y validación posterior en código |
| Compras no autorizadas | Sin emisión automática; aprobación humana requerida |
| Falta de datos o documentos | Recomendar `REVISAR_DATOS` y explicar las causas |
| Acceso a documentos o credenciales | Publicar JSON sin IDs de documentos ni referencias de credenciales; limitar permisos |
| Dependencia de nube y costo variable | Monitorear consumo, presupuesto, disponibilidad y errores |

## 8. Nota sobre la numeración de los nodos

Durante el desarrollo se realizaron distintas pruebas de almacenamiento y recuperación vectorial. Inicialmente se utilizó **Simple Vector Store**; luego se migró a **Supabase Vector Store** para contar con persistencia y permitir la consulta desde el workflow principal. Al finalizar las pruebas se eliminaron los nodos antiguos que ya no eran necesarios.

Se decidió **conservar la numeración original de los nodos activos** para preservar las referencias internas de JavaScript y la trazabilidad de la implementación. Por eso, algunos números no aparecen consecutivamente. Estos saltos son intencionales y no afectan las conexiones ni el funcionamiento del sistema.

## 9. Archivos entregables y puesta en marcha

- `01_CONTROL_COMPRAS_IA_RAG_PUBLICABLE.json`: workflow principal.
- `02_CARGA_POLITICAS_SUPABASE_PUBLICABLE.json`: ingesta de políticas en Supabase.
- `README.md`: documentación técnica y funcional.
- `evidencias/`: carpeta sugerida con capturas reales de pruebas.

Los **JSON publicables** se prepararon quitando referencias a credenciales y localizadores/IDs de hojas externas. **No se ejecutan sin configuración:** para reutilizarlos, importar en n8n, volver a vincular las cuentas Google Sheets, OpenAI y Supabase, elegir los archivos y pestañas correspondientes, y crear la tabla `documents` y función `match_documents` con dimensión de embeddings compatible. También es necesario cargar las políticas antes de hacer consultas RAG.

**No publicar** claves API, tokens, credenciales, URLs de documentos privados o datos operativos sensibles. Mantener los JSON originales de trabajo en almacenamiento privado.

## 10. Alcance y próximos desarrollos

El prototipo demuestra integración entre planillas, detección de pendientes, recomendación asistida por IA, recuperación RAG persistente y registro de resultados. Queda fuera del alcance de esta entrega la puesta en producción, automatización de autorizaciones, cálculo consolidado de todas las reservas y órdenes en tránsito, conteo físico multi-lote completo con cierre automático, monitoreo/alertas y pruebas sistemáticas con datos reales en distintos escenarios.
