### 1. Запуск

```bash
docker compose up -d --build
```

### 2. Инициализация Config Server

```bash
docker compose exec -T configSrv mongosh --port 27017 --quiet <<'EOF'
rs.initiate({
  _id: "config_server",
  configsvr: true,
  members: [{ _id: 0, host: "configSrv:27017" }]
});
EOF
```

### 3. Инициализация шардов

```bash
docker compose exec -T shard1 mongosh --port 27018 --quiet <<'EOF'
rs.initiate({
  _id: "shard1",
  members: [{ _id: 0, host: "shard1:27018" }]
});
EOF
```

```bash
docker compose exec -T shard2 mongosh --port 27019 --quiet <<'EOF'
rs.initiate({
  _id: "shard2",
  members: [{ _id: 0, host: "shard2:27019" }]
});
EOF
```

### 4. Регистрация шардов

```bash
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<'EOF'
sh.addShard("shard1/shard1:27018");
sh.addShard("shard2/shard2:27019");
sh.enableSharding("somedb");
sh.shardCollection("somedb.helloDoc", { name: "hashed" });
EOF
```

### 5. Добавление данных

```bash
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<'EOF'
const database = db.getSiblingDB("somedb");
for (let i = 0; i < 1000; i++) {
  database.helloDoc.insertOne({ age: i, name: "ly" + i });
}
EOF
```

### 6. Проверка

```bash
# Shard 1
docker compose exec shard1 mongosh --port 27018 --quiet --eval 'db.getSiblingDB("somedb").helloDoc.countDocuments({})'

# Shard 2
docker compose exec shard2 mongosh --port 27019 --quiet --eval 'db.getSiblingDB("somedb").helloDoc.countDocuments({})'

# Общее количество
docker compose exec mongos_router mongosh --port 27020 --quiet --eval 'db.getSiblingDB("somedb").helloDoc.countDocuments({})'
```
