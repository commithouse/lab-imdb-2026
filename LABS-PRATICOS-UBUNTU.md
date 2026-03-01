# 🧪 LABS PRÁTICOS: Hands-On Exercises (Ubuntu / Linux)

Comandos quebrados por linha para rodar no **terminal** do Ubuntu (bash). Sem Python.

**Pré-requisitos Ubuntu:** Docker instalado. Para `redis-cli` no host: `sudo apt update && sudo apt install -y redis-tools` (ou use apenas dentro dos containers).

---

## ENCONTRO 1: Labs 1-3

### Lab 1: Configurar Redis Cluster 6-Node

**Objetivo:** Montar cluster com failover automático  
**Tempo:** 45 minutos  
**Pré-requisitos:** Docker, `redis-cli`, 6 portas (6379-6384)

---

**Passo 1: Criar 6 containers Redis**

Baixar imagem:

```bash
docker pull redis:7.0-alpine
```

Criar os 6 nodes com loop (rode o bloco inteiro):

```bash
for i in {0..5}; do
  docker run -d \
    --name redis-node-$i \
    -p $((6379+i)):6379 \
    redis:7.0-alpine \
    redis-server --cluster-enabled yes \
    --cluster-config-file nodes-$i.conf \
    --port 6379
done
```

Verificar (deve listar 6 nodes):

```bash
docker ps | grep redis-node
```

---

**Passo 2: Criar Cluster**

Responda **yes** quando pedir confirmação:

```bash
docker exec redis-node-0 redis-cli --cluster create \
  127.0.0.1:6379 127.0.0.1:6380 127.0.0.1:6381 \
  127.0.0.1:6382 127.0.0.1:6383 127.0.0.1:6384 \
  --cluster-replicas 1
```

---

**Passo 3: Testar Cluster**

Conectar ao node 0:

```bash
redis-cli -p 6379
```

Dentro do `redis-cli`:

```
cluster slots
```

```
SET key1 "Redis slot 0"
```

```
SET key2 "Redis slot 5461"
```

```
GET key1
```

Sair do redis-cli (`exit`). Parar node 0:

```bash
docker stop redis-node-0
```

Aguardar 5 segundos:

```bash
sleep 5
```

Tentar conectar de novo:

```bash
redis-cli -p 6379
```

Reiniciar node 0:

```bash
docker start redis-node-0
```

Status do cluster:

```bash
redis-cli -p 6379 cluster info
```

---

**Checkpoints:** 6 nodes | Cluster 3 masters + 3 slaves | SET/GET distribuído | Failover OK

---

### Lab 2: Replicação Master-Slave

**Objetivo:** Replicação M-S com backup automático  
**Tempo:** 30 minutos

---

**Setup – Terminal 1: Master (porta 7000)**

```bash
docker run -d --name redis-master -p 7000:6379 redis:7.0-alpine redis-server
```

**Terminal 2: Slave (porta 7001)**

```bash
docker run -d --name redis-slave -p 7001:6379 redis:7.0-alpine redis-server --slaveof redis-master 6379
```

Aguardar 2 segundos:

```bash
sleep 2
```

Verificar replicação (master):

```bash
redis-cli -p 7000 INFO replication
```

Verificar replicação (slave):

```bash
redis-cli -p 7001 INFO replication
```

---

**Teste de Replicação**

No master:

```bash
redis-cli -p 7000
```

Dentro do redis-cli:

```
SET counter 100
```

```
LPUSH queue "item1" "item2" "item3"
```

```
ZADD leaderboard 1000 "player1" 2000 "player2"
```

No slave:

```bash
redis-cli -p 7001
```

```
GET counter
```

```
LRANGE queue 0 -1
```

```
ZRANGE leaderboard 0 -1
```

---

**Backup automático**

```bash
redis-cli -p 7000
```

Dentro do redis-cli:

```
CONFIG SET save "300 10"
```

```
CONFIG REWRITE
```

```
BGSAVE
```

```
LASTSAVE
```

Sair e listar arquivos no container:

```bash
docker exec redis-master ls -la /data/
```

---

### Lab 3: Persistência (RDB vs AOF)

**Objetivo:** Comparar RDB e AOF  
**Tempo:** 30 minutos

Criar diretórios:

```bash
sudo mkdir -p /tmp/redis-rdb /tmp/redis-aof
sudo chown $USER:$USER /tmp/redis-rdb /tmp/redis-aof
```

---

**Teste RDB**

```bash
docker run -d --name redis-rdb -p 8000:6379 \
  -v /tmp/redis-rdb:/data \
  redis:7.0-alpine redis-server --save "300 10" --dir /data
```

Inserir 100 chaves (loop):

```bash
for i in {1..100}; do redis-cli -p 8000 SET "key:$i" "value:$i"; done
```

Forçar snapshot:

```bash
redis-cli -p 8000 BGSAVE
```

Verificar arquivo:

```bash
docker exec redis-rdb ls -la /data/dump.rdb
```

Tamanho:

```bash
docker exec redis-rdb du -sh /data/dump.rdb
```

---

**Teste AOF**

```bash
docker run -d --name redis-aof -p 8001:6379 \
  -v /tmp/redis-aof:/data \
  redis:7.0-alpine redis-server \
  --appendonly yes --appendfsync everysec --dir /data
```

Inserir 100 chaves:

```bash
for i in {1..100}; do redis-cli -p 8001 SET "key:$i" "value:$i"; done
```

Verificar AOF:

```bash
docker exec redis-aof ls -la /data/appendonly.aof
```

Ver início do arquivo:

```bash
docker exec redis-aof head -20 /data/appendonly.aof
```

---

## ENCONTRO 2: Labs 4-6

### Lab 4: Full-Text Search (RediSearch)

**Objetivo:** Índice de bicicletas, buscas <100ms  
**Tempo:** 40 minutos

---

**Passo 1: Ativar RediSearch**

```bash
docker run -d --name redis-search -p 9000:6379 redislabs/redisearch:latest
```

Conectar:

```bash
redis-cli -p 9000
```

---

**Passo 2: Criar índice (dentro do redis-cli)**

```redis
FT.CREATE bikes:idx ON HASH SCHEMA model TEXT WEIGHT 2.0 SORTABLE brand TAG price NUMERIC SORTABLE color TAG weight NUMERIC stars NUMERIC
```

```redis
FT.INFO bikes:idx
```

---

**Passo 3: Popular com dados (sem Python)**

Exemplos manuais:

```bash
redis-cli -p 9000 HSET bike:0 brand Trek model Carbon color black price 2500 weight 10 stars 4.5
```

```bash
redis-cli -p 9000 HSET bike:1 brand Specialized model Aluminium color red price 1500 weight 12 stars 4.0
```

Inserir 500 bikes com loop (bash; rode o bloco):

```bash
brands=(Trek Specialized Giant Cannondale Scott)
models=(Carbon Aluminium Hybrid Road Mountain)
colors=(black red blue white green)
for i in $(seq 0 499); do
  b=${brands[$((RANDOM % 5))]}
  m=${models[$((RANDOM % 5))]}
  c=${colors[$((RANDOM % 5))]}
  p=$((500 + RANDOM % 4500))
  w=$((8 + RANDOM % 8))
  s=$((30 + RANDOM % 21))
  s="${s:0:1}.${s:1:1}"
  redis-cli -p 9000 HSET "bike:$i" brand "$b" model "$m" color "$c" price "$p" weight "$w" stars "$s"
  [ $(((i+1) % 100)) -eq 0 ] && echo "Inserted $((i+1)) bikes"
done
```

---

**Passo 4: Buscas (redis-cli -p 9000)**

```redis
FT.SEARCH bikes:idx "Carbon"
```

```redis
FT.SEARCH bikes:idx "Carbon" FILTER price 0 2000
```

```redis
FT.SEARCH bikes:idx "@brand:Trek"
```

```redis
FT.SEARCH bikes:idx "Carbon" FILTER brand Trek FILTER price 1000 3000
```

```redis
FT.SEARCH bikes:idx "Carbon" SORTBY price DESC LIMIT 0 10
```

---

**Passo 5: Fuzzy**

```redis
FT.SEARCH bikes:idx "%carbone%"
```

```redis
FT.SEARCH bikes:idx "%trek%"
```

```redis
FT.SEARCH bikes:idx "%carbin|2%"
```

---

### Lab 5: Write-Behind Cache Pattern

**Objetivo:** Cache com persistência assíncrona  
**Tempo:** 45 minutos  

Aqui só os comandos de ambiente (Redis + PostgreSQL). A aplicação write-behind fica em outro arquivo/repo.

---

**Redis**

```bash
docker run -d --name redis-cache -p 10000:6379 redis:7.0-alpine
```

**PostgreSQL**

```bash
docker run -d --name postgres-db \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=ecommerce \
  -p 5432:5432 postgres:15-alpine
```

Criar tabelas:

```bash
docker exec -it postgres-db psql -U postgres -d ecommerce -c "
  CREATE TABLE users (id SERIAL PRIMARY KEY, name VARCHAR(100), email VARCHAR(100), balance DECIMAL(10,2), updated_at TIMESTAMP DEFAULT NOW());
  CREATE TABLE user_updates (id SERIAL PRIMARY KEY, user_id INT, balance DECIMAL(10,2), status VARCHAR(20) DEFAULT 'pending', created_at TIMESTAMP DEFAULT NOW());
"
```

Verificar cache (após rodar a aplicação):

```bash
redis-cli -p 10000 HGET user:1 balance
```

Verificar DB:

```bash
docker exec -it postgres-db psql -U postgres -d ecommerce -c "SELECT COUNT(*) AS pending FROM user_updates;"
```

---

### Lab 6: Time-Series (Sensores IoT)

**Objetivo:** Ingerir dados de sensores  
**Tempo:** 30 minutos

---

**Setup**

```bash
docker run -d --name redis-ts -p 11000:6379 redis:7.0-alpine
```

Conectar:

```bash
redis-cli -p 11000
```

---

**Criar séries (redis-cli)**

```redis
TS.CREATE car:1:speed RETENTION 86400000
```

```redis
TS.CREATE car:1:rpm RETENTION 86400000
```

```redis
TS.CREATE car:1:temp RETENTION 86400000
```

Sensores 1–50 (no terminal; requer módulo RedisTimeSeries na imagem):

```bash
for i in $(seq 1 50); do
  redis-cli -p 11000 TS.CREATE "sensor:$i:temp" RETENTION 3600000
  redis-cli -p 11000 TS.CREATE "sensor:$i:humidity" RETENTION 3600000
done
```

Inserir pontos (exemplo):

```bash
redis-cli -p 11000 TS.ADD sensor:1:temp 1700000000000 22.5
```

```bash
redis-cli -p 11000 TS.ADD sensor:1:humidity 1700000000000 65
```

Exemplo de loop para muitos pontos (um sensor):

```bash
base=$(($(date +%s) * 1000))
for i in $(seq 1 1000); do
  ts=$((base + i))
  val=$(awk "BEGIN { print 20 + (($RANDOM % 100) - 50) / 10 }")
  redis-cli -p 11000 TS.ADD sensor:1:temp $ts $val
done
```

Query (redis-cli -p 11000):

```redis
TS.RANGE sensor:1:temp (now-600000) +
```

---

## ENCONTRO 3: Lab 10 + Projeto

### Lab 10: Benchmark (1M operações)

**Objetivo:** Comparar Redis vs outros (DuckDB, etc.)  
**Tempo:** 45 minutos  

Sem Python: medir Redis no terminal.

**Write speed – Redis (exemplo 10k para teste rápido)**

```bash
time (for i in $(seq 1 10000); do redis-cli -p 10000 SET "key:$i" "value:$i" > /dev/null; done)
```

Para 1M: use o mesmo loop com `seq 1 1000000` em um script .sh e cronometre.

---

### Projeto Final: Design Your Architecture (60 min)

- Opção A: Pinterest Brasil – Recomendações Real-Time  
- Opção B: Gaming – Leaderboard Global  
- Opção C: Chat App – Presença em Tempo Real  

Entregáveis: diagrama, decisões de tech, custo, failover, backup (RTO/RPO).

---

## Resumo Ubuntu

| Lab | Tema        | Comandos |
|-----|-------------|----------|
| 1   | Cluster     | `for i in {0..5}`, `docker run`, `sleep 5`, `grep` |
| 2   | M-S         | `docker run`, `redis-cli` |
| 3   | RDB/AOF     | `-v /tmp/redis-*`, `for i in {1..100}`; em Ubuntu `chown` em /tmp se necessário |
| 4   | RediSearch  | `FT.CREATE`, `FT.SEARCH`, loop com `seq` para 500 bikes |
| 5   | Write-Behind| Docker Redis + Postgres |
| 6   | Time-Series | `TS.CREATE` / `TS.ADD`, loop com `seq` |
| 10  | Benchmark   | `time` + loop `redis-cli SET` |

**Preparado por:** Prof. Daniel Lemeszenski  
**Data:** 27 Fevereiro 2026  
**Para:** MBA FIAP Encontros 1-3 — versão Ubuntu/Linux, sem Python.
