### <a name="_b7urdng99y53"></a>**Название задачи:Выявление и устранение «горячих» шардов** 
### <a name="_hjk0fkfyohdk"></a>**Автор: Кушу Л.Р.**
### <a name="_uanumrh8zrui"></a>**Дата:21.05.2026**
### <a name="_3bfxc9a45514"></a>**Проблематика**
Из-за категории «Электроника» произошла перегрузка одного из шардов MongoDB, так как 70% запросов приходилось именно 
на эти товары. Поэтому сейчас нужно разработать стратегию, как выявлять и устранять такие «горячие» шарды, а ещё 
предложить метрики мониторинга, чтобы в будущем можно было предотвращать такие ситуации. Не забудьте учесть, что товары 
из популярных категорий могут создавать непропорциональную нагрузку на отдельные узлы.

### <a name="_u8xz25hbrgql"></a>**Решение**
Для решения проблемы будет использовать zoned tag sharding. Если у нас например три шарда, то выделим два шарда под электронику
и один шард под все остальные категории, тогда у нас получится примерно одинаковое распределение запросов. На шарды надо
повесить теги:

```shell
sh.addShardToZone("shard01", "ZONE_ELECTRONICS")
sh.addShardToZone("shard02", "ZONE_ELECTRONICS")  
sh.addShardToZone("shard03", "ZONE_OTHER")

sh.shardCollection("shop.products", { "category": 1, "id": 1 })


sh.addTagRange(
    "shop.products",
    { "category": MinKey, "id": MinKey },
    { "category": "Electronics", "id": MinKey },
    "ZONE_OTHER"
)

sh.addTagRange(
    "shop.products",
    { "category": "Electronics", "id": MinKey },
    { "category": "Electronics", "id": MaxKey },
    "ZONE_ELECTRONICS"
)

sh.addTagRange(
    "shop.products",
    { "category": "Electronics\uFFFF", "id": MinKey },
    { "category": MaxKey, "id": MaxKey },
    "ZONE_OTHER"
)
```

Эти настройки помогут распределить категорию electronics между двумя шардами, а в третий шард будут попадать все остальные
категории.


Для мониторинга и своевременного выявления "горячих" шард можно пользоваться готовыми prometheus метриками для mongo db (настроить интеграцию с графаной и мониторить на дашбордах).

Cписок метрик которые можно отслеживтаь:

- node_cpu_seconds_total - позволяет увидеть насколько загружен ЦПУ шарда. Если на одном шарде загрузка CPU 90%, а на остальных по 30% то это классический горячий шард
- mongodb_mongod_metrics_document_total (с лейблом {state="inserted|updated|deleted"}) — позволяет увидеть, на какой шард идет лавинообразная запись
- mongodb_mongod_metrics_operation_total (с лейблом {type="query"}) — показывает интенсивность чтения

