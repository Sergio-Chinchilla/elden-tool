# Elden Tool · Documentación técnica

Este documento describe la arquitectura, el modelo de datos, el motor de progresión, la calculadora de builds y el flujo de trabajo del proyecto. La descripción funcional está en [README.md](README.md).

**Alcance de la v1:** juego base de Elden Ring. El DLC *Shadow of the Erdtree* llega en la 2.0 y el modelo está preparado para añadirlo sin romper datos ni partidas guardadas.

---

## 1. Decisiones principales

| Tema | Decisión | Motivo |
| --- | --- | --- |
| Alojamiento | GitHub Pages (sitio estático) | Gratis, sin servidor, despliegue desde el propio repositorio |
| Aplicación | SPA con Vite + React + TypeScript | Mucha interactividad (filtros, estado de partida, builds); TypeScript tipa el modelo de datos de punta a punta |
| Enrutado | `HashRouter` (`/#/builds`) | GitHub Pages no reescribe rutas; con hash no hace falta el truco del `404.html` |
| Base de datos | **Documental (no relacional)**: ficheros JSON/YAML en el repositorio, compilados a JSON estático | Datos de solo lectura, pocos miles de registros, se editan a mano y se revisan en pull requests. Una BD relacional obligaría a tener servidor o a cargar SQLite en el navegador sin ganar nada |
| Integridad | Esquemas con **Zod** + script de validación en CI | Sustituye a las claves foráneas: comprueba que cada referencia (`id`) exista |
| Estado del usuario | `localStorage` + exportar/importar JSON | Funciona sin cuentas; `localStorage` admite ~5 MB frente a ~4 KB de una cookie |
| Datos numéricos | [erdb](https://github.com/EldenRingDatabase/erdb) (MIT) | Extraídos de los ficheros del juego, ~99 % autogenerados |
| Datos de progresión | Curados a mano | No existen en ninguna fuente estructurada |
| Idiomas | Español (por defecto) e inglés | Nombres oficiales de cada idioma |
| Tests | Vitest (motor y calculadora), Playwright (flujos clave) | — |

Si en el futuro se añaden cuentas (v3.0), se valorará un backend gestionado (por ejemplo Supabase o Firebase) solo para el progreso de usuario; el catálogo seguirá siendo estático.

---

## 2. Estructura del repositorio

```
EldenTool/
├─ README.md
├─ DOCUMENTATION.md
├─ LICENSE                      # MIT (código)
├─ LICENSE-DATA                 # CC BY-SA 4.0 (data/curated)
├─ data/
│  ├─ generated/                # Salida de erdb importada (no se edita a mano)
│  │  ├─ weapons.json
│  │  ├─ shields.json
│  │  ├─ sorceries.json
│  │  ├─ incantations.json
│  │  ├─ ashes-of-war.json
│  │  └─ calc-correct-graphs.json
│  ├─ curated/                  # Editado a mano (YAML)
│  │  ├─ regions.yaml
│  │  ├─ zones.yaml
│  │  ├─ bosses.yaml
│  │  ├─ events.yaml
│  │  ├─ key-items.yaml
│  │  ├─ collectibles.yaml      # Semillas Doradas, Lágrimas Sagradas
│  │  ├─ npcs.yaml              # NPC y pasos de sus misiones
│  │  ├─ endings.yaml
│  │  ├─ tags.yaml
│  │  ├─ item-extras.yaml       # Etiquetas, cómo conseguirlo, vídeo, por id de objeto
│  │  └─ builds.yaml            # Builds predefinidas
│  └─ i18n/
│     ├─ es.json                # Textos de la interfaz
│     └─ en.json
├─ scripts/
│  ├─ import-erdb.ts            # Normaliza la salida de erdb → data/generated
│  ├─ build-data.ts             # Une generated + curated → public/data/*.json
│  └─ validate-data.ts          # Esquemas + integridad referencial + grafo
├─ src/
│  ├─ model/                    # Tipos y esquemas Zod
│  ├─ engine/                   # Motor de progresión (puro, sin React)
│  ├─ builds/                   # Cálculo de requisitos, escalado y AR
│  ├─ storage/                  # localStorage, export/import, migraciones
│  ├─ i18n/
│  ├─ pages/                    # Progreso, Ruta, Bosses, Finales, Builds, Ajustes
│  └─ components/
├─ public/data/                 # JSON compilado que descarga la web (no se edita)
├─ tests/
└─ .github/workflows/
   ├─ ci.yml                    # lint + typecheck + validate-data + tests
   └─ deploy.yml                # build + publicar en GitHub Pages
```

---

## 3. Modelo de datos

Todos los registros tienen un `id` estable en *kebab-case* (en inglés, para no depender del idioma) y un campo `name` con traducciones. Los ids nunca se reutilizan ni se renombran, porque las partidas guardadas los referencian.

```ts
type Id = string;                       // "maliketh", "leyndell-royal-capital"
type Localized = { es: string; en: string };
type Stat = "vig" | "min" | "end" | "str" | "dex" | "int" | "fai" | "arc";
type Dlc = "base" | "sote";             // v1 solo usa "base"
```

### 3.1 Condiciones

Las condiciones son la pieza central: describen requisitos, bloqueos y pasos de finales.

```ts
type Condition =
  | Id                                         // el nodo está completado
  | { all: Condition[] }
  | { any: Condition[] }
  | { not: Condition }
  | { count: { tag: Id; gte: number } }        // p. ej. 2 Grandes Runas
  | { npc: Id; state: NpcState }               // estado de un NPC
  | { level: { gte: number } };
```

Ejemplo en YAML:

```yaml
requires:
  all:
    - { count: { tag: great-rune, gte: 2 } }
    - margit
```

### 3.2 Nodos de progresión

Zonas, bosses, eventos, objetos clave, coleccionables y pasos de misión comparten una base común. Así el motor trata a todos igual.

```ts
type NodeKind = "zone" | "boss" | "event" | "key-item" | "collectible" | "quest-step";

interface ProgressNode {
  id: Id;
  kind: NodeKind;
  name: Localized;
  dlc: Dlc;
  region?: Id;
  zone?: Id;                       // dónde está (para bosses, objetos…)
  requires?: Condition;            // cuándo pasa a estar disponible
  lostIf?: Condition;              // cuándo deja de ser posible
  implies?: Id[];                  // qué se da por hecho si está completado
  tags?: Id[];                     // "great-rune", "remembrance", "golden-seed"…
  spoilerTier: 0 | 1 | 2 | 3;      // para el modo sin spoilers
  notes?: Localized;
  wikiUrl?: string;
  videoUrl?: string;
}

interface Zone extends ProgressNode {
  kind: "zone";
  recommendedLevel: [number, number];
  parent?: Id;                     // subzona (p. ej. una mazmorra dentro de una región)
}

interface Boss extends ProgressNode {
  kind: "boss";
  mandatory: boolean;
  remembrance: boolean;
  recommendedLevel: [number, number];
}
```

Ejemplos:

```yaml
# events.yaml
- id: erdtree-burned
  kind: event
  name: { es: "Quemar el Árbol Áureo", en: "Burn the Erdtree" }
  requires: fire-giant
  implies: [fire-giant]
  spoilerTier: 3

# zones.yaml
- id: leyndell-royal-capital
  kind: zone
  name: { es: "Leyndell, Capital Real", en: "Leyndell, Royal Capital" }
  recommendedLevel: [70, 90]
  requires: { count: { tag: great-rune, gte: 2 } }
  lostIf: erdtree-burned
  spoilerTier: 2

# bosses.yaml
- id: maliketh
  kind: boss
  name: { es: "Maliketh, la Espada Negra", en: "Maliketh, the Black Blade" }
  zone: crumbling-farum-azula
  mandatory: true
  remembrance: true
  recommendedLevel: [110, 130]
  requires: erdtree-burned
  implies: [erdtree-burned]
  spoilerTier: 3
```

> Los niveles y requisitos de estos ejemplos son ilustrativos; los valores reales se curan y revisan en la fase de datos.

### 3.3 NPC y misiones

```ts
type NpcState = "alive" | "dead" | "quest-failed" | "quest-completed";

interface Npc {
  id: Id;
  name: Localized;
  questSteps: Id[];             // ids de nodos kind="quest-step", en orden
  failsIf?: Condition;          // p. ej. matar al NPC o quemar el Árbol antes de cierto paso
}
```

Cada paso de misión es un `ProgressNode` con `requires` (paso anterior + condiciones) y `lostIf` (punto de no retorno).

### 3.4 Finales

```ts
interface Ending {
  id: Id;
  name: Localized;
  steps: Id[];                   // nodos a completar, en orden recomendado
  avoid: AvoidRule[];            // lo que hay que evitar
  blockedIf: Condition;          // si se cumple, el final es imposible
}

interface AvoidRule {
  condition: Condition;          // lo que NO hay que hacer
  before?: Id;                   // "evitar hasta haber completado X"
  reason: Localized;
}
```

### 3.5 Coleccionables

Semillas Doradas (30) y Lágrimas Sagradas (12) son nodos `kind: "collectible"` con la etiqueta `golden-seed` o `sacred-tear`, su zona y sus requisitos. No bloquean la historia; sirven para la ruta sugerida y el seguimiento.

### 3.6 Catálogo de objetos (builds)

Los datos numéricos vienen de `data/generated` y los datos editoriales de `data/curated/item-extras.yaml`. El script `build-data.ts` los une por `id`.

```ts
type ItemCategory = "weapon" | "shield" | "catalyst" | "sorcery" | "incantation" | "ash-of-war";
type Grade = "S" | "A" | "B" | "C" | "D" | "E" | "-";
type Affinity = "standard" | "heavy" | "keen" | "quality" | "fire" | "flame-art" | "lightning"
  | "sacred" | "magic" | "cold" | "poison" | "blood" | "occult";

interface CatalogItem {
  id: Id;
  category: ItemCategory;
  name: Localized;
  dlc: Dlc;
  subtype?: string;                         // "straight-sword", "colossal-weapon"…
  requirements: Partial<Record<Stat, number>>;
  // Armas
  scaling?: Partial<Record<Stat, Grade>>;   // con mejora +0 y afinidad estándar
  scalingByAffinity?: Partial<Record<Affinity, Partial<Record<Stat, Grade>>>>;
  canInfuse?: boolean;
  upgrade?: "smithing" | "somber";
  baseDamage?: Partial<Record<"physical" | "magic" | "fire" | "lightning" | "holy", number>>;
  statusBuildup?: Partial<Record<"bleed" | "frost" | "poison" | "rot" | "sleep" | "madness" | "death", number>>;
  // Hechizos
  fpCost?: number;
  slots?: number;
  // Datos curados
  tags: Id[];
  acquisition: Acquisition;
}

interface Acquisition {
  description: Localized;
  zone?: Id;
  requires?: Condition;          // se reutiliza el motor: ¿puedo conseguirlo ya?
  lostIf?: Condition;            // objetos que se pierden
  videoUrl?: string;             // YouTube, con marca de tiempo si procede
  wikiUrl?: string;
}

interface Tag {
  id: Id;
  name: Localized;
  kind: "mechanic" | "theme" | "type" | "stat";
}

interface PresetBuild {
  id: Id;
  name: Localized;
  description: Localized;
  tags: Id[];
  items: Id[];
  suggestedStats?: Partial<Record<Stat, number>>;
}
```

Etiquetas *mechanic* se pueden derivar en parte automáticamente (daño elemental y acumulación de estados de erdb). Las *theme* se asignan siempre a mano.

---

## 4. Motor de progresión

Código en `src/engine/`, sin dependencias de React, totalmente cubierto por tests.

### 4.1 Entradas

- El catálogo de nodos, NPC y finales.
- El estado del jugador: nodos marcados, estado de NPC, nivel.

### 4.2 Cierre de implicaciones

Antes de evaluar nada, se calcula el conjunto **efectivo** de nodos completados recorriendo `implies` (y los `requires` simples de tipo `Id`) hasta un punto fijo. Así, marcar `maliketh` añade `erdtree-burned`, `fire-giant`, etc. Los nodos deducidos se muestran en la interfaz como "deducido" para que el usuario sepa de dónde salen.

### 4.3 Estado de cada nodo

| Estado | Regla |
| --- | --- |
| `completed` | Está en el conjunto efectivo |
| `lost` | `lostIf` se cumple y no está completado |
| `available` | `requires` se cumple, no está completado ni perdido |
| `locked` | `requires` no se cumple todavía |

El motor devuelve también el **motivo** (`explain`): qué parte de la condición falla o qué evento provocó la pérdida. Con eso se generan los mensajes "Ya no puedes: la capital quedó en cenizas al quemar el Árbol Áureo".

### 4.4 Ruta sugerida y próximos bosses

- **Ruta:** zonas `available` ordenadas por distancia entre el nivel del jugador y `recommendedLevel`, priorizando las que contienen bosses obligatorios o coleccionables pendientes.
- **Próximos bosses:** bosses `available`, con filtros (obligatorio, región, nivel, estado).

### 4.5 Guía de finales

Para el final elegido:

1. Si `blockedIf` se cumple → **bloqueado**, con el motivo.
2. **Pasos pendientes**: los `steps` no completados. Se destacan los "próximos" (`available` ya, o a una sola condición de estarlo).
3. **Riesgos**: las `avoid` cuya `condition` es alcanzable ahora (sus requisitos están cumplidos) y cuyo `before` aún no está hecho. Se ordenan por cercanía.
4. Los `lostIf` de los pasos pendientes también se muestran como riesgos ("si quemas el Árbol antes de este paso, pierdes la misión").

### 4.6 Validaciones del grafo (CI)

`validate-data.ts` falla el build si:

- Una condición referencia un `id` inexistente.
- Hay ciclos en `requires`/`implies`.
- Un nodo nunca puede llegar a estar disponible partiendo de una partida vacía.
- Algún final no se puede completar desde una partida vacía (se simula siguiendo sus `steps`).
- Faltan traducciones en `name`.

---

## 5. Calculadora de builds

Código en `src/builds/`.

### 5.1 Requisitos mínimos (v1.0)

- Stats mínimas = máximo de cada requisito entre los objetos seleccionados.
- A dos manos, la Fuerza efectiva es `floor(fuerza × 1.5)`; se muestra la Fuerza necesaria a una y a dos manos.
- Nivel de runas resultante a partir de la clase inicial elegida (la clase con menor nivel que cumple los mínimos se sugiere automáticamente).

### 5.2 Escalado (v1.0)

- Letras de escalado por atributo con la afinidad seleccionada (`scalingByAffinity`).
- Aviso si el objeto no admite afinidades (`canInfuse: false`) o se mejora con piedras de forja sombría.

### 5.3 Daño (AR) (v1.x)

Se implementa la fórmula del juego con los parámetros de erdb: daño base por mejora, coeficientes de escalado por afinidad y curvas `CalcCorrectGraph`. Se validará con una tabla de casos medidos en el juego (`tests/fixtures/ar-cases.json`).

### 5.4 Filtros por etiquetas

- Filtro AND/OR por etiquetas (por defecto OR dentro de un mismo tipo y AND entre tipos), para permitir builds mixtas como "Muerte + Congelación".
- Filtros adicionales: categoría, tipo de arma, atributo principal, "solo lo que puedo conseguir ya" (usa `acquisition.requires` con el motor de progresión).

### 5.5 Compartir

La build se serializa (ids + stats + afinidades) en JSON comprimido y codificado en base64url en la URL: `/#/builds?b=…`.

---

## 6. Estado del usuario

### 6.1 Formato

```json
{
  "schemaVersion": 1,
  "app": "elden-tool",
  "dataVersion": "1.0.0",
  "updatedAt": "2026-11-19T15:00:00+01:00",
  "player": {
    "name": "Mi partida",
    "level": 72,
    "startingClass": "astrologer",
    "stats": { "vig": 30, "min": 20, "end": 15, "str": 12, "dex": 14, "int": 40, "fai": 7, "arc": 9 }
  },
  "completed": ["margit", "godrick", "rennala", "erdtree-burned"],
  "npcStates": { "ranni": "alive", "fia": "quest-failed" },
  "targetEnding": "age-of-stars",
  "build": { "items": ["meteorite-staff", "rock-sling"], "affinities": {} },
  "settings": { "spoilerMode": true, "language": "es" }
}
```

Solo se guardan los nodos marcados por el usuario; los deducidos se recalculan al cargar.

### 6.2 Almacenamiento

- Guardado automático en `localStorage` (clave `elden-tool:save:v1`), con *debounce*. Todas las lecturas y escrituras van en `try/catch`: si el navegador no permite almacenamiento, la web sigue funcionando y avisa de que hay que exportar.
- **Exportar**: descarga `elden-tool-<fecha>.json`.
- **Importar**: se valida con Zod; si hay ids desconocidos se descartan y se avisa.
- Varias partidas: lista de guardados en `localStorage` (opcional en v1.0).

### 6.3 Migraciones

`src/storage/migrations.ts` contiene una función por versión (`1 → 2`, …). Al cargar un JSON antiguo se aplican en cadena. La 2.0 (DLC) añade campos sin cambiar los ids existentes.

---

## 7. Fuentes de datos y pipeline

1. **erdb** genera JSON a partir de los ficheros del juego (o se usa su API pública `api.erdb.wiki`). Se fija la versión del juego usada.
2. `scripts/import-erdb.ts` normaliza nombres de campos, ids y traducciones → `data/generated/`.
3. Los datos curados se escriben a mano en `data/curated/*.yaml`, usando como referencia la [Elden Ring Wiki de wiki.gg](https://eldenring.wiki.gg/) (CC BY-SA 4.0). Los textos se redactan con palabras propias y se enlaza a la fuente.
4. `scripts/build-data.ts` une ambas partes y escribe `public/data/*.json` (un fichero por colección, minificado; con hash en el nombre para la caché).
5. `scripts/validate-data.ts` valida todo (sección 4.6).

**No se usa** contenido de Fextralife (licencia propia, no abierta) ni se redistribuyen ficheros del juego. No se scrapea wiki.gg sin permiso de sus administradores.

Volumen estimado para la v1: ~60 zonas y subzonas, ~40 bosses principales (más opcionales), ~20 eventos, ~15 misiones de NPC, 6 finales, 42 coleccionables, ~300 armas, ~150 hechizos y encantamientos.

---

## 8. Interfaz

| Ruta | Página |
| --- | --- |
| `/#/` | Inicio: resumen de la partida, próximos pasos y riesgos del final elegido |
| `/#/progress` | Marcar bosses, eventos, objetos clave, NPC y coleccionables |
| `/#/route` | Ruta sugerida por zonas |
| `/#/bosses` | Próximos bosses con filtros |
| `/#/endings` | Elegir final y ver pasos, riesgos y bloqueos |
| `/#/missables` | Lo que se pierde si no se hace a tiempo |
| `/#/builds` | Catálogo con etiquetas y constructor de builds |
| `/#/settings` | Idioma, modo sin spoilers, exportar/importar |

- Diseño *mobile first*, tema oscuro y claro, accesible (contraste AA, navegación por teclado).
- **Modo sin spoilers**: los nodos `locked` con `spoilerTier` mayor que el progreso actual se muestran como "???".
- Sin imágenes del juego: iconos propios o de una librería libre (por ejemplo Lucide).
- Los vídeos se enlazan, no se incrustan, para no cargar YouTube por defecto.

---

## 9. Desarrollo, CI y despliegue

```bash
npm install
npm run data:build      # genera public/data
npm run data:validate
npm run dev             # servidor local
npm test                # Vitest
npm run build           # build de producción
```

- **ci.yml** (en cada pull request): lint (ESLint + Prettier), `tsc --noEmit`, `data:validate`, tests.
- **deploy.yml** (en cada push a `main`): build y publicación con `actions/deploy-pages`. En `vite.config.ts`, `base: "/EldenTool/"` (nombre del repositorio).
- Versionado semántico; `dataVersion` en el JSON del usuario indica con qué datos se guardó.

---

## 10. Licencias y avisos

- Código: **MIT** (`LICENSE`).
- Datos curados (`data/curated`): **CC BY-SA 4.0** (`LICENSE-DATA`), compatible con la wiki de wiki.gg.
- Datos generados: derivados del juego mediante erdb (MIT); se publican solo los valores necesarios.
- Pie de la web: aviso de no afiliación con FromSoftware ni Bandai Namco y créditos a erdb y wiki.gg.
- Sin anuncios ni monetización.

---

## 11. Plan de trabajo

| Fase | Contenido |
| --- | --- |
| 0 | Esqueleto del repo, esquemas Zod, pipeline de datos, CI y despliegue vacío en Pages |
| 1 | Motor de progresión con tests; datos de Necrolimbo como prueba de concepto |
| 2 | Resto de regiones del juego base, bosses, eventos, objetos clave, coleccionables |
| 3 | NPC, misiones y los 6 finales; guía de finales y lista de cosas que se pierden |
| 4 | Catálogo importado de erdb, etiquetas, constructor de builds y builds predefinidas |
| 5 | Pulido: modo sin spoilers, inglés, accesibilidad → **v1.0** |
| 2.0 | *Shadow of the Erdtree* (incluye Bendición del Scaduárbol como eje de progresión) |
