# Revisión de exposición de datos sensibles

- **Repositorio:** `andresochoa1/google-workspace-mcp-secure` (público)
- **Fecha de la revisión:** 2026-10-29
- **Alcance:** árbol actual + historial completo de git (todas las ramas y etiquetas publicadas)
- **Regla de enmascarado:** ningún valor se reproduce completo; solo se muestran sus 4 primeros caracteres seguidos de `…`.

## 1. Resumen

| | |
|---|---|
| Commits revisados | **1 898** (1 472 normales + 426 de merge) |
| Referencias revisadas | 167 ramas remotas y 92 etiquetas |
| Rango de fechas | 2025-04-27 → 2026-04-08 |
| Blobs (versiones de archivo) revisados | 2 790, más el contenido descomprimido de las 57 versiones del paquete `google_workspace_mcp.dxt` |
| Secretos reales encontrados | **2** (en un mismo archivo comprimido, en un solo commit, ambos de un tercero) |
| Datos personales / de clientes | Correos de colaboradores del proyecto original; ningún dato de clientes del dueño |
| Árbol actual (`main`) | **Limpio**: ningún secreto |

**Contexto clave.** El repositorio es un *fork* (GitHub lo marca así: `fork: true`, creado el 2026-04-11) del proyecto público `taylorwilsdon/google_workspace_mcp`. **Ninguno de los 1 898 commits es del dueño**: no hay commits con su nombre ni con su correo, y la última actividad del historial (2026-04-08) es anterior a la fecha de creación del fork. Todo lo que aparece abajo viene del proyecto original, y por eso ya estaba publicado allí antes de que existiera este fork.

**Recomendación: dejar público** (o archivarlo si el dueño ya no lo va a usar). **No compensa limpiar el historial.** Los motivos están en la sección 5.

## 2. Hallazgos

### H1. Token de GitHub (`ghu_`) dentro del paquete `.dxt`. Severidad: alta en origen, baja hoy (ver nota)

| Campo | Valor |
|---|---|
| Archivo | `google_workspace_mcp.dxt` (zip) → dentro: `.mcpregistry_github_token` |
| Commit | `7cc281e26a42f4f2d2a2dafed7b8a7838ef2fb8d` (2026-02-15, autor: mantenedor del proyecto original, mensaje "dxt") |
| Tipo | Token de usuario de GitHub App (*user-to-server*, prefijo `ghu_`), 40 caracteres. Es el que crea `mcp-publisher login github` para publicar en el MCP Registry |
| Valor enmascarado | `ghu_…` |
| Alcance | Solo en esa versión del `.dxt`. Las otras 56 versiones del paquete no lo contienen. El archivo `.dxt` se borró en el commit `438bc1d` (2026-03-28), pero esa versión sigue en el historial, accesible desde `main`, desde 38 ramas más y desde las etiquetas `v1.11.2` en adelante |

**Nota.** Los tokens `ghu_` de GitHub Apps suelen caducar a las 8 horas cuando la app tiene activada la caducidad. Además, el proyecto original añadió `.mcpregistry_*` a `.dxtignore` el 2026-02-17 (commit `b86dba4`), dos días después de la fuga, lo que indica que la detectó. **No comprobé si el token sigue siendo válido**: hacerlo exige usar una credencial ajena contra la API de GitHub, y eso no corresponde.

### H2. JWT del MCP Registry dentro del mismo paquete. Severidad: baja (caducado)

| Campo | Valor |
|---|---|
| Archivo | `google_workspace_mcp.dxt` → dentro: `.mcpregistry_registry_token` |
| Commit | `7cc281e26a42f4f2d2a2dafed7b8a7838ef2fb8d` |
| Tipo | JWT emitido por `mcp-registry`, con permisos de publicación |
| Valor enmascarado | `{"to…` (JSON que envuelve el JWT) |
| Caducidad | `exp` = 2026-02-13 19:40:48 UTC (5 minutos después de emitirse). **Caducado.** |

### H3. Log de depuración con datos de uso de Gmail. Severidad: baja (dato personal de un tercero)

| Campo | Valor |
|---|---|
| Archivo | `mcp_server_debug.log` (408 líneas) |
| Commits | añadido en `a92748b`, borrado en `11e5516` (ambos 2025-07-14) |
| Tipo | Correo corporativo de un colaborador del proyecto original (`jihu…`, 42 apariciones), rutas locales de su equipo, 21 IDs de hilos y mensajes de Gmail y las consultas `in:inbox` e `is:unread in:inbox` |
| Lo que **no** contiene | Ni tokens, ni asuntos, ni remitentes, ni cuerpos de correo (lo comprobé con búsquedas de `ya29.`, `1//`, `refresh_token`, `Bearer`, `subject`, `from:` y `snippet`) |

El mismo correo aparece en los metadatos del autor de esos commits, así que su exposición no depende del archivo.

### H4. Otros correos de terceros en el contenido. Severidad: informativa

| Archivo(s) | Commit de ejemplo | Dato (enmascarado) | Qué es |
|---|---|---|---|
| `.beads/issues.jsonl` (ya ignorado) | `9dc9b1c` y anteriores | `tbar…` (Gmail) | Correo personal del mantenedor original, campo `owner` del gestor de tareas *beads*. También está en los metadatos de sus commits |
| `AGENTS.md`, `PROGRESS.md` | `e930874` (2026-01-20) | `jh@c…`, `info…` (dominio de empresa) | Cuentas de un colaborador usadas como ejemplo en un plan de pruebas multicuenta. El primero también está en los metadatos de sus commits |
| `tests/gmail/test_message_body_content.py` | `9451ffa` | `Pilo…` (lista de correo de una asociación) | Pie de una lista de correo pública, usado como dato de prueba |
| `pyproject.toml`, `manifest.json`, `SECURITY.md`, `helm-chart/.../Chart.yaml` | — | `tayl…` | Contactos públicos del mantenedor (los declara él mismo) |

Los otros 38 correos encontrados son de ejemplo (`example.com`, `yourdomain.com`, `user@…`, `your-email@gmail.com`) o datos sintéticos de los tests (`evil…`, `msg1…`, `x00…`).

### Falsos positivos descartados

| Detección | Herramienta | Por qué no es un secreto |
|---|---|---|
| 10 "tokens de GitHub" de 40 caracteres hexadecimales (`da65…`, `5461…`, `b0be…`, etc.) en 14 versiones del `.dxt` | trufflehog | Son hashes SHA-1 de `.mypy_cache/**/*.meta.json` (caché de mypy que se coló en el paquete) |
| `curl-auth-header` en `README.md`, línea 1039, commit `241f0987ae` | gitleaks | Ejemplo de documentación: `Bearer ya29…` de 14 caracteres. Un token real de Google mide más de 150 |
| `"type": "service_account"` en `README.md` | barrido propio | Ejemplo con valores `"..."` |
| `client_secret`, `refresh_token`, etc. en `.env.oauth21`, `auth/`, `core/server.py` y `tests/` | barrido propio | Marcadores (`your…`, `not-…`, `dumm…`, `${us…`) y datos falsos de los tests |
| `.credentials/.test` | revisión de rutas | Archivo vacío (0 bytes) |
| `.vscode/launch.json`, `.vscode/mcp.json`, `.mcp.json`, `helm-chart/.../secret.yaml` | revisión de rutas | Configuración sin credenciales (solo rutas locales y puertos) o plantillas de Helm |

### Lo que no apareció

En ninguna versión de ningún archivo ni en los mensajes de commit aparecen claves de API de Google (`AIza…`), secretos de cliente OAuth (`GOCSPX-…`), IDs de cliente OAuth reales (`*.apps.googleusercontent.com`), tokens de acceso o de refresco de Google (`ya29.…`, `1//0…`), claves privadas PEM, JSON de cuentas de servicio con clave, claves de AWS, Slack, OpenAI o Anthropic, ni PATs clásicos de GitHub. Tampoco hay datos de clientes del dueño.

## 3. Método

1. **Historial completo.** El clon inicial era superficial (63 commits), así que lo completé con `git fetch --unshallow` y traje todas las ramas y etiquetas: **1 898 commits** alcanzables desde 167 ramas y 92 etiquetas. El remoto no publica referencias `refs/pull/*`.
2. **gitleaks 8.21.2** sobre `git log -p --all` (diffs de los 1 472 commits normales; gitleaks informa de 1 404 commits con contenido escaneable) y en modo `dir` sobre el árbol actual. Resultado: 1 detección en el historial (falso positivo) y 0 en el árbol.
3. **trufflehog 3.88.0** (`git file://…`, **sin verificación** para no usar credenciales ajenas). Descomprime archivos, y así encontró H1 dentro del `.dxt`. Resultado: 141 detecciones, de las que 140 son los hashes de mypy y 1 es el token real.
4. **Barrido propio** de los **2 790 blobs** del repositorio con `git cat-file --batch-all-objects`. Cubre también lo que solo entra por commits de merge, que `git log -p` no muestra. Usé expresiones regulares para credenciales de Google, GitHub, AWS, Slack, OpenAI/Anthropic, JWT, claves privadas y asignaciones genéricas (`client_secret=…`, `password=…`), más extracción de correos.
5. **Las 57 versiones del `.dxt`**, descomprimidas una por una, con revisión de nombres de archivo sensibles (`.mcpregistry_*`, `.env`, `client_secret*`, `credentials.json`, `token.json`, `*.pem`, `*.key`, `*.p12`). Solo apareció la versión de `7cc281e`.
6. **Rutas sensibles**: lista de los 260 nombres de archivo que han existido en el historial e inspección manual de los sospechosos (`.env*`, `.credentials/`, `*.log`, `.vscode/`, `.mcp.json`, `.beads/`).
7. **Mensajes de commit**: las mismas expresiones regulares sobre `git log --all --format=%B`. Sin resultados.
8. **Autoría**: 98 autores y 101 correos distintos en los metadatos. Ninguno es del dueño.

## 4. Límites

- No comprobé si el token `ghu_…` (H1) sigue activo: hacerlo implica usar una credencial de un tercero.
- No revisé el contenido de *issues*, wikis, *releases* ni *Actions* de GitHub, solo git. El fork tiene las *issues* desactivadas.
- No comparé este historial con el del repositorio original, porque está fuera del alcance de esta sesión. La afirmación de que todo viene del original se basa en que GitHub marca el repo como fork, en los autores, en las fechas y en los mensajes de merge (`Merge branch … of github.com:taylorwilsdon/google_workspace_mcp`).
- Las herramientas automáticas no detectan secretos con formatos propios. El barrido propio cubre los patrones habituales, no todos los posibles.

## 5. Recomendación: dejar público (o archivar si no se usa). No limpiar el historial

**Por qué no compensa limpiar el historial:**

1. **No hay nada del dueño expuesto.** No hay commits suyos, ni credenciales suyas, ni datos de sus clientes.
2. **Limpiar este fork no retira la exposición.** El mismo historial, con el mismo `.dxt`, es el del repositorio original público. Además, GitHub comparte objetos entre los repos de una misma red de forks, así que un commit puede seguir accesible por su hash aunque se reescriba aquí. Reescribir solo daría una falsa sensación de limpieza y rompería la relación con el original.
3. **Los secretos encontrados probablemente ya no sirven.** El JWT está caducado. El token `ghu_` es de corta duración por diseño y el original reaccionó a la fuga dos días después. Aun así, esto último no lo pude verificar.

**Cuándo convendría archivar.** El fork no tiene cambios propios, aunque el nombre (`-secure`) sugiere que se pensaba endurecerlo. Si ya no hay intención de trabajar en él, archivarlo (o borrarlo) reduce superficie sin perder nada, porque el original sigue existiendo. Si se va a usar, puede seguir público.

**Acciones opcionales (las decide el dueño):**

- Avisar al mantenedor del proyecto original del token del commit `7cc281e`, si no lo revocó ya. Lo adecuado es por su canal de seguridad (`SECURITY.md`), no en una *issue* pública.
- Activar en este repo *Secret scanning* y *Push protection* (Settings → Code security) para frenar fugas futuras si se empieza a trabajar en él.
- Antes de cualquier cambio propio, mantener `.env`, `credentials*.json`, `.mcpregistry_*`, `*.log` y los artefactos `.dxt` fuera de git, como ya hacen `.gitignore` y `.dxtignore`.
