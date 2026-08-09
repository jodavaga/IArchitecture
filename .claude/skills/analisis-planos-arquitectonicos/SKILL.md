---
name: analisis-planos-arquitectonicos
description: Analiza planos arquitectónicos (plantas, fachadas, cortes, cubiertas, detalles) de proyectos de vivienda, reforma o remodelación — ejes, niveles, cotas, áreas, tipología constructiva, materiales y programa de espacios. Úsalo SIEMPRE que el usuario suba o mencione un PDF/imagen de un plano, pida "analizar planos", "leer plano", "revisar plano", "extraer medidas", "clasificar materiales", "despiece" de un elemento (pérgolas, carpinterías, muros), presupuesto o cubicación a partir de planimetría, o trabaje como arquitecto/diseñador/presupuestista revisando CAD/Revit/SketchUp. Crítico para resaltar datos faltantes, ilegibles o inconsistentes ANTES de presupuestar o construir. Renderiza el PDF a alta resolución y cruza plantas por eje antes de concluir que un elemento no existe o no está diferenciado — nunca decide eso solo con el texto ya aplanado. No inventa valores — si un dato no se puede leer con confiabilidad, lo señala en vez de asumirlo.
---

# Análisis de Planos Arquitectónicos (vivienda / reforma)

Skill para leer, clasificar y auditar técnicamente planos arquitectónicos entregados como PDF o imagen — plantas, fachadas, cortes, plantas de cubierta, localización — con foco en proyectos de reforma/remodelación de vivienda. El objetivo no es solo describir el plano, sino producir una lectura utilizable para toma de decisiones de diseño y para presupuesto, **dejando explícitamente marcado todo lo que no se puede confirmar con lo que hay en la lámina.**

## Principio rector: nunca inventar un valor

Esta es la regla más importante del skill. Si una cota, material, escala o nivel no se puede leer de forma confiable en el documento:
- No se completa con un valor "razonable" o "típico de la industria".
- Se marca como vacío y se explica qué falta y qué lámina lo resolvería (estructural, hidrosanitaria, cuadro de carpintería, detalle a mayor escala, etc.).

Usa este sistema de etiquetas en cada afirmación no trivial:
- **[Confirmado]** — el dato está explícito y legible en el plano (cota acotada, texto claro, tabla de áreas).
- **[Probable]** — inferencia razonable a partir de convenciones de dibujo o repetición en el mismo set (p. ej. "las dos casas son espejadas porque los callouts se repiten"), pero no está declarado explícitamente.
- **[Suponiendo]** — se está llenando un vacío de información; requiere confirmación del usuario o de otra lámina antes de usarse en presupuesto o ejecución.

No mezclar niveles de confianza sin etiqueta. Si la mayoría de una sección es [Suponiendo], decirlo al inicio de esa sección, no enterrarlo.

## Principio rector 2: no confundir "no lo veo en el texto extraído" con "no está en el plano"

Corrección aprendida de un caso real: leer solo el texto/imagen ya aplanado que entrega la plataforma (render comprimido, callouts solapados o repetidos sin resolución de dónde apunta cada uno) puede llevar a concluir por error que un elemento no existe o no está diferenciado, cuando en realidad sí lo está — solo que a nivel de pixel, no de texto. Esto pasó concretamente con una planta de cubierta donde el mismo callout de material aparecía repetido 3 veces en 3 zonas distintas (pérgola de deck, de acceso y de parqueadero), y una lectura superficial del texto extraído concluyó erróneamente que era "una sola pérgola descrita una vez".

Regla: **antes de afirmar que un elemento no existe, no está diferenciado, o no se puede identificar, renderiza la lámina a alta resolución y haz zoom a la zona en cuestión.** No te quedes con la primera pasada de lectura de texto/imagen entregada en el contexto si la pregunta del usuario es específica sobre un elemento que "debería" verse en el plano.

### Cómo hacer la verificación visual (técnica)
Si tienes acceso a bash y el PDF original está en disco:
```bash
pdftoppm -png -r 300 "archivo.pdf" salida
```
Esto genera una imagen PNG por página a 300 dpi (mucho más nítida que el texto ya extraído). Luego recorta la zona de interés con Python/PIL antes de verla:
```python
from PIL import Image
Image.MAX_IMAGE_PIXELS = None
im = Image.open('salida-1.png')
w, h = im.size
crop = im.crop((x0, y0, x1, y1))  # usar fracciones de w,h para ubicar el cuadrante de interés
crop.save('recorte.png')
```
Luego usa la herramienta de vista sobre el recorte, no sobre la página completa (perderías resolución en el detalle). Repite el recorte en distintos cuadrantes si el primero no resuelve la duda.

### Cuando el mismo callout aparece repetido varias veces en una lámina
No lo colapses automáticamente en "un solo elemento descrito más de una vez". Cada repetición suele corresponder a una instancia física distinta (una pérgola distinta, un vano distinto, un muro distinto) que comparte especificación pero no ubicación. Para asignar función a cada instancia:
1. Ubica la posición de cada callout por eje/cuadrante en la lámina donde aparece (ej. planta de cubierta).
2. Cruza esa posición contra la planta arquitectónica del mismo nivel, que sí suele rotular la función del espacio (deck, acceso, parqueadero, terraza, etc.) en la misma zona de ejes.
3. Asigna función por coincidencia de eje/cuadrante, no por suposición — y marca [Probable] si la coincidencia de eje es aproximada, [Confirmado] si el rótulo cae exactamente en la misma zona.

## Flujo de análisis (8 pasos)

Ejecuta estos pasos en orden. Si el usuario pide "revisa este punto puntual" (p. ej. solo materiales, o solo cotas de un vano), igual corre mentalmente el paso relevante con el mismo rigor, pero no es necesario mostrar los 8 pasos completos — sí es obligatorio mostrar el hallazgo de vacíos de ese punto específico. **Si la pregunta del usuario es sobre un elemento específico (un componente, un despiece, un conteo de instancias), aplica primero la verificación visual a alta resolución del principio anterior antes de concluir cualquier ausencia o indiferenciación.**

### 1. Metadatos y control de láminas
Extraer y tabular: número de lámina, contenido, escala (ojo con "As indicated" — verificar si mezcla escalas entre vistas de la misma hoja), versión/revisión, fecha, firma/diseñador, alcance ("válido para..."). Si hay más de una lámina, verificar que la versión y fecha coincidan entre todas — una discrepancia de versión entre hojas del mismo set es una bandera roja de coordinación.

### 2. Sistema de ejes
Identificar ejes verticales (letras) y horizontales (números), o el criterio que use el plano. Registrar ejes intermedios o con prima (ej. "2'") — estos casi siempre responden a un desfase estructural o de partición que no se justifica en la planta arquitectónica; marcar como [Suponiendo] su origen y señalar que se debe cruzar con la planta estructural.

### 3. Niveles
Tabular todos los niveles con cota (Nivel 1, Enrase, Cubierta, etc.). Cualquier cota numérica suelta sin etiqueta clara (ej. un número cerca de una nota de urbanismo) no se asume como nivel de piso — se marca [Suponiendo] y se pide la lámina de topografía/urbanismo para resolverla.

### 4. Áreas y cotas
- Tabular áreas licenciadas, área de reforma, área total, área privada — por unidad si hay más de una.
- Si hay asimetría entre unidades "gemelas" (misma área licenciada pero distinta área de reforma), señalarlo explícitamente: esto es una diferencia real de costo, no se promedia entre unidades.
- Registrar las cadenas de cotas visibles en fachadas/cortes, pero **no cerrar cubicación de muros con ellas si el espesor de muro no está acotado de forma independiente.**

### 5. Clasificación de espacios / programa
Listar, por unidad si aplica, cada intervención de reforma agrupada por ambiente (cocina, alcobas, baños, social, exteriores) y por disciplina si el plano lo separa (arquitectónico vs eléctrico vs hidrosanitario). Cruzar dimensiones de vanos/mobiliario contra criterios ergonómicos reales (ver `references/ergonomia.md`) y marcar cualquier incumplimiento.

**Chequeo de inconsistencia obligatorio:** comparar cada dimensión que aparezca tanto en nota/leyenda como en cota de dibujo (ej. tipo de ventana + medida en el texto vs. la medida acotada en la sección). Si no coinciden, no elegir una — reportar ambas y marcar como pendiente de reconciliar contra el cuadro de carpintería.

### 6. Tipología constructiva y materiales
Tabular por componente (muro, estructura, cubierta, carpintería, cielo, pisos si están indicados) el sistema constructivo y acabado, diferenciando por unidad si los colores/specs cambian. Si el escaneo tiene texto cortado, espejado o superpuesto (común en plantas simétricas con anotaciones a ambos lados), no completar por analogía sin marcarlo [Suponiendo] — pedir export vectorial limpio si se necesita para contrato.

Si el usuario pide un **despiece por elemento específico** (una pérgola, una carpintería, un tipo de muro) y el mismo callout de material aparece más de una vez en la lámina, no asumas que es un solo elemento — aplica el procedimiento de verificación visual y cruce por eje del principio rector 2 antes de responder cuántas instancias hay y qué función cumple cada una.

### 7. Cortes, fachadas y referencias cruzadas
Listar cada corte/fachada/planta con su lámina de origen y verificar que el paquete de envolvente esté completo (todas las fachadas + cubierta + cortes suficientes para entender el volumen). Señalar cualquier corte o fachada referenciado en una lámina pero no encontrado en el set entregado.

### 8. Detalles constructivos
Listar cada detalle referenciado (llamado con un globo/referencia) y verificar si el contenido del detalle está realmente presente y legible en el set. Priorizar detalles en puntos de riesgo (impermeabilización, claraboyas, remates de cubierta, encuentros de carpintería con muro) — si no están a resolución/escala legible, marcarlo como bloqueante antes de presupuestar o entregar a obra, no como una nota menor.

## Salida esperada

- Responder siempre en español salvo que el usuario pida explícitamente otro idioma en el chat (la instrucción más reciente del usuario en la conversación manda sobre configuraciones previas).
- Usar tablas para: metadatos de láminas, áreas, materiales por componente, comparativas.
- Cerrar siempre con una sección **"Pendientes antes de presupuestar / construir"** listando cada vacío detectado, qué lámina o dato lo resuelve, y por qué importa (riesgo técnico, riesgo de costo, o coordinación).
- No pasar a estructurar partidas de presupuesto (APUs) mientras haya vacíos [Suponiendo] que afecten cantidades — ofrecer al usuario la opción de continuar con una línea de contingencia explícita si decide avanzar de todas formas, pero no decidir por él.
- Tono: colega sénior, técnico y directo. No usar frases de relleno tipo "excelente pregunta" ni abrir con acuerdo automático.

## Recursos adicionales

- `references/ergonomia.md` — dimensiones ergonómicas reales de referencia (circulaciones, alturas de instalación, profundidades de mobiliario, radios de apertura) para contrastar contra lo que muestra el plano. Consultar cuando el paso 5 detecte un espacio o mueble que parezca ajustado.
- `references/vacios_frecuentes.md` — checklist de los vacíos de información que más se repiten en sets de planos de reforma residencial (para no depender solo de la memoria durante el análisis).