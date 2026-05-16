# T-Display Stack

Backend services pro dashboard físico LilyGo T-Display-S3:
- **data-collector**: poller Open-Meteo, publica `data/weather/*` no MQTT.
- **telegraf-broker**: coleta CPU/mem/disk/net/uptime do host broker.
- **telegraf-router**: SNMP polling do router, publica `stats/<router>/*`.
- **firmware-server**: nginx servindo binários OTA em `:8081`.

## Pré-requisitos

- App **Mosquitto** instalado e configurado (com auth + ACL).
- Router OpenWRT com `snmpd` instalado e configurado (ler/aceitar community).

## Setup (ANTES do primeiro start)

Este app NÃO hardcoded credenciais ou IPs no compose. Você cria 3 arquivos `.env` em `${APP_DATA_DIR}` (caminho normalmente `~/umbrel/app-data/camarigor-tdisplay-stack/`).

### 1. Copie os 3 templates

Via SSH no host Umbrel:

```bash
APP_DIR=~/umbrel/app-data/camarigor-tdisplay-stack
mkdir -p $APP_DIR

# data-collector.env
cat > $APP_DIR/data-collector.env <<'EOF'
TZ=America/Sao_Paulo
MQTT_HOST=<IP_DO_BROKER>
MQTT_PORT=1883
MQTT_USER=collector-data
MQTT_PASS=<SENHA_REAL_collector-data>
OPEN_METEO_URL=https://api.open-meteo.com/v1/forecast
WEATHER_CITIES=<id1>,<id2>
WEATHER_<id1>_LABEL=NOME EXIBIDO
WEATHER_<id1>_LAT=
WEATHER_<id1>_LON=
WEATHER_<id2>_LABEL=NOME EXIBIDO
WEATHER_<id2>_LAT=
WEATHER_<id2>_LON=
POLL_INTERVAL_S=600
TOPIC_PREFIX_WEATHER=data/weather
LOG_LEVEL=INFO
EOF

# telegraf-broker.env
cat > $APP_DIR/telegraf-broker.env <<'EOF'
HOSTNAME=<hostname_do_broker>
MQTT_HOST=<IP_DO_BROKER>
MQTT_PORT=1883
MQTT_PASS=<SENHA_collector-<broker_host>>
EOF

# telegraf-router.env
cat > $APP_DIR/telegraf-router.env <<'EOF'
HOSTNAME=<hostname_do_router>
MQTT_HOST=<IP_DO_BROKER>
MQTT_PORT=1883
MQTT_PASS=<SENHA_collector-<router_host>>
SNMP_COMMUNITY=<community_snmp>
ROUTER_IP=<IP_do_router>
EOF

chmod 600 $APP_DIR/*.env
```

### 2. Copie os arquivos de config (Telegraf + nginx)

Os 3 `.conf` (`telegraf-broker.conf`, `telegraf-router.conf`, `nginx.conf`) precisam estar em `${APP_DATA_DIR}`:

```bash
# Local (na sua máquina):
scp -O umbrel-app-store-entries/camarigor-tdisplay-stack/{telegraf-broker.conf,telegraf-router.conf,nginx.conf} \
    umbrel@<UMBREL_IP>:/tmp/

# Remoto:
ssh umbrel@<UMBREL_IP> "mv /tmp/{telegraf-broker.conf,telegraf-router.conf,nginx.conf} $APP_DIR/"
mkdir -p $APP_DIR/firmware  # nginx serve OTA binaries daqui
```

### 3. Install via Umbrel UI

App Store → Community → Camarigor → T-Display Stack → Install.

### 4. Validar

```bash
mosquitto_sub -h <IP_BROKER> -u admin -P "$ADMIN_PASS" -t 'stats/+/#' -v
mosquitto_sub -h <IP_BROKER> -u admin -P "$ADMIN_PASS" -t 'data/weather/+' -v
curl -I http://<IP_BROKER>:8081/
```

## Topologia esperada

Os tópicos MQTT publicados refletem a topologia hostname declarada em cada `HOSTNAME=`:
- `stats/<broker_hostname>/{cpu,mem,disk,net,uptime}` (Telegraf system+docker)
- `stats/<router_hostname>/snmp` (Telegraf SNMP)
- `data/weather/<city_id>` (per cidade em WEATHER_CITIES)

## Atualização

`docker pull ghcr.io/camarigor/tdisplay-data-collector:latest` + restart app via Umbrel UI.
