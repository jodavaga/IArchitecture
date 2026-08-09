---
name: analisis-planos-arquitectonicos
description: Analiza planos arquitectónicos (plantas, fachadas, cortes, cubiertas, detalles) de vivienda/reforma: ejes, niveles, cotas, áreas, materiales y programa. Úsalo cuando el usuario suba/mencione un PDF o imagen de plano, pida analizar, leer o revisar un plano, extraer medidas, clasificar materiales, hacer despiece de un elemento, o presupuestar/cubicar a partir de planimetría. No inventa valores faltantes; los marca como pendientes. Al terminar, siempre guarda el informe y evidencia visual curada en la carpeta outputs del proyecto (subcarpeta espejo de proyectos/); cada pregunta puntual de seguimiento sobre ese mismo plano también se guarda, en su propia subcarpeta descriptiva dentro de esa carpeta de outputs.
---

# Análisis de Planos Arquitectónicos (vivienda / reforma)

Skill para leer, clasificar y auditar técnicamente planos arquitectónicos entregados como PDF o imagen — plantas, fachadas, cortes, plantas de cubierta, localización — con foco en proyectos de reforma/remodelación de vivienda. El objetivo no es solo describir el plano, sino producir una lectura utilizable para toma de decisiones de diseño y para presupuesto, **dejando explícitamente marcado todo lo que no se puede confirmar con lo que hay en la lámina**, y **dejando ese análisis guardado como archivo**, no solo como respuesta de chat — esto aplica tanto al análisis inicial completo como a cada pregunta puntual posterior.

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
Luego usa la herramienta de vista sobre el recorte, no sobre la página completa (perderías resolución en el detalle). Repite el recorte en distintos cuadrantes si el primero no resuelve la duda. Si el texto extraído/OCR falla o es ambiguo en un recorte, apóyate en `pytesseract` (idioma `eng` suele funcionar mejor que `spa` si el paquete de idioma español no está instalado) para confirmar cifras y notas antes de darlas por buenas.

**Si la sesión agota el límite de imágenes que puede mostrar en el chat** (la herramienta de lectura de imágenes empieza a rechazar archivos nuevos, incluso pequeños, con un error de "many-image requests" o similar), no insistas reintentando ni reduciendas de tamaño indefinidamente. Cambia de inmediato a extracción por OCR (`pytesseract.image_to_string` sobre el recorte) para seguir leyendo cotas y notas sin depender de la vista de imagen, y sigue aplicando el mismo rigor de no inventar valores: si el OCR es ambiguo, repórtalo como no confirmado en vez de adivinar.

### Cuando el mismo callout aparece repetido varias veces en una lámina
No lo colapses automáticamente en "un solo elemento descrito más de una vez". Cada repetición suele corresponder a una instancia física distinta (una pérgola distinta, un vano distinto, un muro distinto) que comparte especificación pero no ubicación. Para asignar función a cada instancia:
1. Ubica la posición de cada callout por eje/cuadrante en la lámina donde aparece (ej. planta de cubierta).
2. Cruza esa posición contra la planta arquitectónica del mismo nivel, que sí suele rotular la función del espacio (deck, acceso, parqueadero, terraza, etc.) en la misma zona de ejes.
3. Asigna función por coincidencia de eje/cuadrante, no por suposición — y marca [Probable] si la coincidencia de eje es aproximada, [Confirmado] si el rótulo cae exactamente en la misma zona.

## Flujo de análisis (8 pasos)

Ejecuta estos pasos en orden. Si el usuario pide "revisa este punto puntual" (p. ej. solo materiales, o solo cotas de un vano), igual corre mentalmente el paso relevante con el mismo rigor, pero no es necesario mostrar los 8 pasos completos — sí es obligatorio mostrar el hallazgo de vacíos de ese punto específico. **Si la pregunta del usuario es sobre un elemento específico (un componente, un despiece, un conteo de instancias), aplica primero la verificación visual a alta resolución del principio anterior antes de concluir cualquier ausencia o indiferenciación.**

Si el archivo entregado contiene varias unidades/casas en la misma lámina o set (por ejemplo un PDF de varias páginas donde cada página es "VÁLIDO PARA CASA X") y el usuario pide analizar solo una unidad puntual, identifica primero cuál página/lámina corresponde a esa unidad (por el rótulo "VÁLIDO PARA..." o equivalente) y limita todo el análisis y el informe a esa página — no proceses ni describas el resto de unidades salvo que se te pida.

### 1. Metadatos y control de láminas
Extraer y tabular: número de lámina, contenido, escala (ojo con "As indicated" — verificar si mezcla escalas entre vistas de la misma hoja), versión/revisión, fecha, firma/diseñador, alcance ("válido para..."). Si hay más de una lámina, verificar que la versión y fecha coincidan entre todas — una discrepancia de versión entre hojas del mismo set es una bandera roja de coordinación.

### 2. Sistema de ejes
Identificar ejes verticales (letras) y horizontales (números), o el criterio que use el plano. Registrar ejes intermedios o con prima (ej. "2'") — estos casi siempre responden a un desfase estructural o de partición que no se justifica en la planta arquitectónica; marcar como [Suponiendo] su origen y señalar que se debe cruzar con la planta estructural.

### 3. Niveles
Tabular todos los niveles con cota (Nivel 1, Enrase, Cubierta, etc.). Cualquier cota numérica suelta sin etiqueta clara (ej. un número cerca de una nota de urbanismo) no se asume como nivel de piso — se marca [Suponiendo] y se pide la lámina de topografía/urbanismo para resolverla. Si un nivel aparece rotulado como una "opción" entre varias (p. ej. "Nivel -1 Opción 3"), señalarlo explícitamente como pendiente de confirmar cuál opción rige para la unidad/vivienda analizada.

### 4. Áreas y cotas
- Tabular áreas licenciadas, área de reforma, área total, área privada — por unidad si hay más de una.
- Si hay asimetría entre unidades "gemelas" (misma área licenciada pero distinta área de reforma), señalarlo explícitamente: esto es una diferencia real de costo, no se promedia entre unidades.
- Registrar las cadenas de cotas visibles en fachadas/cortes, pero **no cerrar cubicación de muros con ellas si el espesor de muro no está acotado de forma independiente.**
- Verificar que las cotas parciales de una cadena sumen la cota total marcada; si no cierran, reportar la discrepancia numérica exacta en vez de ignorarla.

### 5. Clasificación de espacios / programa
Listar, por unidad si aplica, cada intervención de reforma agrupada por ambiente (cocina, alcobas, baños, social, exteriores) y por disciplina si el plano lo separa (arquitectónico vs eléctrico vs hidrosanitario). Cruzar dimensiones de vanos/mobiliario contra criterios ergonómicos reales (ver `references/ergonomia.md`) y marcar cualquier incumplimiento.

**Chequeo de inconsistencia obligatorio:** comparar cada dimensión que aparezca tanto en nota/leyenda como en cota de dibujo (ej. tipo de ventana + medida en el texto vs. la medida acotada en la sección). Si no coinciden, no elegir una — reportar ambas y marcar como pendiente de reconciliar contra el cuadro de carpintería.

### 6. Tipología constructiva y materiales
Tabular por componente (muro, estructura, cubierta, carpintería, cielo, pisos si están indicados) el sistema constructivo y acabado, diferenciando por unidad si los colores/specs cambian. Si el escaneo tiene texto cortado, espejado o superpuesto (común en plantas simétricas con anotaciones a ambos lados), no completar por analogía sin marcarlo [Suponiendo] — pedir export vectorial limpio si se necesita para contrato.

Si el usuario pide un **despiece por elemento específico** (una pérgola, una carpintería, un tipo de muro) y el mismo callout de material aparece más de una vez en la lámina, no asumas que es un solo elemento — aplica el procedimiento de verificación visual y cruce por eje del principio rector 2 antes de responder cuántas instancias hay y qué función cumple cada una.

**Chequeo de concordancia de especificación entre vistas:** cuando el mismo elemento (misma marca de perfil, misma viga) aparece rotulado en más de una vista de la lámina, comparar literalmente el texto de cada rótulo — acabado (p. ej. "negro mate" vs "negro semimate") y orden de dimensiones (p. ej. "150X100X4" vs "100X150X4"). No asumir que es la misma especificación solo porque el nombre del perfil coincide a simple vista; reportar la discrepancia textual exacta y marcarla como pendiente de reconciliar contra el plano estructural.

### 7. Cortes, fachadas y coordinación de medidas
Listar cada corte/fachada/planta con su lámina de origen y verificar que el paquete de envolvente esté completo (todas las fachadas + cubierta + cortes suficientes para entender el volumen). Señalar cualquier corte o fachada referenciado en una lámina pero no encontrado en el set entregado.

**Chequeo de concordancia obligatorio entre vistas** — no basta con que la vista exista, sus cotas tienen que coincidir con las de las otras vistas del mismo elemento:
- **Alturas**: la altura de piso a techo / entrepiso / cumbrera en el corte debe coincidir con la misma altura leída en la fachada. Si no coinciden, no promediar ni elegir una — reportar ambas con su lámina de origen.
- **Niveles (paso 3)**: cada nivel tabulado debe aparecer con la misma cota en corte y en fachada. Un nivel que solo aparece en una vista es una bandera roja de coordinación.
- **Vanos (puertas/ventanas)**: posición (por eje) y dimensión de cada vano deben coincidir entre planta, fachada y corte. Si un vano cambia de ancho/alto entre vistas, marcarlo como inconsistencia a reconciliar contra el cuadro de carpintería.
- **Espesores de muro**: el espesor que se deduce del corte debe ser coherente con lo acotado (o deducible) en planta. Si en planta el muro no está acotado de forma independiente (ver paso 4), no cerrar esta verificación — dejarla marcada [Suponiendo].
- **Ejes (paso 2)**: los mismos ejes deben poder rastrearse entre planta, fachada y corte. Si un eje presente en planta no se identifica en el corte correspondiente, señalarlo explícitamente en vez de asumir que es el mismo punto.
- **Vistas 3D / isométricos vs. vistas acotadas (planta, alzado, corte)**: cuando la lámina incluye un render o isométrico junto a las vistas técnicas acotadas, verificar que ambos representen la misma configuración. Es un error real y ya observado que el isométrico refleje una reforma (p. ej. una isla de cocina nueva con un equipo reubicado) mientras que la planta y los alzados/cortes acotados —los que realmente sirven para fabricar y presupuestar— sigan mostrando la configuración anterior sin ese elemento. Si el render y las vistas acotadas no coinciden, es un hallazgo bloqueante, no un detalle menor: repórtalo en la sección de pendientes con máxima prioridad, porque presupuestar o fabricar con la vista acotada desactualizada contradice lo que el render (y probablemente el cliente) espera.

Cada discrepancia encontrada en este chequeo se reporta con su etiqueta de confianza y pasa directo a la sección "Pendientes antes de presupuestar / construir" — es, junto con los detalles del paso 8, la fuente más frecuente de sobrecostos por coordinación deficiente.

### 8. Detalles constructivos
Listar cada detalle referenciado (llamado con un globo/referencia) y verificar si el contenido del detalle está realmente presente y legible en el set. Priorizar detalles en puntos de riesgo (impermeabilización, claraboyas, remates de cubierta, encuentros de carpintería con muro) — si no están a resolución/escala legible, marcarlo como bloqueante antes de presupuestar o entregar a obra, no como una nota menor.

## Salida esperada (respuesta en el chat)

- Responder siempre en español salvo que el usuario pida explícitamente otro idioma en el chat (la instrucción más reciente del usuario en la conversación manda sobre configuraciones previas).
- Usar tablas para: metadatos de láminas, áreas, materiales por componente, comparativas (incluida la comparativa de cotas cruzadas entre planta/fachada/corte del paso 7).
- Cerrar siempre con una sección **"Pendientes antes de presupuestar / construir"** listando cada vacío detectado, qué lámina o dato lo resuelve, y por qué importa (riesgo técnico, riesgo de costo, o coordinación).
- No pasar a estructurar partidas de presupuesto (APUs) mientras haya vacíos [Suponiendo] que afecten cantidades — ofrecer al usuario la opción de continuar con una línea de contingencia explícita si decide avanzar de todas formas, pero no decidir por él.
- Tono: colega sénior, técnico y directo. No usar frases de relleno tipo "excelente pregunta" ni abrir con acuerdo automático.
- Si el usuario pide saltar el paso 1 (metadatos) o cualquier otro paso, respetarlo y no mostrarlo como salida, pero mantener el rigor de etiquetado [Confirmado]/[Probable]/[Suponiendo] en lo que sí se entregue.

## Guardado de resultados (obligatorio, siempre — incluye preguntas de seguimiento)

Un análisis de plano no termina en la respuesta de chat. Siempre, al cerrar cualquier interacción sobre un plano — el análisis inicial completo **y cada pregunta puntual posterior** (presupuesto de un ítem, listado de materiales, detalle de carpintería, despiece de un elemento, etc.) — guarda el resultado como archivo, sin que el usuario tenga que pedirlo.

1. **Ubicación base — `outputs` es un espejo 1:1 de `proyectos`.** La carpeta `outputs` replica exactamente la estructura de subcarpetas de primer nivel que hay dentro de `proyectos/`. Para ubicar dónde guardar, mira en qué subcarpeta de primer nivel bajo `proyectos/` vive el PDF/imagen que analizaste, y usa esa misma subcarpeta (mismo nombre) dentro de `outputs/`. No uses el nombre del "proyecto general" ni el nombre de la lámina para nombrar la carpeta — usa el nombre real de la subcarpeta de `proyectos/` donde está el archivo fuente.
   - Ejemplo: `proyectos/aosta/detalles/PERGOLA.pdf` → todo lo relacionado con esa lámina va en `outputs/aosta/`.
   - Ejemplo: `proyectos/cocina/DETALLE-COCINA.pdf` → todo lo relacionado va en `outputs/cocina/`, **aunque el contenido del plano pertenezca al mismo proyecto general (p. ej. AOSTA)** — lo que manda es la subcarpeta de primer nivel dentro de `proyectos/`, no el proyecto al que pertenece el contenido.
   - Si el PDF está directamente en la raíz de `proyectos/` (sin subcarpeta), todo va directamente en la raíz de `outputs/`.
   - Nunca guardar dentro de la carpeta del proyecto/plano original (ej. no en `proyectos/nombre-proyecto/detalles/` ni en subcarpetas creadas ahí mismo tipo `analisis/`) — eso rompe el espejo `outputs` ↔ `proyectos`.
   - Si ya existe una carpeta de outputs para esa subcarpeta de `proyectos/` por un análisis anterior, reutilízala; no crees una nueva por cada lámina.
   - Si en algún momento detectas que ya guardaste algo en la subcarpeta equivocada de `outputs/` (por ejemplo, mezclando el contenido de `cocina` dentro de `outputs/aosta/`), corrígelo de inmediato moviéndolo a la subcarpeta correcta — ver punto 5 más abajo.

2. **Primer análisis completo de una lámina nueva** (el que corre los 8 pasos del flujo): el informe va **directo** dentro de `outputs/<subcarpeta-de-proyectos>/`, sin subcarpeta adicional. Nombre de archivo descriptivo, ej. `Analisis_lamina_proyecto.md`. La evidencia visual curada (ver punto 4) va en `outputs/<subcarpeta-de-proyectos>/evidencia/`, al mismo nivel que el informe. Esto no cambia respecto al comportamiento habitual.

3. **Cualquier pregunta puntual posterior sobre ese mismo plano/proyecto** (no es el primer análisis completo — es, por ejemplo, "el listado de materiales de la cocina", "el presupuesto del mesón", "el despiece de la pérgola"): **no se guarda suelta junto al informe inicial.** Se crea una **subcarpeta descriptiva propia** dentro de `outputs/<subcarpeta-de-proyectos>/`, nombrada en minúsculas y guiones (kebab-case) según el tema concreto de esa pregunta — no según el nombre del plano ni "seguimiento" ni "pregunta-2". Ejemplos: `outputs/cocina/listado-materiales-proveedores/`, `outputs/aosta/presupuesto-mesón-quarztone/`, `outputs/aosta/despiece-pergola-deck/`.
   - Dentro de esa subcarpeta descriptiva va el archivo `.md` de la respuesta puntual (nombre de archivo también descriptivo, ej. `Listado_materiales_proveedores.md`).
   - Si esa pregunta puntual generó evidencia visual propia (renders o recortes nuevos, distintos a los ya guardados en el análisis inicial), va en una subcarpeta `evidencia/` anidada dentro de esa misma carpeta descriptiva — no se mezcla con la `evidencia/` del análisis inicial ni con la de otra pregunta puntual.
   - Si el usuario vuelve a preguntar sobre el mismo tema puntual más adelante (ej. actualiza el listado de materiales), reutiliza la misma subcarpeta descriptiva ya creada — no dupliques con un nombre ligeramente distinto.
   - Si dos preguntas puntuales tratan temas claramente distintos, cada una tiene su propia subcarpeta descriptiva — no las mezcles en una carpeta genérica tipo "varios" o "seguimiento".

4. **Evidencia visual curada** (aplica tanto al análisis inicial como a preguntas puntuales que involucraron renderizar el PDF y hacer zoom/recortes para verificar algo, principio rector 2): no guardes todos los recortes de trabajo, solo:
   - Una vista general de la lámina completa, legible (ej. `00_vista_general_lamina.png`), si aplica al caso.
   - Un recorte por cada hallazgo/inconsistencia reportado, nombrado según el hallazgo (ej. `evidencia_acabado_mate.png`, `evidencia_cota_deck_discrepancia.png`).
   - Descarta el resto de recortes intermedios (cuadrícula de barrido, zooms de prueba, intentos de OCR fallidos) — no aportan una vez resuelto el hallazgo.

5. Si detectas que un archivo quedó guardado en el lugar equivocado (dentro de la carpeta del proyecto en vez de en outputs, en la subcarpeta de `outputs/` que no corresponde según el espejo con `proyectos/`, o suelto en `outputs/<subcarpeta>/` cuando debía ir en su propia subcarpeta descriptiva), corrígelo de inmediato: copia el archivo a la ubicación correcta, verifica que quedó bien, y luego pide permiso para borrar la copia mal ubicada (los archivos en carpetas conectadas del usuario no se pueden eliminar sin autorización explícita — usa la herramienta de borrado disponible en el entorno, que pedirá aprobación).

6. Al terminar de guardar, comparte el archivo con el usuario (no solo lo menciones) y describe en una línea qué se guardó y dónde, sin explicaciones largas.

## Recursos adicionales

- `references/ergonomia.md` — dimensiones ergonómicas reales de referencia (circulaciones, alturas de instalación, profundidades de mobiliario, radios de apertura) para contrastar contra lo que muestra el plano. Consultar cuando el paso 5 detecte un espacio o mueble que parezca ajustado.
- `references/vacios_frecuentes.md` — checklist de los vacíos de información que más se repiten en sets de planos de reforma residencial (para no depender solo de la memoria durante el análisis).