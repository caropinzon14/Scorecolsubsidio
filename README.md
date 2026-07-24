# Perfilador de Clientes — Colsubsidio

App de una sola página (`index.html`, vanilla JS, sin build ni dependencias más allá de una fuente de Google Fonts) que expone el motor de reglas de Seggu para segmentar clientes de seguros de Colsubsidio, perfilarlos como afiliados o no afiliados y enrutar la venta a cierre automatizado sin intermediario o a asesoría personalizada con intermediario, según la categoría de póliza recomendada.

Fuentes originales del modelo: `Motor_Scoring_Seguros_Colsubsidio.xlsx` (matriz de pesos) y `datos-cotizacion-colsubsidio_md.pdf` (checklist de datos por producto). No están incluidos en este paquete — si vas a recalibrar pesos o agregar productos, pide esos archivos de nuevo o mantén `index.html` como fuente de verdad mientras tanto.

## Cómo correrlo

No requiere servidor: abre `index.html` directamente en el navegador. Si prefieres sportear un live-reload durante el desarrollo:

```bash
npx serve .
# o
python3 -m http.server 8000
```

## Arquitectura (todo vive en `index.html`)

- **`<style>`** — tokens de diseño en `:root` (colores, tipografías). Paleta: `--ink` navy, `--accent` naranja quemado (línea Familia / marca Colsubsidio), `--teal` (línea Patrimonio). Tipografías: Fraunces (display), IBM Plex Sans (cuerpo), IBM Plex Mono (datos/scores).
- **Datos del motor** (dentro de `<script>`, primeras ~350 líneas):
  - `PRODUCTS` — los 12 productos, con su línea (Familia/Patrimonio). `educacion` es el único que no viene del Excel original — se agregó para cubrir la categoría "con intermediario" pedida por negocio, con pesos provisionales (ver más abajo).
  - `MAXS` — puntaje máximo teórico por producto (paralelo a `PRODUCTS`).
  - `VARIABLES` — las 11 variables de segmentación (V1–V11) con sus categorías posibles.
  - `WEIGHTS` — objeto `"Vx|categoría" → [12 pesos, uno por producto]`. Las primeras 11 columnas son la Matriz de Pesos del Excel tal cual; la 12ª (`educacion`) es una adición razonada por analogía, no calibrada.
  - `RATIONALE` — mismo keying que `WEIGHTS`, texto del racional/fuente resumida.
  - `CHECKLIST` — datos requeridos para cotizar por producto, tomados de `datos-cotizacion-colsubsidio_md.pdf`, y el campo `modo` (`'Sin intermediario'` / `'Con intermediario'`) que define el modo de cierre comercial de cada producto — única fuente de verdad, ver `modoCierre()`. `accidentes`, `renta`, `cancer` y `arrendamiento` no aparecen en ese PDF, así que siguen con `items:null`; su `modo` sí es una asunción de proceso comercial declarada como tal.
  - `SOURCES` — tabla de fuentes (Fasecolda, DANE, INC, etc.) para la pestaña de referencia.
- **Funciones núcleo**:
  - `computeScores(profile)` — dado un perfil `{V1: categoría, ..., V11: categoría}`, suma los pesos por producto y devuelve ranking + desglose. Ordena por score crudo descendente (así rankea el Excel original, no por %).
  - `reorderForExplicit(results, key)` — si el cliente ya pidió un producto puntual, lo sube al puesto 1 (regla de negocio "entregar + sugerir").
  - `computeInfluence()` — calcula cuánto cambia el score de cada producto según la categoría (rango máx–mín entre categorías de una variable, sumado en los 12 productos). Es la base de "variables más influyentes", de la priorización para clientes no afiliados y del formulario corto de la pestaña 1.
  - `modoCierre(productKey)` — deriva `'auto'` o `'asesoria'` desde `CHECKLIST[key].modo`. Es el enrutador entre cierre automatizado y asesoría personalizada; no toca los pesos ni el ranking.
  - `topRulesForProduct(productKey, n)` — lee la matriz "al revés": para un producto dado, para cada una de las 11 variables toma la categoría de mayor peso y devuelve las `n` variables que más lo mueven. Es el motor de reglas leído por producto en vez de por variable (que es como ya se lee en el Explorador); alimenta el visor interactivo de la pestaña 4.
- **Cuatro pestañas** (`.tabpanel`, mostradas/ocultadas por JS, sin router):
  1. **Perfilar y vender** (`#tab-sell`) — formulario adaptativo afiliado/no afiliado (para afiliados marca qué datos "ya están en Colsubsidio"; para no afiliados prioriza las 5-6 preguntas de mayor influencia). Calcula el ranking en vivo y muestra el producto top con su badge de modo de cierre; genera una ficha de texto copiable/descargable (cierre automatizado o resumen para asesor, según el caso).
  2. **Explorador de variables** (`#tab-individual`) — sin formulario. Matriz de reglas en acordeón (variable → categorías × productos, mapa de calor), lista de productos clickeable para filtrar columnas (con badge de modo de cierre), lista de variables por influencia, toggle afiliado/no afiliado que cambia solo el texto de contexto y las etiquetas de prioridad (no los pesos).
  3. **Referencia y fuentes** (`#tab-ref`) — metodología resumida, tabla de fuentes, checklist de cotización por producto en `<details>`.
  4. **Recorrido de compra** (`#tab-journey`) — customer journey map completo, desde "no sabe qué seguro necesita" hasta cliente fiel. Arriba, un `.phase-map` estático de 5 fases (Conciencia/Awareness, Consideración/Consideration, Decisión/Decision, Compra/Purchase, Postventa/Post-purchase) con qué hace el cliente, touchpoints, fricción típica y oportunidad por fase. Debajo, el **simulador de casos hipotéticos**: 4 casos de ejemplo (`CASE_PRESETS`, incluye una madre soltera con 2 hijos que trabaja en un banco con 1 SMMLV) o un formulario de las 11 variables para armar un caso propio. Al final, el stepper vertical de 7 etapas (agrupadas bajo las 5 fases con `.phase-divider`) con notas paralelas afiliado/no afiliado donde el camino difiere. La etapa 3 embebe un selector de los 12 productos que muestra, para el elegido, sus variables de mayor peso (`topRulesForProduct`); la etapa 7 (Postventa) muestra la oportunidad de venta cruzada. Al simular un caso, cada etapa se anota con una franja "Para este caso" (`setCaseOverlay`) con el resultado real de `computeScores`/`modoCierre`, y las etapas 5 y 6 atenúan (`.outcome-dim`) la salida que no aplica y marcan la que sí con la etiqueta "Este caso".

## Decisiones de diseño a respetar si sigues iterando

- **La pestaña 2 (Explorador) no es un formulario de captura.** Se diseñó a propósito para explorar el motor de reglas, no para "calcular un perfil" con inputs — eso ya vive en la pestaña 1 (Perfilar y vender). Si agregas algo al Explorador, que siga siendo exploratorio (filtros, resaltados), no un submit.
- **El campo "afiliado" no tiene peso propio en la matriz.** Es una etiqueta/filtro que además decide qué preguntas mostrar en el formulario de la pestaña 1. La guía original del Excel señala que probablemente el peso de "Formal dependiente" deba subir una vez haya datos reales de afiliados — está documentado en `RATIONALE` y en las notas de la app, no lo cambies sin ese dato real.
- **El ranking se ordena por score crudo, no por %**, para calzar con el simulador del Excel (ver `computeScores`). Si algún día se normaliza distinto, hay que decidirlo con el equipo, no cambiarlo silenciosamente.
- **El modo de cierre (`CHECKLIST.modo`) es una capa de proceso comercial, no de scoring.** Cambiarlo no debe tocar `WEIGHTS` ni el ranking — solo decide si la pestaña 1 muestra el flujo de cierre automatizado o el de asesoría.
- **Los productos sin checklist documentado** (`accidentes`, `renta`, `cancer`, `arrendamiento`) deben seguir marcados como "sin datos documentados" hasta que llegue el checklist real — no rellenar con supuestos. Su `modo` de cierre sí es una asunción declarada (ver comentario junto a `CHECKLIST` en el código), a validar con Colsubsidio.
- **Los pesos de `educacion` no vienen del Excel original**, aunque su checklist ya sí (viene de `datos-cotizacion-colsubsidio_md.pdf`, que confirma que sí es un producto real del portafolio). Sus pesos siguen siendo un punto de partida razonado por analogía con vida/salud, no un dato calibrado — no los trates como si tuvieran el mismo respaldo que el resto de la matriz.
- **`datos-cotizacion-colsubsidio_md.pdf` también lista "Viajes"** (sin intermediario: cédula, fecha de nacimiento, fechas de salida/regreso, país de destino) como producto del portafolio. No está en `PRODUCTS` — agregarlo requeriría estimar sus 11 pesos sin respaldo del Excel, igual que se hizo con Educación. No agregarlo sin decidirlo con el equipo primero.
- **El simulador de casos es aditivo, no reemplaza la explicación genérica.** Las franjas "Para este caso" se agregan sobre el contenido afiliado/no afiliado que ya explica la regla; no lo ocultan. Si agregas más casos a `CASE_PRESETS`, son perfiles ilustrativos para mostrar cómo resuelve el motor, no datos de clientes reales.
- **El `.phase-map` de las 5 fases es contenido estático**, no depende del caso simulado — describe el journey en general (touchpoints, fricción, oportunidad), no el resultado de un perfil concreto. Si el simulador se extiende, que siga siendo el stepper de abajo el que muestre el estado por caso, no el mapa de fases.

## Ideas abiertas / próximos pasos razonables

- Conectar `WEIGHTS` a Supabase en vez de tenerlo hardcodeado, para poder recalibrar sin tocar código (ver preguntas abiertas de la Guía y Metodología del Excel original sobre datos reales de afiliados).
- Sumar el matiz "afiliado" como variable real de la matriz de pesos cuando haya datos de conversión histórica para calibrarlo.
- Recalibrar los pesos de `educacion` con datos reales (el checklist ya está confirmado, faltan los pesos).
- Decidir con el equipo si se agrega Viajes como producto 13, y con qué pesos de partida.
- Integrar con Hola Seggu / n8n para automatizar el paso de la ficha de cierre (o el resumen para asesor) generada en la pestaña 1 a cotización/pago en línea o a agendamiento de la asesoría.
- Persistir el perfil y la ficha generada (hoy vive solo en memoria del navegador, se pierde al recargar).
