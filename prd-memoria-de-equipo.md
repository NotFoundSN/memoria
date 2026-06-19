# PRD — Memoria de equipo (contexto de proyectos con IA)

> **Estado:** en revisión activa. Última sesión: 2026-06-18.
> **Sesión 2026-06-16 mañana:** Búsqueda, Sync, No-funcionales.
> **Sesión 2026-06-16 tarde:** Modelo de datos — entidad Memory completa, topic como string, tag como valor único,
> lista preferente de tags (v1), relaciones entre memorias, lifecycle de memorias,
> SLA de búsqueda unscoped cualificado.
> **Sesión 2026-06-17:** Correcciones de consistencia — `supersede` → `replaces` (VAL-2, VAL-3, CAP-3);
> conflicto de sync: lifecycle_state=conflict + relación `caused` automática (SYNC-9, DM-12);
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
> backstop de `replaces` vía conflicto (DM-12), `diverges` sin backstop (juicio puro); UI local edita relaciones propias.
> Identidad/login local (gap #5 cerrado): first-run del visor pide **nombre + contraseña** (ID-11) — contraseña = gate
> del visor, nombre = identidad local; el MCP no usa login. `author_id` local = sentinel reservado **`-1`** (no `0` ni
> `null`); el cloud reescribe al `id_dev` real desde la key al subir (ID-12); "propia" = `-1` o mi `id_dev` remapeado.
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
  - **`caused`** — A originó a B (ej: una decisión de diseño causó un bug). No requiere que ambas existan: B puede apuntar a una causa no documentada.
  - **`diverges`** — B (proyecto/repo) se sale de la norma establecida por A (org) en ese contexto específico. A sigue `active` y vigente para el resto. El motivo va en el texto de B (SEM-6).
- `DM-11` (MUST — principio) — Una relación existe **solo si agrega información que el anclaje no provee por sí solo.** No todas las memorias necesitan estar relacionadas.
  Se descartan: `conflicts_with` (si dos memorias se contradicen, una reemplaza a la otra), `related` (sin caso concreto que topic+tag no cubra → ruido).
- `DM-12` (MUST) — Cuando el cloud detecta un conflicto de sync, **marca la memoria nueva con
  `lifecycle_state = conflict`** y crea automáticamente una relación `caused` (nueva → vieja ganadora).
  El `lifecycle_state` es la señal para distinguir esta arista de una `caused` creada por un dev.
  Se resuelve desde la UI (CAP-3): el dev decide cuál reemplaza a cuál y crea la relación `replaces` correspondiente.

### Lifecycle de memorias

- `DM-13` (MUST) — Una memoria puede estar en uno de estos estados:

  | Estado | Descripción |
  |--------|-------------|
  | `draft` | Borrador de sesión activa. Solo vive en local, nunca se sincroniza al cloud. Aparece en búsquedas locales. |
  | `active` | Vigente. Devuelta por defecto en búsquedas. |
  | `replaced` | Fue reemplazada por otra vía relación `replaces`. Disponible como historial bajo demanda. |
  | `inactive` | Dada de baja manualmente sin reemplazo. Caso típico: norma de org que se elimina sin sucesor. |
  | `conflict` | Conflicto de sync pendiente de resolución desde la UI (CAP-3). |
  | `deleted` | Borrado **cloud-only** (como `upload_at`, DM-15). En cloud: borrado **lógico** — retiene el contenido para auditoría del admin y lo oculta de todo recall. En el local NO es un estado guardado: dispara la **purga física** del contenido. Exclusivo del admin (DM-16). |

- `DM-14` (MUST) — La búsqueda devuelve memorias `active` y `draft` por defecto. `replaced`, `inactive` y `conflict` disponibles bajo demanda (parámetro explícito). `deleted` **NUNCA** se devuelve: no existe en el local (purgado) y en cloud solo es accesible por el path de auditoría del admin (DM-16).

### Campos de la entidad Memory

- `DM-15` (MUST) — Campos de la entidad Memory:

  | Campo | Descripción |
  |-------|-------------|
  | `id` | UUID v7 (time-ordered). Generado localmente al crear la memoria. El cloud usa el mismo ID como canónico. Sin ID local/cloud separados. |
  | `title` | Título de la memoria |
  | `content` | Contenido |
  | `topic` | String libre — el "sobre qué". Opcional. |
  | `tag` | Un solo valor del vocabulario controlado (CON-8) |
  | `author_id` | Autor original. Trazabilidad únicamente. Seguridad se gestiona via key en el push. |
  | `editor_id` | Quién editó por última vez. Si ≠ `author_id`, fue curación del admin (ID-10). |
  | `org` | Siempre presente — tenant boundary |
  | `proyecto` | Opcional — parte del anclaje |
  | `repo` | Opcional — parte del anclaje |
  | `lifecycle_state` | draft / active / replaced / inactive / conflict / deleted (este último cloud-only, DM-16) |
  | `created_at` | Timestamp de creación en el local |
  | `edited_at` | Timestamp de la última edición (ID-1). Nullable si nunca se editó. |
  | `upload_at` | Timestamp asignado por el cloud al recibir la memoria. Solo vive en cloud. |

> La organización es el **tenant boundary** del sistema (aislamiento de datos y de búsqueda).

### Borrado de memorias

- `DM-16` (MUST) — **El borrado es exclusivo del admin, vía cloud.** El dev no puede borrar (CAP-3). El líder
  técnico con rol permisado borra una memoria desde la **UI admin del cloud** (login cloud, no la key de sync;
  mismo path que la escritura org-level, SYNC-3b). Motivos típicos: la memoria **no refleja lo que realmente pasó**, contiene **datos sensibles**
  de una persona o empresa, o **envenena el proyecto** y no se quiere seguir alimentándolo.
  - **Cloud → borrado lógico:** `lifecycle_state = deleted`. Retiene el contenido para auditoría del admin y lo
    oculta de todo recall (DM-14, SRCH-6).
  - **Local → purga física:** al bajar el delta (SYNC-13) el local **borra físicamente** el contenido. No queda
    estado: la memoria desaparece del store local.
  - **Relaciones:** al purgar la memoria en el local se **eliminan también las relaciones que la apuntaban**, en
    ambas direcciones. No se conserva un puntero a una memoria que ya no existe.

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
  - **El admin (líder técnico) puede editar cualquier memoria**, incluidas las de otros, para curación/de-dup.
    Ocurre **cloud-side** (UI admin + login cloud, mismo path privilegiado que DM-16/SYNC-3b) y baja por fan-out
    a los locales.
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
  `no_clasificado` ≠ sin tag: es una **señal diagnóstica** (si abundan, hay locales con la tabla vieja).
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
- `CFG-3` (MUST) — Una org puede ser **local-only** (sin URL, sin sync): proyectos solo locales.
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

---

## 12. Mecánica de captura

- `CAP-1` (MUST) — Captura **silenciosa y basada en sesión**: la IA acumula contexto durante la
  conversación sin interrumpir. **Sesión = conversación con el agente.**
  - La sesión existe como **borrador (draft)** en el store local desde el inicio. El agente lo va
    actualizando a medida que ocurren cosas. Al cierre se finaliza como memoria permanente.
  - Ventaja: si la conversación se corta, el draft preserva lo acumulado hasta ese momento.
  - **Sub-agentes:** guardan su memoria al terminar la tarea, con el tag que corresponde a lo que hicieron.
  - **Agente principal:** actualiza el draft durante la sesión y lo finaliza al cierre.
- `CAP-2` (MUST) — El contenido que la síntesis debe cubrir: decisión, bugfix (con causa raíz), gotcha,
  convención, intento-descartado, hito de proceso… (catálogo definido por la skill, LOCAL-5).
- `CAP-3` (MUST) — Control **post-hoc** en la UI: **revisar, editar (in-place, ver CAP-12), fusionar/reemplazar.** El dev edita **solo sus propias** memorias (ID-10) y **NO puede borrar** — el borrado es exclusivo del admin vía cloud (DM-16).
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
- `CAP-11` (MUST) — **Reanudación de sesión:** si el dev retoma una conversación interrumpida (corte de luz,
  crash, cierre inesperado), el agente detecta el draft activo existente y lo continúa en lugar de crear uno
  nuevo. La memoria y el contexto de la sesión se reanudan desde donde quedaron.
- `CAP-10` (MUST) — **Draft huérfano:** si un borrador de sesión lleva más de **5 días sin cerrarse**,
  el sistema lo finaliza automáticamente. Criterio: cubre fines de semana y feriados largos (hasta 4 días)
  sin penalizar al dev que retoma el lunes o el martes.
  el contexto de negocio y los involucrados son señal, no riesgo; el residuo semántico real es ultra angosto
  y queda cubierto por A + curación post-hoc. El caso de alta severidad (credencial viva) lo garantiza B.

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

- `CAP-13` (MUST — degradación elegante) — La autoría de relaciones (`extends`, `diverges`, `replaces`) en
  captura es **best-effort del agente**: el agente las **propone** con el contexto fresco de la sesión. Pero
  **la captura NUNCA se bloquea ni se pierde por no poder resolver una relación.** Principio:
  **relación faltante > memoria faltante.** Si el agente no reconoce el target o duda, guarda la memoria
  **igual, sin la arista**; jamás "no pude relacionarla, no la guardo". Coherente con DM-11 (una relación
  existe solo si agrega info) y con la filosofía no-destructiva.
  - **Backstop para `replaces`:** una memoria guardada sin su `replaces` no queda huérfana. Si nace otra con
    **mismo anclaje + topic + tag**, el mecanismo de conflicto (DM-12) la marca `conflict` en el sync y la
    sube a la UI para que el dev resuelva (CAP-3). El `replaces` que el agente no puso, el sistema lo caza
    río abajo.
  - **`diverges` no tiene backstop automático:** reconocer que la org tiene una norma y que el contexto
    actual se desvía es **juicio puro del agente** (vive en la skill, LOCAL-5). Si no lo reconoce, se guarda
    la memoria sin la arista y el dev la agrega a mano (CAP-14).
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
- `SRCH-6` (MUST) — Recall por defecto devuelve memorias `active` + `draft`. `replaced`, `inactive` y
  `conflict` disponibles bajo demanda como parámetro explícito. `deleted` nunca se devuelve (no existe en el
  local; en cloud solo vía auditoría admin, DM-16). Resuelto por la DB, no por el modelo (VAL-1/3).

---

## 14. Sync

> Modelo: local es la tabla de verdad del dev. El cloud es fuente de verdad compartida. El sync es bidireccional.

- `SYNC-4` (MUST) — Ciclo periódico, configurable por el dev (ver LOCAL-9). Valor por defecto: ~10 minutos. El local corre como daemon persistente.
- `SYNC-5` (MUST) — **Pull:** delta de memorias + relaciones. Scope: org-level + **todos los proyectos
  de la org** (no solo los del dev). ~150MB estimado para la org de referencia (75K memorias), tamaño manejable.
  > Topics no se sincronizan por separado: viajan como string en la memoria. La lista de topics conocidos
  > se deriva localmente con una query al store (DM-9).
- `SYNC-6` (MUST) — **Push:** memorias nuevas/editadas (`active`, `replaced`, `inactive`, `conflict`) + relaciones
  acumuladas desde el último sync. Los `draft` **nunca se sincronizan** — son locales hasta que se finalizan.
- `SYNC-7` (MUST) — **Cursor de delta:** el cloud asigna `upload_at` a cada memoria al recibirla.
  Los clientes usan ese campo como cursor ("dame todo con `upload_at` > mi último sync").
  `upload_at` vive **solo en el cloud**, nunca se replica a local. Resuelve el caso del dev
  que estuvo offline días y subió memorias "viejas" — otros las bajan igual porque el cursor
  es cuándo llegaron al cloud, no cuándo se crearon. Elimina también el clock skew entre máquinas.
- `SYNC-8` (MUST) — **Primera sync:** al instalar y configurar credenciales, baja todas las memorias
  de la org desde el inicio (sin delta).
  > **Nota de implementación:** definir estrategia de paginación/chunking para orgs grandes cuando se programe.
- `SYNC-9` (MUST) — **Conflictos de edición:** como solo el autor edita su memoria (ID-10), el caso "dos devs
  editan la misma memoria" **no puede ocurrir entre devs distintos**; queda acotado al **mismo autor en dos
  máquinas** (multi-device). Si llegan dos ediciones del mismo `id`, la API compara timestamps: la primera gana
  `active`, la segunda queda `lifecycle_state = conflict` + relación `caused` automática hacia la ganadora (DM-12).
  El dev lo resuelve desde la UI (CAP-3). La edición del **admin** sobre memoria ajena es cloud-side y baja por
  pull → su conflicto con una edición local no pusheada del autor lo maneja SYNC-12.
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
- `SYNC-12` (MUST) — **Conflicto detectado en pull:** si el pull trae una versión actualizada de una memoria que el dev también editó localmente (aún no pusheada), el local aplica la versión cloud y marca la edición local como `conflict`. El dev resuelve desde la UI antes de que esa memoria se pushee.
- `SYNC-13` (MUST) — **Borrado admin (fan-out):** cuando el admin borra una memoria (DM-16), el cloud la marca `deleted` y le asigna un `upload_at` nuevo. Baja por el cursor de pull como cualquier delta; cada local que la tenía **purga físicamente** el contenido y las relaciones que la apuntaban (DM-16). El **first-sync** (SYNC-8) no envía memorias `deleted` — en un local nuevo no hay nada que purgar. Como el ciclo es **pull primero** (SYNC-11), el borrado le gana a una edición local no pusheada: el local purga y descarta su edición.

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

## Decisiones abiertas / a definir en diseño
- Mandatos duros de seguridad/compliance (campo `fuerza` en memoria org) — diferido, v1 todo overridable.
- Motor del store local (`LOCAL-6`) — a definir en diseño.
- Stack / lenguaje — sugerencia: binario único tipo Go. No cerrado; a definir en diseño.
- Formato exacto del archivo `.memoria/config.*` — a definir en diseño.
- **Visor web de memorias** — UI local que consume la API local (SQLite). Requiere su propia sección de requerimientos (qué muestra, qué acciones permite el dev, relación con CAP-3).
