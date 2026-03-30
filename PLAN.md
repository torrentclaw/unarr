# unarr — Plan de Desarrollo

**Fecha:** 14 Feb 2026
**Lenguaje:** Go
**Proyecto:** ~/Proyectos/torrentclaw/unarr
**Módulo Go:** `github.com/torrentclaw/unarr`
**Depende de:** `github.com/torrentclaw/go-client`
**Binario:** `unarr` (alias: `un`)
**Licencia:** MIT
**Organización GitHub:** torrentclaw

---

## Visión

No es un wrapper de API. Es un **asistente local de torrents** que:
- Conoce tu biblioteca local de medios
- Habla con tu cliente de torrents (qBittorrent, Transmission, Deluge)
- Analiza archivos descargados con mediainfo/ffprobe
- Usa unarr como motor de búsqueda inteligente
- Toma decisiones por ti (mejor seed, mejor calidad, upgrades)

---

## Comandos

### Búsqueda y Descubrimiento

#### `unarr search <query>`
Búsqueda con filtros avanzados.

```bash
unarr search "breaking bad" --type show --quality 1080p --lang es --sort seeders
```

**Filtros:**
- `--type movie|show`
- `--quality 480p|720p|1080p|2160p`
- `--lang es|en|fr|...` (idioma del audio)
- `--genre Action|Comedy|Drama|...`
- `--year-min 2020 --year-max 2026`
- `--min-rating 7`
- `--sort relevance|seeders|year|rating|added`
- `--limit 10 --page 2`

**Output:**
- Default: tabla formateada con colores
- `--json`: raw JSON para scripting/piping
- `--compact`: tabla compacta
- `--no-color`: sin colores

#### `unarr popular [--limit N]`
Contenido popular/trending.

#### `unarr recent [--limit N]`
Últimas adiciones al catálogo.

#### `unarr stats`
Estadísticas del sistema unarr.

---

### Análisis de Torrents (Superpoderes Locales)

#### `unarr inspect <.torrent | magnet | hash>`
TrueSpec: analiza un torrent y muestra las specs REALES.

```
$ unarr inspect magnet:?xt=urn:btih:ABC123...

  📋 Oppenheimer (2023)
  ─────────────────────
  Quality:    1080p BluRay
  Codec:      x265 (HEVC) / AAC 5.1
  Size:       4.2 GB
  Seeds:      1,245 | Leechers: 89
  Languages:  Spanish, English
  Source:     YTS
  Score:      9.2/10 (unarr Quality Score)
  Health:     🟢 Healthy (ratio 14:1)
  Files:      1 video (mkv), 2 subs (srt)
```

**Fuentes de datos:**
1. `parse-torrent` → estructura del .torrent (files, trackers, piece length)
2. `parse-torrent-title` → extraer calidad/codec/idioma del nombre
3. unarr API → quality score, metadata TMDB, seeds/leechers
4. Si el archivo ya existe localmente → `mediainfo` para specs reales del video

#### `unarr upgrade <.torrent | magnet | hash>`
Busca una versión MEJOR del mismo contenido (mejor calidad, más resolución).

```
$ unarr upgrade magnet:?xt=urn:btih:ABC123...

  📊 Upgrade disponible para "Oppenheimer (2023)"
  
  ACTUAL:     720p  x264  2.1 GB   342 seeds
  MEJOR:      1080p x265  4.2 GB  1,245 seeds  ⬆️ +903 seeds, +quality
  PREMIUM:    2160p HDR   16 GB     89 seeds   ⬆️ +quality, -seeds
  
  ¿Descargar upgrade? [1/2/n]:
```

**Lógica:**
1. Parsear el torrent para identificar el contenido (título, año)
2. Buscar en unarr ese contenido
3. Filtrar torrents con MEJOR calidad que el actual
4. Ordenar por quality score
5. Presentar opciones y permitir acción directa

#### `unarr moreseed <.torrent | magnet | hash>`
Misma calidad, mismo idioma, pero con más seeders.

```
$ unarr moreseed magnet:?xt=urn:btih:ABC123...

  🔄 Alternativas con más seeds para "Breaking Bad S01E01 1080p ES"
  
  ACTUAL:        12 seeds (fuente: EliteTorrent)
  ALTERNATIVA 1:  1,890 seeds (fuente: 1337x)  ← same quality, same lang
  ALTERNATIVA 2:    456 seeds (fuente: YTS)     ← same quality, same lang
  
  ¿Reemplazar en qBittorrent? [1/2/n]:
```

**Lógica:**
1. Parsear torrent → identificar contenido + calidad + idioma
2. Buscar en unarr con mismos filtros (calidad, idioma)
3. Filtrar torrents con MÁS seeds que el actual
4. Excluir el torrent actual (por infoHash)
5. Ofrecer reemplazo directo en el cliente

#### `unarr compare <hash1|magnet1> <hash2|magnet2>`
Compara dos torrents del mismo contenido lado a lado.

```
$ unarr compare abc123 def456

  ┌──────────────┬─────────────────┬─────────────────┐
  │              │ Torrent A       │ Torrent B       │
  ├──────────────┼─────────────────┼─────────────────┤
  │ Quality      │ 1080p           │ 1080p           │
  │ Codec        │ x264            │ x265 (50% less) │
  │ Size         │ 8.4 GB          │ 4.2 GB     ✅   │
  │ Seeds        │ 342             │ 1,245      ✅   │
  │ Audio        │ AAC 2.0         │ AAC 5.1    ✅   │
  │ Source       │ WEBRip          │ BluRay     ✅   │
  │ Score        │ 6.8             │ 9.2        ✅   │
  └──────────────┴─────────────────┴─────────────────┘

  Recomendación: Torrent B es superior en todo.
```

---

### Biblioteca Local

#### `unarr scan <directorio>`
Escanea tu biblioteca de películas/series y detecta oportunidades.

```
$ unarr scan ~/Movies/

  🔍 Escaneando 234 archivos...

  ⬆️ UPGRADES DISPONIBLES (14):
  • The Matrix (1999) — Actual: 720p → Disponible: 2160p HDR (892 seeds)
  • Inception (2010) — Actual: 1080p x264 → Disponible: 1080p x265 (mitad tamaño)
  ...

  📺 SERIES INCOMPLETAS (3):
  • Breaking Bad — Tienes S01-S04, falta S05 (disponible, 2,340 seeds)
  • The Office — Tienes S01-S07, faltan S08-S09
  ...

  💀 SIN SEEDS (2):
  • documental-raro.mkv — 0 seeds, sin alternativa encontrada
  ...
```

**Lógica:**
1. Recorrer directorio recursivamente
2. Identificar archivos de video (.mkv, .mp4, .avi)
3. Parsear nombre del archivo → título, calidad, idioma, temporada/episodio
4. Opcionalmente: `mediainfo` para specs reales
5. Buscar cada título en unarr
6. Comparar calidad local vs disponible
7. Detectar episodios faltantes en series
8. Reportar en formato legible

#### `unarr doctor [--qbittorrent URL] [--transmission URL]`
Diagnostica tu cliente de torrents.

```
$ unarr doctor --qbittorrent http://localhost:8080

  🏥 Diagnóstico de tu cliente de torrents

  Total torrents: 47
  ✅ Saludables (>10 seeds): 31
  ⚠️ Bajos seeds (1-10): 9
  💀 Muertos (0 seeds): 7

  Para los 7 muertos, encontré alternativas:
  • pelicula-x.torrent → 3 alternativas con seeds
  • serie-y-S02E03 → 1 alternativa con 456 seeds
  ...

  ¿Reemplazar automáticamente? [y/n]:
```

**Lógica:**
1. Conectar al cliente (qBittorrent API, Transmission RPC, etc.)
2. Listar todos los torrents activos
3. Clasificar por salud (seeds/leechers ratio)
4. Para los muertos/bajos: buscar alternativas en unarr
5. Ofrecer reemplazo automático (quitar viejo + añadir nuevo)

---

### Acciones Directas

#### `unarr add <query> [--to qbittorrent|transmission|deluge]`
Busca + descarga directo a tu cliente en un solo comando.

```
$ unarr add "breaking bad s01" --quality 1080p --lang es --to qbittorrent

  ✅ Añadidos 7 torrents a qBittorrent:
  • Breaking Bad S01E01 1080p ES — 1,890 seeds
  • Breaking Bad S01E02 1080p ES — 1,456 seeds
  ...
```

#### `unarr stream <query | hash>`
Streaming directo usando WebTorrent + VLC/mpv.

```
$ unarr stream "oppenheimer 1080p"

  ⚡ Streaming: Oppenheimer (2023) 1080p — 1,245 seeds
  Abriendo en VLC...
  Buffer: ████████░░ 80% | Speed: 12.4 MB/s | ETA: 3s
```

#### `unarr watch <query>`
Te dice dónde verlo legal + opción torrent.

```
$ unarr watch "oppenheimer"

  🎬 Oppenheimer (2023) — Dónde verlo:

  📺 STREAMING (tu país):
  • Amazon Prime Video (incluido en suscripción)
  • Apple TV (alquiler: 3.99€)

  🏴‍☠️ TORRENT (si prefieres):
  • 1080p x265 — 1,245 seeds — 4.2 GB
  • 2160p HDR  — 89 seeds  — 16 GB

  💡 Disponible en Prime Video. ¿Descargar igualmente? [y/n]:
```

#### `unarr monitor <serie> [--quality Q] [--lang L]`
Vigila una serie y avisa cuando sale episodio nuevo.

```
$ unarr monitor "The Last of Us S02" --quality 1080p --lang es

  👀 Monitorizando: The Last of Us S02
  Te avisaré cuando aparezca un nuevo episodio en 1080p ES.
  (Comprobando cada 30 min)
```

---

### Configuración

#### `unarr config`
Setup inicial interactivo.

```
$ unarr config

  🔧 Configuración de unarr

  API URL [https://torrentclaw.com]: 
  API Key []: tc_xxx...

  Cliente de torrents:
  ❯ qBittorrent
    Transmission
    Deluge
    Ninguno

  qBittorrent URL [http://localhost:8080]: 
  qBittorrent User [admin]: 
  qBittorrent Pass: ****

  Directorio de medios [~/Movies]: ~/NAS/Películas

  ✅ Configuración guardada en ~/.config/unarr/config.json
```

#### `unarr open <id>`
Abre contenido en torrentclaw.com en el navegador.

---

## Integraciones Locales (Go)

| Integración | Librería Go | Para qué |
|---|---|---|
| qBittorrent | `github.com/autobrr/go-qbittorrent` | Listar, añadir, reemplazar torrents |
| Transmission | `github.com/hekmon/transmissionrpc` | Mismo |
| Deluge | `github.com/gdm85/go-libdeluge` | Mismo |
| Torrent parsing | `github.com/anacrolix/torrent/metainfo` | Parsear .torrent y magnets |
| Torrent title | Parser propio o port | Extraer calidad/codec/idioma del nombre |
| Streaming | `github.com/anacrolix/torrent` | Streaming directo (mejor que WebTorrent en Go) |
| mediainfo | exec `mediainfo --Output=JSON` | Analizar archivos ya descargados |
| Abrir URLs | `github.com/pkg/browser` | Abrir en navegador/VLC |
| CLI framework | `github.com/spf13/cobra` | Parsing de comandos CLI |
| Colores | `github.com/fatih/color` | Colores en terminal |
| Tablas | `github.com/olekukonez/tablewriter` | Tablas formateadas |
| Prompts | `github.com/AlecAivazis/survey/v2` | Prompts interactivos |
| Config | `github.com/spf13/viper` | Config persistente (~/.config/unarr/) |

---

## Arquitectura

```
unarr/
├── cmd/
│   └── unarr/
│       └── main.go           ← Entry point
├── internal/
│   ├── commands/
│   │   ├── root.go           ← Cobra root command + global flags
│   │   ├── search.go
│   │   ├── inspect.go
│   │   ├── upgrade.go
│   │   ├── moreseed.go
│   │   ├── compare.go
│   │   ├── scan.go
│   │   ├── doctor.go
│   │   ├── stream.go
│   │   ├── watch.go
│   │   ├── add.go
│   │   ├── monitor.go
│   │   ├── popular.go
│   │   ├── recent.go
│   │   ├── stats.go
│   │   ├── config.go
│   │   └── open.go
│   ├── clients/
│   │   ├── factory.go        ← Factory pattern para clientes torrent
│   │   ├── qbittorrent.go
│   │   ├── transmission.go
│   │   └── deluge.go
│   ├── parser/
│   │   ├── torrent.go        ← Parsear .torrent y magnets
│   │   └── title.go          ← Extraer calidad/codec/idioma del nombre
│   ├── scanner/
│   │   ├── scanner.go        ← Escanear directorio de medios
│   │   └── mediainfo.go      ← Wrapper mediainfo/ffprobe
│   └── ui/
│       ├── table.go          ← Output formateado (tablas, colores)
│       ├── format.go         ← Formateo de tamaños, duraciones
│       └── prompt.go         ← Prompts interactivos
├── go.mod
├── go.sum
├── Makefile
├── .goreleaser.yml
├── README.md
├── LICENSE                    ← MIT
└── PLAN.md
```

---

## Fases de Desarrollo

### Fase 1: MVP (Core + Búsqueda + Inspect)
**Estimación: 8-10h**

1. Setup proyecto (TypeScript, commander, chalk)
2. Portar api-client.ts y types.ts del MCP server
3. `unarr config` — setup interactivo
4. `unarr search` — con todos los filtros y output formateado
5. `unarr inspect` — TrueSpec de torrents
6. `unarr popular`, `recent`, `stats`
7. Output `--json` para piping
8. Publicar en npm como `unarr`

### Fase 2: Inteligencia (Upgrade + MoreSeed + Compare)
**Estimación: 6-8h**

1. `unarr upgrade` — buscar mejor versión
2. `unarr moreseed` — más seeds, misma calidad
3. `unarr compare` — comparar torrents lado a lado
4. `unarr watch` — streaming legal + torrents

### Fase 3: Cliente Local (Add + Doctor)
**Estimación: 6-8h**

1. Integración qBittorrent (add, list, remove, replace)
2. Integración Transmission
3. `unarr add` — buscar + añadir a cliente
4. `unarr doctor` — diagnosticar cliente

### Fase 4: Biblioteca (Scan + Monitor + Stream)
**Estimación: 8-10h**

1. `unarr scan` — escanear biblioteca local
2. `unarr monitor` — vigilar series (daemon/cron)
3. `unarr stream` — WebTorrent + VLC/mpv
4. Integración mediainfo para análisis real de archivos

---

## Ecosistema unarr (Open Source)

```
GitHub: torrentclaw/
├── go-client   ← Librería Go compartida (API client + types) — MIT
├── unarr (unarr)  ← CLI terminal (Go, importa go-client) — MIT
├── torrentclaw-mcp          ← MCP server (TypeScript) — ya existe
├── torrentclaw-skill        ← Skill para agentes — ya existe
└── truespec                 ← Verificador specs (Go, migrará a go-client) — ya existe

GitHub: buryni/
└── torrent-aggregator       ← Web app (privado)
```

### go-client (Librería compartida)

```
go-client/
├── go.mod                   ← module github.com/torrentclaw/go-client
├── client.go                ← NewClient, config, HTTP base, retry, rate limiting
├── search.go                ← Search, Autocomplete
├── content.go               ← Popular, Recent, WatchProviders, Credits
├── torrent.go               ← TorrentDownloadURL
├── stats.go                 ← Stats
├── types.go                 ← SearchResult, TorrentInfo, etc.
├── errors.go                ← ApiError, manejo de errores
└── client_test.go           ← Tests
```

**Uso:**
```go
import tc "github.com/torrentclaw/go-client"

client := tc.NewClient("https://torrentclaw.com", "tc_apikey...")
results, _ := client.Search(tc.SearchParams{Query: "breaking bad", Quality: "1080p"})
```

**Consumidores:**
- unarr / unarr (este proyecto)
- truespec (migrar del HTTP client propio)
- Cualquier herramienta Go de terceros

## Publicación

- **Binarios precompilados:** GitHub Releases (linux/mac/windows amd64+arm64)
- **Homebrew tap:** `brew install torrentclaw/tap/unarr`
- **Go install:** `go install github.com/torrentclaw/unarr@latest`
- **Alias:** `un` (alias integrado en el binario)
- **README con GIFs animados** mostrando el CLI en acción
- **Goreleaser** para automatizar builds y releases

---

## Diferenciadores vs Competencia

Ningún CLI de torrents existente ofrece:
1. **Quality Score** — puntuación de calidad calculada por unarr
2. **Upgrade inteligente** — "tengo esto, dame algo mejor"
3. **MoreSeed** — "misma calidad pero con más seeds"
4. **Doctor** — diagnóstico de tu cliente de torrents
5. **Scan** — análisis de tu biblioteca con sugerencias
6. **Watch** — streaming legal vs torrent lado a lado
7. **Multi-cliente** — qBittorrent, Transmission, Deluge desde el mismo CLI
8. **30+ fuentes** — agregación de unarr con metadata TMDB

---

*Plan creado: 14 Feb 2026*
*Estado: Pendiente de aprobación para implementar*
