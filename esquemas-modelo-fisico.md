# Esquemas / Modelo físico — estado y reanudación

> **Última actualización:** 2026-06-22. **Doc vivo.** Punto de reanudación para definir el **DDL** del proyecto
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
| `memories` | ✅ aprobada (concepto) | `observations` | **single UUID v7** vs **dual id** de engram (INTEGER PK + `sync_id` TEXT); suma `org/proyecto/repo` (engram: `project` string plano); `tag` controlado + `topic` (engram: solo `topic_key`, **sin tags**); `lifecycle_state` explícito (engram: `deleted_at` soft + estado computado) |
| `users` + `api_keys` + `project_admin_grants` | ✅ aprobada (org/credenciales) | `cloud_users` | **3 credenciales separadas** (key de sync / password del visor ID-11 / login cloud ADM-6); engram tiene solo `cloud_users` y la key es config, no vive en DB |
| `memory_relations` | ❌ falta | `memory_relations` | vocab `replaces/extends/caused/diverges` + **validez resuelta por la DB** (VAL-1); engram usa `related/compatible/supersedes/...` + `judgment_status` + `confidence`, multi-actor **sin UNIQUE** en `(source,target)` |
| `organizations` (tenant boundary) | ❌ falta | **no existe** | engram no tiene tenant/org; acá org = aislamiento duro (CFG-4, NFR-11) |
| `projects` + `repos` + `repo_topology` + mapping `origin→proyecto` | ❌ falta | **no existe** | engram **adivina por nombre de carpeta** (RES-4, criticado); acá mapping explícito por `origin` (RES-7/RES-8/ADM-11) |
| `tag_vocabulary` (normalización local) | ❌ falta | **no existe** | engram no tiene tags; whitelist derivada de la skill (CON-8/CON-10) |
| sync: `sync_state` + outbox + cursors + audit | ❌ falta | `sync_state`, `sync_mutations`, `cloud_mutations`, `cloud_sync_audit_log` | engram lo tiene **muy maduro** — **adoptar el patrón** (3 cursores: enqueued/acked/pulled + outbox FIFO) |
| `memories_fts` (FTS5) | ❌ falta definir | `observations_fts` | mismo patrón FTS5+BM25 (prd:583) — copiable casi directo |

## Orden de trabajo propuesto (por dependencias de FK)

1. **Estructura de anclaje** — `organizations → projects → repos → repo_topology → mapping origin→proyecto`.
   Es la base de la que cuelgan las FK de `memories`, y es donde engram **no tiene nada** (máximo diferencial).
2. **`memories`** — escribir el DDL real con las FK de anclaje (concepto ya aprobado, DM-15).
3. **`memory_relations`** — aristas dirigidas (DM-10) + validez por DB (VAL-1).
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
- **Pendientes abiertos (del commit de hoy):** reconciliación proyecto-local provisional ↔ mapping org (cerca de RES-8);
  fallback local opcional de credencial cifrada at-rest (no hasheada) para entornos sin keychain (CFG-6/LOCAL-8). `prd:934-950`

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
