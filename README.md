# DevOps-Traefik


// Создание контейнера
docker-compose.yml


// ------------------------------------------------------------------


# Сервисы
services:
  # Название сервиса 
  reverse-proxy:
    # The official v3 Traefik docker image
    image: traefik:v3.4
    container_name: traefik
    # Enables the web UI and tells Traefik to listen to docker - парамметры вынесены в traefik.yml файл
    # command: 
      #- "--api.insecure=false" 
      #- "--providers.docker"
      # Уровень логов для откладки
      #- "--log.level=DEBUG"
      # Не публиковать Docker-контейнеры по умолчанию. То есть, контейнеры не будут автоматически доступны через Traefik, если явно не указать, что они должны быть "exposed". Это повышает безопасность, так как не все контейнеры автоматически становятся доступными извне.
      #- "--providers.docker.exposedByDefault=false"
      # Заставляет Traefik использовать конкретную Docker-сеть (в этом случае — proxynet) при создании маршрутов к контейнерам.
      #- "--providers.docker.network=proxynet"
      # Entrypoints:
      #- "--entrypoints.http.address=:80"
      #- "--entrypoints.https.address=:443"
      #- "--entrypoints.http.http.redirections.entrypoint.to=https"
      #- "--entrypoints.http.http.redirections.entrypoint.scheme=https"
      # SSL Let's encrypt
      #- "--entrypoints.https.http.tls.certResolver=le"
      #- "--certificatesresolvers.le.acme.tlschallenge=true"
      #- "--certificatesresolvers.le.acme.email=test@test.lv"
      #- "--certificatesresolvers.le.acme.storage=/letsencrypt/acme.json"
     labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.rule=Host(`traefik.test.lv`)"
      - "traefik.http.routers.traefik.entrypoints=https"
      - "traefik.http.routers.traefik.service=api@internal"
      - "traefik.http.routers.traefik.middlewares=auth"
    ports:
      # The HTTP port
      - "80:80"
        # The HTTPS port
      - "443:443"
      # The Web UI (enabled by --api.insecure=true)
      #- "8080:8080"
    volumes:
      # So that Traefik can listen to the Docker events
      # Делаем проброс сокет для докера, чтобы наш Traefik видел бы на этом сервере другие контейнеры
      - /var/run/docker.sock:/var/run/docker.sock
      # Только для чтения (read-only) — защита от случайной перезаписи
      - ./traefik.yml:/traefik.yml:ro
      - ./letsencrypt:/letsencrypt
#Сеть которая будет управляться с помощью Traefik, те контейнеры которые будут помещаться в эту сеть, Traefik сможет перебрасывать на них запросы.
networks:
 default:
  # Название сети, если она не создана, её нужно будет создать.
  name: proxynet
  external: true

// Запуск контейнера
>> docker compose up
// Создание сети
>> docker network create proxynet




// Конфигурация Nginx - nginx-compose.yml

version: '3'
services:
 nginx:
  image: nginx:latest
  container_name: nginx
  lables:
   # контейнер указывает Traefik, что этот контейнер можно проксировать, используется в связке с --providers.docker.exposedByDefault=false
   - "traefik.enable=true"
   - "traefik.http.routes.nginx.rule=Host('nginx.test.lv')"
# Запуск в сети контейнера proxynet, где находится и наш Traefik
networks:
 default:
  name: proxynet

# указыкаем нужные парамметры, которые нам нужно передать для Traefik, чтобы Traefik знал к какому запросу перенаправлять наши request'ы на этот контейнер, для каждого контейнера нужно указывать по какому адресу URL запрос будет отправляться в nginx

// Запуск контейнера 
>> docker compose -f nginx-compose.yml up


// Конфигурация Apache - apache-compose.yml

version: '3'
services:
 apache:
  image: httpd:latest
  container_name: apache
  lables:
   # контейнер указывает Traefik, что этот контейнер можно проксировать, используется в связке с --providers.docker.exposedByDefault=false
   - "traefik.enable=true"
   - "traefik.http.routes.apache.rule=Host('apache.test.lv')"
# Запуск в сети контейнера proxynet, где находится и наш Traefik
networks:
 default:
  name: proxynet



// Для выноса конфигурацмм можно использовать traefik.yml файл
# EntryPoints в Traefik — это входные точки, через которые трафик попадает в систему.
entryPoints:
 http:
  address: ":80"
  http:
   redirections:
    entryPoint:
     to: https
     scheme: https
 https:
  address: ":443"
  http:
    tls:
     certResolver: le
     
certificatesResolvers:
 le:
  acme:
   email: test@test.lv
   storage: /letsencrypt/acme.json
   tlsChallenge: true
   
api:
 # Только https соединение
 insecure: false
 dashboard: true
log: 
 level: DEBUG
providers:
 docker:
  exposedByDefault: false
  network: proxynet
middlewares:
 auth:
  basicAuth:
   users:
    - "admin:$apr1$9XW9ABCx$abcdefg..."  # Заменить на свой htpasswd-хеш

#Комента для генирации htpasswd-хеш
>> echo $(htpasswd -nb admin Parole123) | sed -e s/\\$/\\$\\$/g

