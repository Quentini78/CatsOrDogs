# Guide de Déploiement de l'Application

Ce guide explique comment déployer l'application en utilisant des commandes Docker directes. Assurez-vous d'avoir Docker installé sur votre machine avant de commencer.

## Étapes de Déploiement avec les commandes Docker (version Manuelle)

### 1. Construction du réseau 
```bash
docker network create --driver=bridge cats-or-dogs-network --subnet=192.168.49.130/24
```
### 3. Construction de la database (docker Volume)
```bash
docker volume create db-data
```
### 4. Construction des images Docker 
```bash
cd worker
docker build -t worker .
cd ../seed-data
docker build -t alexisfr-virtu-seed-data:cmd ./seed-data/
cd ../result
docker build -t result .
cd ../vote
docker build -t vote .
```
### 5. Lancement des conteneurs 

Lancer le db 
```bash
docker run \
        --network=cats-or-dogs-network \
        --ip 192.168.49.134 \
        --name db \
        -v db-data:/var/lib/postgresql/data \
        -v "$(pwd)/healthchecks:/healthchecks" \
        --health-cmd="/healthchecks/postgres.sh" \
        --health-interval="5s" \
        -e POSTGRES_PASSWORD=postgres \
        -p 5432:5432 \
        -d \
        postgres:15-alpine
```
Lancer le redis 
```bash 
docker run \
        --network=cats-or-dogs-network \
        --ip 192.168.49.135 \
        --name redis \
        -v "$(pwd)/healthchecks:/healthchecks" \
        --health-cmd="/healthchecks/redis.sh" \
        --health-interval="5s" \
        -p 6379:6379 \
        -d \
        redis
```
Lancer le vote 
```bash
docker run \
        --network=cats-or-dogs-network \
        --ip 192.168.49.136 \
        --name vote \
        -v "$(pwd)/vote:/usr/local/app" \
        -p 5002:80 \
        -d \
        vote
```
Lancer le result 
```bash
docker rm result || true
docker run \
        --network=cats-or-dogs-network \
        --ip 192.168.49.137 \
        --name result \
        --health-cmd="curl -f http://localhost/ || exit 1" \
        --health-interval="30s" \
        -p 5001:80 \
        -p 127.0.0.1:9229:9229 \
        -d \
        result
```
Lancer le worker
```bash
docker rm worker || true
docker run \
        --network=cats-or-dogs-network \
        --ip 192.168.49.138 \
        --name worker \
        -d \
        worker
```
Lancer le seed-data
```bash
docker rm seed-data || true
sleep 2
docker run \
        --network=cats-or-dogs-network \
        --ip 192.168.49.139 \
        --name seed-data \
        -d \
        seed-data
```

Une fois les conteneurs lancer il ne reste plus qu'à aller sur :  
Page résultat : http://<docker-host>:5001 \
Page vote : http://<docker-host>:5002 

### 6. Arrêter les conteneurs 
```bash
docker stop db redis result vote worker
```
---
## Étapes de Déploiement avec le fichier Docker compose (version automatique)


### 2. Création du Docker compose
```bash
nano docker-compose.yml
```
```yml
#docker-compose.yml
version: '3'

services:
  worker:
    build:
      context: ./worker
      dockerfile: Dockerfile
    depends_on:
      redis:
        condition: service_healthy
      db:
        condition: service_healthy
    networks:
      - back-tier

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: postgres
    volumes:
      - "db-data:/var/lib/postgresql/data"
      - "./healthchecks:/healthchecks"
    healthcheck:
      test: /healthchecks/postgres.sh
      interval: "5s"
    networks:
      - back-tier

  redis:
    image: redis
    volumes:
      - "./healthchecks:/healthchecks"
    healthcheck:
      test: /healthchecks/redis.sh
      interval: "5s"
    networks:
      - back-tier

  vote:
    build:
      context: ./vote
      dockerfile: Dockerfile
    ports:
      - "5002:80"
    volumes:
      - ./vote:/usr/local/app
    networks:
      - front-tier
      - back-tier
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 10s

  seed-data:
    build:
      context: ./seed-data
      dockerfile: Dockerfile
    depends_on:
      vote:
        condition: service_healthy
    restart: "no"
    networks:
      - front-tier

  result:
    build:
      context: ./result
      dockerfile: Dockerfile
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - ./result:/usr/local/app
    ports:
      - "5001:80"
      - "127.0.0.1:9229:9229"
    entrypoint: ["nodemon", "--inspect=0.0.0.0", "server.js"]
    networks:
      - front-tier
      - back-tier

networks:
  back-tier:
  front-tier:

volumes:
  db-data:
```
### 3. Lancer le Docker compose
```bash
docker compose up --build
```
### 4. Arrêter le Docker compose
```bash
docker compose down
```
Une fois les conteneurs lancer il ne reste plus qu'à aller sur :  
Page résultat : http://<docker-host>:5001 \
Page vote : http://<docker-host>:5002 
