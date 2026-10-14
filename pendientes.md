# Lista de Pendientes y Mejoras - File Browser

Documento de seguimiento de tareas, optimizaciones y mejoras arquitectónicas para el proyecto.

---

## 1. Tareas Completadas

- [x] **Eliminación total del Command Runner y Event Hooks**
  - Eliminado el paquete Go `runner/` (`commands.go`, `commands_test.go`, `parser.go`, `runner.go`) y handler websocket `http/commands.go`.
  - Eliminados componentes de interfaz: terminal emergente (`Shell.vue`), editor de comandos de usuario (`Commands.vue`), llamadas API y estilos (`_shell.css`).
  - Removidos comandos CLI `filebrowser cmds*` y flags (`--disableExec`, `--shell`, `--commands`, `--perm.execute`).
  - Purgadas dependencias en `go.mod` / `go.sum`: `github.com/flynn/go-shlex` y `github.com/gorilla/websocket`.
  - Limpieza de claves obsoletas en 33 archivos i18n y actualización de la documentación.
- [x] **Modernización de Dockerfile (Multi-stage build autónomo)**
  - **Etapa 1:** Compilación de frontend en `node:24-alpine` con `pnpm` (genera `frontend/dist`).
  - **Etapa 2:** Compilación de binario estático en `golang:1.25-alpine` incrustando `frontend/dist` (`CGO_ENABLED=0`).
  - **Etapa 3:** Obtención de certificados, dependencias y `tini-static` con `alpine:3.23`.
  - **Etapa 4:** Contenedor final ligero en `busybox:1.37.0-musl` (~16.7 MB) sin privilegios de root (UID/GID 1000).

---

## 2. Seguridad (Prioridad Alta)

- [ ] **Gestión de Sesiones y Revocación de JWTs**
  - **Diagnóstico:** Los tokens de autenticación son JWTs *stateless*. Si un usuario cambia de contraseña o hace *logout*, los tokens emitidos previamente siguen siendo válidos hasta que expire su tiempo de vida configurado.
  - **Propuesta de solución:**
    - Implementar una lista de revocación (*denylist/blocklist*) en memoria (usando el caché TTL existente) o en la base de datos.
    - O agregar un contador de versión de credenciales (`token_version`) por usuario en la base de datos y en los *claims* del JWT; al cerrar sesión o cambiar credenciales se incrementa la versión, invalidando de inmediato cualquier token anterior.

- [ ] **Autenticación Moderna (OIDC / OAuth2 / 2FA)**
  - **Diagnóstico:** El sistema actual solo ofrece autenticación interna por contraseña, modo sin autenticación y lectura de cabeceras de proxy inverso.
  - **Propuesta de solución:**
    - Agregar integración nativa con OpenID Connect (OIDC) para proveedores de identidad estándar (Keycloak, Google, GitHub, Authelia, Authentik, Microsoft Entra ID).
    - Implementar doble factor de autenticación (2FA / TOTP) mediante códigos temporales (RFC 6238) y/o llaves de seguridad WebAuthn / Passkeys.

---

## 3. Rendimiento y Escalabilidad (Prioridad Media / Alta)

- [ ] **Motor de Búsqueda Indexado**
  - **Diagnóstico:** La búsqueda actual (`search/search.go`) recorre recursivamente el sistema de archivos (`afero.Walk`) en tiempo real cada vez que un usuario realiza una consulta. En carpetas masivas o discos de red (NFS/SMB), esto satura el I/O del servidor.
  - **Propuesta de solución:**
    - Crear un índice de búsqueda en segundo plano utilizando SQLite FTS5, Bleve (librería nativa de Go) o integración con Meilisearch.
    - Mantener el índice actualizado de manera reactiva mediante eventos del sistema de archivos o interceptando las operaciones de la API.

- [ ] **Paginación y Virtual Scrolling en Directorios Masivos**
  - **Diagnóstico:** El endpoint `/api/resources` entrega todo el contenido de una carpeta en una única respuesta JSON. En directorios con miles de archivos, se generan cuellos de botella en memoria y la interfaz de Vue se vuelve lenta al renderizar.
  - **Propuesta de solución:**
    - Modificar la API para admitir parámetros de paginación (`limit`, `offset` o cursor).
    - Implementar *virtual scrolling* (mediante `@vueuse/core` o librerías de virtual list) en el componente de listado de archivos en el frontend.

- [ ] **Persistencia Desacoplada (Soporte SQL / Multi-instancia)**
  - **Diagnóstico:** BoltDB mantiene un bloqueo exclusivo a nivel de archivo sobre `filebrowser.db`, lo que impide desplegar múltiples réplicas detrás de un balanceador de carga o implementar alta disponibilidad (HA).
  - **Propuesta de solución:**
    - Abstraer la capa de almacenamiento (`users.Store`, `settings.Store`, `share.Store`) para permitir motores de bases de datos relacionales (PostgreSQL, MySQL) o SQLite con replicación distribuida (LiteFS / rqlite).

- [ ] **Soporte de Almacenamiento en la Nube (S3 / MinIO / Cloudflare R2)**
  - **Diagnóstico:** El almacenamiento está acoplado al disco local del servidor.
  - **Propuesta de solución:**
    - Permitir montar *buckets* compatibles con la API de Amazon S3 como backend de archivos, aprovechando la abstracción de Afero o un adaptador directo con el SDK oficial.

---

## 4. Experiencia de Usuario y Nuevas Funcionalidades (Prioridad Media)

- [ ] **Papelera de Reciclaje (Soft Delete)**
  - **Diagnóstico:** Al eliminar un archivo se llama directamente a `os.RemoveAll`, provocando la pérdida irreversible e instantánea del recurso sin posibilidad de recuperación ante errores accidentales.
  - **Propuesta de solución:**
    - Implementar una papelera de reciclaje (`.trash` oculta por usuario o a nivel de sistema).
    - Añadir botones para "Restaurar" y "Vaciar papelera", con purga automática programable tras X días.

- [ ] **Editor de Código Moderno (Monaco Editor)**
  - **Diagnóstico:** Ace Editor ofrece prestaciones básicas comparado con editores web modernos.
  - **Propuesta de solución:**
    - Migrar de Ace Editor a Monaco Editor (el editor de Visual Studio Code) para dotar a la plataforma de mejor resaltado de sintaxis, minimapa, autocompletado avanzado y posibilidad de conectar con servidores de lenguaje (LSP).

- [ ] **Visualización de Documentos Ofimáticos**
  - **Diagnóstico:** Actualmente los documentos de oficina (`.docx`, `.xlsx`, `.pptx`, `.odt`) solo pueden descargarse.
  - **Propuesta de solución:**
    - Integrar visores en el navegador (JavaScript / WASM) para hojas de cálculo y documentos de texto en modo solo lectura, o soporte para el protocolo WOPI (Collabora Online / OnlyOffice).

---

## 5. Modernización de Código y Mantenimiento

- [ ] **Migración a Enrutador HTTP Nativo (Go 1.22+)**
  - **Diagnóstico:** Se utiliza la librería externa `gorilla/mux`. Desde Go 1.22, el enrutador estándar `net/http` incluye soporte nativo para métodos y coincidencia de patrones en rutas.
  - **Propuesta de solución:**
    - Reemplazar `gorilla/mux` por `net/http.ServeMux` nativo o un router ultraligero y mantenido como `chi`, reduciendo dependencias externas.

