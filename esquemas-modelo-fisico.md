# Esquemas / Modelo físico — estado y reanudación

> **Última actualización:** 2026-06-24. **Doc vivo.** Punto de reanudación para definir el **DDL** del proyecto
> (las tablas SQL reales), usando los modelos de **engram** como contrapunto.
> Fuente conceptual: `prd-memoria-de-equipo.md` (modelo de datos, secciones 2-3, DM-*, ID-*, ADM-*).

## Cómo reanudar

- Definimos el DDL **una tabla a la vez**, comparando contra engram en cada paso (qué adoptar, qué descartar).
- Recomendación de artefacto: volcar el DDL en un **`schema.sql` versionado** (el SQL no puede vivir como prosa).
- **Importante:** el DDL **NO existe todavía** como artefacto en el repo. Lo "aprobado" (memoria, org/credenciales)
  son **decisiones**, no SQL escrito. Hay que escribir `schema.sql` igual, incluso para lo ya aprobado.
- Memoria de engram relacionada: `mem_search` → topic keys `memoria/schema-inventory` y `engram-schema/reference`.

## Inventario de tablas vs engram

| Tabla del proyecto | Estado | Equivalente en engram | Diferencia clave |
|---|---|---|---|
| `memories` | ✅ **cerrada** (DDL abajo) | `observations` | `local_id INTEGER PK` + `id TEXT UNIQUE` (uuid v7 global, **NO** el dual local/cloud de engram); anclaje `org/project/repo` por **FK compuestas** (tenant CFG-4 + coherencia repo∈proyecto); `tag`+`topic` (engram: solo `topic_key`, **sin tags**); `lifecycle` sin `deleted` en local (purga física, DM-13); **#1554** LWW + `memory_versions` |
| `memory_versions` (#1554) | ✅ **cerrada** (DDL abajo) | **no existe** | append-only keyed por `(memory_id, version)`; guarda **cada edición** (audit completo, no solo el perdedor de LWW) — reemplaza la UI de `conflict`, ahora eliminado |
| `users` + `api_keys` + `project_admin_grants` | ✅ aprobada (org/credenciales) | `cloud_users` | **3 credenciales separadas** (key de sync / password del visor ID-11 / login cloud ADM-6); engram tiene solo `cloud_users` y la key es config, no vive en DB |
| `memory_relations` | ✅ **cerrada** (DDL abajo) | `memory_relations` | vocab `replaces/extends/caused/diverges` + **validez resuelta por la DB** (VAL-1); engram usa `related/compatible/supersedes/...` + `judgment_status` + `confidence`, multi-actor **sin UNIQUE** en `(source,target)` |
| `org` (tenant boundary) | ✅ **cerrada** (DDL abajo) | **no existe** | engram no tiene tenant/org; acá org = aislamiento duro (CFG-4, NFR-11). **LOCAL-only, sin token** (secreto en keychain del OS, CFG-6) |
| `project` + `repo` | ✅ **cerrada** (DDL abajo) | **no existe** | id local + clave estable (mecanismo 1): `repo.origin` (RES-7), `project.sync_id` cloud-assigned; engram **adivina por carpeta** (RES-4, criticado) |
| `repo_topology` | ✅ **cerrada** (DDL abajo) | **no existe** | aristas dirigidas **repo→repo** (DM-5); ancla en **org** (cross-proyecto OK, cross-org no); relación proyecto↔proyecto **derivada** (query, no tabla) |
| mapping `origin→proyecto` (autoritativo cloud) | 🟡 parcial (local: `repo.origin`+`project_id`) | **no existe** | mapping org-level que baja por sync (RES-8/ADM-11) — se cierra con sync |
| `tag_vocabulary` (normalización local) | ✅ **cerrada** (DDL abajo) | **no existe** | whitelist **universal** (mismo vocab para todas las orgs); 1 columna `tag TEXT PK COLLATE NOCASE`; vive en 2 artefactos (skill + tabla, CON-10); LOCAL-only, sin `org_id`, sin FK; validación en código → default `no_clasificado` (CON-11) |
| sync: `sync_state` + outbox + cursors + audit | ❌ falta | `sync_state`, `sync_mutations`, `cloud_mutations`, `cloud_sync_audit_log` | engram lo tiene **muy maduro** — **adoptar el patrón** (3 cursores: enqueued/acked/pulled + outbox FIFO) |
| `memories_fts` (FTS5) | ✅ **cerrada** (DDL abajo, **Opción A**) | `observations_fts` | external-content + `content_rowid` entero estable; **impone `local_id INTEGER PK` en `memories`**; índice **LOCAL-only/derivado** (nunca sincroniza); filtros por JOIN, no en el índice |

## Orden de trabajo propuesto (por dependencias de FK)

1. **Estructura de anclaje** — ✅ `org → project → repo` + `repo_topology` **cerradas** (DDL abajo). Falta el
   mapping `origin→proyecto` autoritativo (cloud, baja por sync) — se cierra con sync. Es la base de la que
   cuelgan las FK de `memories`, y es donde engram **no tiene nada** (máximo diferencial).
2. ~~**`memories`** — escribir el DDL real con las FK de anclaje (concepto aprobado, DM-15).~~ ✅ **cerrada** (DDL
   abajo): identidad Opción A (`local_id INTEGER PK` + `id TEXT UNIQUE`), anclaje por **FK compuestas** (tenant +
   coherencia repo∈proyecto), #1554 (LWW + `memory_versions`).
3. ~~**`memory_relations`** — aristas dirigidas (DM-10) + validez por DB (VAL-1).~~ ✅ **cerrada** (ver DDL abajo).
4. ~~**`tag_vocabulary`** — normalización local derivada de la skill (CON-10).~~ ✅ **cerrada** (DDL abajo).
5. **Sync** — `sync_state` + outbox + cursors + audit (reusar patrón de engram).
6. ~~**`memories_fts`** — virtual table FTS5 + triggers.~~ ✅ **cerrada** (Opción A, DDL abajo). Impuso
   `local_id INTEGER PK` en `memories` (paso 2).

> Nota: lo aprobado de "organización" suena a **users/keys/grants** (auth), distinto de la **estructura de anclaje**
> (org/proyecto/repo/topología). Confirmar al reanudar qué cubrió exactamente la aprobación previa.

## Decisiones de modelo ya tomadas (del PRD)

- **Identidad:** UUID v7 único (no dual id local/cloud como engram). Unicidad por validación server-side; colisión
  accidental se resuelve re-asignando el id (SYNC-14), no mergeando. `prd:DM-15, 46, 671`
- **Búsqueda:** FTS5 + BM25 en SQLite local, igual que engram. Índice **local-only y derivado** (nunca sincroniza);
  external-content sobre `local_id` (**Opción A**, cerrada 2026-06-23). `prd:583`
- **Resolución repo→proyecto:** mapping org-level explícito keyed por `origin` de git; sin heurística por carpeta. `prd:RES-7/RES-8`
- **Credenciales:** tres separadas — key de sync (online, ID-2/ID-8) / password del visor local (ID-11) / login cloud admin (ADM-6).
  Tablas online que nunca bajan a local: `users`, `project_admin_grants`, tabla de keys. `prd:ID-2/ID-5, ADM-6`
- **Tags:** vocabulario controlado (CON-8); storage varchar; validación contra tabla de normalización local materializada de la skill (CON-10).
- **Conflicto de edición (LWW) — ✅ incorporado (2026-06-23) — hilo #1554:** dos ediciones de la MISMA
  memoria (mismo uuid, multi-device del mismo autor) se resuelven **last-write-wins** por timestamp, **NO** con UI
  de reconciliación. **Se ELIMINA el estado `conflict`.** El historial completo de ediciones vive en
  **`memory_versions`** (append-only, audit completo: cada edición versiona, no solo el perdedor). 
  Barrido pendiente en el PRD: DM-12/13/14, SYNC-6/9/12, VIS-9, SRCH-6.
- **Anclaje (org/project/repo) — cerrado, mecanismo 1:** `memory` guarda **FK enteros LOCALES**; el sync traduce a
  **clave estable** (`repo.origin` RES-7 / `project.sync_id` cloud / org implícita). Los ids locales nunca viajan.
  El **doble-id local↔cloud** de engram (`sync_id`) **NO** se replica en `memory`: las `memory_relations` cuelgan del
  **uuid** (`id`), no de un id de sync. Matiz (Opción A, 2026-06-23): `memory` sí suma `local_id INTEGER` además del uuid,
  pero es **rowid local del FTS que no sincroniza** (nada lógico cuelga de él), no el dual de engram. `org` es
  **LOCAL-only** y **sin token** (secreto en el keychain del OS, CFG-6). Detalle en engram, topic `prd/data-model/anchor-tables`.
- **Pendientes abiertos (del commit de hoy):** reconciliación proyecto-local provisional ↔ mapping org (cerca de RES-8);
  fallback local opcional de credencial cifrada at-rest (no hasheada) para entornos sin keychain (CFG-6/LOCAL-8). `prd:934-950`
- **Reabrir PRD por `repo_topology` (decidido 2026-06-23):** DM-5/DM-6 pasan de "aristas entre repos **de un proyecto**"
  a "aristas **repo→repo intra-org**" (cross-proyecto permitido); la relación proyecto↔proyecto es **derivada**, no entidad.
  VIS-4 suma capa **opcional** de mapa de org (aristas que cruzan proyecto). Permisos: la arista saliente la declara el dueño
  del repo source (suaviza ADM-5).   ✅ **Cerrado (2026-06-23):** `repo` queda sin `role` (es relacional, vive en `repo_topology.relation_type`) y suma `name` como alias amigable para humanos, no derivado del `origin`.

---

## DDL cerrado

### Anclaje — `org` / `project` / `repo` (cerrado 2026-06-23, LOCAL/SQLite)

```sql
CREATE TABLE org (
  id        INTEGER PRIMARY KEY,
  name      TEXT NOT NULL,
  cloud_url TEXT                        -- NULL = pseudo-org LOCAL (CFG-3); el secreto NO va acá (keychain OS, CFG-6)
);
CREATE UNIQUE INDEX idx_org_one_local ON org((cloud_url IS NULL)) WHERE cloud_url IS NULL;  -- a lo sumo una LOCAL

CREATE TABLE project (
  id      INTEGER PRIMARY KEY,
  org_id  INTEGER NOT NULL REFERENCES org(id),
  name    TEXT NOT NULL,
  sync_id TEXT,                         -- id estable del cloud; NULL = provisional/LOCAL (lo crea el admin, ADM-11)
  UNIQUE (org_id, id)                   -- target de la FK compuesta (org_id, project_id) de memories (redundante con la PK, SQLite lo exige)
);

CREATE TABLE repo (
  id         INTEGER PRIMARY KEY,
  org_id     INTEGER NOT NULL REFERENCES org(id),        -- denormalizado (habilita el UNIQUE y el lookup origin→proyecto)
  project_id INTEGER NOT NULL REFERENCES project(id),    -- DM-2: no hay repo sin proyecto
  origin     TEXT    NOT NULL,                           -- git origin = identidad estable (RES-7)
  name       TEXT,                                       -- alias amigable para humanos; puede diferir del nombre en GitHub (no derivado del origin)
  UNIQUE (org_id, origin),                               -- un origin = un repo por org (RES-7/ADM-11)
  UNIQUE (org_id, id),                                   -- target de las FK compuestas de repo_topology (redundante con la PK, pero SQLite lo exige)
  UNIQUE (project_id, id)                                -- target de la FK compuesta (project_id, repo_id) de memories (coherencia repo∈proyecto)
);
```

**Mecanismo 1 (id local + clave estable):** `memory` referencia el anclaje por **FK entero local** (joins compactos
para scope/topología); el **sync traduce** a clave estable (`repo.origin`, `project.sync_id`, org implícita). Los ids
locales nunca cruzan el wire → la dualidad de ids no genera conflicto. Ver `repo_topology` abajo.

### Topología — `repo_topology` (cerrado 2026-06-23, LOCAL/SQLite)

Aristas **dirigidas repo→repo** (DM-5). El **nodo es el repo** (código real, identidad por `origin`), **no** el
proyecto: el proyecto no se relaciona "al aire", lo **plasma el código**. Ancla en **org** (tenant boundary, CFG-4),
**no** en proyecto → una arista puede **cruzar proyectos** (el back de A consume el back de B), nunca cruzar org. La
relación **proyecto↔proyecto es derivada** (query), no tabla.

```sql
CREATE TABLE repo_topology (
  org_id         INTEGER NOT NULL,                 -- tenant boundary: la arista vive en una org (CFG-4), nunca la cruza
  source_repo_id INTEGER NOT NULL,
  target_repo_id INTEGER NOT NULL,
  relation_type  TEXT,                             -- "saca info de", "sirve a", "consume"... varchar libre (DM-6, ≈CON-9); nullable
  -- FK COMPUESTAS: ambos repos en la MISMA org (permite distinto proyecto). El project de cada punta sale de repo.project_id
  FOREIGN KEY (org_id, source_repo_id) REFERENCES repo(org_id, id) ON DELETE CASCADE,
  FOREIGN KEY (org_id, target_repo_id) REFERENCES repo(org_id, id) ON DELETE CASCADE,
  PRIMARY KEY (source_repo_id, target_repo_id),    -- una arista dirigida por par ordenado; relation_type es etiqueta, no identidad
  CHECK (source_repo_id <> target_repo_id)         -- sin auto-loops
);
CREATE INDEX idx_topology_target ON repo_topology(target_repo_id);  -- traversal inverso: "¿quién apunta a este repo?" (la salida la cubre la PK)
```
> **Requiere en `repo`:** `UNIQUE (org_id, id)` (target de las FK compuestas; redundante con la PK, pero SQLite lo exige).

**Relación proyecto↔proyecto = VISTA derivada, NO tabla** (no declarar lo que el código ya plasma, DM-11):
```sql
SELECT DISTINCT rs.project_id AS from_project, rt.project_id AS to_project, t.relation_type
FROM repo_topology t
JOIN repo rs ON rs.id = t.source_repo_id
JOIN repo rt ON rt.id = t.target_repo_id
WHERE rs.project_id <> rt.project_id;   -- las aristas que cruzan proyecto = mapa A↔B emergente
```

**Decisiones y porqué:**
- **El nodo es el repo, no el proyecto.** El proyecto es **gobierno/anclaje** (memorias transversales DM-3, permisos ADM-5, mapping RES-8), **no conectividad**. La relación entre proyectos **emerge** de las aristas de código; declararla aparte sería relación "al aire" + riesgo de **drift** (dos fuentes de verdad). Derivada = **single source of truth, no puede mentir** respecto del código. Además preserva dirección y multiplicidad (A→B con N aristas) que una tabla declarativa colapsaría.
- **Ancla en `org`, NO en `project` (cambia DM-5).** El único límite duro es el tenant (CFG-4). Cross-proyecto sí (caso real microservicios), cross-org nunca. La FK compuesta `(org_id, repo_id)` hace **irrepresentable** la arista cross-org; el proyecto de cada punta sale de `repo.project_id`. **Sin `project_id` propio en la arista.**
- **`relation_type` TEXT libre, nullable.** DM-6 es SHOULD, ejemplos en lenguaje natural ("saca info de"), no vocab cerrado (≈ `repo.role`). La **dirección ya tiene sentido sin etiqueta**. Contrasta con `memory_relations` (set cerrado + validez por DB, VAL-1): acá no hay semántica que validar.
- **PK `(source, target)` — una arista por par ordenado.** `relation_type` es etiqueta, no identidad (más simple que `memory_relations`). **[abierto:** si se quiere multi-tipo por par, la PK pasa a `(source, target, relation_type)`].
- **Permisos (ADM-5/ADM-11):** la arista es **saliente** (la plasma el repo source). El `project-admin` del repo source la declara — no necesita ser dueño del target, solo apunta a un `origin` (no lo saca del dominio de nadie, ADM-11). El target la recibe como entrante (lectura/visualización, no toca su gobierno).
- **`ON DELETE CASCADE`** protege el **LOCAL** (purga física). Re-mapear un repo a otro proyecto **NO** borra la arista (sigue válida: ancla en org, no en project) — solo cambia de qué proyecto "sale" en la vista derivada.
- **Engram no tiene NADA.** Sin repos, sin topología (`project` = string plano). Greenfield puro, **máximo diferencial** — igual que `org/project/repo`.

**Cloud (Postgres): BLOQUEADO** — depende del modelo de anclaje cloud (`org/project/repo` en Postgres, **aún no escrito**). Forma futura: keyear por `origin` (RES-7, clave estable) + `upload_at` (cursor de pull, SYNC-7) + `author_id` (admin que la declaró, ADM-11).

---

### `memories` — entidad raíz (cerrada 2026-06-24, LOCAL/SQLite)

Tabla raíz del modelo (DM-15). Identidad **Opción A**: `local_id INTEGER PK` (rowid del FTS, local-only) + `id TEXT
UNIQUE` (uuid v7, identidad lógica de la que cuelgan `memory_relations` y `memory_versions`). Anclaje por **FK enteras
locales** (mecanismo 1: el sync traduce a clave estable, los ids locales no viajan), con **FK compuestas** que hacen
irrepresentable cruzar tenant (CFG-4) o anclar un repo fuera de su proyecto.

```sql
CREATE TABLE memories (
    local_id        INTEGER PRIMARY KEY,                    -- rowid estable = content_rowid del FTS; LOCAL-only, NO sincroniza
    id              TEXT    NOT NULL UNIQUE,                -- uuid v7; identidad lógica/portable (DM-15); cuelgan relations/versions

    title           TEXT    NOT NULL,
    content         TEXT    NOT NULL,
    topic           TEXT,                                   -- string libre, opcional, sin entidad propia (DM-7/DM-8)
    tag             TEXT,                                   -- vocab controlado validado en CÓDIGO (CON-9/10); NULL = sin tag

    org_id          INTEGER NOT NULL,                       -- DM-1: org SIEMPRE (tenant boundary, CFG-4)
    project_id      INTEGER,                                -- DM-1: opcional
    repo_id         INTEGER,                                -- DM-1: opcional (si existe ⇒ project_id NOT NULL, DM-2)

    author_id       INTEGER NOT NULL DEFAULT -1,            -- autor original; sentinel -1 local, cloud remapea (ID-12)
    editor_id       INTEGER,                                -- último editor; NULL = nunca editado; ≠ author_id = curación admin (ID-10)

    lifecycle_state TEXT    NOT NULL                        -- sin DEFAULT: el código lo setea SIEMPRE explícito
                    CHECK (lifecycle_state IN ('draft','active','replaced','inactive')),  -- 'deleted' NO: en local es purga física (DM-13)

    created_at      INTEGER NOT NULL,                       -- epoch millis UTC (creación local)
    edited_at       INTEGER,                                -- NULL si nunca se editó (DM-15)
    -- upload_at: NO vive en el local (cloud-only, cursor de pull SYNC-7)

    -- Integridad de anclaje
    FOREIGN KEY (org_id)              REFERENCES org(id),                 -- org siempre (cubre la memoria org-only)
    FOREIGN KEY (org_id, project_id)  REFERENCES project(org_id, id),    -- proyecto en la org correcta (tenant)
    FOREIGN KEY (project_id, repo_id) REFERENCES repo(project_id, id),   -- repo en el PROYECTO correcto (⇒ org, transitivo)

    CHECK (repo_id IS NULL OR project_id IS NOT NULL)       -- DM-2: si hay repo, hay proyecto
);

CREATE INDEX idx_mem_scope     ON memories(org_id, project_id, repo_id);  -- filtros de anclaje/scope
CREATE INDEX idx_mem_topic     ON memories(topic);                        -- recall por topic (DM-9 / SRCH)
CREATE INDEX idx_mem_tag       ON memories(tag);                          -- facetado por tag (SRCH-4)
CREATE INDEX idx_mem_lifecycle ON memories(lifecycle_state);              -- active/draft por defecto (DM-14)
```

**Requiere en las tablas de anclaje (para las FK compuestas):** `project` suma `UNIQUE (org_id, id)`; `repo` suma
`UNIQUE (project_id, id)` (además del `UNIQUE (org_id, id)` que ya tenía para `repo_topology`).

**Decisiones y porqué:**
- **Identidad Opción A.** `local_id` entero (rowid estable del FTS, local-only, NO sincroniza) + `id` uuid v7 (identidad
  lógica). NO es el dual local/cloud de engram: nada lógico cuelga del entero. Ver sección `memories_fts`.
- **Anclaje por FK compuestas, no simples.** El tenant boundary (CFG-4) es límite duro:
  `(org_id, project_id)→project(org_id, id)` hace **irrepresentable cruzar org**. `(project_id, repo_id)→repo(project_id,
  id)` cierra además la **coherencia repo∈proyecto** (un repo no puede anclarse a un proyecto que no es el suyo). La FK
  simple `org_id→org` cubre la memoria **org-only** (cuando `project_id` es NULL las compuestas no se evalúan).
- **`lifecycle_state` sin DEFAULT, CHECK sin `deleted`.** El estado nunca es implícito. `deleted` es **cloud-only**
  (borrado lógico para auditoría, DM-16); en local dispara **purga física** (DM-13), por eso no es valor almacenable acá.
  El filtro `<> 'deleted'` del query FTS es defensa/no-op en local.
- **`tag` sin FK ni CHECK.** Se valida en código contra `tag_vocabulary` (CON-9): un tag retirado no rompe memorias
  viejas. `topic` sin entidad ni unicidad (DM-8): string libre que se reutiliza por búsqueda (DM-9).
- **`author_id` sentinel -1 / `editor_id` nullable.** Mismo patrón que `memory_relations` y `memory_versions` (ID-12).
- **`upload_at` fuera del local.** Cursor de pull asignado por el cloud al recibir (SYNC-7) — no vive en SQLite.

**Cloud (Postgres): BLOQUEADO** — depende del modelo de anclaje cloud (org/project/repo en Postgres, aún no escrito).
Forma futura: anclaje por clave estable (`repo.origin` RES-7, `project.sync_id`), `author_id`/`editor_id` remapeados al
id_dev real (ID-12), suma `upload_at TIMESTAMPTZ` y el estado `deleted` (borrado lógico para auditoría, DM-16).

---

### `memory_versions` — historial de ediciones (cerrada 2026-06-23, LOCAL/SQLite)

Tabla **append-only** keyed por `(memory_id, version)`. Guarda **cada edición** de una memoria (audit completo,
no solo el perdedor de LWW). Reemplaza el estado `conflict` (eliminado del lifecycle). `memories.content` y
`memories.title` son la **versión actual** (ganadora); `memory_versions` es el historial completo.

```sql
CREATE TABLE memory_versions (
    memory_id  TEXT    NOT NULL REFERENCES memories(id) ON DELETE CASCADE,
    version    INTEGER NOT NULL,                        -- número incremental dentro de ese uuid (1, 2, 3...)
    content    TEXT    NOT NULL,                        -- snapshot del contenido en esa versión
    title      TEXT    NOT NULL,                        -- snapshot del título
    author_id  INTEGER NOT NULL DEFAULT -1,             -- sentinel local; cloud remapea al id_dev real (ID-12)
    edited_at  INTEGER NOT NULL,                        -- epoch millis UTC
    edited_on  TEXT,                                    -- dispositivo (hostname o lo que mande el MCP)
    PRIMARY KEY (memory_id, version)
);
```

**Decisiones:**
- **Audit completo (no solo el perdedor).** Cada vez que una memoria se actualiza, la versión saliente se inserta
  acá antes del UPDATE. Si hubo 7 ediciones y ganó la 7ª, las 6 anteriores son recuperables. Más storage y ruido en
  sync, pero es el diseño más prolijo y da trazabilidad real.
- **`author_id` con sentinel `-1` local** (mismo patrón que `memory_relations`, ID-12). El cloud remapea al id_dev
  real al subir.
- **Sin `upload_at` en local** — mismo criterio que `memories` y `memory_relations`: el sello de recepción es
  cursor cloud-only (SYNC-7).
- **`ON DELETE CASCADE`** — si se purga físicamente una memoria (local, SYNC-13), todo su historial se va con ella.
- **Cloud (Postgres):** misma estructura, suma `upload_at TIMESTAMPTZ`. **BLOQUEADO** hasta tener el modelo de
  anclaje cloud.

---

### `memory_relations` (cerrada 2026-06-22)

**LOCAL (SQLite):**
```sql
CREATE TABLE memory_relations (
    source_id  TEXT    NOT NULL REFERENCES memories(id) ON DELETE CASCADE,  -- uuid v7 (id TEXT, coherente con memories local)
    target_id  TEXT    NOT NULL REFERENCES memories(id) ON DELETE CASCADE,
    relation   TEXT    NOT NULL CHECK (relation IN ('replaces','extends','caused','diverges')),
    author_id  INTEGER NOT NULL DEFAULT -1,            -- sentinel local (ID-12); el cloud remapea al id_dev real
    created_at INTEGER NOT NULL,                        -- epoch millis UTC (creación local)
    PRIMARY KEY (source_id, target_id, relation),
    CHECK (source_id <> target_id)
);
CREATE INDEX idx_memrel_target ON memory_relations(target_id);  -- traversal inverso (SRCH-5); el de salida (VIS-5) lo cubre la PK
```
> **Sin `upload_at` en el local** — es cursor cloud-only (SYNC-7), igual que en `memories`.

**CLOUD (PostgreSQL):**
```sql
CREATE TABLE memory_relations (
    source_id  UUID        NOT NULL REFERENCES memories(id) ON DELETE CASCADE,
    target_id  UUID        NOT NULL REFERENCES memories(id) ON DELETE CASCADE,
    relation   TEXT        NOT NULL CHECK (relation IN ('replaces','extends','caused','diverges')),
    author_id  BIGINT      NOT NULL REFERENCES users(id),   -- id_dev real (remapeado desde la key al subir, ID-12)
    created_at TIMESTAMPTZ NOT NULL,                        -- creación local, viaja con la arista
    upload_at  TIMESTAMPTZ NOT NULL DEFAULT now(),          -- sello del cloud al recibir = cursor de pull (SYNC-7)
    PRIMARY KEY (source_id, target_id, relation),
    CHECK (source_id <> target_id)
);
CREATE INDEX idx_memrel_target ON memory_relations(target_id);
```

**Decisiones y porqué:**
- **No se copia engram.** Su `memory_relations` es una tabla de **juicio de compatibilidad semántica** (LLM judge: `confidence`, `judgment_status`, `evidence`, `marked_by_model`, multi-actor sin UNIQUE). La nuestra modela **relaciones narrativas declaradas**, con la **validez resuelta por la DB** (VAL-1, `VAL-5` descartado). Casi ningún campo de engram aplica → de ~16 columnas a 6.
- **Sin `id` propio.** `(source_id, target_id, relation)` ya identifica unívocamente. Un surrogate solo serviría para darle al sync una `entity_key` de una columna (como el `sync_id` `rel-` de engram), pero eso es **uniformidad de la capa de sync, no identidad**. Se revisita SOLO si el diseño de la tabla de sync lo pide. Nada referencia a una arista (no versionamos aristas; la supersesión vive en la memoria, `lifecycle=replaced`).
- **`FK NOT NULL` en ambos extremos + `ON DELETE CASCADE`.** Habilitado porque `caused` exige que **ambas memorias existan** (DM-10, reescrito en commit `422f255`). Engram permite aristas colgantes (`target` NULL); nosotros las prohibimos → integridad referencial pura.
- **`upload_at` solo en el cloud** = sello al recibir y cursor de pull (SYNC-7), mismo patrón que `memories` (**no vive en el local**). **Sin `edited_at`:** una arista no se edita — "corregir" es borrar + crear (su identidad **es** la terna).
- **Caveat local/cloud:** el `CASCADE` protege solo el **LOCAL** (purga física, SYNC-13). En cloud el borrado es **lógico** (`lifecycle=deleted`, un `UPDATE`) → ocultar las aristas de una memoria `deleted` es **por query**, no por FK.

**Deuda que traslada a la skill (LOCAL-5, aún no escrita):** al sacar el judge de la DB, el juicio de *cuándo/cómo* nace cada arista recae entero en la skill de captura. Por relación: `replaces` (reconocer corrección/reversión; recall por anclaje+topic+tag; estructural ≠ semántico, VIS-10); `extends` (amplía sin invalidar; recall por topic); `caused` (causa **documentada** → arista, si no → al texto, CAP-2); `diverges` (norma org + desvío; recall org-level; **sin backstop**). Principio: **relación faltante > memoria faltante** (CAP-13); red de seguridad = dev en el visor (CAP-14/VIS-7).

---

### `tag_vocabulary` — whitelist de tags (cerrada 2026-06-24, LOCAL/SQLite)

Vocabulario controlado **universal**: el MISMO para todas las orgs (CON-8). Un único vocabulario canónico para todo el
producto — si cada org definiera el suyo, cada org necesitaría su propia skill y se rompería la fuente de verdad única.
**Vive en 2 artefactos** (CON-10, "skill + tabla, no 3"): la **skill de captura** (LOCAL-5) sabe *cuándo* usar cada tag;
esta **tabla** valida que no se ingrese cualquier cosa. **LOCAL-only**, no sincroniza, **sin `org_id`**, **sin FK**
(`memories.tag` es varchar libre, CON-9).

```sql
CREATE TABLE tag_vocabulary (
    tag TEXT PRIMARY KEY COLLATE NOCASE   -- vocabulario controlado (CON-8); valores canónicos en MAYÚSCULA
);

-- Seed inicial = vocabulario v1 (CON-8), valores canónicos en MAYÚSCULA. NO es algo que se llene en runtime:
-- son valores iniciales que vienen con la skill. `NO_CLASIFICADO` NO va acá (sentinel del sistema, CON-11).
INSERT INTO tag_vocabulary (tag) VALUES
    ('SPEC'),           -- Especificación de requerimientos de una funcionalidad
    ('DESIGN'),         -- Diseño técnico y decisiones de arquitectura
    ('VERIFICATION'),   -- Verificación, pruebas y validación del trabajo realizado
    ('DECISION'),       -- Decisión puntual sobre tecnología, enfoque o diseño
    ('DISCOVERY'),      -- Algo aprendido o descubierto en el código base, no obvio
    ('BUGFIX'),         -- Bug encontrado y resuelto, con causa raíz documentada
    ('PATTERN'),        -- Patrón o convención establecida en el proyecto
    ('RETROSPECTIVE'),  -- Aprendizaje post-entrega o lección aprendida
    ('INCIDENT'),       -- Falla en producción con análisis, causa raíz y resolución
    ('CONFIG'),         -- Setup de entorno, variables y dependencias necesarias
    ('PREFERENCE');     -- Inclinación o preferencia del equipo, sin ser una decisión cerrada
```

**Flujo de actualización (manual, sin migración, CON-9):** se mete una actualización → se tocan LOS DOS artefactos
juntos: (1) se agregan los tags nuevos a esta tabla (re-seed); (2) se actualiza la skill para que sepa usarlos. **Sin
versión trackeada en DB:** un "local con la tabla vieja" (CON-11) es un local con el seed/skill desactualizado.

**Validación al escribir (MCP o API):** el escritor consulta esta tabla con el tag que mandó el modelo —
- **existe** → se guarda;
- **no existe** (typo / local desactualizado) → se guarda con el default **`no_clasificado`** (CON-11);
- el guardado **NUNCA falla por el tag** — es metadata best-effort.

`no_clasificado` **NO está en esta tabla**: es un sentinel diagnóstico (como el `-1` del autor, ID-12), no un tag real.
Si abundan en `memories`, hay locales con el seed viejo (señal de CON-11).

**Decisiones y porqué:**
- **`TEXT`, NO enum de DB (CON-9).** Agregar/quitar un tag = re-seed + actualizar skill, **sin migración**. Una memoria
  vieja con un tag retirado **no se rompe** (es solo un string). Puerta abierta a vocabulario dinámico a futuro sin tocar
  el schema.
- **Sin FK desde `memories.tag`.** Si fuera FK, retirar un tag rompería/bloquearía memorias viejas — justo lo que CON-9
  prohíbe. La tabla valida en **código**, no por constraint de DB.
- **Canónico en MAYÚSCULA + `COLLATE NOCASE`.** El valor almacenado/canónico es UPPER (`DESIGN`, no `design`). El
  código normaliza el input a mayúscula y guarda el canónico de la tabla (no el string crudo del modelo) → el facetado
  por tag (SRCH-4) no se fragmenta por casing. NOCASE es la red de seguridad en el schema por si el código no normaliza.
- **Universal, sin `org_id`.** Decisión de **producto**: un vocabulario único para todas las orgs (una sola skill
  canónica). No es configurable por tenant.
- **Sin equivalente en engram.** Engram solo tiene `topic_key`, no maneja tags — invención del proyecto.

**Cloud (Postgres): N/A.** El vocabulario es universal y vive en la skill + la tabla local; la validación ocurre en el
escritor local/MCP, no en el cloud. Nada que sincronizar.

---

### `memories_fts` — búsqueda full-text (cerrada 2026-06-23, LOCAL/SQLite, **Opción A**)

FTS5 **external-content**: el índice **NO guarda copia del texto**; lo lee de `memories` vía `content_rowid`.
**Índice LOCAL-only y DERIVADO** — nunca sincroniza (un índice no es dato: se reconstruye, no se replica).

**Impone en `memories`** (decisión de identidad, Opción A): rowid entero estable + uuid portable, separados.
```sql
-- memories (fragmento que la FTS EXIGE; el resto de la tabla sigue 🟡 pendiente):
--   local_id INTEGER PRIMARY KEY   -- rowid estable (inmune a VACUUM); es el content_rowid del FTS; LOCAL-only, NO sincroniza
--   id       TEXT NOT NULL UNIQUE  -- uuid v7, identidad lógica/portable; de acá cuelgan memory_relations y memory_versions
```
> El **uuid** sigue siendo la identidad referenciada (las FK de relations/versions apuntan a `memories(id)`, ahora columna
> UNIQUE). El **entero** no lo referencia nada lógico — solo el índice. **No** es el dual-id prohibido en `memory`.

```sql
CREATE VIRTUAL TABLE memories_fts USING fts5(
    title,
    content,
    content='memories',           -- external content: lee el texto de la fila real, no lo duplica
    content_rowid='local_id'      -- puente al entero estable
);

-- Triggers: external-content NO se autoindexa; hay que alimentarlo a mano.
CREATE TRIGGER memories_fts_ai AFTER INSERT ON memories BEGIN
  INSERT INTO memories_fts(rowid, title, content)
  VALUES (new.local_id, new.title, new.content);
END;

CREATE TRIGGER memories_fts_ad AFTER DELETE ON memories BEGIN              -- solo purga física local (SYNC-13)
  INSERT INTO memories_fts(memories_fts, rowid, title, content)           -- delete necesita los valores VIEJOS
  VALUES ('delete', old.local_id, old.title, old.content);
END;

CREATE TRIGGER memories_fts_au AFTER UPDATE OF title, content ON memories BEGIN   -- LWW: la versión ganadora re-indexa
  INSERT INTO memories_fts(memories_fts, rowid, title, content)
  VALUES ('delete', old.local_id, old.title, old.content);
  INSERT INTO memories_fts(rowid, title, content)
  VALUES (new.local_id, new.title, new.content);
END;
```

**Patrón de búsqueda (filtros por JOIN de vuelta, NO en el índice):**
```sql
SELECT m.id, m.title, bm25(memories_fts, 5.0, 1.0) AS rank   -- title pesa 5x sobre content (tunable)
FROM memories_fts
JOIN memories m ON m.local_id = memories_fts.rowid
WHERE memories_fts MATCH :q
  AND m.lifecycle_state <> 'deleted'    -- soft-delete filtrado acá: gratis, el índice ni se entera
  AND m.project_id = :project           -- scope/anclaje/tag/topic: todos por JOIN, valor SIEMPRE fresco
ORDER BY rank                           -- bm25 da negativo; más negativo = más relevante
LIMIT :n;
```

**Decisiones y porqué:**
- **Solo `title` + `content` en el índice.** Todo lo estructural (scope, anclaje org/project/repo, tag, topic, lifecycle)
  se filtra por **JOIN de vuelta a `memories`** con el valor fresco. Beneficio central de Opción A: un cambio de metadata
  (p.ej. `active→replaced`) **NO toca el FTS** — el trigger de update dispara solo con `OF title, content`.
- **Soft-delete = gratis.** La memoria `deleted` (lógico, cloud) sigue en el índice; el visor la oculta con
  `lifecycle_state <> 'deleted'` en el JOIN. Solo la **purga física local** (SYNC-13) dispara el delete trigger.
- **LWW sin tablas extra para el índice.** Gane quien gane, el `UPDATE` final de `content`/`title` re-indexa.
  `memory_versions` (append-only) **NO se indexa** — el índice solo refleja la versión vigente.
- **Triggers external-content quisquillosos (costo de una vez).** El delete/update necesita los valores **viejos**
  (`'delete', old.local_id, old.title, old.content`). Mal escritos → corrupción silenciosa del índice. Reconstrucción de
  emergencia: `INSERT INTO memories_fts(memories_fts) VALUES('rebuild');`.
- **BM25 con `title` ponderado** (arranque 5x; tunable). El piso de rank tipo engram (descartar matches > -2.0) es
  **filtro de query**, no de schema.

**Cloud (Postgres): N/A.** El índice es **local-only y derivado**. Postgres no tiene FTS5; si el cloud necesitara
búsqueda usaría `tsvector`+GIN construido de SUS datos — pero **VIS-1** manda al visor a leer del local, así que el cloud
probablemente no indexa texto. **Nada que sincronizar.**

---

## Referencia — schema real de engram

> Clonado de `https://github.com/Gentleman-Programming/engram.git` (HEAD 2026-06-22). Verbatim donde fue posible.
> Archivos: `internal/store/store.go` (SQLite local, `migrate()`), `internal/store/relations.go`,
> `internal/cloud/cloudstore/cloudstore.go` (Postgres cloud, `migrate()`), `internal/cloud/cloudstore/audit_log.go`.

### Local (SQLite, `~/.engram/engram.db`)

**`sessions`** — `id TEXT PK`, `project TEXT NOT NULL`, `directory TEXT NOT NULL`, `started_at TEXT DEFAULT datetime('now')`, `ended_at TEXT`, `summary TEXT`.

**`observations`** (≈ nuestra `memories`):
```
id INTEGER PK AUTOINCREMENT          -- rowid local (joins rápidos)
sync_id TEXT                         -- 'obs-' + 16hex, UNIQUE, portable entre máquinas
session_id TEXT NOT NULL             -- FK sessions(id)
type TEXT NOT NULL
title TEXT NOT NULL
content TEXT NOT NULL
tool_name TEXT
project TEXT
scope TEXT NOT NULL DEFAULT 'project' -- project | personal
topic_key TEXT
normalized_hash TEXT
revision_count INTEGER DEFAULT 1
duplicate_count INTEGER DEFAULT 1
last_seen_at TEXT
pinned BOOLEAN DEFAULT 0
review_after TEXT                    -- decay por tipo: decision +6m, policy +12m, preference +3m
expires_at TEXT                      -- reservado, NULL en fase 1
embedding BLOB / embedding_model TEXT / embedding_created_at TEXT  -- reservado
created_at / updated_at TEXT DEFAULT datetime('now')
deleted_at TEXT                      -- soft delete
```
Índices: por session, type, project, created DESC, scope, sync_id, `(topic_key,project,scope,updated_at DESC)`, deleted, dedupe `(normalized_hash,project,scope,type,title,created_at DESC)`.

**`observations_fts`** (FTS5 external-content): `title, content, tool_name, type, project, topic_key` con `content='observations'`, `content_rowid='id'` + triggers insert/delete/update. BM25 vía `fts.rank` (floor -2.0).

**`user_prompts`** — `id INTEGER PK`, `sync_id TEXT` ('prompt-'+16hex), `session_id FK`, `content`, `project`, `created_at`. FTS `prompts_fts`. Tombstones en `prompt_tombstones (sync_id PK, session_id, project, deleted_at)`.

**`memory_relations`** (≈ nuestras aristas):
```
id INTEGER PK AUTOINCREMENT
sync_id TEXT NOT NULL UNIQUE          -- 'rel-' + 16hex
source_id TEXT / target_id TEXT       -- sync_id de las observations (portable)
relation TEXT DEFAULT 'pending'       -- pending|related|compatible|scoped|conflicts_with|supersedes|not_conflict
reason / evidence TEXT
confidence REAL
judgment_status TEXT DEFAULT 'pending' -- pending|judged|orphaned|ignored
marked_by_actor / marked_by_kind (agent|human|system) / marked_by_model TEXT
session_id TEXT
superseded_at TEXT
superseded_by_relation_id INTEGER REFERENCES memory_relations(id) ON DELETE SET NULL
created_at / updated_at TEXT
-- SIN UNIQUE en (source_id, target_id): desacuerdo multi-actor es objetivo de diseño
```

**Sync local:** `sync_state` (target_key PK, lifecycle idle|pending|running|healthy|degraded, **3 cursores** `last_enqueued_seq / last_acked_seq / last_pulled_seq`, backoff, lease, last_error). `sync_mutations` (outbox FIFO: seq PK, entity session|observation|prompt|relation, op upsert|delete, payload JSON). `sync_chunks`, `sync_enrolled_projects`, `sync_apply_deferred`, `cloud_upgrade_state`.

### Cloud (PostgreSQL)

**`cloud_users`** — `id BIGSERIAL PK`, `username TEXT UNIQUE`, `email TEXT UNIQUE`, `password_hash TEXT DEFAULT ''`, `created_at TIMESTAMPTZ`.

**`cloud_chunks`** — blobs JSONB inmutables content-addressed: `chunk_id = SHA256(payload)`, verificado al escribir. `(project_name, chunk_id)` UNIQUE + contadores.

**`cloud_project_sessions`** (dedup), **`cloud_project_controls`** (sync_enabled, paused_reason), **`cloud_mutations`** (journal fino: seq BIGSERIAL, entity, op, payload JSONB), **`cloud_sync_audit_log`** (occurred_at, contributor, project, action, outcome, reason_code, metadata JSONB).

### Observaciones de diseño de engram

1. **Dual id:** INTEGER PK local + `sync_id` TEXT con prefijo tipado; relaciones referencian por `sync_id` (portable). → nosotros: **single UUID v7**.
2. **Sin org/proyecto/repo:** `project` es string plano, membership implícita, **cero control de acceso en la capa de datos**. → nuestro diferencial: anclaje estructurado + tenant.
3. **Auth solo en cloud:** local sin tabla de users/auth; la key es config. → coincide con ID-2/ID-5.
4. **Sync:** outbox local (seq) + 3 cursores; cloud recibe en dos rieles (chunks content-addressed para bootstrap + mutations para incremental).
5. **Relaciones:** grafo dirigido sin UNIQUE en el par (desacuerdo multi-actor first-class); upsert lo hace la app.
6. **Tags/topics:** sin tags; solo `topic_key` TEXT con upsert por `(topic_key, project, scope)`.
7. **Decay:** `review_after` por tipo; estado active/needs_review computado, no almacenado.
8. **FTS5+BM25:** primitiva de similitud antes de la capa opcional de juicio LLM.
9. **Soft delete:** `deleted_at` en observations; tombstones aparte para prompts; hard delete por flag en payload.
10. **Scope:** `project` | `personal` (cross-project privado), enforced en query, no por FK.
