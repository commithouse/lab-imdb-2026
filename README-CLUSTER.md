# Redis Cluster com Docker Compose

## 1. Subir os 6 nós

```bash
docker compose -f docker-compose.cluster.yml up -d
```

Aguarde alguns segundos para todos os containers iniciarem.

## 2. Criar o cluster (obrigatório depois de subir os nodes)

**Sem este passo o cluster fica em `cluster_state:fail` e `cluster_known_nodes:1`.** Faça em dois passos: primeiro entre no container em modo interativo; depois rode o comando de criar o cluster. Quando aparecer *"Can I set the above configuration? (type 'yes' to accept):"*, digite **yes** e Enter.

**Passo 2.1 – Entrar no container (modo interativo):**
```powershell
docker exec -it redis-node-0 sh
```

**Passo 2.2 – Dentro do container, criar o cluster:**
```bash
redis-cli --cluster create redis-node-0:6379 redis-node-1:6379 redis-node-2:6379 redis-node-3:6379 redis-node-4:6379 redis-node-5:6379 --cluster-replicas 1
```

Digite **yes** quando pedir confirmação. Para sair do shell do container: `exit`.

## 3. Testar

Como o cluster usa os nomes dos containers, conecte de dentro da rede (para seguir redirecionamentos):
```bash
docker exec -it redis-node-0 redis-cli -c -p 6379
```
(Opção `-c` ativa o modo cluster e segue redirecionamentos entre os nós.)

No redis-cli:
```
cluster info
cluster slots
SET key1 "hello"
GET key1
```

## 4. Parar e remover

```bash
docker compose -f docker-compose.cluster.yml down
```

Para apagar também os volumes (dados):
```bash
docker compose -f docker-compose.cluster.yml down -v
```

---

**Se `cluster info` mostrar `cluster_state:fail` e `cluster_known_nodes:1`:** você ainda não rodou o passo 2. Saia do `redis-cli` (`exit`) e execute o comando da seção 2 (criar o cluster) no PowerShell/terminal.
