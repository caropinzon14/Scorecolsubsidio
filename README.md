# Perfilador de Clientes — Colsubsidio

App de una sola página (`index.html`, vanilla JS, sin build ni dependencias más allá de una fuente de Google Fonts) que expone el motor de reglas de Seggu para segmentar clientes de seguros de Colsubsidio. Pensada para uso interno del analista de datos.

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
- **Datos del motor** (dentro de `<script>`, primeras ~200 líneas):
  - `PRODUCTS` — los 11 productos, con su línea (Familia/Patrimonio).
  - `MAXS` — puntaje máximo teórico por producto (paralelo a `PRODUCTS`).
  - `VARIABLES` — las 11 variables de segmentación (V1–V11) con sus categorías posibles.
  - `WEIGHTS` — objeto `"Vx|categoría" → [11 pesos, uno por producto]`. Esta es la Matriz de Pesos del Excel, tal cual.
  - `RATIONALE` — mismo keying que `WEIGHTS`, texto del racional/fuente resumida.
  - `CHECKLIST` — datos requeridos para cotizar por producto (del PDF). Algunos productos (`accidentes`, `renta`, `cancer`, `arrendamiento`) no tienen checklist documentado — se marca explícitamente en vez de inventarse.
  - `SOURCES` — tabla de fuentes (Fasecolda, DANE, INC, etc.) para la pestaña de referencia.
- **Funciones núcleo**:
  - `computeScores(profile)` — dado un perfil `{V1: categoría, ..., V11: categoría}`, suma los pesos por producto y devuelve ranking + desglose. Ordena por score crudo descendente (así rankea el Excel original, no por %).
  - `reorderForExplicit(results, key)` — si el cliente ya pidió un producto puntual, lo sube al puesto 1 (regla de negocio "entregar + sugerir").
  - `computeInfluence()` — para la pestaña 1: calcula cuánto cambia el score de cada producto según la categoría (rango máx–mín entre categorías de una variable, sumado en los 11 productos). Es la base de "variables más influyentes" y de la priorización para clientes no afiliados.
- **Tres pestañas** (`.tabpanel`, mostradas/ocultadas por JS, sin router):
  1. **Explorador de variables** (`#tab-individual`) — sin formulario. Matriz de reglas en acordeón (variable → categorías × productos, mapa de calor), lista de productos clickeable para filtrar columnas, lista de variables por influencia, toggle afiliado/no afiliado que cambia solo el texto de contexto y las etiquetas de prioridad (no los pesos).
  2. **Carga masiva** (`#tab-batch`) — sube CSV, parser propio (`parseCSV`, soporta comillas), normaliza texto libre a categorías (`matchCategory`/`matchProduct`, tolerante a tildes/mayúsculas), tabla filtrable/ordenable, exporta resultados a CSV.
  3. **Referencia y fuentes** (`#tab-ref`) — metodología resumida, tabla de fuentes, checklist de cotización por producto en `<details>`.

## Decisiones de diseño a respetar si sigues iterando

- **No es un formulario de captura.** La pestaña 1 se rediseñó a propósito para explorar el motor de reglas, no para "calcular un perfil" con inputs. Si agregas algo ahí, que siga siendo exploratorio (filtros, resaltados), no un submit.
- **El campo "afiliado" no tiene peso propio en la matriz.** Es una etiqueta/filtro. La guía original del Excel señala que probablemente el peso de "Formal dependiente" deba subir una vez haya datos reales de afiliados — está documentado en `RATIONALE` y en las notas de la app, no lo cambies sin ese dato real.
- **El ranking se ordena por score crudo, no por %**, para calzar con el simulador del Excel (ver `computeScores`). Si algún día se normaliza distinto, hay que decidirlo con el equipo, no cambiarlo silenciosamente.
- **Los 4 productos sin checklist documentado** (`accidentes`, `renta`, `cancer`, `arrendamiento`) deben seguir marcados como "sin datos documentados" hasta que llegue el checklist real — no rellenar con supuestos.

## Ideas abiertas / próximos pasos razonables

- Conectar `WEIGHTS` a Supabase en vez de tenerlo hardcodeado, para poder recalibrar sin tocar código (ver preguntas abiertas de la Guía y Metodología del Excel original sobre datos reales de afiliados).
- Persistir resultados de carga masiva (hoy vive solo en memoria del navegador, se pierde al recargar).
- Sumar el matiz "afiliado" como variable real de la matriz de pesos cuando haya datos de conversión histórica para calibrarlo.
- Exportar la ficha de cotización (checklist) junto con el CSV de resultados de carga masiva.
- Integrar con Hola Seggu / n8n para que el perfil resultante alimente el flujo de conversación con Caro.
