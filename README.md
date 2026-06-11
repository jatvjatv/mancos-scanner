# Mancos Scanner

Portal cripto **single-file** (un solo `index.html`) que rankea ~200 tokens del top 250 de CoinGecko con un score 0-100 estilo **Buffett: castigar promesa, premiar prueba on-chain**. Sin build step, sin dependencias, sin backend.

Live data desde CoinGecko (top 250 por mcap) + DefiLlama (fees por protocolo). Refresca cada 60 segundos. Persiste un snapshot en `localStorage` para sobrevivir sin red.

---

## Qué hace

- **Ranking de oportunidades** ordenado por score, con tiers `TOP / SÓLIDO / ESPECULATIVO / EVITAR`.
- **Trampas de valor** detectadas dinámicamente (dilución, captura nula, hype enfriado, timing cerca de ATH). Click en el ticker abre un popup con argumento completo.
- **Reserva** — tarjetas vivas de BTC/ETH como ancla del portafolio.
- **Portafolio recomendado** en tres versiones (Reservado / Balanceado / Agresivo) con pesos calculados.
- **Filtros** por tier y categoría (DeFi, L1, IA, RWA, DePIN, etc.).
- **Búsqueda** externa on-demand vía CoinGecko `/search` para tokens fuera del top 250.

## Filosofía del scoring (resumen)

El modelo evalúa 8 criterios y los pondera 100 puntos en total. Luego aplica penalties multiplicativos por dilución y categoría problemática.

**Calidad (51 pts)** — fundamentos del negocio:
- Oferta sin dilución (≥80% circulante)
- Uso real (fees reales en DefiLlama o turnover ≥5%)
- Crecimiento (Δ fees mensual ≥10% **con piso absoluto $250K** para evitar ruido de bases pequeñas)
- Rentabilidad estructural (annualized fees / mcap ≥5%, continuo)

**Oportunidad (49 pts)** — entrada asimétrica:
- Narrativa correcta (categoría hot Y no en narrativas muertas)
- Descuento vs ATH ≥70%
- Demanda externa (turnover ≥3%)
- Asimetría restante (mcap chico con espacio al ATH)

**Penalties multiplicativos:**
- Dilución: ×0.78 si <50% circ, ×0.88 si <70%, ×0.95 si <85%, ×1.00 si ≥85%
- Categoría: ×0.50 meme, ×0.45 legacy, ×0.60 unknown

**Arquetipos** — cada uno mide captura distinto:
- `defi` (slug en DefiLlama): fees reales y rentabilidad continua
- `l1` (chain): turnover por uso real; sin slug DefiLlama del chain, crit[2]/[3]=0 (techo natural SÓLIDO)
- `infra` (narrativa hot sin slug): igual que l1
- `legacy` (sin narrativa hot, sin slug): no acreditamos crecimiento/rentabilidad

**Tiers:** TOP ≥78 / SÓLIDO ≥60 / ESPECULATIVO ≥45 / EVITAR <45.

## Setup (humano)

### 1. Conseguir API key de CoinGecko (gratis, 1 min)

Ve a https://www.coingecko.com/en/developers/dashboard, crea cuenta, copia tu **Demo API key** (formato `CG-xxxx...`). Plan Demo: 30 calls/min — suficiente para uso personal. Sin key, el portal cae al plan público anónimo (5-15 calls/min) y choca con rate-limit (429) seguido.

### 2. Servir el archivo (no abrir directo con `file://`)

Algunos browsers bloquean `fetch` desde `file://`. Mejor servir por HTTP local. Desde la carpeta del repo:

```bash
# Opción A — Python (ya viene en Mac/Linux y en Windows con Python instalado)
python -m http.server 8000

# Opción B — Node
npx serve .

# Opción C — VS Code
# Instalar extensión "Live Server" y click derecho en index.html → "Open with Live Server"
```

Abre `http://localhost:8000` (o el puerto que indique).

### 3. Configurar la API key una vez

Con el portal abierto, F12 → Console → pega:

```js
localStorage.setItem('cgApiKey', 'CG-TU-KEY-AQUI')
```

Recarga la página. Queda persistente en ese navegador.

## Verificar que funciona

- Header debe decir "**● DATOS EN VIVO**" en verde
- Estado del universo debe mostrar "**● ~150-200 tokens evaluados · fuente: top 250 mcap**"
- Ranking debe mostrar tokens con badges `EN VIVO` (no `REF`)
- BTC/ETH en la sección Reserva con precios actuales

Si ves `REF` en todos los tokens y "Mostrando curaduría (43 tokens)" → la key no está configurada o hay bloqueo de red.

## Estructura del repo

```
index.html      # toda la app (HTML + CSS + JS embebidos, ~3400 líneas)
.gitignore      # excluye .claude/, logs, basura de SO
README.md       # este archivo
AGENTS.md       # instrucciones para agentes IA (Claude, Cursor, etc.)
```

**No hay**: package.json, build step, framework, backend, base de datos. Es deliberadamente un solo archivo.

## Stack

- HTML + CSS (custom, sin framework) + Vanilla JS
- CoinGecko Public API v3 (Demo plan) — markets, global, search
- DefiLlama Pro API (gratuito) — fees por protocolo
- localStorage para cache (universo + reserva + key)

## Licencia

MIT. Úsalo, fórkealo, modifícalo, monetízalo. Sin garantías — no es asesoría financiera.
