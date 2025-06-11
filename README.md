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
    # Enables the web UI and tells Traefik to listen to docker
    command: 
      - "--api.insecure=true" 
      - "--providers.docker"
      # Уровень логов для откладки
      - "--log.level=DEBUG"
    ports:
      # The HTTP port
      - "80:80"
      # The Web UI (enabled by --api.insecure=true)
      - "8080:8080"
    volumes:
      # So that Traefik can listen to the Docker events
      # Делаем проброс сокет для докера, чтобы наш Traefik видел бы на этом сервере другие контейнеры
      - /var/run/docker.sock:/var/run/docker.sock
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
