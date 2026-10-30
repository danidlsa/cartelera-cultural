# Diseño: modelo de datos, pipeline y stack

Propuesta para el MVP de la cartelera. Se apoya en [fuentes.md](fuentes.md) y en datos reales de Tickantel, RedTickets y el portal IM revisados el 30/10/2026.

**Estado: aprobado el 31/10/2026.** El lenguaje es **Python** en todo lo que se pueda, y los flyers se muestran.

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
- **`imagen_url`**: **el flyer se muestra** (decisión). Se enlaza la imagen original de la fuente, sin copiarla al repo; así el repo no crece y el flyer siempre lleva a la fuente. Si una imagen no carga (porque el sitio bloquea que se muestre desde otra web o porque la borraron), la web muestra un placeholder por categoría.
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

## Almacenamiento y retención

**No hay base de datos con servidor. La "base" son archivos JSON en la rama `datos` del repo.**

- **Por qué alcanza**: la cartelera activa son del orden de **1.000 a 2.000 eventos**, o sea unos pocos MB de JSON. La web los lee directamente y filtra en el navegador. No hay usuarios, ni escrituras desde la web, ni consultas complejas: no hace falta un motor de base de datos.
- **Qué hay en `datos`**:
  - `fuentes/<fuente>.json`: el último resultado bueno de cada scraper.
  - `eventos.json`, `lugares.json` y `ciclos.json`: la cartelera unificada.
- **Limpieza automática**: en cada unificación, **se descartan las funciones que ya pasaron** (con un margen de un día) y **los eventos que se quedan sin funciones futuras**. Los archivos siempre tienen solo lo vigente, así que no crecen con el tiempo.
- **El historial es gratis**: cada corrida es un commit en `datos`, así que lo que sale de los archivos sigue en el historial de git. Se puede consultar cualquier día pasado sin guardar un archivo histórico aparte.
- **El tamaño del repo**: git guarda solo las diferencias, comprimidas, entre un commit y el siguiente. Con dos corridas por día, la rama `datos` crecería **del orden de decenas de MB por año**, muy lejos del límite recomendado por GitHub (1 GB). Si algún día pesa demasiado, se puede **reiniciar el historial de `datos`** (los datos se regeneran solos), sin tocar `main`.
- **Cuándo cambiaría esto**: si en el futuro hubiera funciones con usuarios (favoritos, avisos por mail) o estadísticas históricas pesadas, se puede sumar **SQLite** como archivo o la capa gratuita de **Supabase**. Para el MVP no hace falta.
  - Ojo con la capa gratuita de Supabase: **pausa los proyectos sin actividad** después de un tiempo.

---

## Seguridad del job en la Mac

**El riesgo es bajo**, y se puede bajar más con algunas reglas.

### Qué hace el job

Una vez por día, `launchd` ejecuta código **de este repo**, que hace pedidos HTTP **salientes** a 4 sitios públicos, guarda JSON y hace `git push`.

- **No abre puertos ni recibe conexiones.**
- **Nadie puede ejecutar algo en la Mac desde GitHub.** Por eso se eligió `launchd` y no un runner self-hosted.

### Riesgos y cómo se cubren

| Riesgo | Qué podría pasar | Cómo se cubre |
|--------|------------------|---------------|
| **Código malicioso en el repo** | Si alguien lograra meter código en `main`, la Mac lo ejecutaría con tus permisos (acceso a tus archivos). | Solo vos tenés permiso de escritura. **No se mergean pull requests de terceros sin revisarlos.** Se protege `main` con una regla de GitHub para que no se pueda pushear directo. El job corre siempre desde `main`, no desde otras ramas. |
| **Una dependencia comprometida** (un paquete de Python malicioso) | Lo mismo: código ajeno corriendo en tu Mac. | **Pocas dependencias** y conocidas (`httpx`, `selectolax`, `pydantic`). **Versiones fijadas con hash** (`uv.lock`), así que una actualización maliciosa no entra sola. Un entorno virtual aislado. |
| **Contenido hostil en los sitios scrapeados** | Una página armada para atacar al que la lee. | El job **solo lee JSON y HTML como texto**: no ejecuta JavaScript, no usa navegador headless y no hace nada parecido a `eval` o `pickle` con lo que baja. Parsear texto en Python no ejecuta código. Las 4 fuentes de la Mac son APIs JSON o HTML simple. |
| **Credenciales de GitHub demasiado amplias** | Hoy `git push` usa tus credenciales guardadas, que dan acceso a **todos** tus repos. | El job usa una **deploy key** (una llave SSH) **solo para este repo**. Si se filtrara, solo afectaría a `cartelera-cultural`, y la regla sobre `main` impide que se use para cambiar el código. |
| **Que se suba algo de más** | Un log con rutas o datos de tu Mac terminando en el repo público. | El job hace `git add` **solo** de sus 4 archivos en `datos/fuentes/` y nunca de otra cosa. Los logs quedan en la Mac. |
| **Tu IP de casa** | Los 4 sitios ven pedidos diarios desde tu IP. Si alguno la bloqueara, también podría afectarte al navegar ese sitio. | Volumen muy bajo (una vez por día), User-Agent identificable y pausas entre pedidos. Es poco probable. |

**Opcional, si querés más aislamiento**: correr el job con un **usuario de macOS aparte**, sin acceso a tus documentos. No me parece necesario para empezar.

---

## Stack

### Python para scrapers y datos, sitio estático simple (decisión)

| Parte | Herramienta | Por qué |
|-------|-------------|---------|
| HTTP | `httpx` | Moderno, con reintentos y timeouts. |
| HTML | `selectolax` | Rápido y simple. |
| JSON-LD | `extruct` | Lo usan Tickantel, Ticketmaster, Livepass y Fever. |
| Esquema | `pydantic` | Valida que cada fuente cumpla el modelo de datos. |
| Dependencias | `uv` | Instala y fija versiones exactas, con hashes (ver Seguridad). |
| Deduplicación | `rapidfuzz` | Similitud de títulos. |
| Tests | `pytest` | Contra el HTML guardado. |
| Headless (fase 3) | `playwright` | Para Movistar Arena y Primavera Sound. |
| Web | **Jinja2** (genera HTML estático) y un poco de JavaScript para los filtros | Todo el build en Python. Sin servidor, gratis en GitHub Pages. |

**Regla: Python siempre que se pueda.** Es el lenguaje que mejor conocés, y además tiene el mejor ecosistema para scraping y limpieza de datos.

**La web** también se puede armar en Python para no sumar otro lenguaje: un script que genera páginas HTML con plantillas **Jinja2** a partir de `eventos.json`. Solo los filtros del navegador (categoría, fecha) necesitan unas pocas líneas de JavaScript, que es inevitable en una web. Astro queda descartado.

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

Ninguna por ahora.
