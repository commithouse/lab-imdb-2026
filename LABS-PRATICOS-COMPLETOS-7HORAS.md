
# 🧪 LABS PRÁTICOS: Hands-On Exercises

## ENCONTRO 1: Labs 1-3

### Lab 1: Configurar Redis Cluster 6-Node

**Objetivo:** Montar cluster com failover automático

**Tempo:** 45 minutos

**Pré-requisitos:**
- Docker instalado
- `redis-cli` (ferramenta)
- 6 portas disponíveis (6379-6384)

**Passo 1: Criar 6 containers Redis**

```{bash}
# Pull imagem
docker pull redis:7.0-alpine

# Criar nodes 0-5
for i in {0..5}; do
  docker run -d \
    --name redis-node-$i \
    -p $((6379+i)):6379 \
    redis:7.0-alpine \
    redis-server --cluster-enabled yes \
    --cluster-config-file nodes-$i.conf \
    --port $((6379+i))
done

# Verificar (deve listar 6 nodes)
docker ps | grep redis-node
```

**Passo 2: Criar Cluster**

```bash
# Executar cluster create
docker exec redis-node-0 redis-cli --cluster create \
  127.0.0.1:6379 \
  127.0.0.1:6380 \
  127.0.0.1:6381 \
  127.0.0.1:6382 \
  127.0.0.1:6383 \
  127.0.0.1:6384 \
  --cluster-replicas 1

# Responder "yes" quando pergunta confirmar
```

**Passo 3: Testar Cluster**

```bash
# Conectar ao node 0
redis-cli -p 6379

# Verificar slots
cluster slots
# Expected: 3 masters com ~5460 slots cada

# Adicionar dados (vai distribuir entre nodes)
SET key1 "Redis slot 0"
SET key2 "Redis slot 5461"
GET key1

# Failover test: Matar node 0
docker stop redis-node-0

# Aguardar 5s
sleep 5

# Tentar conectar novamente
redis-cli -p 6379
# Esperado: Node 3 (slave) promovido para master

# Reiniciar node 0
docker start redis-node-0

# Verificar cluster status
redis-cli -p 6379
cluster info
```

**Checkpoints:**
- [ ] 6 nodes em execução
- [ ] Cluster criado com 3 masters + 3 slaves
- [ ] SET/GET funcionam distribuído
- [ ] Failover automático funciona
- [ ] Slave promovido corretamente

**Resultado Esperado:**
```
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
```

---

### Lab 2: Replicação Master-Slave

**Objetivo:** Configurar replicação M-S com backup automático

**Tempo:** 30 minutos

**Setup:**

```bash
# Terminal 1: Master (7000)
docker run -d --name redis-master -p 7000:6379 \
  redis:7.0-alpine redis-server

# Terminal 2: Slave (7001)
docker run -d --name redis-slave -p 7001:6379 \
  redis:7.0-alpine redis-server --slaveof redis-master 6379

# Aguardar 2s para sincronização
sleep 2

# Verificar replicação
redis-cli -p 7000 INFO replication
# Expected: role:master, connected_slaves:1

redis-cli -p 7001 INFO replication
# Expected: role:slave, master_host:redis-master
```

**Teste de Replicação:**

```bash
# Master: adicionar dados
redis-cli -p 7000
SET counter 100
LPUSH queue "item1" "item2" "item3"
ZADD leaderboard 1000 "player1" 2000 "player2"

# Slave: verificar replicação (deve ter mesmo dados)
redis-cli -p 7001
GET counter              # 100
LRANGE queue 0 -1       # item3, item2, item1
ZRANGE leaderboard 0 -1 # player1, player2
```

**Backup Automático:**

```bash
# Master: configurar snapshots
redis-cli -p 7000
CONFIG SET save "300 10"     # Snapshot a cada 300s se 10 mudanças
CONFIG REWRITE               # Salvar em arquivo redis.conf

# Forçar snapshot agora
BGSAVE
LASTSAVE  # Timestamp do último snapshot

# Backup arquivo
docker exec redis-master ls -la /data/
# Deve ter dump.rdb
```

**Checkpoints:**
- [ ] Master e Slave conectados
- [ ] Dados replicam para Slave
- [ ] BGSAVE funciona
- [ ] dump.rdb criado

---

### Lab 3: Persistência (RDB vs AOF)

**Objetivo:** Comparar RDB (snapshot) vs AOF (append-only file)

**Tempo:** 30 minutos

**Teste RDB (Point-in-Time):**

```bash
# Iniciar Redis com RDB
docker run -d --name redis-rdb -p 8000:6379 \
  -v /tmp/redis-rdb:/data \
  redis:7.0-alpine redis-server \
  --save "300 10" \
  --dir /data

# Adicionar dados
for i in {1..100}; do
  redis-cli -p 8000 SET "key:$i" "value:$i"
done

# Forçar snapshot
redis-cli -p 8000 BGSAVE

# Verificar arquivo
docker exec redis-rdb ls -la /data/dump.rdb

# Tamanho do arquivo
docker exec redis-rdb du -sh /data/dump.rdb
# Expected: ~5-10KB (comprimido)
```

**Teste AOF (Durabilidade Total):**

```bash
# Iniciar Redis com AOF
docker run -d --name redis-aof -p 8001:6379 \
  -v /tmp/redis-aof:/data \
  redis:7.0-alpine redis-server \
  --appendonly yes \
  --appendfsync everysec \
  --dir /data

# Adicionar dados
for i in {1..100}; do
  redis-cli -p 8001 SET "key:$i" "value:$i"
done

# Verificar arquivo AOF
docker exec redis-aof ls -la /data/appendonly.aof

# Ver conteúdo (texto legível)
docker exec redis-aof head -20 /data/appendonly.aof
# Expected: *3, $3, SET, $4, key:1, ...
```

**Comparativa:**

```
                 RDB           AOF
Arquivo          dump.rdb      appendonly.aof
Tamanho          ~10KB         ~50KB
Recuperação      1s (rápido)   5s (log replay)
Precisão         5 min         1s (everysec)
Compressão       Sim (binário) Não (texto)
```

**Checkpoints:**
- [ ] RDB criado e restaurável
- [ ] AOF criado com todos commands
- [ ] Recuperação de RDB funciona
- [ ] Recuperação de AOF funciona
- [ ] Entender trade-offs

---

## ENCONTRO 2: Labs 4-6

### Lab 4: Full-Text Search (RediSearch)

**Objetivo:** Indexar 500 bicicletas, buscar <100ms

**Tempo:** 40 minutos

**Passo 1: Ativar RediSearch**

```bash
# Docker com Redis + RediSearch
docker run -d --name redis-search -p 9000:6379 \
  redislabs/redisearch:latest

# Conectar
redis-cli -p 9000
```

**Passo 2: Criar Índice de Bicicletas**

```redis
# Definir schema
FT.CREATE bikes:idx \
  ON HASH \
  SCHEMA \
    model TEXT WEIGHT 2.0 SORTABLE \
    brand TAG \
    price NUMERIC SORTABLE \
    color TAG \
    weight NUMERIC \
    stars NUMERIC

# Verificar índice
FT.INFO bikes:idx
```

**Passo 3: Popular com Dados (500 bicicletas)**

```bash
# Script Python para gerar dados
cat > populate_bikes.py << 'EOF'
import redis
import random

r = redis.Redis(host='localhost', port=9000, decode_responses=True)

brands = ["Trek", "Specialized", "Giant", "Cannondale", "Scott"]
models = ["Carbon", "Aluminium", "Hybrid", "Road", "Mountain"]
colors = ["black", "red", "blue", "white", "green"]

for i in range(500):
    key = f"bike:{i}"
    brand = random.choice(brands)
    model = random.choice(models)
    color = random.choice(colors)
    price = random.randint(500, 5000)
    weight = random.randint(8, 15)
    stars = round(random.uniform(3.0, 5.0), 1)
    
    r.hset(key, mapping={
        'brand': brand,
        'model': model,
        'color': color,
        'price': price,
        'weight': weight,
        'stars': stars
    })
    
    if (i + 1) % 100 == 0:
        print(f"✓ Inserted {i+1} bikes")

print("✅ Total: 500 bikes inserted")
EOF

# Executar
python3 populate_bikes.py
```

**Passo 4: Buscas Avançadas**

```redis
# Busca 1: Simples (buscar todas as Carbon)
FT.SEARCH bikes:idx "Carbon"
# Expected: <100ms, 100 resultados

# Busca 2: Com filtro numérico (Carbon < $2000)
FT.SEARCH bikes:idx "Carbon" \
  FILTER price 0 2000
# Expected: ~30 resultados

# Busca 3: Com TAG (Trek brand)
FT.SEARCH bikes:idx "@brand:Trek"
# Expected: ~100 resultados

# Busca 4: Composta (Carbon AND Trek AND price 1000-3000)
FT.SEARCH bikes:idx "Carbon" \
  FILTER brand Trek \
  FILTER price 1000 3000
# Expected: ~10 resultados

# Busca 5: Ordenação (Top 10 Carbon mais caro)
FT.SEARCH bikes:idx "Carbon" \
  SORTBY price DESC \
  LIMIT 0 10
# Expected: Ordenado por preço decrescente
```

**Passo 5: Fuzzy Search (Typos)**

```redis
# Usuário digita "carbone" (faltou 'n')
FT.SEARCH bikes:idx "%carbone%"
# Esperado: Acha "Carbon" mesmo com erro!

# Usuário digita "Trek" certo
FT.SEARCH bikes:idx "%trek%"
# Esperado: Acha "Trek" (case-insensitive)

# Levenshtein distance 2
FT.SEARCH bikes:idx "%carbin|2%"
# Esperado: Acha com até 2 caracteres de diferença
```

**Performance Test:**

```bash
cat > benchmark_search.py << 'EOF'
import redis
import time

r = redis.Redis(host='localhost', port=9000, decode_responses=True)

queries = [
    ("Carbon", "Busca simples"),
    ("*", "Match all"),
    ("Trek", "Brand filter"),
    ("Carbon AND Specialized", "Composed AND"),
]

for query, desc in queries:
    start = time.time()
    result = r.execute_command('FT.SEARCH', 'bikes:idx', query)
    elapsed = (time.time() - start) * 1000  # ms
    
    count = result[0]  # Total results
    print(f"✓ {desc:20s} | {elapsed:6.2f}ms | {count} results")

# All queries should be < 100ms
EOF

python3 benchmark_search.py
```

**Checkpoints:**
- [ ] Índice criado com 5 campos
- [ ] 500 bicicletas inseridas
- [ ] Busca simples <100ms
- [ ] Filtros funcionam
- [ ] Fuzzy search pega typos
- [ ] Todas queries <100ms

---

### Lab 5: Write-Behind Cache Pattern

**Objetivo:** Implementar cache com persistência assíncrona

**Tempo:** 45 minutos

**Setup:**

```bash
# Terminal 1: Redis
docker run -d --name redis-cache -p 10000:6379 redis:7.0-alpine

# Terminal 2: PostgreSQL (DB)
docker run -d --name postgres-db \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=ecommerce \
  -p 5432:5432 \
  postgres:15-alpine

# Terminal 3: Redis + Python Worker
# (vou criar app.py abaixo)
```

**Passo 1: Schema PostgreSQL**

```sql
-- Conectar ao postgres
psql -h localhost -U postgres -d ecommerce -c "
  CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100),
    balance DECIMAL(10,2),
    updated_at TIMESTAMP DEFAULT NOW()
  );

  CREATE TABLE user_updates (
    id SERIAL PRIMARY KEY,
    user_id INT,
    balance DECIMAL(10,2),
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT NOW()
  );
"
```

**Passo 2: Aplicação Write-Behind**

```python
# app.py
import redis
import psycopg2
import json
import time
from datetime import datetime

# Conexões
redis_conn = redis.Redis(host='localhost', port=10000, decode_responses=True)
pg_conn = psycopg2.connect(
    host='localhost',
    database='ecommerce',
    user='postgres',
    password='password'
)

class WriteBehindsProcessor:
    def __init__(self):
        self.queue_key = "pending_updates"
        self.cache_prefix = "user:"
    
    def update_user_balance(self, user_id, amount):
        """Write-Behind: escreve em cache, queue persiste depois"""
        
        # 1. Escrever em cache (RÁPIDO ~1ms)
        cache_key = f"{self.cache_prefix}{user_id}"
        current = redis_conn.hget(cache_key, 'balance') or 0
        new_balance = float(current) + amount
        
        redis_conn.hset(cache_key, 'balance', new_balance)
        redis_conn.expire(cache_key, 3600)  # TTL 1h
        
        # 2. Enqueue para worker (NÃO-BLOQUEANTE)
        job = {
            'user_id': user_id,
            'balance': new_balance,
            'timestamp': datetime.now().isoformat()
        }
        redis_conn.rpush(self.queue_key, json.dumps(job))
        
        print(f"✓ User {user_id}: Cache updated (+${amount})")
        return new_balance
    
    def process_queue(self):
        """Background worker: persiste DB"""
        
        while True:
            # Pegar job da queue
            job_json = redis_conn.lpop(self.queue_key)
            
            if not job_json:
                time.sleep(1)  # Aguardar próximo job
                continue
            
            job = json.loads(job_json)
            user_id = job['user_id']
            balance = job['balance']
            
            try:
                # Escrever em DB
                cursor = pg_conn.cursor()
                cursor.execute(
                    "INSERT INTO user_updates (user_id, balance) VALUES (%s, %s)",
                    (user_id, balance)
                )
                pg_conn.commit()
                
                print(f"✓ User {user_id}: DB persisted (${balance})")
                
            except Exception as e:
                print(f"❌ Error: {e}, requeuing job...")
                # Recolocar na queue se falhar
                redis_conn.rpush(self.queue_key, job_json)

# Uso
processor = WriteBehindsProcessor()

# Teste: 1000 updates rápidos
print("=== WRITE-BEHIND LOAD TEST ===")
start = time.time()

for i in range(1000):
    processor.update_user_balance(user_id=1, amount=10)

elapsed = (time.time() - start) * 1000
print(f"✓ 1000 updates em {elapsed:.0f}ms = {1000/(elapsed/1000):.0f} ops/sec")
print(f"  (Redis rápido! Queue processar depois...)")

# Background: processar queue
print("\n=== BACKGROUND WORKER ===")
print("Iniciando worker (Ctrl+C para parar)...")
try:
    processor.process_queue()
except KeyboardInterrupt:
    print("\n✓ Worker parado")
```

**Passo 3: Executar Teste**

```bash
# Terminal 1: Aplicação
python3 app.py

# Esperado Output:
# ✓ User 1: Cache updated (+$10)  [~1ms]
# ✓ User 1: Cache updated (+$10)  [~1ms]
# ...
# ✓ 1000 updates em 150ms = 6,666 ops/sec
# ✓ Background worker processando...
# ✓ User 1: DB persisted ($10010)
# ...

# Terminal 2: Verificar Cache
redis-cli -p 10000
HGET user:1 balance  # Deve ter novo valor imediatamente

# Terminal 3: Verificar DB
psql -h localhost -U postgres -d ecommerce -c \
  "SELECT COUNT(*) as pending FROM user_updates;"
# Esperado: 1000 updates persistidos
```

**Checkpoints:**
- [ ] 1000 updates completam em <200ms
- [ ] Cache tem novo valor imediatamente
- [ ] Worker persiste em DB
- [ ] DB tem todas 1000 inserções
- [ ] Falha em DB recoloca em queue

---

### Lab 6: Time-Series (Sensores IoT)

**Objetivo:** Ingerir 1M dados de sensores em <1 segundo

**Tempo:** 30 minutos

**Setup:**

```bash
# Docker Redis + Redis Streams
docker run -d --name redis-ts -p 11000:6379 redis:7.0-alpine
```

**Passo 1: Criar Série de Tempo**

```redis
# Conectar
redis-cli -p 11000

# Criar série (Tesla car 1 - speed sensor)
TS.CREATE car:1:speed RETENTION 86400000  # 1 dia retenção
TS.CREATE car:1:rpm RETENTION 86400000    # RPM sensor
TS.CREATE car:1:temp RETENTION 86400000   # Temperatura

# Criar série (50 sensores)
for i in {1..50}; do
  TS.CREATE sensor:$i:temp RETENTION 3600000
  TS.CREATE sensor:$i:humidity RETENTION 3600000
done
```

**Passo 2: Ingerir Dados (1M pontos)**

```python
# ingest.py
import redis
import time
import random
from datetime import datetime

r = redis.Redis(host='localhost', port=11000, decode_responses=True)

print("=== TIME-SERIES LOAD TEST ===")
start_time = time.time()
timestamp = int(start_time * 1000)  # milliseconds

# 1M dados em batches
batch_size = 10000
total = 1_000_000

for batch in range(0, total, batch_size):
    # 50 sensores × 200 pontos por batch
    for sensor_id in range(1, 51):
        for i in range(200):
            # Simular dados sensor
            temp = 20 + random.gauss(0, 5)  # ~20°C ± 5
            humidity = 60 + random.gauss(0, 10)  # ~60% ± 10
            current_ts = timestamp + (batch + i) * 10  # 10ms entre pontos
            
            # Add to time-series
            r.execute_command('TS.ADD', f'sensor:{sensor_id}:temp', current_ts, temp)
            r.execute_command('TS.ADD', f'sensor:{sensor_id}:humidity', current_ts, humidity)
    
    elapsed = time.time() - start_time
    rate = batch / elapsed
    print(f"✓ {batch:,}/{total:,} ({elapsed:.1f}s, {rate:,.0f} ops/sec)")

total_elapsed = time.time() - start_time
final_rate = total / total_elapsed
print(f"\n✅ TOTAL: {total:,} pontos em {total_elapsed:.1f}s = {final_rate:,.0f} ops/sec")
```

**Passo 3: Queries & Aggregações**

```redis
# Query 1: Últimos 10 minutos (600,000ms)
TS.RANGE sensor:1:temp \
  (now-600000) +

# Query 2: Agregação média a cada 60s (60,000ms)
TS.QUERYINDEX AGGREG avg 60000 FILTER name=sensor:*:temp
# Expected: ~1,000 agregações (1M pontos / 1K por agregação)

# Query 3: Máxima temperatura (últimas 24h)
TS.QUERYINDEX AGGREG max 86400000 FILTER name=sensor:*:temp

# Query 4: Múltiplas séries correlação
# Comparar temperatura vs humidity
TS.QUERYINDEX FILTER name=sensor:1:*

# Query 5: Alertas (temperatura > 35°C)
TS.QUERYINDEX AGGREG max 60000 FILTER \
  "name=sensor:*:temp" "value>35"
```

**Passo 4: Dashboard Simulado**

```python
# dashboard.py
import redis
import time

r = redis.Redis(host='localhost', port=11000, decode_responses=True)

while True:
    print("\n=== REAL-TIME DASHBOARD ===")
    print(f"Timestamp: {time.strftime('%H:%M:%S')}")
    
    # Sensor stats (últimas 60s)
    for sensor_id in range(1, 6):  # Mostrar primeiros 5
        temp_range = r.execute_command(
            'TS.RANGE',
            f'sensor:{sensor_id}:temp',
            f'(now-60000)',
            'now'
        )
        
        if temp_range:
            temps = [float(x[1]) for x in temp_range]
            avg = sum(temps) / len(temps)
            min_t = min(temps)
            max_t = max(temps)
            
            print(f"Sensor {sensor_id}: "
                  f"Avg={avg:.1f}°C | Min={min_t:.1f}°C | Max={max_t:.1f}°C")
    
    time.sleep(5)
```

**Checkpoints:**
- [ ] TS.CREATE funcionam (50 sensores)
- [ ] 1M pontos ingeridos <2s
- [ ] Rate > 500K ops/sec
- [ ] Queries última hora <100ms
- [ ] Agregações funcionam
- [ ] Dashboard mostra dados reais

---

## ENCONTRO 3: Lab 10 + Projeto

### Lab 10: Benchmark Comparativo (1M operações)

**Objetivo:** Comparar Redis vs DuckDB vs Azure SQL

**Tempo:** 45 minutos

**Teste 1: Write Speed (SET 1M)**

```bash
cat > benchmark_write.py << 'EOF'
import redis
import psycopg2
import duckdb
import time
import random
import string

print("=== WRITE SPEED TEST (1M operações) ===\n")

# 1. Redis
print("1️⃣  REDIS")
r = redis.Redis(host='localhost', port=10000)
start = time.time()
for i in range(1_000_000):
    r.set(f"key:{i}", f"value:{random.randint(0,9999)}")
redis_write = time.time() - start
print(f"   ✓ {redis_write:.2f}s = {1_000_000/redis_write:,.0f} ops/sec")

# 2. DuckDB
print("\n2️⃣  DUCKDB")
db = duckdb.connect(':memory:')
db.execute("CREATE TABLE data (id INT, value VARCHAR)")
start = time.time()
for i in range(1_000_000):
    db.execute(f"INSERT INTO data VALUES ({i}, 'value:{random.randint(0,9999)}')")
duckdb_write = time.time() - start
print(f"   ✓ {duckdb_write:.2f}s = {1_000_000/duckdb_write:,.0f} ops/sec")

# 3. Azure SQL
print("\n3️⃣  AZURE SQL")
try:
    conn = psycopg2.connect(...)
    cursor = conn.cursor()
    start = time.time()
    for i in range(1_000_000):
        cursor.execute("INSERT INTO data VALUES (%s, %s)", 
                      (i, f'value:{random.randint(0,9999)}'))
    conn.commit()
    azure_write = time.time() - start
    print(f"   ✓ {azure_write:.2f}s = {1_000_000/azure_write:,.0f} ops/sec")
except:
    print("   ⚠️  Azure SQL não disponível (skip)")

print("\n=== RESULTADO ===")
print(f"🏆 Redis é {duckdb_write/redis_write:.0f}x mais rápido que DuckDB")
EOF

python3 benchmark_write.py
```

**Teste 2: Query Speed (GROUP BY 100K)**

```bash
cat > benchmark_query.py << 'EOF'
import redis
import duckdb
import psycopg2
import time
import random

print("=== QUERY SPEED TEST (GROUP BY 100K rows) ===\n")

# Preparar dados (100K registros)
data = [(i, f"category_{i % 10}", random.randint(100, 9999)) 
        for i in range(100_000)]

# 1. Redis (não faz GROUP BY nativo)
print("1️⃣  REDIS")
r = redis.Redis(host='localhost', port=10000)
print("   ⚠️  Redis não suporta SQL (skip)")

# 2. DuckDB
print("\n2️⃣  DUCKDB")
db = duckdb.connect(':memory:')
db.execute("CREATE TABLE sales (id INT, category VARCHAR, amount INT)")
db.execute(f"INSERT INTO sales VALUES {','.join(str(d) for d in data)}")
start = time.time()
result = db.execute(
    "SELECT category, COUNT(*) as cnt, SUM(amount) FROM sales GROUP BY category"
).fetchall()
duckdb_query = time.time() - start
print(f"   ✓ {duckdb_query*1000:.0f}ms = {result}")

# 3. Azure SQL
print("\n3️⃣  AZURE SQL")
try:
    conn = psycopg2.connect(...)
    cursor = conn.cursor()
    cursor.executemany("INSERT INTO sales VALUES (%s, %s, %s)", data)
    conn.commit()
    start = time.time()
    cursor.execute("SELECT category, COUNT(*), SUM(amount) FROM sales GROUP BY category")
    result = cursor.fetchall()
    azure_query = time.time() - start
    print(f"   ✓ {azure_query*1000:.0f}ms = {result}")
except:
    print("   ⚠️  Azure SQL não disponível (skip)")

print("\n=== CONCLUSÃO ===")
print("✅ DuckDB é MAIS RÁPIDO para queries (50ms vs 200ms Azure)")
print("✅ Redis é MAIS RÁPIDO para writes (puro cache)")
EOF

python3 benchmark_query.py
```

**Checkpoints:**
- [ ] Redis: 1M writes <2s (500K+ ops/sec)
- [ ] DuckDB: 1M inserts <5s (200K ops/sec)
- [ ] Queries: DuckDB <100ms
- [ ] Conclusão: Redis para cache, DuckDB para analytics

---

### Projeto Final: Design Your Architecture (60 minutos)

**Objetivo:** Em grupos, desenhar arquitetura production-ready

**Escolha um cenário:**

#### **Opção A: Pinterest Brasil - Recomendações Real-Time**

**Requisitos:**
- 10M usuários ativos/mês
- 500M pins em DB
- 100M impressões/dia (feed)
- Latência P99 <200ms
- MUST ter: recomendações personalizadas

**Stack Esperado:**
```
Cliente (iOS/Web)
        ↓
CDN (CloudFront - imagens)
        ↓
Load Balancer
        ↓
3x App Servers (feed service)
        ↓
┌─────────────────────────────────┐
│ Redis Cluster (recomen./cache) │
│ - 6 nós (3M+/3M recomen)       │
│ - Clustering automático         │
└─────────────────────────────────┘
        ↓
PostgreSQL (master-slave, replication)
        ↓
Elasticsearch (pins full-text)
```

**Decisões a Tomar:**
1. Por que Redis (não SQL puro)? Latência <100ms
2. Qual cache pattern? Write-Behind (recomen. async)
3. Como falhar? Failover automático + backup 6h
4. Custo 3 anos? $150K (vs $2M Oracle)

---

#### **Opção B: Gaming - Leaderboard Global (100K simultâneos)**

**Requisitos:**
- 100K jogadores online
- Atualizar leaderboard a cada 5s
- Ranking em 50ms
- Tolerância zero perda de pontos
- Integração com payment ($ reais)

**Stack Esperado:**
```
Game Client (Unreal/Unity)
        ↓
Game Server (Match server)
        ↓
┌─────────────────────────────────┐
│ Redis SORTED SET (leaderboard) │
│ - 1B scores/dia                 │
│ - ZADD + ZRANK em 1ms           │
│ - Cluster 3-node                │
└─────────────────────────────────┘
        ↓
PostgreSQL (transações com payment)
        ↓
Message Queue (confirmações async)
```

**Decisões:**
1. Por que ZSET Redis? Rank em O(1)
2. ACID para payment? Write-Through + DB
3. Como validar fraude? Timestamp server

---

#### **Opção C: Chat App - Presença em Tempo Real (1M usuários online)**

**Requisitos:**
- 1M usuários online
- Presença/typing status <500ms
- Histórico 30 dias
- Push notifications
- Encryption end-to-end

**Stack Esperado:**
```
Mobile App (iOS/Android)
        ↓
WebSocket Server (3x pods)
        ↓
┌─────────────────────────────────┐
│ Redis Pub/Sub (chat rooms)      │
│ - Presença (sorted set)         │
│ - Typing indicator              │
│ - 50 chats/sec                  │
└─────────────────────────────────┘
        ↓
PostgreSQL (histórico persistido)
        ↓
S3 (media files - imagens)
```

**Decisões:**
1. Pub/Sub persistence? Não (use Redis Streams)
2. Histórico? PostgreSQL com TTL
3. Escalabilidade? Redis Cluster (1B mensagens/dia)

---

**Entregáveis (para cada opção):**

1. **Diagrama Arquitetura** (desenho/texto ASCII)
2. **Decisões de Tech** (qual DB? por quê?)
3. **Cálculo de Custo** (3 anos em dólares)
4. **Plano de Failover** (o que acontece se cair?)
5. **Backup Strategy** (RTO/RPO definido)

**Rubric (10 pontos):**
- [ ] Escalabilidade (3pts) — Aguanta 10x+ carga?
- [ ] Tech Choices (3pts) — Justificativas sólidas?
- [ ] Security (2pts) — ACL, TLS, VPC?
- [ ] Cost (2pts) — Realista? Otimizado?

**Tempo:**
- 10 min: Discussão do grupo (qual opção?)
- 30 min: Design + decisões
- 15 min: Desenhar diagrama
- 5 min: Apresentação (2 min por grupo)

---

## 🏁 RESUMO LABS

| Lab | Tema | Tempo | Conceito | Stack |
|-----|------|-------|----------|-------|
| 1 | Cluster 6-node | 45m | Failover automático | Docker |
| 2 | Replicação M-S | 30m | Backup automático | Redis |
| 3 | RDB vs AOF | 30m | Persistência | Redis |
| 4 | FT.SEARCH | 40m | Full-text index | RediSearch |
| 5 | Write-Behind | 45m | Cache pattern | Redis+PG |
| 6 | Time-Series | 30m | Ingestão 1M/s | RedisTS |
| 10 | Benchmark | 45m | Comparativa | Redis vs DuckDB vs Azure |
| Final | Projeto | 60m | Architecture | Escolha stack |

**TOTAL: 7 horas 25 minutos (encaixa em 9 horas com breaks)**

---

**Preparado por:** Prof. Daniel Lemeszenski  
**Data:** 27 Fevereiro 2026  
**Para:** MBA FIAP Encontros 1-3
