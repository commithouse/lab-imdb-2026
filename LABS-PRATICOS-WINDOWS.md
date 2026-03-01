# 🧪 LABS PRÁTICOS: Hands-On Exercises (Windows)

Comandos quebrados por linha para rodar no **PowerShell** ou **CMD** do Windows. Sem Python.

---

## ENCONTRO 1: Labs 1-3

### Lab 1: Configurar Redis Cluster 6-Node

**Objetivo:** Montar cluster com failover automático  
**Tempo:** 45 minutos  
**Pré-requisitos:** Docker instalado, `redis-cli`, 6 portas (6379-6384)

---

**Passo 1: Criar 6 containers Redis**

Baixar imagem:

```powershell
docker pull redis:7.0-alpine
```

Criar cada node (rode um comando por vez):

```powershell
docker run -d --name redis-node-0 -p 6379:6379 redis:7.0-alpine redis-server --cluster-enabled yes --cluster-config-file nodes-0.conf --port 6379
```

```powershell
docker run -d --name redis-node-1 -p 6380:6379 redis:7.0-alpine redis-server --cluster-enabled yes --cluster-config-file nodes-1.conf --port 6379
```

```powershell
docker run -d --name redis-node-2 -p 6381:6379 redis:7.0-alpine redis-server --cluster-enabled yes --cluster-config-file nodes-2.conf --port 6379
```

```powershell
docker run -d --name redis-node-3 -p 6382:6379 redis:7.0-alpine redis-server --cluster-enabled yes --cluster-config-file nodes-3.conf --port 6379
```

```powershell
docker run -d --name redis-node-4 -p 6383:6379 redis:7.0-alpine redis-server --cluster-enabled yes --cluster-config-file nodes-4.conf --port 6379
```

```powershell
docker run -d --name redis-node-5 -p 6384:6379 redis:7.0-alpine redis-server --cluster-enabled yes --cluster-config-file nodes-5.conf --port 6379
```

Verificar (deve listar 6 nodes):

```powershell
docker ps | findstr redis-node
```

---

**Passo 2: Criar Cluster**

Um único comando (responda **yes** quando pedir confirmação):

```powershell
docker exec redis-node-0 redis-cli --cluster create 127.0.0.1:6379 127.0.0.1:6380 127.0.0.1:6381 127.0.0.1:6382 127.0.0.1:6383 127.0.0.1:6384 --cluster-replicas 1
```

---

**Passo 3: Testar Cluster**

Conectar ao node 0:

```powershell
redis-cli -p 6379
```

Dentro do `redis-cli`, ver slots:

```
cluster slots
```

Adicionar dados (dentro do redis-cli):

```
SET key1 "Redis slot 0"
```

```
SET key2 "Redis slot 5461"
```

```
GET key1
```

Sair do redis-cli (digite `exit`). Parar node 0:

```powershell
docker stop redis-node-0
```

Aguardar 5 segundos:

```powershell
Start-Sleep -Seconds 5
```

Tentar conectar de novo (esperado: outro node assumiu):

```powershell
redis-cli -p 6379
```

Reiniciar node 0:

```powershell
docker start redis-node-0
```

Ver status do cluster:

```powershell
redis-cli -p 6379 cluster info
```

---

**Checkpoints:** 6 nodes em execução | Cluster 3 masters + 3 slaves | SET/GET distribuído | Failover OK | Slave promovido

---

### Lab 2: Replicação Master-Slave

**Objetivo:** Replicação M-S com backup automático  
**Tempo:** 30 minutos

---

**Setup – Terminal 1: Master (porta 7000)**

```powershell
docker run -d --name redis-master -p 7000:6379 redis:7.0-alpine redis-server
```

**Terminal 2: Slave (porta 7001)**

```powershell
docker run -d --name redis-slave -p 7001:6379 redis:7.0-alpine redis-server --slaveof redis-master 6379
```

Aguardar 2 segundos:

```powershell
Start-Sleep -Seconds 2
```

Verificar replicação (master):

```powershell
redis-cli -p 7000 INFO replication
```

Verificar replicação (slave):

```powershell
redis-cli -p 7001 INFO replication
```

---

**Teste de Replicação**

No master (conectar e depois os comandos abaixo):

```powershell
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

No slave, conferir (conectar e depois os comandos):

```powershell
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

Conectar no master:

```powershell
redis-cli -p 7000
```

Configurar snapshot e salvar:

```
CONFIG SET save "300 10"
```

```
CONFIG REWRITE
```

Forçar snapshot:

```
BGSAVE
```

```
LASTSAVE
```

Sair do redis-cli. Listar arquivos no container (deve ter `dump.rdb`):

```powershell
docker exec redis-master ls -la /data/
```

---

### Lab 3: Persistência (RDB vs AOF)

**Objetivo:** Comparar RDB e AOF  
**Tempo:** 30 minutos

No Windows use um diretório local para volume, por exemplo `C:\temp\redis-rdb`. Crie a pasta se não existir:

```powershell
New-Item -ItemType Directory -Force -Path C:\temp\redis-rdb
```

```powershell
New-Item -ItemType Directory -Force -Path C:\temp\redis-aof
```

---

**Teste RDB**

Subir Redis com RDB (ajuste o caminho se usar outra pasta):

```powershell
docker run -d --name redis-rdb -p 8000:6379 -v C:\temp\redis-rdb:/data redis:7.0-alpine redis-server --save "300 10" --dir /data
```

Inserir dados em lote (PowerShell, 100 chaves). Rode em um bloco ou linha a linha se preferir:

```powershell
1..100 | ForEach-Object { redis-cli -p 8000 SET "key:$_" "value:$_" }
```

Forçar snapshot:

```powershell
redis-cli -p 8000 BGSAVE
```

Verificar arquivo:

```powershell
docker exec redis-rdb ls -la /data/dump.rdb
```

Tamanho (em container Linux):

```powershell
docker exec redis-rdb du -sh /data/dump.rdb
```

---

**Teste AOF**

Subir Redis com AOF:

```powershell
docker run -d --name redis-aof -p 8001:6379 -v C:\temp\redis-aof:/data redis:7.0-alpine redis-server --appendonly yes --appendfsync everysec --dir /data
```

Inserir 100 chaves:

```powershell
1..100 | ForEach-Object { redis-cli -p 8001 SET "key:$_" "value:$_" }
```

Verificar AOF:

```powershell
docker exec redis-aof ls -la /data/appendonly.aof
```

Ver início do arquivo:

```powershell
docker exec redis-aof head -20 /data/appendonly.aof
```

---

## ENCONTRO 2: Labs 4-6

### Lab 4: Full-Text Search (RediSearch)

**Objetivo:** Índice de bicicletas, buscas &lt;100ms  
**Tempo:** 40 minutos

---

**Passo 1: Ativar RediSearch**

```powershell
docker run -d --name redis-search -p 9000:6379 redislabs/redisearch:latest
```

Conectar:

```powershell
redis-cli -p 9000
```

---

**Passo 2: Criar índice (dentro do redis-cli)**

```redis
FT.CREATE bikes:idx ON HASH SCHEMA model TEXT WEIGHT 2.0 SORTABLE brand TAG price NUMERIC SORTABLE color TAG weight NUMERIC stars NUMERIC
```

Verificar índice:

```redis
FT.INFO bikes:idx
```

---

**Passo 3: Popular com dados (sem Python)**

Inserir bicicletas uma a uma via redis-cli. Exemplos (repita o padrão para chegar perto de 500 ou use o loop PowerShell abaixo).

Exemplo manual (algumas bikes):

```powershell
redis-cli -p 9000 HSET bike:0 brand Trek model Carbon color black price 2500 weight 10 stars 4.5
```

```powershell
redis-cli -p 9000 HSET bike:1 brand Specialized model Aluminium color red price 1500 weight 12 stars 4.0
```

Inserir muitas com PowerShell (ajuste o total se quiser menos que 500):

```powershell
$brands = @("Trek","Specialized","Giant","Cannondale","Scott"); $models = @("Carbon","Aluminium","Hybrid","Road","Mountain"); $colors = @("black","red","blue","white","green"); 1..500 | ForEach-Object { $b = $brands[(Get-Random -Maximum 5)]; $m = $models[(Get-Random -Maximum 5)]; $c = $colors[(Get-Random -Maximum 5)]; $p = Get-Random -Minimum 500 -Maximum 5000; $w = Get-Random -Minimum 8 -Maximum 15; $s = [math]::Round((Get-Random -Minimum 30 -Maximum 50)/10, 1); redis-cli -p 9000 HSET "bike:$($_-1)" brand $b model $m color $c price $p weight $w stars $s }
```

---

**Passo 4: Buscas (no redis-cli -p 9000)**

Busca simples (Carbon):

```redis
FT.SEARCH bikes:idx "Carbon"
```

Carbon com preço &lt; 2000:

```redis
FT.SEARCH bikes:idx "Carbon" FILTER price 0 2000
```

Marca Trek:

```redis
FT.SEARCH bikes:idx "@brand:Trek"
```

Carbon + Trek + preço 1000–3000:

```redis
FT.SEARCH bikes:idx "Carbon" FILTER brand Trek FILTER price 1000 3000
```

Top 10 Carbon mais caros:

```redis
FT.SEARCH bikes:idx "Carbon" SORTBY price DESC LIMIT 0 10
```

---

**Passo 5: Fuzzy (typos)**

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

Este lab usa aplicação (Redis + PostgreSQL + worker). A parte “direto no MD” é só o setup de containers e DB; a lógica do write-behind continua em código (Python ou outra linguagem) em outro repositório/arquivo. Aqui só os comandos de ambiente.

---

**Setup – Redis**

```powershell
docker run -d --name redis-cache -p 10000:6379 redis:7.0-alpine
```

**PostgreSQL**

```powershell
docker run -d --name postgres-db -e POSTGRES_PASSWORD=password -e POSTGRES_DB=ecommerce -p 5432:5432 postgres:15-alpine
```

Conectar e criar tabelas (uma vez):

```powershell
docker exec -it postgres-db psql -U postgres -d ecommerce -c "CREATE TABLE users (id SERIAL PRIMARY KEY, name VARCHAR(100), email VARCHAR(100), balance DECIMAL(10,2), updated_at TIMESTAMP DEFAULT NOW()); CREATE TABLE user_updates (id SERIAL PRIMARY KEY, user_id INT, balance DECIMAL(10,2), status VARCHAR(20) DEFAULT 'pending', created_at TIMESTAMP DEFAULT NOW());"
```

Verificar cache (após rodar a aplicação em outro terminal):

```powershell
redis-cli -p 10000 HGET user:1 balance
```

Verificar DB:

```powershell
docker exec -it postgres-db psql -U postgres -d ecommerce -c "SELECT COUNT(*) AS pending FROM user_updates;"
```

---

### Lab 6: Time-Series (Sensores IoT)

**Objetivo:** Ingerir muitos dados de sensores  
**Tempo:** 30 minutos

---

**Setup**

```powershell
docker run -d --name redis-ts -p 11000:6379 redis:7.0-alpine
```

Conectar:

```powershell
redis-cli -p 11000
```

---

**Criar séries (redis-cli)**

Carro 1:

```redis
TS.CREATE car:1:speed RETENTION 86400000
```

```redis
TS.CREATE car:1:rpm RETENTION 86400000
```

```redis
TS.CREATE car:1:temp RETENTION 86400000
```

Sensores 1 a 50 (no PowerShell, fora do redis-cli):

```powershell
1..50 | ForEach-Object { redis-cli -p 11000 TS.CREATE "sensor:${_}:temp" RETENTION 3600000; redis-cli -p 11000 TS.CREATE "sensor:${_}:humidity" RETENTION 3600000 }
```

*(Nota: se o Redis não tiver módulo RedisTimeSeries, TS.CREATE não existirá; use imagem com RedisTimeSeries.)*

Inserir pontos manualmente (exemplo):

```powershell
redis-cli -p 11000 TS.ADD sensor:1:temp 1700000000000 22.5
```

```powershell
redis-cli -p 11000 TS.ADD sensor:1:humidity 1700000000000 65
```

Para ingerir 1M pontos sem Python: use um loop PowerShell chamando `TS.ADD` em lote (vários sensores e timestamps). Exemplo mínimo para um sensor:

```powershell
$base = [long]((Get-Date -UFormat %s) * 1000); 1..1000 | ForEach-Object { redis-cli -p 11000 TS.ADD sensor:1:temp ($base + $_) (20 + (Get-Random -Minimum -50 -Maximum 50)/10) }
```

Queries (no redis-cli -p 11000, se usar RedisTimeSeries):

```redis
TS.RANGE sensor:1:temp (now-600000) +
```

---

## ENCONTRO 3: Lab 10 + Projeto

### Lab 10: Benchmark (1M operações)

**Objetivo:** Comparar Redis vs DuckDB vs Azure SQL  
**Tempo:** 45 minutos  

Sem Python: você pode medir só o Redis direto no terminal.

**Write speed – Redis (1M SETs)**

Exemplo em PowerShell (pode demorar; reduzir o range para teste rápido):

```powershell
$start = Get-Date; 1..10000 | ForEach-Object { redis-cli -p 10000 SET "key:$_" "value:$_" }; (Get-Date) - $start
```

Para 1M, use um script .ps1 com o mesmo loop e `1..1000000` e cronometre. Esperado: Redis muito rápido (centenas de milhares de ops/segundo).

**Query speed:** DuckDB e Azure SQL exigem cliente (ODBC, psql, etc.); compare manualmente com as ferramentas que você tiver. Redis não faz GROUP BY; use para cache/writes.

---

### Projeto Final: Design Your Architecture (60 min)

*(Mantido como no documento original – apenas desenho e decisões, sem comandos.)*

- Opção A: Pinterest Brasil – Recomendações Real-Time  
- Opção B: Gaming – Leaderboard Global  
- Opção C: Chat App – Presença em Tempo Real  

Entregáveis: diagrama, decisões de tech, custo, failover, backup (RTO/RPO). Rubric: escalabilidade, tech choices, security, cost.

---

## Resumo

| Lab | Tema        | Comandos no Windows                          |
|-----|-------------|----------------------------------------------|
| 1   | Cluster     | `docker run` por node, `redis-cli`, `Start-Sleep` |
| 2   | M-S         | `docker run` master/slave, `redis-cli`       |
| 3   | RDB/AOF     | `docker run -v C:\temp\...`, loops PowerShell |
| 4   | RediSearch  | `redis-cli`, `FT.CREATE`, `FT.SEARCH`, loop PowerShell para dados |
| 5   | Write-Behind| Docker Redis + Postgres, app em outro repo   |
| 6   | Time-Series | `TS.CREATE` / `TS.ADD` via redis-cli e PowerShell |
| 10  | Benchmark   | Loop PowerShell com `redis-cli SET` e cronômetro |

**Preparado por:** Prof. Daniel Lemeszenski  
**Data:** 27 Fevereiro 2026  
**Para:** MBA FIAP Encontros 1-3 — versão Windows, sem Python, direto no MD.
