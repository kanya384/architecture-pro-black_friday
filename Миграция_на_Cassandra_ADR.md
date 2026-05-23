### <a name="_b7urdng99y53"></a>**Название задачи: Миграция на Cassandra**
### <a name="_hjk0fkfyohdk"></a>**Автор: Кушу Л.Р.**
### <a name="_uanumrh8zrui"></a>**Дата:23.05.2026**
### <a name="_3bfxc9a45514"></a>**Проблематика**
Во время «чёрной пятницы» интернет-магазин использовал MongoDB с шардированием на основе диапазонов (Range-Based Sharding). При резком увеличении нагрузки (50 000 запросов/сек.) возникла высокая задержка при масштабировании:

При добавлении новых шардов MongoDB полностью перераспределяла данные между всеми узлами, что вызывало просадку latency в пик нагрузки, так как система тратила ресурсы на перемещение данных.

Руководство решило перейти на БД Cassandra, чтобы обеспечить:
- Высокую отказоустойчивость (leaderless‑репликация).
- Быстрое горизонтальное масштабирование без полного перераспределения данных.
- Равномерное распределение данных.

### <a name="_u8xz25hbrgql"></a>**Решение**

1. Определение критически важных данных интернет-магазина  
Критически важными для интернет-магазина являются Товары (products), Корзины (carts), Заказы (orders). Так как все эти данные 
используются для покупок и соответсвенно приносят прибыль интернет-магазину.  
Наиболее высоконагруженными являются товары и корзины, для них применение Cassandra имеет смысл. Заказы будут менее нагружены и их
перенос в cassandra не столь критичен. 


2. Модель для выбранных данных

```puml
@startuml
title ER-диаграма «Мобильный мир» Cassandra

entity Products {
  *product_id : uuid <<PK>>
  --
  name : text
  description : text
  category : text
  price : decimal
  status : text
  geo_zone: text
  count: number
  attributes : map<text, text>
  --
  PK: (product_id)
  CK: (geo_zone, price)
}

entity Carts_guest {
  *session_id : uuid
  --
  product_id : uuid
  added_at : timestamp
  quantity : int
  --
  PK: (session_id)
  CK: (product_id, added_at)
}

entity Carts_users {
  *user_id : uuid
  product_id : uuid
  added_at : timestamp
  quantity : int
  --
  PK: (session_id)
  CK: (product_id, added_at)
}

Carts_guest ||--o{ Products : "product_id"

Carts_users ||--o{ Products : "product_id"

@enduml
```

Для Products partition key выбран product_id, что поможет равномерно распределить продукты по шардам и минимизарует шансы
появления горячих шардов, а clustering key выбраны geo_zone и price так как по этим полям часто будет применяться фильтрация.

Carts были разбиты на две таблицы, Carts_guest и Carts_users так как partition_key будет отличаться у Carts_guest это будет
session_id, а Carts_users user_id, в качестве clustring key выбраны product_id и added_at, т.к. по ним ожидается фильтрация

3. Какие стратегии можно применить для обеспечения целостности данных

Для таблиц Carts и Products нужно применять стратегию Read Repair с высоким уровнем согласованности, так как это у нас 
будут часто читаемые данные на основе которых будут формироваться заказы, в данном случае прийдется пожертвовать latency
или подумать над способом кеширования данных.
