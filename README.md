> [!WARNING]
> **File Browser fue archivado oficialmente el 2026-09-01**. Este repositorio funciona como un **fork optimizado, modernizado y asegurado** para producción y uso personal.

<p align="center">
  <img src="./branding/banner.png" width="550"/>
</p>

File Browser proporciona una interfaz web moderna, ligera y privada para administrar archivos dentro de un directorio específico de tu servidor: permite subir, descargar, previsualizar, editar y compartir archivos fácilmente (una alternativa ligera a Nextcloud o Google Drive).

---

## 🚀 Cambios y Optimizaciones en este Fork

### 1. Eliminación total del Command Runner y Event Hooks (Seguridad Crítica)
* **Eliminado el vector principal de RCE:** Se eliminó por completo el paquete de ejecución de comandos (`runner/`), el endpoint `/api/command` y las llamadas intermedias `d.RunHook(...)`.
* **Interfaz y CLI depurados:** Se eliminaron la terminal emergente (`Shell.vue`), el selector de comandos de usuario (`Commands.vue`), los botones de shell y los comandos CLI `filebrowser cmds*` junto con sus banderas (`--disableExec`, `--shell`, `--commands`, `--perm.execute`).
* **Dependencias y estilos purgados:** Removidos `github.com/flynn/go-shlex` y `github.com/gorilla/websocket`, así como `_shell.css` y claves obsoletas en los 33 archivos de traducción i18n.

### 2. Dockerfile Multi-etapa Autónomo
* **Compilación integral sin dependencias en el host:**
  * **Etapa 1 (`frontend-builder`):** Compilación del frontend con `node:24-alpine` y `pnpm` (genera `frontend/dist`).
  * **Etapa 2 (`backend-builder`):** Compilación estática del binario Go con `golang:1.25-alpine` incrustando los assets (`CGO_ENABLED=0`).
  * **Etapa 3 (`fetcher`):** Obtención de certificados SSL y utilidades en `alpine:3.23`.
  * **Etapa 4:** Contenedor final minimalista en `busybox:1.37.0-musl` (~16.7 MB) ejecutado como usuario sin privilegios (`UID=1000`).

### 3. Pila de Despliegue Seguro (`compose.yaml`)
* **Eliminación de Redis:** Se eliminó el contenedor de Redis al ser redundante para instancias individuales; File Browser utiliza de forma nativa su caché en memoria interna de alto rendimiento.
* **Reverse Proxy Nginx integrado:**
  * **Aislamiento:** File Browser no expone puertos al host, solo se comunica con Nginx dentro de la red interna de Docker.
  * **Rate Limiting:** Protección anti fuerza bruta en `/api/login` (máximo 5 peticiones/minuto) y anti DoS en `/api/search` (máximo 10 peticiones/minuto).
  * **Optimización de transferencias:** Soporte para subida de archivos gigantes (`client_max_body_size 0;`) y transmisión en streaming sin búfer en disco (`proxy_request_buffering off;`).
  * **Cabeceras de seguridad:** Protección X-Frame-Options, X-Content-Type-Options, X-XSS-Protection y Referrer-Policy.
* **Protección con Fail2ban:** Monitorización de logs de Nginx para bloquear mediante `iptables` a cualquier IP que falle 3 intentos de inicio de sesión durante 1 hora.

---

## 📦 Despliegue Rápido con Docker Compose

Para levantar el entorno completo con File Browser, Nginx y Fail2ban:

```bash
docker compose up -d --build
```

El servicio estará disponible en el puerto **80** (o el puerto configurado en [compose.yaml](compose.yaml)).

---

## 📋 Lista de Tareas y Mejoras Futuras

Para consultar el roadmap técnico y las próximas tareas planificadas (gestión de sesiones JWT, autenticación OIDC/2FA, motor de búsqueda indexado, papelera de reciclaje y Monaco Editor), consulta:

👉 [**pendientes.md**](pendientes.md)

---

## 🔒 Consideraciones de Seguridad

- **Sesiones y JWTs:** Los tokens de sesión son autónomos (*stateless*). Se recomienda mantener tiempos de expiración prudentes (`--tokenExpirationTime`).
- **Exposición a Internet:** Utiliza el proxy Nginx provisto con certificados TLS/HTTPS (o mediante Cloudflare Tunnels / VPN Tailscale) antes de exponer el servicio a redes públicas.

---

## Documentación

* [Instalación](docs/installation.md)
* [Personalización](docs/customization.md)
* [Autenticación](docs/authentication.md)
* [Uso de CLI](docs/cli/filebrowser.md)
* [Despliegue](docs/deployment.md)

---

## Licencia

[Apache License 2.0](LICENSE) © File Browser Contributors
