# Esquemas / Modelo físico — estado y reanudación

> **Última actualización:** 2026-06-23. **Doc vivo.** Punto de reanudación para definir el **DDL** del proyecto
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
| `memories` | 🟡 concepto (falta #1554) | `observations` | **single UUID v7** (id `TEXT` en SQLite) vs **dual id** de engram (INTEGER PK + `sync_id` TEXT); suma `org/proyecto/repo` (engram: `project` string plano); `tag` controlado + `topic` (engram: solo `topic_key`, **sin tags**); `lifecycle_state` explícito. **⚠️ incorporar #1554:** lifecycle SIN `conflict` (LWW) |
| `memory_versions` (perdedor LWW, #1554) | ❌ falta | **no existe** | append-only keyed por uuid; guarda el perdedor de LWW recuperable (reemplaza la UI de `conflict`, ahora eliminado) |
| `users` + `api_keys` + `project_admin_grants` | ✅ aprobada (org/credenciales) | `cloud_users` | **3 credenciales separadas** (key de sync / password del visor ID-11 / login cloud ADM-6); engram tiene solo `cloud_users` y la key es config, no vive en DB |
| `memory_relations` | ✅ **cerrada** (DDL abajo) | `memory_relations` | vocab `replaces/extends/caused/diverges` + **validez resuelta por la DB** (VAL-1); engram usa `related/compatible/supersedes/...` + `judgment_status` + `confidence`, multi-actor **sin UNIQUE** en `(source,target)` |
| `org` (tenant boundary) | ✅ **cerrada** (DDL abajo) | **no existe** | engram no tiene tenant/org; acá org = aislamiento duro (CFG-4, NFR-11). **LOCAL-only, sin token** (secreto en keychain del OS, CFG-6) |
| `project` + `repo` | ✅ **cerrada** (DDL abajo) | **no existe** | id local + clave estable (mecanismo 1): `repo.origin` (RES-7), `project.sync_id` cloud-assigned; engram **adivina por carpeta** (RES-4, criticado) |
| `repo_topology` + mapping `origin→proyecto` | ❌ falta (topología = **próximo tema**) | **no existe** | aristas dirigidas repo→repo (DM-5) + mapping explícito (RES-8/ADM-11) |
| `tag_vocabulary` (normalización local) | ❌ falta | **no existe** | engram no tiene tags; whitelist derivada de la skill (CON-8/CON-10) |
| sync: `sync_state` + outbox + cursors + audit | ❌ falta | `sync_state`, `sync_mutations`, `cloud_mutations`, `cloud_sync_audit_log` | engram lo tiene **muy maduro** — **adoptar el patrón** (3 cursores: enqueued/acked/pulled + outbox FIFO) |
| `memories_fts` (FTS5) | ❌ falta definir | `observations_fts` | mismo patrón FTS5+BM25 (prd:583) — copiable casi directo |

## Orden de trabajo propuesto (por dependencias de FK)

1. **Estructura de anclaje** — ✅ `org → project → repo` **cerradas** (DDL abajo). Falta `repo_topology` +
   mapping `origin→proyecto` (**próximo tema**). Es la base de la que cuelgan las FK de `memories`, y es donde
   engram **no tiene nada** (máximo diferencial).
2. **`memories`** — escribir el DDL real con las FK de anclaje (concepto aprobado, DM-15). ⚠️ **Antes incorporar
   #1554** (LWW / eliminar `conflict` / tabla `memory_versions`) — ver "Decisiones ya tomadas".
3. ~~**`memory_relations`** — aristas dirigidas (DM-10) + validez por DB (VAL-1).~~ ✅ **cerrada** (ver DDL abajo).
4. **`tag_vocabulary`** — normalización local derivada de la skill (CON-10).
5. **Sync** — `sync_state` + outbox + cursors + audit (reusar patrón de engram).
6. **`memories_fts`** — virtual table FTS5 + triggers (copiar patrón de engram).

> Nota: lo aprobado de "organización" suena a **users/keys/grants** (auth), distinto de la **estructura de anclaje**
> (org/proyecto/repo/topología). Confirmar al reanudar qué cubrió exactamente la aprobación previa.

## Decisiones de modelo ya tomadas (del PRD)

- **Identidad:** UUID v7 único (no dual id local/cloud como engram). Unicidad por validación server-side; colisión
  accidental se resuelve re-asignando el id (SYNC-14), no mergeando. `prd:DM-15, 46, 671`
- **Búsqueda:** FTS5 + BM25 en SQLite local, igual que engram. `prd:583`
- **Resolución repo→proyecto:** mapping org-level explícito keyed por `origin` de git; sin heurística por carpeta. `prd:RES-7/RES-8`
- **Credenciales:** tres separadas — key de sync (online, ID-2/ID-8) / password del visor local (ID-11) / login cloud admin (ADM-6).
  Tablas online que nunca bajan a local: `users`, `project_admin_grants`, tabla de keys. `prd:ID-2/ID-5, ADM-6`
- **Tags:** vocabulario controlado (CON-8); storage varchar; validación contra tabla de normalización local materializada de la skill (CON-10).
- **Conflicto de edición (LWW) — hilo #1554 (engram, pendiente de bajar al PRD):** dos ediciones de la MISMA
  memoria (mismo uuid, multi-device del mismo autor) se resuelven **last-write-wins** por timestamp, **NO** con UI
  de reconciliación. El perdedor queda **recuperable** como snapshot → **se ELIMINA el estado `conflict`**. Falta
  una tabla **`memory_versions` append-only keyed por uuid** (fork abierto: solo el perdedor de LWW vs cada edición
  = audit completo; recomendado: **mínimo**). **Esto cambia el DDL de `memories`** (lifecycle sin `conflict`) y
  agrega `memory_versions`. Barrido pendiente en el PRD: DM-12/13/14, SYNC-6/9/12, VIS-9, SRCH-6.
- **Anclaje (org/project/repo) — cerrado, mecanismo 1:** `memory` guarda **FK enteros LOCALES**; el sync traduce a
  **clave estable** (`repo.origin` RES-7 / `project.sync_id` cloud / org implícita). Los ids locales nunca viajan.
  El **doble-id** (local int + sync_id) es OK en project/repo —nada cuelga de su id— pero **NO** en `memory`
  (las `memory_relations` cuelgan del uuid) → single UUID. `org` es **LOCAL-only** y **sin token** (secreto en el
  keychain del OS, CFG-6). Detalle en engram, topic `prd/data-model/anchor-tables`.
- **Pendientes abiertos (del commit de hoy):** reconciliación proyecto-local provisional ↔ mapping org (cerca de RES-8);
  fallback local opcional de credencial cifrada at-rest (no hasheada) para entornos sin keychain (CFG-6/LOCAL-8). `prd:934-950`

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
  sync_id TEXT                          -- id estable del cloud; NULL = provisional/LOCAL (lo crea el admin, ADM-11)
);

CREATE TABLE repo (
  id         INTEGER PRIMARY KEY,
  org_id     INTEGER NOT NULL REFERENCES org(id),        -- denormalizado (habilita el UNIQUE y el lookup origin→proyecto)
  project_id INTEGER NOT NULL REFERENCES project(id),    -- DM-2: no hay repo sin proyecto
  origin     TEXT    NOT NULL,                           -- git origin = identidad estable (RES-7)
  role       TEXT,                                       -- back/front/service... varchar, no enum (≈CON-9)
  UNIQUE (org_id, origin)                                -- un origin = un repo por org (RES-7/ADM-11)
);
```

**Mecanismo 1 (id local + clave estable):** `memory` referencia el anclaje por **FK entero local** (joins compactos
para scope/topología); el **sync traduce** a clave estable (`repo.origin`, `project.sync_id`, org implícita). Los ids
locales nunca cruzan el wire → la dualidad de ids no genera conflicto. Falta `repo_topology` (próximo tema).

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
