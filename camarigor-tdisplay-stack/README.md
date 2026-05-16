# T-Display Stack

Backend services pro dashboard físico LilyGo T-Display-S3:
- **data-collector**: poller Open-Meteo, publica `data/weather/*` no MQTT.
- **telegraf-broker**: coleta CPU/mem/disk/net/uptime do host broker.
- **telegraf-router**: SNMP polling do router, publica `stats/<router>/*`.
- **firmware-server**: nginx servindo binários OTA em `:8081`.

## Pré-requisitos

- App **Mosquitto** instalado e configurado (com auth + ACL).
- Router OpenWRT com `snmpd` instalado e configurado (ler/aceitar community).

## Install

App Store → Community → Camarigor → T-Display Stack → Install.

Na primeira inicialização, o init container cria os 3 arquivos de configuração (`telegraf-broker.conf`, `telegraf-router.conf`, `nginx.conf`) em `${APP_DATA_DIR}` automaticamente.

## Setup (DEPOIS do install)

Por questões de validação do Umbrel (env_file paths são checados pre-start), as variáveis de ambiente ficam inline no `docker-compose.yml`. Após install, edite o compose deployado:

```bash
ssh umbrel@<UMBREL_IP>
APP_DIR=~/umbrel/app-data/camarigor-tdisplay-stack
vi $APP_DIR/docker-compose.yml
```

Preencha os campos vazios em cada service:

### Service `camarigor-tdisplay-stack` (data-collector)

```yaml
environment:
  TZ: "America/Sao_Paulo"
  LOG_LEVEL: "INFO"
  MQTT_HOST: "<IP_DO_BROKER>"
  MQTT_PORT: "1883"
  MQTT_USER: "collector-data"
  MQTT_PASS: "<SENHA_collector-data>"
  OPEN_METEO_URL: "https://api.open-meteo.com/v1/forecast"
  POLL_INTERVAL_S: "600"
  TOPIC_PREFIX_WEATHER: "data/weather"
  WEATHER_CITIES: "<id1>,<id2>"
  # Adicione 1 par LABEL/LAT/LON por id:
  WEATHER_<id1>_LABEL: "NOME EXIBIDO"
  WEATHER_<id1>_LAT: "<latitude>"
  WEATHER_<id1>_LON: "<longitude>"
  WEATHER_<id2>_LABEL: "NOME EXIBIDO"
  WEATHER_<id2>_LAT: "<latitude>"
  WEATHER_<id2>_LON: "<longitude>"
```

### Service `_telegraf-broker`

```yaml
environment:
  HOSTNAME: "<hostname_do_broker>"
  MQTT_HOST: "<IP_DO_BROKER>"
  MQTT_PORT: "1883"
  MQTT_USER: "collector-<hostname_broker>"
  MQTT_PASS: "<SENHA_collector-broker>"
```

### Service `_telegraf-router`

```yaml
environment:
  HOSTNAME: "<hostname_do_router>"
  MQTT_HOST: "<IP_DO_BROKER>"
  MQTT_PORT: "1883"
  MQTT_USER: "collector-<hostname_router>"
  MQTT_PASS: "<SENHA_collector-router>"
  SNMP_COMMUNITY: "<community_snmp_v2c>"
  ROUTER_IP: "<IP_do_router>"
```

Restart o app via Umbrel UI após salvar.

## Validar

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

## Update

A edição manual no `docker-compose.yml` do app-data sobrevive uninstall/reinstall apenas se o app-data não for limpo. Pra updates de imagem:

```bash
docker pull ghcr.io/camarigor/tdisplay-data-collector:latest
docker pull ghcr.io/camarigor/tdisplay-init:latest
```

E restart o app via Umbrel UI.

## Notas técnicas

- v1.1.0: env vars inline. Tentativa anterior com `env_file:` apontando pra arquivos criados via init container falhou — Umbrel valida paths pre-start, antes do init rodar.
- O init container apenas semeia configs Telegraf/nginx (idempotente, preserva edições do user).
- Hardening: read_only rootfs + tmpfs + no-new-privileges em todos os services.
