---
title: "Compound и Partial индексы в spring-data-mongodb"
date: 2020-03-28
---

#### Используемый стек:
* spring-boot 2.2.4.RELEASE
* spring-data-mongodb 2.2.4.RELEASE

#### Введение
Статья призвана помочь разобраться с основными ошибками и неявностями при работе
с индексами монги при помощи spring-data-mongodb

#### Что такое Compound Index
Compound Index - это индекс который индексируется сразу по нескольким полям. 

#### Что такое Partial Index
Partial Index - это индекс который работает, только при встрече
определенного критерия в коллекции.

#### Как просиходит процесс инициализации?

У spring по умолчанию включена автосоздание индексов, при старте вашего приложения
спринг просканирует ваши сущности помеченные Document на наличие аннотаций CompoundIndex
и попытается создать ваш индекс в коллекции. Тут начинается самое интересное:
1) Ваше приложение развалится если уже есть индекс с тем же именем, но с другим определением.

2) Ваше приложение развалится если условия для индекса не выполнено.

#### Когда уже есть индекс с таким же именем, но с другим определением
Например у нас есть такое определение продукта:
```kotlin
@Document("products")
@CompoundIndex(
    name = "product_index",
    unique = true,
    def = "{'productId':1, 'type': 1}"
)

data class Product(
    @field:Id
    val id: ObjectId,
    val type: String,
    val productId: String,
    val weight: Double,
    val amount: Int,
    val createdBy: Instant,
    @field:Version
    val version: Long
)
```
И у нас есть требование, что не может быть двух одинаковых продуктов
одного типа с одним и тем же продуктовым идентификатором. Аннотацией CompundIndex
мы ограничиваем возможность создания таких продуктов.

Далее мы понимаем, что хотели бы, чтобы наш индекс сортировал в другом направлении,
мы переделываем определение аннотации на следующее:
```kotlin
@Document("products")
@CompoundIndex(
    name = "product_index",
    unique = true,
    def = "{'productId':-1, 'type': -1}"
)

data class Product(
    @field:Id
    val id: ObjectId,
    val type: String,
    val productId: String,
    val weight: Double,
    val amount: Int,
    val createdBy: Instant,
    @field:Version
    val version: Long
)
```
Деплоим на стейджинг и наше приложение разваливается.

В итоге чтобы этого избежать надо внимательно следить за изменениями индексов.

#### Ваше приложение разваливается если условие индекса не выполнено

Возьмем то же определение сущности:
```kotlin
@Document("products")
@CompoundIndex(
    name = "product_index",
    unique = true,
    def = "{'productId':1, 'type': 1}"
)

data class Product(
    @field:Id
    val id: ObjectId,
    val type: String,
    val productId: String,
    val weight: Double,
    val amount: Int,
    val createdBy: Instant,
    @field:Version
    val version: Long
)
```

Такая ситуация произойдет, если в базе будет уже две сущности, которые по type и productId совпадают.

Чтобы этого избежать надо создавать индекс самим, тогда приложение не развалится при старте.

#### Partial индексы
В монге можно строить partial index, они бывают полезно, когда вам необходимо, чтобы индекс работал при определенном условии
например:
```kotlin
@Document("products")
@CompoundIndex(
    name = "product_index",
    unique = true,
    def = "{'productId':1, 'type': 1}"
)

data class Product(
    @field:Id
    val id: ObjectId,
    val type: String,
    val productId: String,
    val weight: Double,
    val amount: Int,
    val createdBy: Instant,
    val deleted: Boolean=false,
    @field:Version
    val version: Long
)
```

Мы хотим чтобы индекс строился, только если deleted=false, тогда мы добавляем:
```kotlin
@Document("products")
@CompoundIndex(
    name = "product_index",
    unique = true,
    def = "{'productId':1, 'type': 1}, {partialFilterIndex: {deleted: false}}"
)

data class Product(
    @field:Id
    val id: ObjectId,
    val type: String,
    val productId: String,
    val weight: Double,
    val amount: Int,
    val createdBy: Instant,
    val deleted: Boolean=false,
    @field:Version
    val version: Long
)
```
Но внезапно выясняется, что оно так не работает и при автосоздании спринг не умеет в partial:(

Решение **надо создать индекс самому**

#### Как создать индекс самому

Во-первых надо отключить автосоздание индекса в application.yml:
```yaml
spring:
    data:
      mongodb:
        auto-index-creation: false
```

Во-вторых определить функцию, которая будет исполнять при совершении определенного ивента,
ивент который мы выберем называется ApplicationReadyEvent, генерируется после полного старта приложения.

```kotlin
 @EventListener(ApplicationReadyEvent::class)
 fun initIndicies() {
     val index = Index()
                 .unique()
                 .named("product_index")
                 .on("productId", Sort.DEFAULT_DIRECTION)
                 .on("type", Sort.DEFAULT_DIRECTION)
                 .partial(
                     PartialIndexFilter.of(Criteria.where("deleted").`is`(false))
                 )
     val reactiveIndexOperations = mongoTemplate.indexOps(Product::class.java)
     val result = reactiveIndexOperations.ensureIndex(index).block()
     logger.info { "Init index $result" }
}
```
Таким образом мы создадим partial index на коллекцию и даже если что-то развалится, то приложение все равно стартанет.

### Итого

* Не доверять автосозданию индексов, потому что часть функций индексации при таком создании может просто не работать

* Создавать индекс самому после старта приложения

* Следить за изменением индекса, если меняете то сносите старый или меняйте имя индекса
 
* Проверять базу на дубликаты по индексу при создании дополнительных новых индексов
