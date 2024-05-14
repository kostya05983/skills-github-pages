---
title: "Spring test data generator"
date: 2019-05-10
---

В статье будет обзор на недавно написанную мною либу для упрощения генерации тестовых данных.

Что позволяет делать библиотека:

* Генерировать валидные тестовые данные.

* Расширять стандартный набор резолверов для валидационных аннотаций (об этом ниже).

* Предоставляет способ объяснить как конструировать ваш объект, если ваш тип объекта слишком специфичен.

#### Как мы жили раньше
В общем случае мы в команде создавали данные для тестов следующим образом:
* Объявлили в каком-то файле утилиты для создания объектов
```kotlin
fun createSomeDifficultClass(): DifficultClass {
    return DifficultClass(
        id = ObjectId(),
        someParam = "test",
        ....
    )
}
``` 
такие файлики в среднем занимают по 120-200 строчек,
но когда ваше апи слишком большое получается много лишнего кода и писать его не очень весело.
* Тест выглядит примерно так
```kotlin
@Test
fun test() {
    val difficultClass = createSomeDifficultClass()
    ...
}
```

#### Как мы заживем теперь

Теперь подключив либу, вам не надо больше писать функции для создания объектов, все что нужно это расширить класс
расширением библиотеки, данные она сгенерирует за вас.
```kotlin
@ExtendWith(GenerateParameterResolver::class)
class YourTestClass {

@Test
fun test(@Generate difficultClazz) {
 ...
}
}
```
Теперь о фичах, библиотека генерирует валидные данные основываясь на javax

```kotlin
data class YourClass(
@field:Null
var param: String?
)
```
В данном случае, либа увидит аннотацию Null и вызовет соответствующий генератор для нее и param проинициализируется null значением.


#### Что делать со своими аннотациями?
Библиотеку можно расширять для своих аннотаций, делается это следующим образом, предположим что у нас есть аннотация
ValidObjectId, которая проверяет, что строка является id-шником монги, тогда генератор под такую аннотацию будет
выглядеть следующим образом:
```kotlin
@ResolverFor(ValidObjectId::class)
class ValidObjectIdGenerator : ValidationParamResolver {
    override fun <T> process(generatedParam: T?, clazz: KClass<*>, type: KType, annotation: Annotation): Any? {
        return ObjectId().toHexString()
    }
}
```

#### Мой тип содержит конструктор с некоторой валидацией
Библиотеке можно указать использовать конструктор для конкретного типа
```kotlin
class ObjectIdConstructor : ValidationConstructor<ObjectId> {
    override fun call(): ObjectId {
        return ObjectId()
    }
}
```
она его подхватит и не будет пытаться самостоятельно генерировать тип.

#### Как генератору найти мои расширения?

Для этого необходимо навешать на класс аннотацию с путем
```kotlin
@SpringTestDataGenerator(value = "ru.kontur.spring.test.generator")
```

#### Вместо вывода 

Недавно один друг подсказал, что это похоже на autofixture с c#, но только со своими примочками,
 что ж будем двигаться в этом направлении.
 
[Ссылка на репу] (https://github.com/skbkontur/spring-test-data-generator) 
