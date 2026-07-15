# Implementación de una Infraestructura Open Source para Recolección de Datos con K3s

![Estado](https://img.shields.io/badge/estado-en%20desarrollo-yellow)
![Licencia](https://img.shields.io/badge/licencia-MIT-blue)
![K3s](https://img.shields.io/badge/K3s-v1.29-informational)
![KoboToolbox](https://img.shields.io/badge/KoboToolbox-latest-success)

## Objetivo

Implementar una infraestructura distribuida basada en tecnologías Open Source para el despliegue de **KoboToolbox** sobre un clúster **K3s** de dos nodos, incorporando administración remota (NetBird), monitoreo (Uptime Kuma), respaldos automatizados y documentación técnica reproducible.

## Tabla de contenidos

- [Arquitectura general](#arquitectura-general)
- [Requisitos previos](#requisitos-previos)
- [Etapa 1. Diseño de infraestructura](#etapa-1-diseño-de-infraestructura)
- [Etapa 2. Instalación del sistema operativo](#etapa-2-instalación-del-sistema-operativo)
- [Etapa 3. Configuración de red](#etapa-3-configuración-de-red)
- [Etapa 4. Instalación de Docker](#etapa-4-instalación-de-docker)
- [Etapa 5. Migración de servicios](#etapa-5-migración-de-servicios)
- [Etapa 6. Implementación del clúster K3s](#etapa-6-implementación-del-clúster-k3s)
- [Etapa 7. Despliegue de KoboToolbox](#etapa-7-despliegue-de-kobotoolbox)
- [Etapa 8. Configuración online / offline](#etapa-8-configuración-online--offline)
- [Etapa 9. Instalación de NetBird](#etapa-9-instalación-de-netbird)
- [Etapa 10. Configuración Wake on LAN](#etapa-10-configuración-wake-on-lan)
- [Etapa 11. Estrategia de backups](#etapa-11-estrategia-de-backups)
- [Etapa 12. Actualización de servicios](#etapa-12-actualización-de-servicios)
- [Etapa 13. Documentación con Git](#etapa-13-documentación-con-git)
- [Etapa 14. Implementación de Uptime Kuma](#etapa-14-implementación-de-uptime-kuma)
- [Etapa 15. Sistema de notificaciones](#etapa-15-sistema-de-notificaciones)
- [Etapa 16. Pruebas](#etapa-16-pruebas)
- [Etapa 17. Resultados](#etapa-17-resultados)
- [Estructura del repositorio](#estructura-del-repositorio)
- [Licencia](#licencia)

---

## Arquitectura general

```
                              Internet
                                 │
                          ┌──────────────┐
                          │   NetBird     │
                          │  (VPN mesh)   │
                          └──────┬────────┘
                                 │
              ┌──────────────────┴───────────────────┐
              │                                        │
     ┌──────────────────┐                   ┌──────────────────┐
     │       PC1          │                   │       PC2          │
     │ Master K3s          │◄─────────────────►│ Worker K3s         │
     │ Docker Engine       │   Red interna     │ Docker Engine      │
     │ KoboToolbox (pods)  │   K3s (flannel)   │ Réplica / Pods     │
     │ Uptime Kuma         │                   │                    │
     │ IP: 192.168.1.10    │                   │ IP: 192.168.1.11   │
     └──────────────────┘                   └──────────────────┘
```

**Componentes:**

| Componente     | Función                                   | Nodo        |
|----------------|--------------------------------------------|-------------|
| K3s Master     | Control plane, `kube-apiserver`, `etcd`     | PC1         |
| K3s Worker     | Ejecución de pods, balanceo                 | PC2         |
| KoboToolbox    | Recolección de datos (formularios ODK)      | PC1 / PC2   |
| PostgreSQL     | Base de datos relacional de KoboToolbox     | PC1         |
| MongoDB        | Almacenamiento de submissions               | PC1         |
| NetBird        | VPN mesh para administración remota         | PC1 y PC2   |
| Uptime Kuma    | Monitoreo de servicios                      | PC1         |

---

## Requisitos previos

- 2 equipos (físicos o VM) con mínimo 4 GB RAM / 2 vCPU cada uno.
- Ubuntu Server 22.04 LTS instalado en ambos.
- Acceso a Internet durante la instalación.
- Usuario con privilegios `sudo`.
- Dominio o subdominio (opcional, para Ingress con TLS).

---

## Etapa 1. Diseño de infraestructura

### Objetivo
Definir la arquitectura física y lógica antes de tocar una sola máquina.

### Tabla de direccionamiento

| Host | Rol         | IP interna      | IP NetBird      | Hostname     |
|------|-------------|-----------------|------------------|--------------|
| PC1  | Master K3s  | 192.168.1.10/24 | 100.64.0.1       | k3s-master   |
| PC2  | Worker K3s  | 192.168.1.11/24 | 100.64.0.2       | k3s-worker   |

### Entregables
- Diagrama físico y lógico (ver carpeta `docs/arquitectura/`).
- Tabla de direccionamiento (arriba).
- Definición de roles y flujo de información (Master expone servicios, Worker replica carga).

---

## Etapa 2. Instalación del sistema operativo

### Objetivo
Preparar ambos servidores con Ubuntu Server.

### Actividades

```bash
# Durante la instalación de Ubuntu Server, habilitar OpenSSH Server

# Configurar hostname
sudo hostnamectl set-hostname k3s-master   # en PC1
sudo hostnamectl set-hostname k3s-worker   # en PC2

# Crear usuario de administración
sudo adduser devops
sudo usermod -aG sudo devops

# Configurar SSH (deshabilitar login root y password si se usan llaves)
sudo nano /etc/ssh/sshd_config
# PermitRootLogin no
# PasswordAuthentication no   (solo si ya configuraste llaves)
sudo systemctl restart ssh

# Configurar firewall básico
sudo ufw allow OpenSSH
sudo ufw allow 6443/tcp   # API server K3s
sudo ufw allow 8472/udp   # VXLAN (flannel)
sudo ufw allow 10250/tcp  # kubelet
sudo ufw enable

# Configurar zona horaria
sudo timedatectl set-timezone America/Guayaquil
```

### Validación

```bash
ping -c 4 192.168.1.11        # desde PC1 hacia PC2
ssh devops@192.168.1.11       # acceso remoto
hostname                       # debe mostrar k3s-master / k3s-worker
timedatectl                    # verificar zona horaria y sincronización NTP
```

---

## Etapa 3. Configuración de red

### Objetivo
Garantizar conectividad estable entre nodos.

### Actividades

```bash
# IP fija con netplan (Ubuntu Server)
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 192.168.1.10/24     # 192.168.1.11 en PC2
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

```bash
sudo netplan apply

# Resolución de nombres entre nodos
sudo nano /etc/hosts
```

```
192.168.1.10   k3s-master
192.168.1.11   k3s-worker
```

### Validación

```bash
ping k3s-worker
traceroute k3s-worker
ip route
ip addr show eth0
```

---

## Etapa 4. Instalación de Docker

### Objetivo
Preparar el entorno de contenedores en ambos nodos.

### Actividades

```bash
# Instalación oficial de Docker Engine
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Permitir uso de docker sin sudo
sudo usermod -aG docker $USER
newgrp docker
```

### Validación

```bash
docker --version
docker compose version
docker ps
docker images
docker network ls
```

---

## Etapa 5. Migración de servicios

### Objetivo
Contenerizar los servicios previos a su despliegue en K3s (fase intermedia con Docker Compose antes de migrar a manifiestos Kubernetes).

### `docker/docker-compose.yml`

```yaml
version: "3.9"

services:
  postgres:
    image: postgres:14
    container_name: kobo-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: kobo
      POSTGRES_USER: kobo
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - kobo-net

  mongo:
    image: mongo:6
    container_name: kobo-mongo
    restart: unless-stopped
    volumes:
      - mongo_data:/data/db
    networks:
      - kobo-net

  redis:
    image: redis:7
    container_name: kobo-redis
    restart: unless-stopped
    networks:
      - kobo-net

volumes:
  postgres_data:
  mongo_data:

networks:
  kobo-net:
    driver: bridge
```

### `docker/.env.example`

```
POSTGRES_PASSWORD=changeme
```

### Validación

```bash
docker compose --env-file .env -f docker/docker-compose.yml up -d
docker logs kobo-postgres --tail 50
docker inspect kobo-mongo | grep IPAddress
```

---

## Etapa 6. Implementación del clúster K3s

### Objetivo
Crear un clúster K3s de dos nodos (1 master, 1 worker).

### Instalación del Master (PC1)

```bash
curl -sfL https://get.k3s.io | sh -s - \
  --write-kubeconfig-mode 644 \
  --tls-san 192.168.1.10

# Obtener el token para unir el worker
sudo cat /var/lib/rancher/k3s/server/node-token
```

### Instalación del Worker (PC2)

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.10:6443 \
  K3S_TOKEN=<TOKEN_OBTENIDO_EN_MASTER> sh -
```

### Configurar `kubectl` local

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
```

### Validación

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl cluster-info
kubectl get componentstatuses
```

---

## Etapa 7. Despliegue de KoboToolbox

### Objetivo
Publicar la plataforma KoboToolbox sobre K3s.

### `k3s/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kobo
```

### `k3s/postgres-statefulset.yaml`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: kobo
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:14
          envFrom:
            - secretRef:
                name: kobo-secrets
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

### `k3s/kobo-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kobotoolbox
  namespace: kobo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: kobotoolbox
  template:
    metadata:
      labels:
        app: kobotoolbox
    spec:
      containers:
        - name: kobotoolbox
          image: kobotoolbox/kpi:latest
          ports:
            - containerPort: 80
          envFrom:
            - secretRef:
                name: kobo-secrets
---
apiVersion: v1
kind: Service
metadata:
  name: kobotoolbox-svc
  namespace: kobo
spec:
  selector:
    app: kobotoolbox
  ports:
    - port: 80
      targetPort: 80
```

### `k3s/ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kobo-ingress
  namespace: kobo
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  rules:
    - host: kobo.midominio.org
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kobotoolbox-svc
                port:
                  number: 80
  tls:
    - hosts:
        - kobo.midominio.org
      secretName: kobo-tls
```

### Aplicar manifiestos

```bash
kubectl apply -f k3s/namespace.yaml
kubectl create secret generic kobo-secrets \
  --namespace kobo \
  --from-literal=POSTGRES_PASSWORD=changeme \
  --from-literal=DJANGO_SECRET_KEY=$(openssl rand -hex 32)
kubectl apply -f k3s/postgres-statefulset.yaml
kubectl apply -f k3s/kobo-deployment.yaml
kubectl apply -f k3s/ingress.yaml
```

### Validación

```bash
kubectl get pods -n kobo
kubectl get svc -n kobo
kubectl get ingress -n kobo
curl -I http://kobo.midominio.org
```

- Acceso web: abrir `http://kobo.midominio.org` en el navegador.
- Login con usuario administrador creado durante el primer arranque.
- Crear un formulario de prueba y verificar que se guarda correctamente.

---

## Etapa 8. Configuración online / offline

### Objetivo
Validar el funcionamiento de KoboToolbox en modo desconectado (KoboCollect en Android).

### Actividades

1. Instalar **KoboCollect** en un dispositivo Android.
2. Configurar el servidor: `https://kobo.midominio.org`.
3. Descargar formularios con conexión activa.
4. Desconectar el dispositivo de la red (modo avión).
5. Completar el formulario sin conexión.
6. Reconectar y sincronizar (`Enviar formularios finalizados`).

### Validación

```bash
# En el servidor, revisar que la submission llegó
kubectl exec -n kobo -it deploy/kobotoolbox -- python manage.py shell -c \
  "from kpi.models import Asset; print(Asset.objects.count())"
```

- Sin Internet: el formulario se completa y queda almacenado localmente en el dispositivo.
- Con Internet: la sincronización sube las respuestas al servidor.
- Exportación: descargar datos en formato XLS/CSV desde la interfaz web (`Exportar > XLS`).

---

## Etapa 9. Instalación de NetBird

### Objetivo
Habilitar administración remota mediante una VPN mesh.

### Actividades

```bash
# Crear cuenta en https://app.netbird.io (o self-hosted)

# Instalar agente en cada nodo
curl -fsSL https://pkgs.netbird.io/install.sh | sh

# Iniciar sesión y unir el dispositivo
sudo netbird up
```

### Configurar rutas (acceso a la red interna 192.168.1.0/24 desde afuera)

Desde el panel de NetBird: **Network Routes > Add Route** → `192.168.1.0/24`, asignando PC1 como *routing peer*.

### Validación

```bash
netbird status
ping 100.64.0.2          # IP NetBird del worker
ssh devops@100.64.0.1    # acceso remoto vía VPN
```

---

## Etapa 10. Configuración Wake on LAN

### Objetivo
Permitir el encendido remoto de los equipos.

### Actividades

```bash
# Habilitar WoL en la NIC
sudo apt install ethtool -y
sudo ethtool -s eth0 wol g

# Hacerlo persistente con netplan
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  ethernets:
    eth0:
      wakeonlan: true
```

```bash
sudo netplan apply

# Habilitar también en BIOS/UEFI: "Wake on LAN" / "Power On by PCI-E" = Enabled
```

### Encender el equipo remotamente (desde otro host en la misma red o vía NetBird)

```bash
sudo apt install wakeonlan -y
wakeonlan AA:BB:CC:DD:EE:FF
```

### Validación

```bash
ip link show eth0 | grep -i wol
wakeonlan AA:BB:CC:DD:EE:FF
ping 192.168.1.11   # confirmar que el equipo encendió
```

---

## Etapa 11. Estrategia de backups

### Objetivo
Garantizar la recuperación ante fallos.

### `scripts/backup.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

FECHA=$(date +%Y%m%d_%H%M%S)
DEST="/backups/$FECHA"
mkdir -p "$DEST"

echo ">> Respaldando PostgreSQL"
kubectl exec -n kobo statefulset/postgres -- pg_dumpall -U kobo > "$DEST/postgres.sql"

echo ">> Respaldando MongoDB"
kubectl exec -n kobo deploy/mongo -- mongodump --archive > "$DEST/mongo.archive"

echo ">> Respaldando manifiestos K3s"
cp -r k3s/ "$DEST/k3s"

echo ">> Respaldando volúmenes Docker (si aplica en fase intermedia)"
docker run --rm -v postgres_data:/data -v "$DEST":/backup alpine \
  tar czf /backup/postgres_data.tar.gz -C /data .

echo ">> Backup completo en $DEST"
```

```bash
chmod +x scripts/backup.sh
```

### Automatizar con cron

```bash
crontab -e
# Ejecutar backup todos los días a las 2:00 AM
0 2 * * * /home/devops/repo/scripts/backup.sh >> /var/log/backup.log 2>&1
```

### Restauración

```bash
kubectl exec -i -n kobo statefulset/postgres -- psql -U kobo < /backups/20260101_020000/postgres.sql
kubectl exec -i -n kobo deploy/mongo -- mongorestore --archive < /backups/20260101_020000/mongo.archive
```

### Validación
Restaurar en un entorno de prueba y confirmar el conteo de registros y el acceso funcional a KoboToolbox.

---

## Etapa 12. Actualización de servicios

### Objetivo
Mantener la infraestructura actualizada.

```bash
# Docker
sudo apt update && sudo apt install --only-upgrade docker-ce docker-ce-cli containerd.io -y

# K3s
curl -sfL https://get.k3s.io | sh -

# KoboToolbox (nueva imagen)
kubectl set image deployment/kobotoolbox kobotoolbox=kobotoolbox/kpi:latest -n kobo
kubectl rollout status deployment/kobotoolbox -n kobo

# NetBird
sudo netbird update

# Uptime Kuma
docker compose -f uptime-kuma/docker-compose.yml pull
docker compose -f uptime-kuma/docker-compose.yml up -d
```

Verificar compatibilidad de versiones antes de actualizar en producción; probar primero en un entorno de staging.

---

## Etapa 13. Documentación con Git

### Objetivo
Mantener trazabilidad del proyecto.

```bash
git init
git add .
git commit -m "chore: estructura inicial del repositorio"

git branch -M main
git remote add origin https://github.com/<usuario>/<repositorio>.git
git push -u origin main

# Flujo de ramas
git checkout -b feature/etapa-7-despliegue-kobo
git commit -m "feat: manifiestos de despliegue de KoboToolbox"
git push origin feature/etapa-7-despliegue-kobo
# Pull Request hacia main

# Versionado semántico con tags
git tag -a v1.0.0 -m "Primera versión estable del clúster"
git push origin v1.0.0
```

### Archivos de trazabilidad

- `README.md` — este documento.
- `CHANGELOG.md` — historial de cambios (formato [Keep a Changelog](https://keepachangelog.com/es-ES/1.1.0/)).
- `LICENSE` — licencia del proyecto (MIT sugerida).
- `CONTRIBUTING.md` — guía de contribución.
- `docs/` — documentación detallada por etapa.

---

## Etapa 14. Implementación de Uptime Kuma

### Objetivo
Monitoreo continuo de todos los servicios.

### `uptime-kuma/docker-compose.yml`

```yaml
version: "3.9"

services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - uptime-kuma-data:/app/data

volumes:
  uptime-kuma-data:
```

```bash
docker compose -f uptime-kuma/docker-compose.yml up -d
```

### Monitores a configurar (desde la interfaz web `http://<IP_PC1>:3001`)

| Tipo   | Destino                          | Intervalo |
|--------|------------------------------------|-----------|
| HTTP   | `https://kobo.midominio.org`      | 60 s      |
| TCP    | `192.168.1.10:6443` (API K3s)     | 60 s      |
| Ping   | `192.168.1.11` (worker)           | 30 s      |
| Docker | Contenedores `kobo-postgres`, etc. | 60 s      |

### Validación

```bash
# Generar caída controlada para probar la alerta
kubectl scale deployment/kobotoolbox -n kobo --replicas=0
# Esperar la alerta en Uptime Kuma, luego restaurar:
kubectl scale deployment/kobotoolbox -n kobo --replicas=2
```

---

## Etapa 15. Sistema de notificaciones

### Objetivo
Recibir alertas ante caídas de servicio.

### Configurar en Uptime Kuma → **Settings > Notifications**

**Telegram:**
1. Crear bot con `@BotFather`, obtener el token.
2. Obtener el `chat_id` (con `@userinfobot` o la API de Telegram).
3. En Uptime Kuma: Notification Type = Telegram, ingresar `Bot Token` y `Chat ID`.

**Discord:**
1. Crear un Webhook en el canal: **Configuración del canal > Integraciones > Webhooks**.
2. En Uptime Kuma: Notification Type = Discord, pegar la URL del webhook.

**Email (SMTP):**
```
Notification Type: Email (SMTP)
Host: smtp.gmail.com
Port: 587
Security: STARTTLS
Usuario/Contraseña: <credenciales de aplicación>
```

**Webhook genérico:**
```
Notification Type: Webhook
URL: https://mi-endpoint.example.com/alertas
Content Type: application/json
```

### Validación

```bash
kubectl scale deployment/kobotoolbox -n kobo --replicas=0
# Confirmar recepción de alerta en Telegram/Discord/Email
kubectl scale deployment/kobotoolbox -n kobo --replicas=2
# Confirmar notificación de recuperación
```

---

## Etapa 16. Pruebas

### Pruebas funcionales

```bash
docker ps                                 # servicios activos
kubectl get nodes,pods,svc -A             # estado del clúster
curl -I https://kobo.midominio.org        # KoboToolbox responde
netbird status                            # VPN activa
wakeonlan AA:BB:CC:DD:EE:FF                # encendido remoto
git log --oneline -5                      # historial de commits
curl http://<IP_PC1>:3001                 # Uptime Kuma responde
```

### Pruebas de integración

```bash
kubectl exec -n kobo deploy/kobotoolbox -- ping -c 3 postgres
kubectl get pvc -n kobo                   # persistencia de volúmenes
kubectl get endpoints kobotoolbox-svc -n kobo   # balanceo entre réplicas
```

### Pruebas de disponibilidad

```bash
# Reinicio de nodo worker
ssh devops@192.168.1.11 "sudo reboot"
kubectl get nodes -w                      # observar reincorporación al clúster

# Reinicio de servicio
kubectl rollout restart deployment/kobotoolbox -n kobo

# Restauración de backup (ver Etapa 11)

# Reconexión NetBird
sudo netbird down && sudo netbird up
netbird status
```

Registrar los resultados de cada prueba (éxito/falla, tiempo de recuperación) en `docs/pruebas/`.

---

## Etapa 17. Resultados

Documentar en `docs/anexos/` y `docs/pruebas/`:

- Evidencias fotográficas / capturas de pantalla de cada etapa validada.
- Logs relevantes (`kubectl logs`, `docker logs`, `/var/log/backup.log`).
- Tablas comparativas de tiempos de respuesta y disponibilidad.
- Métricas de Uptime Kuma (uptime %, tiempo de respuesta promedio).
- Conclusiones sobre la viabilidad de la infraestructura Open Source para recolección de datos en campo.

---

## Estructura del repositorio

```text
.
├── README.md
├── LICENSE
├── CHANGELOG.md
├── CONTRIBUTING.md
├── docs/
│   ├── arquitectura/          # diagramas físico/lógico, tabla de direccionamiento
│   ├── instalacion/           # guías paso a paso de SO, red, Docker
│   ├── configuracion/         # configuración de K3s, Kobo, NetBird, WoL
│   ├── mantenimiento/         # backups, actualizaciones
│   ├── pruebas/                # resultados de pruebas funcionales/integración/disponibilidad
│   └── anexos/                 # evidencias, capturas, métricas
├── docker/
│   ├── docker-compose.yml
│   └── .env.example
├── k3s/
│   ├── namespace.yaml
│   ├── postgres-statefulset.yaml
│   ├── kobo-deployment.yaml
│   └── ingress.yaml
├── kubernetes/                 # manifiestos adicionales (cert-manager, storage class, etc.)
├── kobo/                        # configuraciones específicas de KoboToolbox
├── netbird/                      # configuración de rutas y agentes
├── wol/                           # scripts de Wake on LAN
├── uptime-kuma/
│   └── docker-compose.yml
├── backups/                       # (gitignored) respaldos generados
├── scripts/
│   └── backup.sh
└── images/                          # capturas usadas en la documentación
```

### `.gitignore` recomendado

```
backups/
*.env
.env
*.log
kubeconfig
```

---

## Licencia

Este proyecto se distribuye bajo la licencia MIT. Ver el archivo [`LICENSE`](LICENSE) para más detalles.
