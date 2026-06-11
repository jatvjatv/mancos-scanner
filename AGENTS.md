# AGENTS.md

Instrucciones operativas para agentes IA (Claude Code, Cursor, Aider, Codex, etc.) que reciban la consigna **"toma este repo e impleméntalo"**.

Este archivo prioriza acciones concretas sobre prosa. Si el usuario quiere contexto narrativo, mándalo al `README.md`.

---

## Qué es este repo

Una webapp **single-file** (`index.html`, ~3400 líneas, HTML+CSS+JS embebidos). No tiene build step ni dependencias instalables. Tu trabajo como agente al "implementar" es: dejarla corriendo en el browser del usuario con datos en vivo.

## Implementación end-to-end (orden estricto)

### Paso 1 — Verificar el clone
```bash
ls index.html README.md AGENTS.md
```
Si falta `index.html`, el clone se rompió. Reporta y para.

### Paso 2 — Levantar un servidor local

NO abras `index.html` con `file://` — browsers como Chrome y Firefox bloquean `fetch` a APIs externas en ese contexto. Levanta uno de estos (en orden de preferencia, usa el primero que funcione):

```bash
# Preferido: Python (universal)
python -m http.server 8000

# Fallback 1: Node
npx serve . -p 8000

# Fallback 2: PHP (si Python y Node no están)
php -S localhost:8000
```

Lánzalo en background si el harness lo permite. Reporta al usuario el puerto exacto donde sirve.

### Paso 3 — Conseguir la API key de CoinGecko

El portal **necesita** una CoinGecko Demo API key para funcionar bien (sin ella, rate-limit 429 cada pocas cargas).

- **No la inventes ni la pongas hardcoded en el código.** El repo deliberadamente la sacó del source para no exponerla.
- **Pregúntale al usuario** si tiene una. Si no, dale la URL: https://www.coingecko.com/en/developers/dashboard (registro gratis, plan Demo, key formato `CG-xxxx...`).
- **No la guardes en archivos del repo.** Va en `localStorage` del browser del usuario.

### Paso 4 — Configurar la key en el browser

Una vez que tengas la key del usuario, dale estas instrucciones literales:

> 1. Abre http://localhost:8000 (o el puerto que el servidor reportó)
> 2. Presiona F12 → tab Console
> 3. Pega y ejecuta:
>    ```js
>    localStorage.setItem('cgApiKey', 'CG-TU-KEY-AQUI')
>    ```
> 4. Recarga la página

### Paso 5 — Verificar
Pídele al usuario que confirme que ve:
- Header: "● DATOS EN VIVO" en verde
- Estado del universo: "● ~150-200 tokens evaluados"
- Tickers con badge `● EN VIVO` (no `○ REF`)

Si ve `REF` en todos los tokens y banner rojo "Mostrando curaduría (43 tokens)", algo falló: key no configurada, ad-blocker activo, o red sin acceso a `api.coingecko.com`.

---

## Si el usuario pide modificar el scoring

**Lee primero el README** para entender la filosofía Buffett-style. Luego ubica estas zonas críticas en `index.html`:

- `THRESHOLDS` constant — umbrales globales (rentableAnualPct, descuentoAthPct, etc.)
- `STORAGE_CHAINS`, `COMPUTE_NETWORKS`, `DEAD_NARRATIVES`, `HOT_NARRATIVES` — listas nombradas
- `TOKEN_CATEGORIES` — whitelist manual ticker → categoría local
- `CURATED_TOKENS` — overlay editorial (análisis, warnings, tierOverride)
- `CG_IDS`, `LLAMA_IDS` — mapeo ticker → ID de CoinGecko y slug de DefiLlama
- `detectArchetype(t, llama)` — clasifica defi / l1 / infra / legacy
- `computeCritFromData(t, cg, llama)` — calcula los 8 criterios
- `computeScore(t)` — score final con penalties
- `tierOf(t)` — umbrales TOP/SÓLIDO/ESPECULATIVO/EVITAR

**Cosas que NO debes hacer sin discutir con el usuario:**
- Bajar umbrales de score para "llenar" el portafolio recomendado (el slider muestra "Pool agotado" intencionalmente — es señal de honestidad)
- Premiar narrativa o "captación de capital" por encima de captura demostrable (el framework rechaza explícitamente momentum-chasing)
- Quitar el piso absoluto de fees ($250K) en `crit[2]` del archetype defi
- Acreditar `crit[2]` o `crit[3]` para L1s sin slug DefiLlama del chain (es regla de diseño)

## Si el usuario pide agregar un token

1. Buscar su id de CoinGecko (api.coingecko.com/api/v3/search?query=X)
2. Buscar su slug en DefiLlama si aplica (defillama.com/protocols)
3. Agregarlo a `CG_IDS` y `LLAMA_IDS`
4. Si tiene categoría no estándar, agregarlo a `TOKEN_CATEGORIES`
5. Opcionalmente, agregar overlay editorial en `CURATED_TOKENS` con análisis y warnings
6. Si es storage chain → agregar a `STORAGE_CHAINS`
7. Si es compute network → agregar a `COMPUTE_NETWORKS`

## Si el usuario pide deployar a la web

El proyecto es estático (un solo HTML). Cualquiera de estos sirve:

- **GitHub Pages**: en el repo, Settings → Pages → Source: `main` branch, `/` root. Queda en `https://<usuario>.github.io/<repo>/`.
- **Cloudflare Pages**: conectar el repo, sin build command, output directory `/`.
- **Vercel/Netlify**: deploy estático, sin configuración.

⚠ Si lo deployás a un dominio público, **el usuario necesita su propia API key** en `localStorage` de cada navegador que use. NO embebas una key en el código antes de deployar — es deliberado que viva en localStorage.

## Convenciones del proyecto

- **Idioma**: español neutro (no rioplatense, no voseo). Comentarios y UI en español.
- **Comentarios en código**: solo cuando el "por qué" no es obvio. Suelen explicar bugs históricos cerrados o decisiones de diseño no triviales.
- **Single-file deliberado**: no proponer modularización ni build step. Cualquier "mejora" que requiera bundler o framework va contra el espíritu del proyecto.
- **No mocks ni placeholders**: si una fuente de datos falla, fallback honesto (cache + indicador visual), no datos inventados.

## Glosario rápido

- **Archetype**: clasificación interna de cómo medir captura de valor (`defi` / `l1` / `infra` / `legacy`). Determina qué criterios cuentan y cómo.
- **Universo**: el set de ~200 tokens evaluados (top 250 mcap menos excluidos como stablecoins y wrappers).
- **Reserva**: BTC y ETH, viven aparte del ranking — son el ancla del portafolio, no compiten por asimetría.
- **Trampa de valor**: token que parece barato (descuento ATH alto) pero tiene falla estructural que lo condena (dilución, captura nula, hype enfriado, timing).
- **Tier**: TOP ≥78, SÓLIDO ≥60, ESPECULATIVO ≥45, EVITAR <45.
- **Oportunidad real**: multiplicador realista al ATH ajustado por tier y haircut.
