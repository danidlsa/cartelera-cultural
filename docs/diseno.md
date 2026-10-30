# Diseño: modelo de datos, pipeline y stack

Propuesta para el MVP de la cartelera. Se apoya en [fuentes.md](fuentes.md) y en datos reales de Tickantel, RedTickets y el portal IM revisados el 30/10/2026.

**Estado: propuesta para validar.** Hay una pregunta abierta sobre el stack al final.

---

## Principios

Vienen de las decisiones de [fuentes.md](fuentes.md#decisiones-tomadas):

- **Todo por scraping**, sin carga manual de eventos.
- **Público local**, **solo Montevideo** (más Buenos Aires en la fase 3) y **sin costo**: planes gratuitos, sin APIs pagas.
- **Una fuente rota no rompe la cartelera**: cada fuente conserva su último resultado bueno.
- **Siempre se enlaza a la fuente** para comprar o ver más. No se republican textos largos.

---

## Lo que muestran los datos reales

Esto cambia algunas cosas respecto a lo que se veía en las portadas:

| Fuente | Qué trae el detalle de cada evento | Consecuencia |
|--------|------------------------------------|--------------|
| **Tickantel** | Un `Event` en JSON-LD **por función**, con `startDate` y zona horaria, precio, coordenadas e imagen. El payload trae además cada función con `availabilityStatus` (disponible o agotado) y un texto de horario ("Viernes y Sábado a las 17:30h"). | No hace falta interpretar fechas en texto. |
| **RedTickets** | Bloque `Dates` → `Times` → `Tickets`, con fecha ISO, hora, `soldOut`, precio con y sin cargo, `GeoPoint`, dirección y **productor**. | Lo mismo. La fecha en texto ("Viernes 30 de Octubre") solo aparece en la portada; el detalle la trae estructurada. |
| **Portal IM** | Hay **eventos "paraguas"** (por ejemplo "Música en el Parque", de octubre a noviembre) y **actividades** que dependen de ellos. Las funciones vienen en un párrafo `funciones`, con fecha, fin, zona horaria y, a veces, una regla de recurrencia (`rrule`). | El modelo necesita la idea de **ciclo**. Lo recurrente (con `rrule` o sin fecha) se descarta, según lo decidido. |

Todas las fuentes del MVP traen **coordenadas** o una dirección. Eso permite reconocer la sala por ubicación y no solo por nombre.

---

## Modelo de datos

Cuatro entidades. Los nombres de campo van en español, como el dominio.

### `Lugar`

Catálogo de salas y espacios: `data/lugares.json`.

```json
{
  "id": "teatro-el-galpon",
  "nombre": "Teatro El Galpón",
  "alias": ["Teatro Galpón", "El Galpón", "Galpón Sala Cero"],
  "direccion": "Av. 18 de Julio 1618",
  "barrio": "Cordón",
  "lat": -34.90403,
  "lon": -56.181082,
  "ciudad": "Montevideo",
  "web": "https://www.teatroelgalpon.org.uy"
}
```

- **Se genera automáticamente** a partir de lo que traen las fuentes: los `node--lugar` del portal IM (con coordenadas) y las direcciones de RedTickets y Tickantel.
- **La lista de alias es la única parte mantenida a mano.** Es configuración, no carga de eventos: un archivo chico donde se anota que "Auditorio Adela Reta" y "Auditorio Nacional del Sodre" son el mismo lugar.
- Las **salas internas** ("Sala Cero", "Sala Zavala Muniz") se guardan como `sala` dentro de la función, no como lugares distintos.

### `Evento`

Lo que se muestra en la cartelera: `data/eventos.json`.

```json
{
  "id": "ev_7f3a9c",
  "titulo": "La rebelión de la carne",
  "categoria": "teatro",
  "descripcion_corta": "Unipersonal interdisciplinario entre la danza, el teatro y la performance autobiográfica.",
  "lugar_id": "teatro-el-galpon",
  "ciclo_id": null,
  "funciones": [
    { "inicio": "2026-11-14T20:30:00-03:00", "sala": "Sala Cero", "agotada": false },
    { "inicio": "2026-11-15T20:30:00-03:00", "sala": "Sala Cero", "agotada": false }
  ],
  "precio": { "desde": 650, "hasta": 650, "moneda": "UYU", "gratis": false },
  "entradas": [
    { "url": "https://redtickets.uy/evento/LA-REBELION-DE-LA-CARNE-Sala-Cero/34406/", "via": "redtickets" }
  ],
  "beneficios": [
    { "programa": "la diaria", "tipo": "2x1", "url": "https://beneficios.ladiaria.com.uy/la-rebelion-de-la-carne/" }
  ],
  "imagen_url": "https://files.redtickets.uy/imagenes/....jpg",
  "productor": "Teatro El Galpón",
  "region": "montevideo",
  "fuentes": [
    { "fuente": "redtickets", "id_externo": "34406", "url": "https://redtickets.uy/evento/.../34406/", "visto": "2026-10-31T06:00:00-03:00" },
    { "fuente": "ladiaria", "id_externo": "la-rebelion-de-la-carne", "url": "https://beneficios.ladiaria.com.uy/la-rebelion-de-la-carne/", "visto": "2026-10-31T06:00:00-03:00" }
  ],
  "actualizado": "2026-10-31T06:05:00-03:00"
}
```

- **`categoria`** es una lista cerrada: `musica`, `teatro`, `danza`, `humor`, `cine`, `exposicion`, `infantil`, `fiesta`, `festival`, `tour`, `charla`, `otros`. Cada fuente traduce sus categorías a esta lista.
- **`funciones`** es la unidad de tiempo: un evento con 10 funciones es **un** evento, no diez. La cartelera "hoy" muestra los eventos que tienen alguna función hoy.
- **`descripcion_corta`**: como máximo unos 300 caracteres, para no republicar textos ajenos. El resto se lee en la fuente.
- **`imagen_url`** queda guardada, pero **falta decidir si se muestra** (son flyers promocionales de terceros).
- **`region`**: `montevideo` o `buenos_aires`. Los eventos del interior se descartan al scrapear.

### `Ciclo` (opcional)

Para festivales, temporadas y ciclos: "Música en el Parque", la temporada de la Comedia Nacional, Primavera 0, el Día del Patrimonio. Tiene `id`, `titulo`, `desde`, `hasta` y `url`, y los eventos lo referencian con `ciclo_id`. Sirve para agrupar en la web ("parte de…").

### Datos crudos por fuente

Cada scraper escribe `datos/fuentes/<fuente>.json` con los eventos ya normalizados a este mismo esquema y una cabecera:

```json
{ "fuente": "redtickets", "corrida": "2026-10-31T06:00:00-03:00", "ok": true, "cantidad": 168, "eventos": [ ... ] }
```

**Si una corrida falla** o trae sospechosamente pocos eventos (por ejemplo, menos de la mitad que la anterior), **no se sobrescribe el archivo**: queda el último resultado bueno y se registra el error.

---

## Deduplicación

Un mismo evento puede llegar por varias fuentes. El paso de unificación las cruza en este orden:

1. **Coincidencia exacta por ID externo.** Muchas fuentes enlazan a la ticketera: la diaria, Piano Piano, la Sala del Museo y Gaucho enlazan a RedTickets; el Antel Arena enlaza a Tickantel. Si dos registros apuntan al mismo evento de una ticketera, son el mismo.
2. **Coincidencia difusa.** Mismo `lugar_id`, alguna función el mismo día (con ±1 hora de tolerancia) y títulos parecidos. La similitud se calcula sin tildes, en minúsculas y sin ruido del tipo "presenta:" o "- Sala Cero".
3. **Fusión.** Se unen las funciones, se juntan `entradas`, `beneficios` y `fuentes`, y cada campo se toma de la fuente de mayor prioridad:
   - **sala oficial > ticketera > cartelera agregada > productora > beneficios**.

     Por ejemplo, el título y la descripción salen del sitio de la sala si existe, y el precio y si está agotado, de la ticketera.

**El `id` del evento** se deriva de la primera coincidencia estable (ticketera e ID externo). Así no cambia de una corrida a otra y los enlaces a la web no se rompen.

---

## Pipeline

```
 GitHub Actions (cron)              Mac en Uruguay (launchd, días de semana 11 h)
 ─ 15 fuentes ─────────┐            ┌─ 4 fuentes: portal IM, Zitarrosa, Ticketmaster.uy, All Access
                       ▼            ▼
              datos/fuentes/<fuente>.json   (crudo normalizado, último bueno)
                       │
                       ▼  unificar: lugares + dedupe + fusión
              datos/eventos.json · datos/lugares.json · datos/ciclos.json
                       │
                       ▼  build
              sitio estático en GitHub Pages
```

- **Los datos generados van en una rama aparte, `datos`.** Así `main` tiene solo código y su historial no se llena con un commit de datos por día. Además, el historial de `datos` deja ver cómo cambió la cartelera.
- **GitHub Actions** corre dos veces por día (por ejemplo, a las 6 y a las 14 h de Uruguay): sus scrapers, la unificación y el build y deploy de la web.
- **La Mac** corre sus 4 scrapers, hace commit **solo de sus archivos** en `datos/fuentes/` y pushea a la rama `datos`. Ese push dispara la unificación y el build en Actions. Como cada lado escribe archivos distintos, no hay conflictos; igual se hace `git pull --rebase` antes de pushear.

### Estructura del repo

```
cartelera-cultural/
├── docs/                    # fuentes.md, diseno.md
├── scrapers/
│   ├── comun/               # HTTP con User-Agent propio, reintentos, parseo de JSON-LD, fechas, normalización
│   ├── tickantel.py
│   ├── redtickets.py
│   └── ...                  # un archivo por fuente
├── unificar/                # catálogo de lugares, dedupe, fusión
├── config/
│   ├── fuentes.yaml         # qué fuentes hay, dónde corren (actions o local), prioridad
│   └── lugares_alias.yaml   # la única parte mantenida a mano
├── sitio/                   # la web estática
├── scripts/                 # check-fuentes.sh, instalación del job de launchd
├── tests/                   # tests de cada parser con HTML guardado
└── .github/workflows/
```

**Tests con HTML guardado:** cada scraper tiene un ejemplo real de la página guardado en `tests/`. Si un sitio cambia y el parser se rompe, el test nuevo falla con el HTML nuevo y se arregla rápido. Es la mejor defensa contra el mantenimiento.

---

## Stack

### Propuesta: Python para scrapers y datos, sitio estático simple

| Parte | Herramienta | Por qué |
|-------|-------------|---------|
| HTTP | `httpx` | Moderno, con reintentos y timeouts. |
| HTML | `selectolax` | Rápido y simple. |
| JSON-LD | `extruct` | Lo usan Tickantel, Ticketmaster, Livepass y Fever. |
| Esquema | `pydantic` | Valida que cada fuente cumpla el modelo de datos. |
| Deduplicación | `rapidfuzz` | Similitud de títulos. |
| Tests | `pytest` | Contra el HTML guardado. |
| Headless (fase 3) | `playwright` | Para Movistar Arena y Primavera Sound. |
| Web | **Astro**, o HTML con JavaScript simple | Lee `eventos.json` y filtra en el navegador. Sin servidor, gratis en GitHub Pages. |

**Por qué Python:** tiene el mejor ecosistema para scraping y limpieza de datos.

**La alternativa** es TypeScript de punta a punta (`fetch`, `cheerio`, `zod`, Playwright y Astro), con un solo lenguaje también en la web. Técnicamente las dos funcionan igual de bien. **Conviene elegir el lenguaje que te resulte más cómodo para mantenerlo.**

---

## Primera iteración del MVP

1. Esqueleto del repo, `scrapers/comun` y el modelo con `pydantic`.
2. **Scrapers de Tickantel y RedTickets**: juntas cubren la mayoría de las salas. Incluyen sus tests con HTML guardado y el filtro de Montevideo.
3. **Unificación mínima**: catálogo de lugares, deduplicación entre esas dos y `eventos.json`.
4. **Workflow de GitHub Actions** con cron, escribiendo en la rama `datos`.
5. **Página mínima** con "hoy", "este fin de semana" y filtro por categoría, para ver los datos de verdad.

Después se suman de a una las demás fuentes del MVP (portal IM con el job de la Mac, Cartelera de Montevideo Portal, Sodre, Zitarrosa, Solís, Teatro de Verano, Galpón y Cinemateca).

---

## Preguntas abiertas

1. **Stack**: ¿Python (propuesta) o TypeScript? Depende sobre todo de cuál te resulte más fácil de leer y mantener.
2. **Imágenes**: ¿mostramos el flyer de cada evento (enlazado a la fuente) o solo texto? Se puede decidir más adelante.
