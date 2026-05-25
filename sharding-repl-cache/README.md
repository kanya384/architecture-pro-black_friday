# pymongo-api

## Как запустить

Запускаем mongodb и приложение

```shell
docker compose up -d
```

Инициализаруем mongodb и заполняем данными

```shell
./scripts/mongo-init.sh
```

## Как проверить


Выводим общее количество документов на всех шардах

```shell
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
use somedb
db.helloDoc.count()
EOF
```

Выводим количество документов на репликах первой шарды

```shell
docker compose exec -T shard1 mongosh --port 27018 --quiet <<EOF
use somedb
db.helloDoc.count()
EOF
```

```shell
docker compose exec -T shard1-1 mongosh --port 27118 --quiet <<EOF
use somedb
db.helloDoc.count()
EOF
```

```shell
docker compose exec -T shard1-2 mongosh --port 27218 --quiet <<EOF
use somedb
db.helloDoc.count()
EOF
```

Выводим количество документов на репликах второй шарды

```shell
docker compose exec -T shard2 mongosh --port 27019 --quiet <<EOF
use somedb
db.helloDoc.count()
EOF
```

```shell
docker compose exec -T shard2-1 mongosh --port 27119 --quiet <<EOF
use somedb
db.helloDoc.count()
EOF
```

```shell
docker compose exec -T shard2-2 mongosh --port 27219 --quiet <<EOF
use somedb
db.helloDoc.count()
EOF
```

### Проверка приложения

Откройте в браузере http://localhost:8080

## Доступные эндпоинты

Список доступных эндпоинтов, swagger http://localhost:8080/docs