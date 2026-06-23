# PRD — Memoria de equipo (contexto de proyectos con IA)

> **Estado:** 6 gaps de refinamiento cerrados — listo para diseño. Última sesión: 2026-06-23 (incorporado #1554: LWW + `memory_versions`, sin estado `conflict`).
> **Sesión 2026-06-16 mañana:** Búsqueda, Sync, No-funcionales.
> **Sesión 2026-06-16 tarde:** Modelo de datos — entidad Memory completa, topic como string, tag como valor único,
> lista preferente de tags (v1), relaciones entre memorias, lifecycle de memorias,
> SLA de búsqueda unscoped cualificado.
> **Sesión 2026-06-17:** Correcciones de consistencia — `supersede` → `replaces` (VAL-2, VAL-3, CAP-3);
> conflicto de sync: lifecycle_state=conflict + relación `caused` automática (SYNC-9, DM-12) — *[revisado 2026-06-19: el `caused` se removió, ver abajo]*;
> topics fuera del delta de sync, derivados localmente (SYNC-5);
> 4ta relación `diverges` para desviaciones de norma org (DM-10, SEM-6).
> **Sesión 2026-06-18:** Borrado de memorias — exclusivo del admin vía cloud (DM-16); borrado **lógico** en cloud
> (retención + auditoría) y **purga física** en el local (SYNC-13); el dev pierde "borrar" (CAP-3);
> `deleted` es estado **cloud-only** (DM-13, DM-14, SRCH-6); al purgar una memoria, el local elimina también
> las relaciones que la apuntaban.
> Identidad/sync (gap #4 cerrado): MCP no autentica → escritura local siempre libre; **autorización server-side**
> contra la key (NFR-5); key inválida ≠ revocada (SYNC-10b vs SYNC-10); flujo de purga por offboarding con guardia
> de idempotencia `purge_executed_at` + confirmación al cloud (SYNC-10). Nivel 2: la **escritura org-level es
> cloud-only** (UI admin + login cloud, SYNC-3b); los locales nunca tienen key org-write; las memorias org igual
> bajan por fan-out y viven en local offline.
> Editar vs. reemplazar (gap #3 cerrado): editar = in-place para fidelidad/completar; `replaces` = supersesión;
> criterio "¿la vieja alguna vez fue verdad?" vive en la skill (CAP-12). Autorización (ID-10): el dev edita solo
> lo suyo, el admin edita cualquiera cloud-side; se registra `editor_id`/`edited_at` (DM-15). SYNC-9 acotado a
> mismo-autor multi-device.
> Creación de relaciones (gap #6 cerrado): autoría best-effort del agente en captura, **relación faltante > memoria
> faltante** — nunca se bloquea el guardado (CAP-13); el juicio + recall del target vive en la skill (CAP-14, LOCAL-5);
> backstop de `replaces` vía posibles duplicados (VIS-10) *[re-apuntado 2026-06-19; antes vía conflicto DM-12]*,
> `diverges` sin backstop (juicio puro); UI local edita relaciones propias.
> Identidad/login local (gap #5 cerrado): first-run del visor pide **nombre + contraseña** (ID-11) — contraseña = gate
> del visor, nombre = identidad local; el MCP no usa login. `author_id` local = sentinel reservado **`-1`** (no `0` ni
> `null`); el cloud reescribe al `id_dev` real desde la key al subir (ID-12); "propia" = `-1` o mi `id_dev` remapeado.
> **Sesión 2026-06-19:** Modelo de conflictos vs. identidad (avance gap #1 visor) — separación de **tres casos**
> que estaban mezclados: (1) **conflicto de edición** = mismo `id`, misma memoria → reconciliar contenido, **sin
> `caused`** (DM-12, SYNC-9, VIS-9); el `caused` automático se removió porque relacionaba una memoria consigo misma,
> violando DM-10. (2) **Redundancia** = `id` distinto, mismo anclaje+topic+tag → supersesión vía `replaces`, ofrecida
> como **posibles duplicados** en el visor (CAP-13 re-apuntado, VIS-10), no como conflicto. (3) **Colisión accidental
> de `id`** = mismo `id`, memorias distintas → problema de identidad, el cloud re-asigna el UUID discriminando por
> `CREATE` vs `UPDATE` (SYNC-14, nota en DM-15). Se mantuvo **single UUID v7** (no doble id local/cloud): la unicidad
> se garantiza con validación server-side, sin reescribir aristas en cada subida.
> **Estados pasivos del visor (VIS-11):** último sync (NFR-9), read-only vs editable (ID-10/12), draft editable con
> promoción **unidireccional** `draft → active` (el agente abre uno nuevo si se promueve a mitad de sesión).
> **Sesión 2026-06-23:** Incorporado #1554 al PRD — **LWW automático para conflictos de edición**, eliminado el estado
> `conflict` del lifecycle (DM-12, DM-13, DM-14, DM-15). Agregada tabla **`memory_versions`** con historial completo
> de ediciones (audit trail, DM-12). SYNC-9 y SYNC-12 pasan a resolución automática sin intervención del dev.
> VIS-9 cambia de "resolución de conflictos" a "historial de ediciones" (consulta, no resuelve). Barrido completo:
> DM-12/13/14/15, SYNC-6/9/12, SRCH-6, VIS-6, VIS-9, CAP-13.
> **Principio de uso del visor:** primariamente para visualizar; el agente es el canal de trabajo; curación mínima
> en el visor. **Re-anclaje por admin (VIS-8):** puede re-anclar memorias ajenas **sin límite de fecha**, cloud-side;
> el cruce de org sigue imposible (estructural). **→ Gap #1 cerrado: los 6 gaps del PRD quedan completos.**
> **Resuelto:** Orden del ciclo sync — pull primero (SYNC-11).

---

## Resumen

Sistema de **memoria de equipo** que captura automáticamente el *por qué* y el *cómo*
detrás de los cambios de un proyecto, y lo deja recuperable para que el próximo dev (o su IA)
entienda las decisiones sin tener que estar quien las tomó.

Consumo en tres alturas: **repo → proyecto → organización**.

---

## Decisiones de fondo (cerradas)

- **Captura:** automática vía IA (el agente guarda mientras el dev trabaja). Silenciosa.
- **Camino arquitectónico:** proyecto **nuevo** (no se construye sobre engram ni se forkea).
  Se adopta de engram el *patrón* (interop MCP, sync local/cloud), no el código.
- **Motivo del proyecto nuevo:** el concepto "proyecto = N repos con topología" no entra en el
  modelo plano de engram. Owning el modelo de datos es el diferencial.
- **Relación con engram:** engram es memoria general agent-agnostic (no es SDD). Sirve de
  inspiración y precedente probado, no de base de código.

---

## 1. Visión y alcance

**Problema.** El *por qué* detrás de los cambios vive en la cabeza de quien los hizo. Cuando otro
dev toca ese código no entiende qué decisiones lo moldearon, y rompe cosas o reinventa la rueda.

**Objetivo.** Capturar el porqué/cómo **automáticamente** y dejarlo **recuperable** en tres
alturas: repo, proyecto y organización.

**Casos de uso que justifican el proyecto:**
- **(a) Estandarizar (org-wide):** "¿cómo hicimos login en *cualquier* proyecto del equipo?"
- **(b) Auditar con IA:** la IA, mientras analiza código, cruza contra la memoria
  ("esto está hardcodeado… ¿por qué? → fue a propósito por X → aviso al dev").
- **(c) Búsqueda humana on-demand:** el dev en GitHub ve algo raro y va a la UI a buscar si hay
  algo que lo justifique.

**No-objetivos (v1):**
- `NG-1` — No es editor inline / git blame (descartado explícitamente).
- `NG-2` — No reescribe el motor de captura/storage de engram.
- `NG-3` — No modela subconjuntos arbitrarios de repos (el "exactamente repo A y B"); se difiere.

---

## 2-3. Modelo de datos

```
Organización
└── Proyecto (lógico = agrupa N repos)
    ├── Repo (rol: back / front / servicio…)
    │   └── Memorias
    ├── Memorias de proyecto (repo = null)
    └── Topología: aristas dirigidas entre repos (ej: bot-back → sirve a → mobile-front)
```

- `DM-1` (MUST) — Anclaje de toda memoria: **Org obligatoria, Proyecto opcional, Repo opcional.**
- `DM-2` (MUST) — Integridad: si hay repo, debe haber proyecto. No existe repo sin proyecto.
- `DM-3` (MUST) — Tres anclajes válidos:
  - `org` → política de toda la empresa.
  - `org + proyecto` (repo null) → decisión transversal del proyecto.
  - `org + proyecto + repo` → lo concreto (el 90%).
- `DM-4` (MUST) — Un proyecto agrupa N repos, cada uno con un **rol**.
- `DM-5` (MUST) — **Topología:** aristas dirigidas entre repos de un proyecto. Dato de primera
  clase, consultable.
- `DM-6` (SHOULD) — Repos y aristas llevan metadata mínima (rol, tipo de relación).
- `DM-7` (MUST) — Una memoria se describe por **tres ejes ortogonales** (independientes entre sí):
  1. **Anclaje** (lineal — *dónde vive*): `org → ?proyecto → ?repo` (DM-1/2/3).
  2. **Topic** (transversal — *a qué hilo pertenece*): string libre. Puede cruzar repos y repetirse entre memorias. Nunca se renombra.
  3. **Tag** (faceta única): **1 solo valor** del vocabulario controlado (CON-8). Cuando cambia el tag, es una memoria distinta (pueden compartir topic).
  > Ejemplo: memoria anclada en `org/proyecto/repo:back`, topic `agregar-login`, tag `design`.
  > Una feature que toca back+front = dos memorias con distinto anclaje que **comparten el topic**.
  > Dos verificaciones del mismo feature = dos memorias con mismo anclaje, misma topic, mismo tag — la segunda tiene relación `replaces` sobre la primera.
- `DM-7b` (MUST) — El anclaje completo `org + proyecto + repo + topic + tag` **NO es una clave única.** Pueden existir múltiples memorias con el mismo anclaje (ej: dos verificaciones).
- `DM-8` (MUST) — **Topic = string en la entidad Memory, NO entidad separada.** Se puede repetir entre memorias. Nunca se renombra. Una memoria org-only puede no tener topic. La tabla de normalización es la de **tags** (CON-8/CON-10), no la de topics.
  > Decisión: no se necesita renombrar topics ni metadata propia → el costo de entidad sin beneficio. El topic viaja con la memoria en el sync sin necesitar sincronización separada.
- `DM-9` (MUST — anti-fragmentación) — El flujo de captura **muestra a la IA los topics existentes del
  proyecto para que reutilice** antes de crear uno nuevo (evita `formulario-datos` vs `form-datos`).
  Se resuelve vía búsqueda por string, no por entidad. La curación post-hoc (CAP-3) puede editar el campo topic en memorias duplicadas.
- `DM-10` (MUST) — **Relaciones dirigidas entre memorias** = aristas de primera clase. Set v1:
  - **`replaces`** — B reemplaza a A. A queda en estado `replaced` (historial recuperable). Cubre correcciones, revisiones y reversiones (el motivo va en el texto de la memoria).
  - **`extends`** — B agrega información a A sin invalidarla. Ambas válidas; recall puede encadenarlas.
  - **`caused`** — A originó a B (ej: una decisión de diseño causó un bug). **Ambas memorias deben existir** (como toda arista: borrar un nodo elimina sus relaciones, DM-16/SYNC-13). Si la causa **no está documentada** como memoria, va en el **texto** de B (causa raíz, CAP-2), **no** como arista.
  - **`diverges`** — B (proyecto/repo) se sale de la norma establecida por A (org) en ese contexto específico. A sigue `active` y vigente para el resto. El motivo va en el texto de B (SEM-6).
- `DM-11` (MUST — principio) — Una relación existe **solo si agrega información que el anclaje no provee por sí solo.** No todas las memorias necesitan estar relacionadas.
  Se descartan: `conflicts_with` (si dos memorias se contradicen, una reemplaza a la otra), `related` (sin caso concreto que topic+tag no cubra → ruido).
- `DM-12` (MUST) — **Conflicto de edición = misma memoria (mismo `id`), dos versiones divergentes** (edición
  multi-device del mismo autor, SYNC-9; o local-vs-pull, SYNC-12). Se resuelve **automáticamente por LWW**
  (last-write-wins por timestamp): la versión con timestamp más reciente gana y queda `active`; la perdedora
  se guarda como snapshot en **`memory_versions`** (tabla append-only con historial completo de ediciones)
  y queda recuperable, pero **no genera estado `conflict`** ni requiere intervención del dev. **Sin relación
  `caused`:** son dos versiones de la **misma** memoria, no dos memorias distintas (DM-10).
  - **Distinguir de la redundancia:** dos memorias con **`id` distinto** pero mismo anclaje+topic+tag **no son
    un conflicto**; son **supersesión** vía `replaces` (lo propone el agente en captura —gap #6, CAP-13— o lo
    crea el dev curando). La UI las ofrece como **posibles duplicados** (VIS-10).
  - **Distinguir de la colisión accidental de `id`:** mismo `id` pero memorias **distintas** (otro anclaje/topic)
    es un problema de **identidad**, no de contenido: el cloud re-asigna el ID (SYNC-14).

### Lifecycle de memorias

- `DM-13` (MUST) — Una memoria puede estar en uno de estos estados:

  | Estado | Descripción |
  |--------|-------------|
  | `draft` | Borrador de sesión activa. Solo vive en local, nunca se sincroniza al cloud. Aparece en búsquedas locales. |
  | `active` | Vigente. Devuelta por defecto en búsquedas. |
  | `replaced` | Fue reemplazada por otra vía relación `replaces`. Disponible como historial bajo demanda. |
  | `inactive` | Dada de baja manualmente sin reemplazo. Caso típico: norma de org que se elimina sin sucesor. |
  | `deleted` | Borrado **cloud-only** (como `upload_at`, DM-15). En cloud: borrado **lógico** — retiene el contenido para auditoría del admin y lo oculta de todo recall. En el local NO es un estado guardado: dispara la **purga física** del contenido. Exclusivo del admin (DM-16). |

- `DM-14` (MUST) — La búsqueda devuelve memorias `active` y `draft` por defecto. `replaced` e `inactive` disponibles bajo demanda (parámetro explícito). `deleted` **NUNCA** se devuelve: no existe en el local (purgado) y en cloud solo es accesible por el path de auditoría del admin (DM-16).

### Campos de la entidad Memory

- `DM-15` (MUST) — Campos de la entidad Memory:

  | Campo | Descripción |
  |-------|-------------|
  | `id` | UUID v7 (time-ordered). Generado localmente al crear la memoria. El cloud usa el mismo ID como canónico. Sin ID local/cloud separados. El cloud **valida unicidad al recibir**; una colisión accidental de UUID entre memorias distintas se resuelve re-asignando el ID, no mergeando contenido (SYNC-14). |
  | `title` | Título de la memoria |
  | `content` | Contenido |
  | `topic` | String libre — el "sobre qué". Opcional. |
  | `tag` | Un solo valor del vocabulario controlado (CON-8) |
  | `author_id` | Autor original. Trazabilidad únicamente. Seguridad se gestiona via key en el push. |
  | `editor_id` | Quién editó por última vez. Si ≠ `author_id`, fue curación del admin (ID-10). |
  | `org` | Siempre presente — tenant boundary |
  | `proyecto` | Opcional — parte del anclaje |
  | `repo` | Opcional — parte del anclaje |
  | `lifecycle_state` | draft / active / replaced / inactive / deleted (este último cloud-only, DM-16) |
  | `created_at` | Timestamp de creación en el local |
  | `edited_at` | Timestamp de la última edición (ID-1). Nullable si nunca se editó. |
  | `upload_at` | Timestamp asignado por el cloud al recibir la memoria. Solo vive en cloud. |

> La organización es el **tenant boundary** del sistema (aislamiento de datos y de búsqueda).

### Borrado de memorias

- `DM-16` (MUST) — **El borrado de memorias consolidadas es exclusivo del admin, vía cloud.** El dev no puede borrar
  lo que ya es `active` (CAP-3); **sí** puede **descartar sus propios drafts** (excepción al final de DM-16). El admin borra
  desde la **consola admin del cloud** (Sección 17; login cloud, no la key de sync; mismo path que la escritura
  org-level, SYNC-3b), con alcance según el rol (ADM-3/ADM-4): el **`cloud-admin`** borra **cualquier** memoria de
  la org; el **`project-admin`**, solo las de **sus proyectos asignados** (ADM-8). Motivos típicos: la memoria **no refleja lo que realmente pasó**, contiene **datos sensibles**
  de una persona o empresa, o **envenena el proyecto** y no se quiere seguir alimentándolo.
  - **Cloud → borrado lógico:** `lifecycle_state = deleted`. Retiene el contenido para auditoría del admin y lo
    oculta de todo recall (DM-14, SRCH-6).
  - **Local → purga física:** al bajar el delta (SYNC-13) el local **borra físicamente** el contenido. No queda
    estado: la memoria desaparece del store local.
  - **Relaciones:** al purgar la memoria en el local se **eliminan también las relaciones que la apuntaban**, en
    ambas direcciones. No se conserva un puntero a una memoria que ya no existe.
  - **Excepción — descartar drafts propios:** el dev **sí** puede borrar sus **propios** `draft` (DM-13). Un draft
    es **local y privado** (nunca se pushea, SYNC-6) → descartarlo es **purga local sin fan-out**: no toca el grafo
    compartido ni huérfana aristas de nadie (a lo sumo aristas locales propias, que cascadean en su máquina). Es la
    **válvula** que evita que un draft abandonado se **auto-finalice** a `active` (CAP-10). La exclusividad del admin
    aplica a memorias **consolidadas** (`active` en adelante), no al borrador privado.

---

## 4. Identidad y secretos

- `ID-1` (MUST) — Cada memoria guarda **autoría (`author_id`) + timestamps** (creación/edición).
  Se replica a local y online.
- `ID-2` (MUST) — La tabla de **API keys / secretos** vive **solo online**. Nunca se sincroniza a local.
- `ID-3` (MUST) — Directorio de devs replicable a local = **solo `id_dev` + `nombre_dev`**
  (para mostrar "agregado por Juan"). **Sin rol.**
- `ID-4` (MUST) — Un local no conoce las keys de otros devs (ni la propia más allá de lo necesario
  para autenticar su sync).
- `ID-5` (MUST) — **Rol/permisos** viven **online**, junto con las keys (autorización pura).
- `ID-6` (MUST) — La tabla de keys lleva flag **activa/inactiva** (revocar sin borrar).
- `ID-7` (consecuencia) — La UI local muestra **nombre**, no rol. Para mostrar rol, se consulta el cloud.
- `ID-8` (MUST) — **Una sola key activa por usuario** a la vez.
- `ID-9` (MUST) — No se rastrea dispositivo ni ubicación. Autoría = el dev (vía su key) + timestamp.
- `ID-10` (MUST) — **Autorización de edición:**
  - **El dev edita solo sus propias memorias** (gate por `author_id`; qué cuenta como "propia", ver ID-12).
    No puede editar las de otro, aunque las tenga bajadas por sync. Esta edición ocurre en el **local** (UI o agente).
  - **El admin puede editar memorias ajenas** para curación/de-dup, con alcance según el rol (ADM-3/ADM-4): el
    **`cloud-admin`** edita **cualquier** memoria de la org; el **`project-admin`**, solo las de **sus proyectos
    asignados** (ADM-7). Ocurre **cloud-side** (consola admin, Sección 17, + login cloud, mismo path privilegiado
    que DM-16/SYNC-3b) y baja por fan-out a los locales.
  - **Trazabilidad:** toda edición actualiza `edited_at` (ID-1) y registra **quién editó** (`editor_id`). Si
    `editor_id` ≠ `author_id`, queda visible que fue curación del admin — la autoría original **no** se reescribe
    en silencio.
- `ID-11` (MUST) — **Identidad local (first-run).** La primera vez que se abre el **visor web local**, el setup
  pide **nombre + contraseña**. Cumplen roles **distintos**:
  - **Nombre** = identidad local del dueño de la máquina (el **autor por defecto**, ver ID-12). No es "login".
  - **Contraseña** = candado de acceso **solo al visor web**. El **MCP no la usa ni la necesita** (NFR-5):
    la escritura del agente nunca pide login.
  - Una sola identidad local por instalación; funciona también en orgs local-only (CFG-3, sin cloud).
- `ID-12` (MUST) — **`author_id` local = sentinel reservado + reescritura en el cloud.**
  - **Write local:** el MCP estampa `author_id = sentinel reservado` (**`-1`**, "dev local"). Funciona offline
    y en orgs local-only (CFG-3) sin depender del cloud. Es un valor **reservado por diseño** —jamás un `id_dev`
    real—; **no se usa `0`** (es el default falsy de una columna entera, colisiona por accidente) **ni `null`**
    (significa "autor desconocido", no "yo"). En diseño va como **constante con nombre**; si los ids terminan
    siendo UUID/string, el sentinel pasa a ser un string reservado.
  - **Upload:** el cloud —que ya identifica al dev por la **key**— **reescribe `author_id` al `id_dev` real** de
    esa org. La autoridad de identidad es el cloud (ID-2/ID-5), no el id que mandó el local.
  - **Qué es "propia" en el local** (gate de ID-10): `author_id == -1` **o** `author_id ==` mi `id_dev` en esa
    org (las que ya volvieron por sync, remapeadas). El gate reconoce **ambos**.
  - **Local-only:** nunca hay remap; el autor queda `-1` siempre (consistente: `-1` = yo).

---

## 5. Semántica de niveles y precedencia

- `SEM-1` (MUST) — Las memorias org-level son **preferencias prioritarias, NO mandatos.**
  (v1: **todas permiten divergencia**. La opción de marcar mandatos duros de seguridad/compliance queda diferida.)
- `SEM-2` (MUST) — Un nivel más específico (proyecto/repo) puede **desviarse** de una preferencia org
  para un caso particular.
- `SEM-3` (MUST) — Toda desviación **documenta su porqué** (caso + motivo, ej. "mejor performance acá").
- `SEM-4` (MUST) — Precedencia al consultar: **repo > proyecto > org.** Lo más específico gana; org como fallback.
- `SEM-5` (MUST) — Una desviación documentada **no es un conflicto a resolver**: es un override válido con contexto.
- `SEM-6` (SHOULD) — La desviación es un **vínculo de primera clase** vía relación `diverges` (DM-10):
  la memoria del proyecto apunta a la norma org de la que se sale. La IA y la UI muestran ambas:
  el default org y el porqué de no seguirlo.

---

## 5.b Validez de memorias y política de recall

> Define DÓNDE vive la validez de una memoria. Decisión de fondo: **la DB resuelve la validez; el modelo
> interpreta el significado.** No se le hace hacer al modelo el trabajo de la DB, ni a la DB el del modelo.

- `VAL-1` (MUST) — La **validez la resuelve la DB** (estructural, vía las aristas de DM-10), **no el modelo
  al leer**. Determinística, consistente entre consumidores y barata (un query, no un LLM).
- `VAL-2` (MUST) — `replaces` es **no destructivo**: la memoria reemplazada queda como **historial recuperable**.
- `VAL-3` (MUST) — **Política de recall:** por defecto devuelve lo **vigente**; el **historial está disponible
  bajo demanda**. También resuelta por la DB, no adjudicada por el modelo.
- `VAL-4` (principio) — El modelo lee el historial **solo cuando necesita la narrativa** ("¿por qué cambió
  esto?"), **no para re-derivar la validez en cada lectura**.
- ~~`VAL-5` — Validez interpretada por el modelo al leer (opción 2)~~ **DESCARTADO** — no determinística
  (misma búsqueda, distinto modelo → distinto veredicto), cara (LLM por recall sobre todo el historial) y
  rompe los casos b/c que necesitan respuestas consistentes y baratas. Mismo modo de falla que el score CON-7.

---

## 6. Interoperabilidad

- `INT-1` (MUST) — **Ecosystem-agnostic vía MCP** (Claude Code, OpenCode, Gemini, Codex, Cursor…).
- `INT-2` (MUST) — **Independiente del flujo:** con SDD de Gentleman, con agentes propios, o sin framework.
  El contrato de memorias es el mismo. No atado a SDD.
- `INT-3` (SHOULD) — Además del MCP, exponer **API/CLI plana** para clientes que no hablen MCP.
- `INT-4` (MUST) — El instalador **configura la skill y el MCP** en los agentes o CLI presentes en la máquina
  (Claude Code, OpenCode, Cursor…). El dev no lo hace a mano.

---

## 7. Alcance del contenido

- `CON-1` (MUST) — No se limita a decisiones: captura **decisiones + proceso de desarrollo**
  (qué se hizo, cómo se avanzó, qué se intentó y descartó y por qué). Documentar el proyecto con la IA.
- `CON-2` (MUST) — Tipos de memoria abiertos/extensibles.
- `CON-3` (MUST — principio) — **Señal sobre ruido:** la IA cura en unidades con sentido, **no vuelca
  transcript crudo.** Documentar ≠ loguear todo.
- `CON-4` (MUST) — Captura **amplia con clasificación**: barra más baja al guardar + **tags** (ver vocabulario en CON-8).
- `CON-5` (MUST) — Refina `CON-3`: la barra baja pero siguen siendo unidades con sentido; la
  **importancia se resuelve al buscar**, no al guardar.
- `CON-6` (MUST) — Los **tags son facetas** → ranking **según la intención de búsqueda**
  (estandarizar pondera decisión/diseño; onboarding pondera planificación…).
- ~~`CON-7` — Score 1-5~~ **DESCARTADO** (poco calibrado entre modelos distintos; multi-agente lo rompe).
- `CON-8` (MUST) — **Vocabulario de tags controlado** (no libre). Fuente de verdad = la **skill de captura**
  (LOCAL-5): contiene los tags + el **criterio** para elegir cada uno. Un vocabulario conocido es lo que hace
  **posible el ranking por intención** (CON-6).
  > **Lista preferente v1** (abierta a revisión con el equipo):

  | Tag | Descripción |
  |-----|-------------|
  | `spec` | Especificación de requerimientos de una funcionalidad |
  | `design` | Diseño técnico y decisiones de arquitectura |
  | `verification` | Verificación, pruebas y validación del trabajo realizado |
  | `decision` | Decisión puntual sobre tecnología, enfoque o diseño |
  | `discovery` | Algo aprendido o descubierto en el código base, no obvio |
  | `bugfix` | Bug encontrado y resuelto, con causa raíz documentada |
  | `pattern` | Patrón o convención establecida en el proyecto |
  | `retrospective` | Aprendizaje post-entrega o lección aprendida |
  | `incident` | Falla en producción con análisis, causa raíz y resolución |
  | `config` | Setup de entorno, variables y dependencias necesarias |
  | `preference` | Inclinación o preferencia del equipo, sin ser una decisión cerrada |
- `CON-9` (MUST) — **Storage del tag = `varchar`, NO enum de DB.** Agregar/cambiar un tag = actualizar la
  skill, **sin migración**. Memorias viejas con un tag retirado no se rompen (es solo un string). Mantiene
  la **puerta abierta** a un vocabulario dinámico (tags-como-dato) a futuro **sin cambio de schema**.
- `CON-10` (MUST) — El **código valida** contra una **tabla de normalización local** (whitelist de tags
  aceptados). Esa tabla está **derivada/materializada de la skill** (se regenera al instalar/actualizar la
  skill), **no se mantiene a mano** → skill y validación **nunca derivan**. La validación lee la tabla
  **como dato**, no hardcodeada → agregar un tag **no recompila el MCP** (2 artefactos: skill + tabla, no 3).
- `CON-11` (MUST — robustez) — **Los tags son metadata best-effort: el guardado de una memoria NUNCA falla
  por tags.** Tres resultados posibles:
  - tag **conocido** → se guarda ese tag;
  - el modelo **no aplica ninguno** (nada encaja) → memoria **sin tag** (decisión legítima);
  - el modelo manda un tag **desconocido** (typo / local desactualizado) → se guarda con **`no_clasificado`**.
  `no_clasificado` ≠ sin tag: es una **señal diagnóstica** (si abundan, hay locales con la tabla vieja). Es además un
  **valor reservado del sistema**: lo asigna el **código** (el modelo nunca lo elige) y **no es parte del vocabulario
  de CON-8** — por eso no figura en la lista de tags ni en el whitelist (CON-10) como un tag normal, igual que el
  sentinel `-1` del autor (ID-12). La skill (CON-8) nunca lo usa como tag real.
- `CON-12` (nota) — Frecuencia esperada de cambios de vocabulario: **alta solo en bootstrap** (mientras se
  define el set), luego **prácticamente congelado**. Por eso se elige el modelo estático (skill) sobre el
  dinámico (tabla cloud): optimizar un cambio que casi no ocurre sería over-engineering.

---

## 8. Arquitectura — Online (cloud)

- `SYNC-1` (MUST) — Store central en **Postgres**.
- `SYNC-2` (MUST) — Expone una **API** a la que los locales se conectan (push/pull + escritura org permisada).
- `SYNC-3` (MUST) — Incluye **endpoints de configuración/administración** (enrolar proyectos,
  gestionar keys/roles, activar/desactivar, **borrar memorias (DM-16)**, etc.).
- `SYNC-3b` (MUST) — **Autoría de memorias org-level = exclusiva del cloud.** El líder técnico con rol permisado
  escribe política org desde una **UI/superficie admin del cloud**, autenticado por **login de cloud** (no por la
  key de sync). El MCP **no** participa en la escritura org. Una vez creada, la memoria org baja por fan-out
  (SYNC-5) y vive en todos los locales, legible offline (NFR-7). Consecuencia de seguridad: **ningún local
  necesita ni guarda una key con permiso org-write** — el privilegio se ejerce server-side, detrás del login de
  cloud. Mismo path que el borrado admin (DM-16): todo lo privilegiado org-wide vive y se ejecuta en el cloud.

---

## 9. Arquitectura — Local (nodo del dev)

- `LOCAL-1` (MUST) — **Multi-OS:** corre igual en macOS, Linux y Windows.
- `LOCAL-2` (MUST) — **Multi-agente:** cualquier cliente de IA vía MCP (ver `INT-1`).
- `LOCAL-3` (MUST) — Hospeda el **MCP server + API local**.
  > Dirección de diseño: un único servidor local que sirve API + MCP + UI por SSR (server-side rendering). Un solo proceso, un solo binario.
- `LOCAL-4` (MUST) — Hospeda la **UI del dev** (local-first; lo org-wide consulta el cloud).
- `LOCAL-5` (MUST) — Se entrega con una **skill/protocolo/preset** que le enseña al modelo qué guardar,
  cuándo, cómo buscar y **qué relaciones crear** (recall del target + juicio `extends`/`diverges`/`replaces`,
  ver CAP-13/CAP-14). **(Pieza crítica: es el cerebro de la captura / calidad señal-sobre-ruido.)**
- `LOCAL-6` (SHOULD) — Mantiene un **store local** de lo sincronizado (motor a definir en diseño).
- `LOCAL-7` (MUST) — El instalador despliega el cliente completo: **API local + MCP server + visor web de memorias.**
  > El visor web es una UI local que consume la API local (SQLite). Pendiente de definir en su propia sección.
- `LOCAL-8` (MUST) — El instalador registra el daemon como **servicio del OS**: systemd (Linux),
  launchd (macOS), Windows Service (Windows). El daemon arranca automáticamente con el sistema.
- `LOCAL-9` (MUST) — Expone una **CLI o TUI de configuración** que permite:
  - Agregar organizaciones.
  - Modificar la API key de una org.
  - Modificar la URL del cloud de una org.
  - Configurar el intervalo de sync (cada N minutos).

> **Precedente fuerte (no requerimiento):** binario único sin dependencias, cross-compilado a los 3 OS
> (como engram). El stack se define en diseño.

---

## 10. Configuración del local / multi-organización

- `CFG-1` (MUST) — El local se configura, **por organización**, con: **credencial (key) + URL de la API cloud.**
- `CFG-2` (MUST) — Soporta **varias organizaciones a la vez** (varias empresas desde la misma máquina).
- `CFG-3` (MUST) — **Existe exactamente UNA org sin URL: la pseudo-org `LOCAL`** (proyectos personales del dev,
  sin cloud, sin sync). **No puede haber más de una** org sin URL. **Toda otra org DEBE** tener configurada la
  **URL del cloud + la credencial** (token/secreto) que la identifica contra ese cloud (CFG-1). Sin URL no es
  org de primera clase: es `LOCAL`/personal.
  - **Org ⇒ existe cloud.** La frontera no es "el dev tiene acceso o no", es "**hay un cloud (URL) o no**". Una
    org con URL pero **sin acceso actual** (key inválida/revocada) **sigue siendo org** — el cloud existe, el dev
    está aislado (caso del gap #4, SYNC-10/SYNC-10b).
- `CFG-4` (MUST — seguridad) — **Aislamiento estricto entre orgs:** memorias y búsquedas de una org
  NO se mezclan con otra. El org-wide search es *dentro de una* org, nunca entre empresas.
- `CFG-5` (MUST) — El local guarda su propia key **por org**; sigue sin conocer keys de otros devs.
- `CFG-6` (SHOULD) — Credenciales locales en **secret store del OS** (keychain / credential manager), no texto plano.

> **Resolución de org activa:** cada proyecto local está atado a una org; el agente usa la config de
> esa org según el `cwd` (autodetección por proyecto, no selección manual constante).

---

## 11. Configuración declarativa y resolución de contexto

> Nace del dolor con la detección heurística de engram (adivinar por nombre de carpeta y degradar
> en silencio). La config declarativa es la **única forma** de que "N repos = 1 proyecto" funcione.

- `RES-1` (MUST) — Un archivo declarativo (ej. `.memoria/config.*`) declara explícito:
  **org, proyecto, repo (identidad + rol)** y contexto extra.
- `RES-2` (MUST) — La **config explícita gana** sobre cualquier heurística por nombre de carpeta o `cwd`.
- `RES-3` (MUST) — Si no hay config para el contexto actual → el sistema **pregunta y la genera** (bootstrap).
- `RES-4` (MUST — anti-dolor) — **Prohibido el degradado silencioso.** Si no puede resolver el target,
  **pregunta**; jamás "no pude, te lo guardo en cualquier lado".
- `RES-5` (MUST) — Resolución: busca la **config gobernante más cercana hacia arriba** en el árbol.
  Resuelve tanto "N repos sueltos" como "carpeta padre con sub-repos".
- `RES-6` (SHOULD) — El **mapping** (org/proyecto/repo + topología) es **committeable** y viaja con el repo;
  **las credenciales jamás** van en ese archivo (siguen en el secret store local, `CFG-6`).
- `RES-7` (MUST) — **Identidad de repo = git `origin`, obligatoria siempre.** Un repo se identifica por la URL
  de su **remoto (`origin`)**. Un directorio con `git init` **sin remoto** **no es trabajable**: el sistema no
  lo ancla (no inventa identidad). **Aplica por igual a repos de org y a personales/LOCAL** — un proyecto
  personal es un repo GitHub del propio dev (o un clon de otro dev) y, por tanto, **también tiene `origin`**.
  La distinción org vs LOCAL **no** es "tiene origin o no" (todos lo tienen), sino **de dónde sale la asignación
  de proyecto** (mapping de org vs config local, ver RES-8). Ventaja: el `origin` es identidad **única y estable**
  en toda la org, válida como **clave del mapping org-level** de proyectos.
- `RES-8` (MUST) — **Asignación de proyecto y precedencia.** El proyecto de un repo (identificado por su `origin`,
  RES-7) sale de dos fuentes posibles:
  - **Mapping org-level:** un admin (`cloud-admin`, o `project-admin` sobre sus proyectos, ver ADM-11) declara en el
    cloud "`origin` X → proyecto Y". **Baja por sync** (mismo
    fan-out que las memorias org, SYNC-5); el local resuelve repo→proyecto **solo**, sin config por dev.
  - **Config local declarativa** (RES-1): el dev lo declara en `.memoria/config.*`.
  - **Precedencia — gana el mapping de la org.** Para repos de una org, el mapping org-level **gana sobre la
    config local**. Razón: "N repos = 1 proyecto" vale por la **consistencia del equipo**; si cada dev reasignara
    el proyecto localmente, la topología de la org se **fractura**.
  - **La config local manda solo donde NO hay mapping org:** pseudo-org `LOCAL`/personal, o un repo de org que el
    admin **todavía no mapeó**.
  - **El mapping org solo existe donde hay cloud** (org con URL, CFG-3). En `LOCAL` no hay mapping → siempre
    config local.
  - **No contradice RES-2:** RES-2 le gana a la **heurística de nombre de carpeta**; el mapping org es **dato
    autoritativo**, no heurística. Orden: **mapping org > config local > sugerencia por nombre** (esta última nunca
    autoritativa, solo bootstrap RES-3).

---

## 12. Mecánica de captura

- `CAP-1` (MUST) — Captura **silenciosa y basada en sesión**: la IA acumula contexto durante la
  conversación sin interrumpir. **Sesión = conversación con el agente.**
  - La sesión existe como **borrador (draft)** en el store local desde el inicio. El agente lo va
    actualizando a medida que ocurren cosas. Al cierre se finaliza como memoria permanente.
  - Ventaja: si la conversación se corta, el draft preserva lo acumulado hasta ese momento.
  - **Sub-agentes:** cada uno crea **su propia memoria en estado `draft`** (con el tag que corresponde a su
    tarea) para que pueda ir creciendo. No se consolida al terminar la tarea: la promoción a `active` la hace
    el agente principal al cierre de la sesión.
  - **Agente principal:** actualiza su draft durante la sesión y, al cierre, **consolida (promueve a `active`)**
    su draft y los de los sub-agentes.
  - **Múltiples drafts:** durante una sesión pueden coexistir **varios drafts vivos** (el del principal + uno
    por sub-agente). La mecánica exacta de promoción —flag, fecha, estado normalizado— se define en diseño.
- `CAP-2` (MUST) — El contenido que la síntesis debe cubrir: decisión, bugfix (con causa raíz), gotcha,
  convención, intento-descartado, hito de proceso… (catálogo definido por la skill, LOCAL-5).
- `CAP-3` (MUST) — Control **post-hoc** en la UI: **revisar, editar (in-place, ver CAP-12), fusionar/reemplazar.** El dev edita **solo sus propias** memorias (ID-10) y **NO puede borrar memorias consolidadas** — el borrado es exclusivo del admin vía cloud (DM-16). **Excepción:** sí puede **descartar sus propios drafts** (local/privado, DM-16).
- `CAP-4` (SHOULD) — **Transparencia sin fricción:** vista del draft de la sesión activa y de memorias
  finalizadas recientes. El dev puede ojear qué acumuló el agente antes del cierre.
- `CAP-5` (MUST) — La calidad recae en la **skill de captura** (`LOCAL-5`), que aplica señal-sobre-ruido.
- `CAP-6` (SHOULD) — El dev puede pedir al agente que actualice el draft en cualquier momento de la sesión
  (guardado explícito = actualización del borrador activo).
- `CAP-7` (MUST — privacidad, capa dura **B**) — **Filtro determinístico de secretos.** Bloquea/enmascara
  secretos **con forma** (API keys, tokens, passwords, private keys, connection strings) **antes de escribir
  en local y antes de sincronizar**. **Redacción quirúrgica:** en un connection string enmascara solo la
  credencial y **preserva host/IP/puerto** (dato útil para identificar proveedores).
- `CAP-8` (MUST — privacidad, capa blanda **A**) — La **skill de captura** instruye al modelo a no volcar
  credenciales/secretos. **IPs, rutas, DNI y nombres NO se filtran ni se redactan, pero tampoco se piden
  ni se priorizan:** no son datos obligatorios ni preferentes; si aparecen naturalmente en el contexto, se
  conservan tal cual (pueden identificar proveedores, involucrados o casos de uso de ejemplo). Lo único que
  el sistema oculta son secretos/API keys.
- `CAP-9` (decisión) — Se **descarta** la clasificación de sensibilidad / sync con scope (opción "C"):
  el contexto de negocio y los involucrados son señal, no riesgo; el residuo semántico real es ultra angosto
  y queda cubierto por A + curación post-hoc. El caso de alta severidad (credencial viva) lo garantiza B.
- `CAP-10` (MUST) — **Draft huérfano:** si un draft lleva más de **5 días sin cerrarse**, el sistema lo
  **finaliza automáticamente** (lo promueve a `active`) y entra al ciclo de sync (SYNC-6). Criterio del umbral:
  cubre fines de semana y feriados largos (hasta 4 días) sin penalizar al dev que retoma el lunes o el martes.
  **Fundamento:** una memoria a medio terminar vale más como documentación que no tener ninguna — se asume
  deliberadamente el costo de consolidar un borrador incompleto antes que perderlo.
- `CAP-11` (MUST) — **Reanudación de sesión:** al retomar trabajo en un proyecto (incluido un corte de luz,
  crash o cierre inesperado), el agente busca un draft **pertinente al cambio en curso** —no necesariamente el
  último abierto, ya que pueden coexistir varios (CAP-1)— antes de crear uno nuevo. Dos caminos:
  - **Draft aún activo** (< 5 días, CAP-10): lo **continúa** desde donde quedó, preservando el contexto acumulado.
  - **Draft ya auto-finalizado** por antigüedad (CAP-10): esa memoria ya es `active` y sincronizó → el agente
    **abre un draft nuevo** para el trabajo actual (la transición `draft → active` es unidireccional, VIS-11c).
    Si corresponde, lo vincula a la memoria anterior vía `extends` (CAP-13) para no perder el hilo narrativo.
  > El recall del draft pertinente (por anclaje/topic) vive en la **skill de captura** (LOCAL-5), igual que el
  > resto del juicio de captura.

### Editar vs. reemplazar

- `CAP-12` (MUST) — **Editar (in-place) ≠ reemplazar (`replaces`).** Son las dos formas en que una memoria
  finalizada evoluciona, y NO son intercambiables:
  - **Editar** — corrige el registro para que sea **fiel a lo que pasó** o lo **completa**: muta el contenido
    **en el lugar** (mismo `id`). La versión vieja **nunca fue verdad** (un plan que no se cumplió, un dato mal
    escrito, pasos que faltaban) → no hay historia que preservar.
    > Ej.: la memoria decía "se instala la 1.0" y en la práctica se instaló la 2.5; o se agregaron pasos sobre la marcha.
  - **Reemplazar** — supersesión vía relación `replaces` (DM-10): la vieja **fue verdad en su momento** y el
    mundo **cambió**. Memoria nueva, la vieja pasa a `replaced` (historial recuperable, VAL-2).
    > Ej.: las contraseñas se codificaban con X y meses después, por seguridad, se hashean con Y.
  - **Criterio (vive en la skill, LOCAL-5):** *"¿la versión vieja alguna vez fue verdad?"* → nunca = editar;
    sí, y dejó de serlo = reemplazar. Es un **juicio de autoría al escribir** (como elegir el tag, CON-8), **no**
    una re-derivación de validez al leer → no reedita el anti-patrón descartado VAL-5. Coherente con VAL-1.
  - **Quién edita:** el **agente** puede editar **sus propias** memorias finalizadas durante el trabajo (para
    darles consistencia con lo realmente hecho), además del humano post-hoc en la UI (CAP-3). La autorización de
    quién puede editar qué la define ID-10.

> Al elegir captura silenciosa, la red de seguridad se mueve a: (1) la skill de captura (capa blanda **A**),
> (2) el **filtro determinístico de secretos** (capa dura **B**, write + sync) y (3) la curación post-hoc en
> la UI. No son opcionales.

### Creación de relaciones

- `CAP-13` (MUST — degradación elegante) — La autoría de relaciones (`extends`, `diverges`, `replaces`, `caused`) en
  captura es **best-effort del agente**: el agente las **propone** con el contexto fresco de la sesión. Pero
  **la captura NUNCA se bloquea ni se pierde por no poder resolver una relación.** Principio:
  **relación faltante > memoria faltante.** Si el agente no reconoce el target o duda, guarda la memoria
  **igual, sin la arista**; jamás "no pude relacionarla, no la guardo". Coherente con DM-11 (una relación
  existe solo si agrega info) y con la filosofía no-destructiva.
  - **Backstop para `replaces`:** una memoria guardada sin su `replaces` no queda huérfana. Si nace otra con
    **mismo anclaje + topic + tag** (memorias con **`id` distinto**), el visor las **sugiere como posibles
    duplicados** (VIS-10). **El código nunca crea el `replaces` ni afirma que son duplicados:** mismo
    anclaje+topic+tag es coincidencia **estructural, no semántica** — esas dos memorias pueden ser `extends`
    (complementarias) o no tener relación. La heurística solo **filtra candidatos baratos para revisar**; la
    decisión de crear el `replaces` exige **juicio semántico sobre el contenido**, que vive en dos lugares y
    **jamás en inferencia automática**: (a) el **agente en captura**, con contenido fresco + recall (CAP-14);
    (b) el **dev en el visor**, mirando el contenido y pudiendo descartar (VIS-10). No se marca
    `lifecycle_state = conflict` (ese estado ya no existe — DM-12 usa LWW automático) ni se crea `caused`.
  - **`diverges` no tiene backstop automático:** reconocer que la org tiene una norma y que el contexto
    actual se desvía es **juicio puro del agente** (vive en la skill, LOCAL-5). Si no lo reconoce, se guarda
    la memoria sin la arista y el dev la agrega a mano (CAP-14).
  - **`caused` no tiene backstop automático** (igual que `diverges`): el caso típico es documentar un `bugfix`
    cuya causa raíz rastrea a una memoria existente (`design`/`decision`) que el agente encuentra por recall
    (CAP-14). El agente la **propone** memoria→memoria, con ambos extremos existentes (DM-10). Si la causa **no
    está documentada** como memoria, no se infiere arista: va en el **texto** de la memoria (causa raíz, CAP-2).
    Si el agente no reconoce el target, se guarda sin la arista y el dev la agrega a mano (CAP-14).
- `CAP-14` (MUST) — El **juicio de qué relación crear** (y el recall para encontrar el target) vive en la
  **skill de captura (LOCAL-5)**, igual que la elección de tag (CON-8) y el editar-vs-reemplazar (CAP-12).
  La UI local es la **red de seguridad editable**: el dev puede **agregar, corregir o quitar** relaciones de
  **sus propias** memorias post-hoc (ID-10), además de revisar las que el agente propuso (CAP-3).

---

## Modelo de escritura/sync (esbozado — pendiente de detalle)

```
LOCAL (cualquier dev, captura IA sin fricción)
  ├── org+proyecto+repo   ← el 90%
  └── org+proyecto        ← transversal del proyecto
        │ sync ↑
        ▼
      CLOUD (fuente de verdad, Postgres)
        ▲
        │ admin → UI admin del cloud (login cloud + rol permisado) → escribe org-only
  org   ← política de empresa (se AUTORA en cloud; baja por fan-out y vive en local, offline)
        │ fan-out ↓
        ▼
  se descarga a TODOS los locales
```

- repo/proyecto → nacen en local, suben por sync.
- org → **se autora en el cloud** (UI admin, login cloud + rol permisado, ver SYNC-3b), baja a todos por fan-out
  y **vive en local** (se lee offline como cualquier memoria, NFR-7). "Nace en cloud" = el *origen/escritura* es
  cloud-only, **no** que viva solo ahí.
- Los locales **nunca** tienen una key con permiso org-write: la corona se queda detrás del login del cloud.
- Joya de consistencia: `proyecto = null` (memoria org-only) es exactamente la única que el path
  privilegiado puede crear. Modelo de datos y modelo de permisos encajan sin forzarlos.

---

## 13. Búsqueda

> Motor: **FTS5 + BM25** (SQLite), igual que engram. Offline-first, binario único, sin dependencias externas.
> La búsqueda es siempre local — nunca va al cloud.

- `SRCH-1` (MUST) — Motor: FTS5 con ranking BM25.
  - Búsqueda **con scope** (proyecto/repo/topic): sub-10ms garantizado.
  - Búsqueda **org-wide sin scope**: best-effort, sin SLA estricto. A 1M memorias puede ir a 50-200ms. Revisar si FTS5 sigue cumpliendo cuando la org supere 500K memorias.
- `SRCH-2` (MUST) — Scoping por **filtros opcionales** sobre los 3 ejes del anclaje. `org` siempre presente
  (tenant boundary). `proyecto`, `repo` y `topic` son filtros opcionales e independientes entre sí.
  No hay cascade — el llamador aplica los filtros que correspondan a la intención de la pregunta.
- `SRCH-3` (MUST) — Un solo endpoint de búsqueda sirve tanto al agente (MCP) como a la UI del dev.
- `SRCH-4` (MUST) — Tags en búsqueda, separados por consumidor:
  - **MCP (agente):** recibe resultados con campo `tag` incluido; razona internamente según su intención.
    Sin parámetro de intent en el endpoint.
  - **UI (dev):** mismo endpoint + filtro opcional por `tag` para navegación/facetado manual.
- `SRCH-5` (MUST) — Relaciones en resultados, diferenciadas por endpoint:
  - **`search`:** relaciones **shallow** — array `{id, title, type, direction}`. Sin contenido de memorias relacionadas.
  - **`get memory` (detalle):** relaciones con **contenido completo** incluido.
  - Flujo agente: busca → identifica → fetchea detalle cuando necesita profundidad (2 llamadas).
- `SRCH-6` (MUST) — Recall por defecto devuelve memorias `active` + `draft`. `replaced` e `inactive`
  disponibles bajo demanda como parámetro explícito. `deleted` nunca se devuelve (no existe en el
  local; en cloud solo vía auditoría admin, DM-16). Resuelto por la DB, no por el modelo (VAL-1/3).

---

## 14. Sync

> Modelo: local es la tabla de verdad del dev. El cloud es fuente de verdad compartida. El sync es bidireccional.

- `SYNC-4` (MUST) — Ciclo periódico, configurable por el dev (ver LOCAL-9). Valor por defecto: ~10 minutos. El local corre como daemon persistente.
- `SYNC-5` (MUST) — **Pull:** delta de memorias + relaciones. Scope: org-level + **todos los proyectos
  de la org** (no solo los del dev). ~150MB estimado para la org de referencia (75K memorias), tamaño manejable.
  > Topics no se sincronizan por separado: viajan como string en la memoria. La lista de topics conocidos
  > se deriva localmente con una query al store (DM-9).
- `SYNC-6` (MUST) — **Push:** memorias nuevas/editadas (`active`, `replaced`, `inactive`) + relaciones
  acumuladas desde el último sync. Los `draft` **nunca se sincronizan** — son locales hasta que se finalizan.
- `SYNC-7` (MUST) — **Cursor de delta:** el cloud asigna `upload_at` a cada memoria al recibirla.
  Los clientes usan ese campo como cursor ("dame todo con `upload_at` > mi último sync").
  `upload_at` vive **solo en el cloud**, nunca se replica a local. Resuelve el caso del dev
  que estuvo offline días y subió memorias "viejas" — otros las bajan igual porque el cursor
  es cuándo llegaron al cloud, no cuándo se crearon. Elimina también el clock skew entre máquinas.
- `SYNC-8` (MUST) — **Primera sync:** al instalar y configurar credenciales, baja todas las memorias
  de la org desde el inicio (sin delta).
  > **Nota de implementación:** definir estrategia de paginación/chunking para orgs grandes cuando se programe.
- `SYNC-9` (MUST) — **Conflictos de edición (LWW automático):** como solo el autor edita su memoria (ID-10),
  el caso "dos devs editan la misma memoria" **no puede ocurrir entre devs distintos**; queda acotado al
  **mismo autor en dos máquinas** (multi-device). Si llegan dos ediciones del mismo `id`, la API compara
  timestamps: la más reciente gana (`active`), la perdedora se guarda como snapshot en **`memory_versions`**
  (DM-12). **Resolución automática, sin intervención del dev, sin estado `conflict`.** La edición del
  **admin** sobre memoria ajena es cloud-side y baja por pull → su conflicto con una edición local no
  pusheada del autor lo maneja SYNC-12 (también LWW automático).
- `SYNC-10` (MUST) — **Key revocada (offboarding) — flujo de purga:**
  1. El local sincroniza y el cloud responde `"key inactiva"` (ID-6).
  2. Si la `purge_executed_at` de esa org está **vacía**, el local **purga TODAS las memorias de la org** — corte
     limpio: lo bajado del cloud **y** lo que el propio dev autoró para esa org.
  3. El local **avisa al cloud** que purgó. El cloud acepta la key inactiva **solo** para registrar la confirmación
     (ya la identificó en el paso 1; no sirve para ninguna otra operación).
  4. El cloud **registra la purga confirmada** para esa key → señal de offboarding consultable: "purga confirmada"
     vs. "pendiente".
  5. El local setea `purge_executed_at`. De ahí en más el sync **sigue intentando y fallando** (NFR-8) pero **no
     vuelve a purgar** mientras `purge_executed_at` tenga valor (guardia de idempotencia).
  - **Después de la purga:** el dev puede seguir generando memorias nuevas en esa org si quiere — quedan **locales
    y nunca sincronizan**. Es cosa suya.
  - **Limitación conocida (local-first):** la purga solo se ejecuta cuando el local **reconecta**. Una máquina que
    nunca vuelve a conectarse conserva la data hasta que lo haga; el cloud **no puede forzar** el borrado remoto.
    La confirmación (paso 4) hace el estado **observable**, no lo garantiza.
  - **Re-contratación:** se emite una **key nueva** (ID-8) → config de org nueva → first-sync completo (SYNC-8).
    La `purge_executed_at` vieja muere con la key vieja.
  - **Campo:** `purge_executed_at` es **estado local por org** (no es campo de la entidad Memory): nullable,
    vacío = purga pendiente, con fecha = purga hecha.
- `SYNC-10b` (MUST) — **Key inexistente / inválida (≠ revocada):** si la key nunca fue válida (typo, no enrolada),
  el local **no sincroniza** (ni baja ni sube) y **NO purga nada** — las memorias locales quedan intactas. La purga
  (SYNC-10) se dispara **solo** con la respuesta específica `"key inactiva"` de una key reconocida-pero-revocada.
  Distinguir ambos casos es **obligatorio**: jamás purgar por un error de tipeo.
- `SYNC-11` (MUST) — **Orden del ciclo:** pull primero, luego push. Se baja el estado del cloud antes de subir cambios locales para minimizar conflictos.
- `SYNC-12` (MUST) — **Conflicto detectado en pull (LWW automático):** si el pull trae una versión actualizada
  de una memoria que el dev también editó localmente (aún no pusheada), el local aplica la versión cloud y guarda
  la edición local como snapshot en **`memory_versions`**. Resolución automática, sin intervención del dev, sin
  estado `conflict` (DM-12). El dev puede consultar `memory_versions` si necesita recuperar su edición descartada.
- `SYNC-13` (MUST) — **Borrado admin (fan-out):** cuando el admin borra una memoria (DM-16), el cloud la marca `deleted` y le asigna un `upload_at` nuevo. Baja por el cursor de pull como cualquier delta; cada local que la tenía **purga físicamente** el contenido y las relaciones que la apuntaban (DM-16). El **first-sync** (SYNC-8) no envía memorias `deleted` — en un local nuevo no hay nada que purgar. Como el ciclo es **pull primero** (SYNC-11), el borrado le gana a una edición local no pusheada: el local purga y descarta su edición.
- `SYNC-14` (MUST) — **Colisión accidental de `id` (no es conflicto de contenido):** con UUID v7 generado en local,
  la probabilidad de que dos memorias **distintas** saquen el mismo `id` es ínfima (requiere mismo milisegundo +
  mismos ~74 bits aleatorios), pero el cloud **no la asume imposible**. El discriminador es el **tipo de operación**
  con que llega el upload:
  - **`CREATE` con `id` ya existente** → es una **colisión accidental** (una memoria nueva nunca debería reclamar
    un `id` ya tomado). El cloud **conserva el `id` de la memoria que ya estaba** (llegó primero, es la canónica),
    **re-asigna un UUID nuevo a la entrante** y le responde al local: *"ese `id` ya existe, usá este otro"*. El local
    **actualiza la memoria y reescribe sus aristas locales** apuntando al `id` nuevo. Es un **fallback de emergencia**
    (probabilidad ≈ cero, acotado a una sola memoria), no la mecánica normal de sync.
  - **`UPDATE` con `id` ya existente** → es la **misma** memoria, dos versiones → **conflicto de edición** (SYNC-9),
    se resuelve automáticamente por LWW; la perdedora va a `memory_versions` (DM-12). No se re-asigna nada.
  - **Por qué no doble `id` (local + cloud):** la garantía de unicidad se obtiene con esta validación server-side
    sobre el ID único; un segundo ID obligaría a una tabla de traducción local↔cloud y a **reescribir aristas en
    cada subida** (no solo en la colisión). El UUID v7 mantiene identidad estable desde el nacimiento (DM-15).

---

## 15. No-funcionales

### Performance
- `NFR-1` (MUST) — Round-trip completo (MCP → servidor local → SQLite → respuesta) sub-100ms. Motor FTS5 garantiza sub-10ms internamente (SRCH-1).
- `NFR-2` (MUST) — El servidor local corre como **daemon/servicio persistente**. Sin startup por consulta.
- `NFR-3` (MUST) — Multi-OS: macOS, Linux y Windows. (Stack a definir en diseño; Go como sugerencia, no cerrado.)

### Privacidad
- `NFR-4` (SHOULD) — Datos en tránsito sobre **HTTPS/TLS**. Preferente; evaluar vs otros protocolos en diseño.
- `NFR-5` (MUST) — **Capas de identidad independientes:**
  - **Login local:** **nombre + contraseña** definidos en el first-run del visor (ID-11). La **contraseña**
    da acceso a la UI **únicamente**; el **nombre** es la identidad local / autor por defecto (ID-12). Funciona
    también en org local-only (CFG-3, sin sync). No requiere cloud. El MCP no usa este login.
  - **API key de org:** conecta el local al cloud para sync. Puede estar ausente en setups local-only.
    **Solo autoriza sync** — nunca escritura org-level.
  - **Login admin del cloud:** autentica al líder técnico contra el cloud para las operaciones privilegiadas
    org-wide (escribir política org, SYNC-3b; borrar, DM-16). Independiente del login local y de la key de sync;
    vive solo en el cloud.
  - **MCP:** sin autenticación. Acceso local libre — el agente IA lo consume directamente sin login.
  - **Principio (autenticación ≠ autorización):** autenticar al llamador local **no es lo mismo** que autorizar la
    operación. El MCP no autentica → la **escritura local es siempre libre** (no se bloquea ni con key inválida).
    La **autorización de todo lo que toca el cloud vive server-side**, contra la key de org (ID-5). Consecuencia:
    la key de org es la llave para **sincronizar**, no para escribir. Key inválida = local aislado pero 100%
    funcional (NFR-7).
- `NFR-6` (MUST) — Ver `SYNC-10`: key revocada → purga local de memorias de esa org.

### Comportamiento offline
- `NFR-7` (MUST) — El agente vía MCP y la UI local funcionan **igual** con o sin conexión al cloud.
  Casos de uso: cloud detrás de VPN, DB cloud caída, red no disponible.
- `NFR-8` (MUST) — El sync es **transparente para el agente**: si falla por falta de conexión,
  reintenta en el próximo ciclo sin interrumpir el trabajo del dev.
- `NFR-9` (SHOULD) — La UI local muestra la **fecha del último sync exitoso** como indicador pasivo.
  Sin alertas ni interrupciones.

### Escala (referencia org)
- `NFR-10` (referencia) — Org de referencia: ~20 usuarios, ~250 repos activos, ~75K memorias totales,
  ~150MB con índice FTS5. El stack simple maneja esta escala sin optimizaciones especiales.
- `NFR-11` (MUST) — El cloud es **una instancia por organización** (no plataforma SaaS multi-tenant).
  Cada org instala y opera su propio cloud. El local puede conectarse a múltiples orgs (CFG-2).

---

## 16. Visor web de memorias

> UI local que consume la **API local** (SQLite). Es el componente con más responsabilidades colgadas a lo
> largo del PRD (CAP-3/CAP-4/CAP-14, ID-7/ID-10/ID-11/ID-12, SRCH-3, SYNC-12, NFR-7/NFR-9); esta sección las
> consolida y define qué muestra y qué deja hacer.
>
> **Principio de uso:** el visor es primariamente para **visualizar**. El canal principal de trabajo sobre
> memorias es el **agente** (captura y edición conversacional). En el visor el dev hace **curación mínima sobre
> sus propias memorias** (VIS-7) — no es la herramienta de trabajo principal. Las acciones editables del visor
> son la excepción, no el flujo habitual.

### Alcance y navegación

- `VIS-1` (MUST) — **Lee exclusivamente del local.** El local replica **toda la org** (SYNC-5) y la búsqueda
  es **siempre local** (sección 13, "nunca va al cloud"). El visor **no** hace consultas vivas al cloud. Anda
  100% offline (NFR-7).
- `VIS-2` (MUST) — **Entrada por organización.** Lo primero es un selector **"¿qué organización querés ver?"**:
  cada org configurada (A, B…) + una pseudo-org **LOCAL** (proyectos personales sin org, CFG-3). Acá se
  materializa el **aislamiento estricto** (CFG-4): se ve **una org por vez**; memorias y búsquedas no cruzan
  el límite de org.
- `VIS-3` (MUST) — **Drill-down por anclaje.** Dentro de la org elegida, la navegación baja por los tres ejes:
  **organización → proyecto → repo**. La vista de proyecto muestra la topología (VIS-4).
- `VIS-4` (MUST) — **Vista de proyecto = topología, no lista plana.** Dentro de un proyecto, el visor muestra
  los repos con su **rol** (DM-4) y las **aristas dirigidas** entre ellos (DM-5), no una bolsa plana de repos.
  - **Es decisión visual, no de datos:** la topología es **dato de primera clase ya almacenado** (DM-5,
    consultable). Mostrarla **no cuesta campos nuevos**; una lista plana solo **desperdiciaría** dato existente.
  - **Fidelidad de render abierta a diseño (MVP-friendly):** el MVP puede ser una **vista estructurada** (cada
    repo con su rol + sus aristas listadas); el **grafo visual completo** es una mejora posterior. La mejora es
    puro front sobre el mismo modelo — **no hay migración de datos** porque la topología ya existe (DM-5).

> **Principio de evolución del visor:** mientras el **dato ya esté en el modelo**, el MVP puede mostrar una vista
> simple y las mejoras de UI no generan cruces ni migraciones — son puro front sobre la misma base. Aplica a la
> topología (VIS-4), a las relaciones (VIS-5) y a todo lo visual del visor.

### Vista de detalle y relaciones

- `VIS-5` (MUST) — **Detalle de memoria con relaciones inline.** Al abrir una memoria, el visor muestra sus
  relaciones (`replaces`/`extends`/`diverges`/`caused`) como **lista inline**: tipo + dirección + título, con
  click para **saltar** a la relacionada (traversal de a un salto). Los datos ya los provee la API (SRCH-5:
  `search` shallow `{id, title, type, direction}`, `get memory` con contenido completo) — no requiere nada nuevo.
  - **Historial:** un **"ver historial"** despliega, desde la memoria vigente, la cadena de versiones `replaced`
    (DM-13, recuperable bajo demanda).
  - **Mejora futura (sin tocar modelo):** una **vista de grafo** de memorias (cadenas de `replaces`, árboles de
    `extends`) es una mejora posterior — mismo dato, puro front (ver principio de evolución).

### Búsqueda en el visor

- `VIS-6` (MUST) — **Búsqueda auto-scope con escape para ensanchar.** La búsqueda del visor aplica por defecto el
  **contexto resuelto** (org/proyecto/repo) como filtro inicial. El visor es el "llamador" al que SRCH-2 le deja
  decidir los filtros, y el contexto es **determinístico** (repo por `origin` RES-7, proyecto por mapping RES-8,
  org por cloud CFG-3): **no adivina por nombre de carpeta** — el scope refleja una **posición conocida**, no una
  corazonada.
  - **Escape innegociable:** un click ensancha a "todo el proyecto" o "toda la org" (caso de uso (a),
    estandarizar org-wide).
  - **Filtros adicionales:** por `tag` (SRCH-4) y por `lifecycle` (SRCH-6, `replaced`/`inactive` bajo
    demanda). Motor BM25 local (SRCH-1), siempre offline.

### Acciones de curación

- `VIS-7` (MUST) — **Acciones sobre memorias propias** (gate por `author_id`, ID-10/ID-12): **editar contenido**
  (in-place, CAP-12), **editar el `tag`**, y **agregar/corregir/quitar relaciones** (CAP-14). **Borrar NO** memorias
  consolidadas — es exclusivo del admin vía cloud (DM-16); **sí** puede **descartar drafts propios** (DM-16). La
  edición del admin sobre memoria ajena es cloud-side (ID-10).
- `VIS-8` (MUST) — **Re-anclaje (mover una memoria en el árbol org/proyecto/repo).**
  - **Cruzar de org: IMPOSIBLE.** Inviable por el aislamiento estricto (CFG-4, tenant boundary): una memoria
    **pertenece a su org**. Para "moverla" a otra org, se **recrea** a mano allá; no se mueve.
  - **Dentro de la misma org, solo el autor** (`author_id`, ID-10/ID-12):
    - `repo → repo` dentro del **mismo proyecto**.
    - `proyecto A → proyecto B` (misma org).
  - **Límite temporal: 30 días desde `created_at`** (DM-15). Pasado el plazo, el anclaje queda **fijo**.
    Fundamento: una memoria fresca con anclaje errado se corrige; una vieja ya está **asentada en el grafo
    compartido** (linkeada, buscada) y moverla disrumpe.
  - **Coherencia con RES-8:** para memorias a nivel repo, el proyecto lo determina el `origin` del repo (mapping
    org, RES-8); el re-anclaje entre proyectos aplica de lleno a **memorias de proyecto** (`repo = null`) o, en
    repos, implica reconocer a qué repo/proyecto corresponde realmente.
  - **Re-anclaje por el admin (excepción):** el admin **puede re-anclar memorias ajenas** y **sin el límite de 30
    días** — coherente con ID-10. El alcance depende del rol (ADM-3/ADM-4, ADM-9): el **`cloud-admin`** re-ancla
    **cualquier** memoria en toda la org; el **`project-admin`**, solo cuando **origen y destino** caen en sus
    proyectos asignados (cruzar fuera de su dominio = `cloud-admin`). El límite temporal es un freno para el
    **autor**; el admin tiene autoridad sobre el grafo y reorganiza cuando haga falta. Ocurre
    **cloud-side** (mismo path privilegiado que DM-16/ID-10) y baja por fan-out; queda trazado en `editor_id`/
    `edited_at` (ID-10). **El cruce de org sigue imposible también para el admin:** es un límite **estructural**
    (CFG-4, NFR-11 — cada org es un cloud aparte), no un permiso que el rol pueda saltar.

### Historial de ediciones y duplicados

- `VIS-9` (MUST) — **Historial de ediciones de una memoria.** El visor permite consultar el historial
  completo de ediciones de una memoria desde **`memory_versions`** (DM-12): lista cronológica de snapshots
  con `version`, `edited_at`, `edited_on` y `author_id`. Lectura solamente — el historial es append-only.
  **No hay UI de resolución de conflictos:** los conflictos de edición se resuelven automáticamente por
  LWW (SYNC-9, SYNC-12), sin intervención del dev. Si el dev necesita recuperar contenido de una versión
  anterior, lo copia manualmente en una nueva edición.
- `VIS-10` (MUST) — **Vista de posibles duplicados (ids distintos).** Aparte del historial de ediciones, el visor
  agrupa memorias con **mismo anclaje + topic + tag** pero **`id` distinto** (backstop de CAP-13) como
  **candidatas de baja confianza** a revisar. **La agrupación es señal estructural, no una afirmación de
  duplicado:** el visor **no infiere la relación ni crea nada automáticamente**. El **dev revisa el contenido**
  y decide: crear un `replaces` (supersesión), un `extends` (son complementarias), o **descartar** la sugerencia
  (no tienen relación). Recién con esa confirmación el visor crea la arista (DM-10). Es **curación opcional**, no
  un estado bloqueante: no hay `lifecycle_state = conflict` involucrado. Solo sobre memorias propias (ID-10/ID-12).

### Estados pasivos

- `VIS-11` (MUST) — **Estados pasivos del visor** = indicadores que el visor **muestra** sin que el dev actúe.
  Tres:
  - **(a) Último sync exitoso (NFR-9):** la fecha/hora del último sync OK, como indicador pasivo **sin alertas
    ni interrupciones**. En una org **LOCAL-only** (CFG-3, sin cloud) no hay sincronización → el indicador muestra
    "local, sin cloud" en lugar de una fecha.
  - **(b) Read-only vs. editable (ID-10/ID-12):** badge visible que distingue las memorias **editables**
    (`author_id == -1` **o** `== mi id_dev` en esa org) de las **solo-lectura** (de otros devs bajadas por
    fan-out, y memorias org que nacen cloud-side del admin). La señal es visible **antes** de intentar editar,
    para que las acciones de VIS-7 no sorprendan al dev.
  - **(c) Drafts activos (CAP-4):** un dev puede tener **varios drafts simultáneos** porque un draft es una
    memoria **anclada** (DM-1) como cualquier otra: varios por sesión (principal + sub-agentes, CAP-1) y, a lo
    largo del tiempo, uno por cada **proyecto/repo** en el que viene trabajando. El visor los muestra **cada uno
    en su lugar del árbol de anclaje** (VIS-3) con badge **"borrador"**, no como un único draft global. Cada
    draft **es editable a mano** por el dev: puede **editarlo y mantenerlo `draft`**, **editarlo y promoverlo a
    `active`**, o **descartarlo** (borrado local de un draft propio — la válvula contra que un draft abandonado se
    auto-finalice, CAP-10/DM-16) cuando quiera. La promoción también la hace el **agente** al cerrar la sesión (CAP-1) o el sistema
    al **auto-finalizar a los 5 días** (CAP-10); todos los caminos confluyen en la misma transición.
    - **Transición unidireccional `draft → active`.** Una memoria consolidada **nunca** vuelve a `draft`.
      Fundamento: `active` sincroniza (SYNC-6) y puede estar ya **linkeada/buscada por otros**; degradarla a
      `draft` (local, no-sync) la **sacaría del cloud** y rompería las relaciones que la apuntan.
    - **Promoción a mitad de sesión:** si el dev promueve el draft mientras la sesión del agente sigue viva, el
      agente **abre un draft nuevo** y continúa acumulando ahí; el draft promovido queda cerrado como memoria.
      No es el flujo habitual —normalmente el draft lo finaliza el agente al cierre (CAP-2)— pero la promoción
      manual **no interrumpe** la sesión.

## 17. Visor/consola cloud (admin)

> Superficie **cloud-side** de las operaciones privilegiadas org-wide. Es la **hermana de la Sección 16** pero del
> otro lado del sync: el visor local (VIS-1) lee del **local** y lo opera el dev por agente; esta consola lee del
> **cloud** y la opera un humano del **tier admin** por web. Consolida responsabilidades dispersas a lo largo del PRD
> (SYNC-3/SYNC-3b, DM-16, ID-2/ID-5/ID-6/ID-10, VIS-8, RES-8, DM-14, y el "login admin del cloud" de NFR-5).
>
> **Principio de uso:** lo privilegiado es **humano y cloud-side**, no agéntico. El cloud **cura** memorias de
> proyecto y **autora** política org; lo que **nunca** hace es **capturar** una memoria de proyecto — la captura
> (CAP-*) es exclusiva del borde (dev local + agente). El admin trabaja **a mano**.

### Naturaleza y alcance

- `ADM-1` (MUST) — **Lee del cloud, no del local.** A diferencia del visor local (VIS-1, que lee SQLite local), la
  consola admin opera contra el **cloud**, donde viven las operaciones privilegiadas y la retención para auditoría
  (DM-16). Es una **UI web** autenticada por **login cloud** (ADM-6), operada **a mano** por humanos del tier admin.
  **No pasa por el agente ni por el MCP del dev.**
- `ADM-2` (MUST) — **v1 = solo visor web; sin MCP admin.** Un eventual MCP que asista al admin es **feature futura**
  y, aun cuando llegue, su alcance queda **acotado a curación sobre lo existente** — **nunca** a capturar memorias de
  proyecto cloud-side. Las memorias de proyecto **nacen solo en el borde** (CAP-*).

### Roles y autorización

- `ADM-3` (MUST) — **Tres roles, por organización** (cierra ID-5; cada org es un cloud aparte, NFR-11/CFG-4 → se
  puede ser `cloud-admin` en una org y `dev` en otra):
  - **`dev`** (default) — captura en el borde (CAP-*), cura **solo lo propio** (VIS-7, ID-10), lee toda la org por
    fan-out (local). **Cero cloud-side.**
  - **`project-admin`** — **todo dentro de sus proyectos asignados** (editar/borrar/auditar/re-anclar memorias
    ajenas, scopeado) **+ crear proyectos nuevos** (quedan a su cargo: self-grant) **+ linkear repos**
    (`origin`→proyecto, RES-8) en su dominio. **No** hace nada org-wide ni asigna roles.
  - **`cloud-admin`** — **todo, siempre.** Org-wide (política, keys/roles), todas las memorias, toda la estructura,
    asignar roles.
- `ADM-4` (MUST) — **Gate de autorización, centralizado y puro-cloud.** Como **toda** acción privilegiada es
  cloud-side, el local **no necesita** conocer roles (y no los conoce, ID-3/ID-7). El chequeo vive solo en el cloud:
  - **Curación por-memoria** (editar/borrar/auditar/re-anclar una memoria): *"¿tu autoridad cubre el proyecto de
    esta memoria?"* — `cloud-admin`: siempre (scope = org); `project-admin`: si el proyecto ∈ sus grants; `dev`:
    nunca.
  - **Acciones org-level** (política org, keys/roles, asignar roles): **`cloud-admin` only**.
  - **Cruces de dominio** (re-anclar o re-mapear fuera del propio dominio): **`cloud-admin`**. El **cruce de org** es
    imposible para todos — límite **estructural** (CFG-4), no un permiso que el rol pueda saltar (VIS-8).
- `ADM-5` (MUST) — **Frontera de seguridad: el rol no se auto-propaga.** Un `project-admin` puede **crear y componer
  sus propios proyectos**, pero **asignar el rol a otra persona es `cloud-admin` only**. Ampliás tu **dominio**,
  nunca el **poder de un tercero**.
- `ADM-6` (MUST) — **Login cloud e identidad** (cierra el "login admin del cloud" de NFR-5):
  - **Credenciales = email + `password_hash`**, independientes de la key de sync (ID-2/ID-8) y del password del
    visor **local** (ID-11): son **tres credenciales distintas**.
  - Login cloud **solo para el tier admin** (`cloud-admin` + `project-admin`), porque ambos curan cloud-side. El
    `dev` **no loguea al cloud** (trabaja local + agente + key de sync). **Futuro:** si el dev gana lectura cloud,
    ahí recién gana login (read-only) — aditivo.
  - **Tablas online** (nunca bajan a local): `users` (`id_dev`, `nombre_dev`, `role`, `email?`, `password_hash?`) y
    `project_admin_grants` (`id_dev`, `project_id`). A local **solo** bajan `id_dev` + `nombre_dev` (ID-3); role,
    grants, email y pw **se quedan online** (ID-5).

### Curación de memorias (cloud-side)

- `ADM-7` (MUST) — **Editar memoria ajena** (ID-10), scopeada por ADM-4. Toda edición actualiza `edited_at` y
  registra `editor_id`; si `editor_id` ≠ `author_id`, queda visible que fue **curación del admin** — la autoría
  original **no** se reescribe (ID-10). Baja por fan-out a los locales.
- `ADM-8` (MUST) — **Borrar** (DM-16), scopeado por ADM-4. Es **borrado lógico** en el cloud (`lifecycle_state =
  deleted`): retiene el contenido para auditoría (ADM-10). Baja por el cursor de pull como cualquier delta y cada
  local que la tenía **purga físicamente** el contenido y sus relaciones (SYNC-13). El `dev` **no** puede borrar
  (CAP-3/DM-16).
- `ADM-9` (MUST) — **Re-anclar memoria ajena sin el límite de 30 días** (VIS-8), scopeado por ADM-4. Para
  `project-admin`, **ambos extremos** (origen y destino) deben caer en su dominio; cruzar fuera = `cloud-admin`. El
  cruce de org sigue imposible (CFG-4). Trazado en `editor_id`/`edited_at` (ID-10).
- `ADM-10` (MUST) — **Auditoría de `deleted`** (cierra DM-14). Las memorias `deleted` retienen contenido en el cloud
  (ADM-8) y son accesibles **solo** por este path de auditoría, scopeado por ADM-4 (un `project-admin` audita las de
  sus proyectos; `cloud-admin`, todas). **Nunca** se devuelven en recall normal ni existen en el local (purgadas,
  DM-14/DM-16).

### Estructura de la org (cloud-side)

- `ADM-11` (MUST) — **Crear proyectos y linkear repos** (`origin`→proyecto, RES-8). Un `project-admin` **crea
  proyectos** (quedan a su cargo: self-grant) y **mapea repos** dentro de su dominio; el `cloud-admin`, en toda la
  org. El mapping **baja por sync** y es lo que permite al agente resolver **repo→proyecto solo** (RES-7/RES-8), sin
  config por dev y sin degradado heurístico (RES-4). **Colisión:** un `origin` es **único** (RES-7) → mapea a **un**
  proyecto; linkear un `origin` **sin mapear** es libre, sacarlo del dominio de otro = `cloud-admin`.
- `ADM-12` (MUST) — **Autorar política org** (SYNC-3b) — **`cloud-admin` only**. Es escritura **org-level**, no
  curación de una memoria de proyecto.
- `ADM-13` (MUST) — **Gestionar keys/roles y enrolamiento** (SYNC-3, ID-2/ID-5/ID-6) — **`cloud-admin` only**:
  enrolar organizaciones/devs, asignar roles y grants (ADM-5) y activar/desactivar keys (revocar sin borrar, ID-6).

## Decisiones abiertas / a definir en diseño
- Mandatos duros de seguridad/compliance (campo `fuerza` en memoria org) — diferido, v1 todo overridable.
- Motor del store local (`LOCAL-6`) — a definir en diseño.
- Stack / lenguaje — sugerencia: binario único tipo Go. No cerrado; a definir en diseño.
- Formato exacto del archivo `.memoria/config.*` — a definir en diseño.
- **Reconciliación de proyecto local provisional ↔ mapping org (cerca de RES-8)** — cuando baja un mapping
  org que cubre un repo de un proyecto local bootstrapeado (`sync_id = NULL`), el local debe **adoptar** la
  identidad del cloud (rellenar `sync_id`, tomar el nombre autoritativo) **keyeando por `origin`**, sin
  re-apuntar memorias (el id local absorbe la identidad cloud → cero rewrite de FKs). Las memorias ya subidas
  se reconcilian solas en el cloud por `origin` (no por nombre). Caso 1-repo: **automático + evento pasivo**
  en el visor (no degradado silencioso, RES-4). Caso multi-repo con destinos distintos: **partir** el proyecto
  local (resolución por `origin` o confirmación del dev) — *pendiente de cerrar*.
- **Almacenamiento de credencial — fallback local opcional (relacionado con CFG-6/LOCAL-8):** por defecto la
  credencial de org vive en el **secret store del OS** (CFG-6, protección más fuerte). **Opción opt-in:** guardarla
  también en el **local** (SQLite o archivo aparte) para entornos **sin keychain** (Linux headless) o sync
  desatendido sin depender del contexto de usuario. Debe guardarse **cifrada at-rest, NO hasheada** — un token que
  se reenvía al cloud tiene que ser **recuperable**; el hash es irreversible y solo sirve para *verificar* (como la
  password del visor, ID-11), no para *reusar*. **Caveat honesto:** sin passphrase del usuario (que rompería el sync
  desatendido), la llave de cifrado queda accesible al daemon → la protección real degrada a **permisos de archivo
  (0600) + ofuscación**, no criptografía fuerte. Trade-off seguridad/conveniencia **explícito y opt-in**; el default
  sigue siendo el keychain del OS. Cubre también la decisión de **contexto del daemon** (user-session vs system
  service, LOCAL-8).