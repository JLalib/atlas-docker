# 🗺️ Atlas Docker - Network Discovery & Visualization

[![GitHub](https://img.shields.io/badge/GitHub-karam--ajaj%2Fatlas-blue?logo=github)](https://github.com/karam-ajaj/atlas)
[![Docker Hub](https://img.shields.io/badge/Docker%20Hub-keinstien%2Fatlas-blue?logo=docker)](https://hub.docker.com/r/keinstien/atlas)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](https://opensource.org/licenses/MIT)

## 📋 Descripción general

**Atlas** es una herramienta full-stack de descubrimiento y visualización de redes autohospedada que escanea automáticamente infraestructura (contenedores Docker, hosts físicos, múltiples subnets) y mapea la topología completa en un grafo interactivo con indicadores de estado en tiempo real. Ideal para homelabs, PYMES y monitoreo de infraestructura compleja.

Combina **escaneo automático de red** con **visualización gráfica interactiva** en una sola solución ligera: backend Go ultra-rápido, API REST FastAPI y frontend React moderno. Todo desplegable con un solo contenedor Docker.

## ✨ Características principales

- **Escaneo automático de infraestructura**: Inspección de contenedores Docker (multi-red, IPs/MACs múltiples), hosts físicos (ping + ARP), subnets configurables
- **Grafo interactivo de topología**: Nodos hosts/contenedores, edges conexiones, zoom/pan/drag (basado en Vis.js)
- **Indicadores de estado en vivo**: Online/offline en tiempo real, última actividad, health checks visuales
- **Multi-interfaz y multi-subnet**: Escanea múltiples redes (192.168.x.0/24, 10.0.0.0/24, etc.), hosts en múltiples networks aparecen separados con labels de interfaz
- **Autenticación JWT opcional**: Tokens con TTL configurable (24h por defecto), login gate antes del dashboard
- **Backend Go ultra-rápido**: Escaneos eficientes, alto rendimiento, baja latencia
- **API REST FastAPI**: Query de datos, trigger manual de escaneos, integrable en aplicaciones externas
- **UI React moderna**: Dashboard responsivo, modos dark/light, interfaz intuitiva
- **Escaneos programados automáticos**: Fast (1h), Docker (1h), Deep (2h) vía timers Go integrados (sin cron)
- **Persistencia SQLite**: Datos locales, históricos, queries built-in, backup trivial
- **Ligero**: Bajo consumo CPU/RAM (típicamente 150-300MB RAM)
- **Open Source MIT**: Código abierto, community-driven, desarrollo activo

## 📋 Requisitos del sistema

- Docker (Docker Compose v2+ recomendado)
- 200 MB - 1 GB RAM mínimo (backend + DB)
- 500 MB espacio disco (imagen + SQLite DB)
- Puerto 8888 (UI) + 8889 (API) - configurables
- `CAP_ADD NET_RAW` + `NET_ADMIN` (para escaneo de red)
- `network_mode: host` (para acceso completo a red)
- Acceso a Docker socket (`/var/run/docker.sock` montado)
- Navegador moderno (Chrome, Firefox, Safari, Edge)

## 🐳 Instalación

### Opción 1: Docker run simple (recomendado)

```bash
docker run -d \
  --name atlas \
  --network=host \
  --cap-add=NET_RAW \
  --cap-add=NET_ADMIN \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e ATLAS_UI_PORT=8888 \
  -e ATLAS_API_PORT=8889 \
  keinstien/atlas:latest
```

**Acceso**: http://localhost:8888

---

### Opción 2: Docker Compose completo (independiente)

```bash
mkdir -p atlas && cd atlas
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  atlas:
    image: keinstien/atlas:latest
    container_name: atlas
    restart: unless-stopped
    network_mode: host
    cap_add:
      - NET_RAW
      - NET_ADMIN
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - atlas_data:/config
    environment:
      # Puertos (UI + API)
      - ATLAS_UI_PORT=8888
      - ATLAS_API_PORT=8889
      # Autenticación (opcional)
      - ATLAS_ADMIN_USER=admin
      - ATLAS_ADMIN_PASSWORD=cambiar_despues_a_fuerte
      - ATLAS_AUTH_TTL_SECONDS=86400
      # Intervalos scans (segundos)
      - FASTSCAN_INTERVAL=3600
      - DOCKERSCAN_INTERVAL=3600
      - DEEPSCAN_INTERVAL=7200
      # Subnets a escanear (auto-detect si no especificado)
      - SCAN_SUBNETS=192.168.1.0/24,10.0.0.0/24
      - TZ=Europe/Madrid
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8888"]
      interval: 30s
      timeout: 5s
      retries: 3

volumes:
  atlas_data:
EOF

docker compose up -d
```

---

### Opción 3: Con Caddy HTTPS reverse proxy

```bash
cat >> docker-compose.yml << 'EOF'

  caddy:
    image: caddy:latest
    container_name: atlas-caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config

volumes:
  caddy_data:
  caddy_config:
EOF

cat > Caddyfile << 'EOF'
atlas.tudominio.com {
  reverse_proxy localhost:8888
}
EOF

docker compose up -d
```

**Accesos**:
- `http://localhost:8888` - UI Local
- `http://localhost:8889` - API Local
- `https://atlas.tudominio.com` - HTTPS remoto (con Caddy)

## ⚙️ Configuración

1. **Puertos UI/API**: `ATLAS_UI_PORT=8888`, `ATLAS_API_PORT=8889`
2. **Autenticación**: `ATLAS_ADMIN_USER=admin`, `ATLAS_ADMIN_PASSWORD=contraseña_fuerte`, `ATLAS_AUTH_TTL_SECONDS=86400` (24h)
3. **Intervalos de escaneo**: `FASTSCAN_INTERVAL=3600`, `DOCKERSCAN_INTERVAL=3600`, `DEEPSCAN_INTERVAL=7200` (segundos)
4. **Subnets a escanear**: `SCAN_SUBNETS=192.168.1.0/24,10.0.0.0/24,172.16.0.0/16` (auto-detecta si no se especifica)
5. **Zona horaria**: `TZ=Europe/Madrid`
6. **Persistencia**: Volumen `atlas_data:/config` para SQLite DB
7. **Permisos de red**: `network_mode: host` + `cap_add: NET_RAW, NET_ADMIN` obligatorios
8. **Docker socket**: `/var/run/docker.sock:/var/run/docker.sock` para descubrimiento de contenedores

## 🚀 Primeros pasos

1. **Acceder a la Web UI**
   - Abre `http://localhost:8888`
   - Si auth habilitado: ingresa usuario/contraseña
   - Dashboard con grafo de red aparece (escaneo en progreso)

2. **Ver topología de red en vivo**
   - Grafo interactivo muestra: hosts, contenedores, subnets
   - Nodos: azul (online), gris (offline)
   - Zoom/pan con mouse
   - Click nodo → detalles host/contenedor

3. **Ver status hosts + contenedores**
   - Panel dashboard: lista hosts y contenedores Docker
   - Status: online/offline, última actividad
   - IP, MAC, interfaces listadas
   - Logs de contenedores integrados (si Docker)

4. **Ver detalles multi-interfaz**
   - Host conectado a múltiples redes aparece como nodos separados
   - Labels muestran nombre interfaz (eth0, vlan1, etc.)
   - Perfecto para visualizar VLANs, VPNs, multi-homed hosts

5. **Triggear escaneo manual**
   - Dashboard → "Scan Now" o "Refresh"
   - API: `POST http://localhost:8889/api/scan`
   - Escaneo inicia (fast + docker + deep secuencial)

6. **Query API (integración externa)**
   ```bash
   # Lista todos hosts
   curl http://localhost:8889/api/hosts
   
   # Lista contenedores Docker
   curl http://localhost:8889/api/containers
   
   # Trigger escaneo
   curl -X POST http://localhost:8889/api/scan
   ```

7. **Configurar autenticación**
   - Editar variables de entorno: `ATLAS_ADMIN_PASSWORD=contraseña_fuerte`
   - Restart container
   - Login gate aparece antes del dashboard
   - Sesiones TTL 24h (configurable `ATLAS_AUTH_TTL_SECONDS`)

8. **Configurar subnets a escanear**
   - `SCAN_SUBNETS=192.168.1.0/24,10.0.0.0/24,172.16.0.0/16`
   - Si no especificado, auto-detecta subnet local

9. **Ver logs de escaneo**
   ```bash
   docker logs -f atlas
   # Muestra: scanning subnets, hosts found, containers detected
   ```

10. **Backup DB (importante)**
    ```bash
    docker cp atlas:/config/db/atlas.db ./atlas-backup-$(date +%Y%m%d).db
    ```

## 💡 Casos de uso

- **Homelabs**: Visibilidad total red + contenedores, topología en un vistazo
- **Redes PYMES**: Descubrimiento automático infraestructura, monitoring distribuido
- **Multi-sitio**: Escanea redes remotas vía subnets configurables, agregación central
- **Troubleshooting**: Ver topología conectividad, identificar hosts sin red
- **Dashboards centralizados**: API integrable en Grafana, Home Assistant, dashboards custom
- **Orquestación contenedores**: Monitor Docker containers + hosts físicos unificado

## 🔒 Acceso remoto seguro

Para exposición pública segura, usa **Caddy reverse proxy** (Opción 3) que proporciona:
- HTTPS automático con Let's Encrypt
- Termination TLS en edge
- Configuración trivial (solo dominio en Caddyfile)

Alternativas: Traefik, Nginx Proxy Manager, Cloudflare Tunnel.

**Nunca expongas puertos 8888/8889 directamente a Internet sin TLS y autenticación fuerte.**

## 🛠️ Gestión y mantenimiento

```bash
# Ver logs en tiempo real
docker logs -f atlas

# Restart container
docker compose restart atlas

# Actualizar a versión más reciente
docker pull keinstien/atlas:latest
docker compose up -d

# Monitorear consumo (ultra-ligero)
docker stats atlas
# Típicamente: 150-300MB RAM

# Acceder a SQLite DB directamente
docker exec atlas sqlite3 /config/db/atlas.db
# SQL queries directo
# SELECT ip, name, interface_name FROM hosts ORDER BY ip;

# Resetear base de datos
docker exec atlas rm /config/db/atlas.db
docker compose restart atlas
# Próximo escaneo recrea DB

# Ver versión instalada
docker inspect keinstien/atlas:latest | grep -i version
```

## 📝 Licencia

MIT License - Código abierto, uso comercial permitido, modificación y distribución libre.

Ver [LICENSE](https://github.com/karam-ajaj/atlas/blob/main/LICENSE) en el repositorio oficial.

---

> **Referencia**: Basado en el post [**Cómo instalar Atlas en Docker - Descubrimiento y visualización de redes autohospedada**](https://genbyte.blogspot.com/2026/08/como-instalar-atlas-en-docker.html) de Genbyte.
>
> **Repositorio oficial**: [karam-ajaj/atlas](https://github.com/karam-ajaj/atlas) | **Docker Hub**: [keinstien/atlas](https://hub.docker.com/r/keinstien/atlas)