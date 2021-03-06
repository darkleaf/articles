# Работа с базой данных из приложения

В начале я обозначу некоторые проблемы и особенности при работе с БД, покажу дыры в абстракциях.
Далее мы разберем более простую абстракцию, основанную на иммутабельности.

Предполагается, что читатель немного знаком с паттернами
[Active Record](https://martinfowler.com/eaaCatalog/activeRecord.html),
[Data Maper](https://martinfowler.com/eaaCatalog/dataMapper.html),
[Identity Map](https://martinfowler.com/eaaCatalog/identityMap.html)
и [Unit of Work](https://martinfowler.com/eaaCatalog/unitOfWork.html).

Проблемы и решения рассматриваются в контексте достаточно больших проектов,
которые нельзя выкинуть и быстро переписать.

## Identity map

Первая проблема - проблема сохранения идентичности.
Идентичность - нечто, что однозначно определяет сущность.
В базе данных - это первичный ключ, а в памяти - ссылка (указатель).
Хорошо, когда ссылки указывают только на один объект.

Для ruby библиотеки
[ActiveRecord](https://guides.rubyonrails.org/active_record_basics.html)
это не так:

```ruby
post_a = Post.find 1
post_b = Post.find 1

post_a.object_id != post_b.object_id # true

post_a.title = "foo"
post_b.title != "foo" # true
```

Т.е. мы получаем 2 ссылки на 2 разных объекта в памяти.

Таким образом мы можем потерять изменения,
если по невнимательности начнем работать с одной и той же сущностью,
но представленной разными объектами.

[Hibernate](http://hibernate.org/orm/)
имеет сессию, фактически кэш первого уровня, которая хранит сопоставление идентификатора сущности
на объект в памяти. Если мы повторно запросим ту же сущность, то получим ссылку на существующий объект.
Т.е. Hibernate реализует паттерн [Identity Map](https://martinfowler.com/eaaCatalog/identityMap.html).

## Долгие транзакции

Но что, если мы делаем выборки не по идентификатору?
Чтобы не допустить рассинхронизации состояния объектов и состояния бд,
Hibernate перед запросом выборки делает
[flush](https://docs.jboss.org/hibernate/stable/core.old/reference/en/html/objectstate-flushing.html),
т.е. сбрасывает в БД "грязные" объекты, чтобы запрос прочитал согласованные данные.

Такой подход вынуждает держать открытой транзакцию БД, пока идет бизнес транзакция.
Если бизнес транзакция долгая, то простаивает, в том числе, и ответственный за соединение процесс в самой БД.
Например, это может случиться, если бизнес транзакция запрашивает данные по сети или выполняет сложные расчеты.

## N+1

Пожалуй, самая большая "дыра" в абстракции ORM - проблема N+1 запроса.

Пример на ruby для библиотеки ActiveRecord:

```ruby
posts = Post.all # select * from posts
posts.each do |post|
  like = post.likes.order(id: :desc).first
  # SELECT * FROM likes WHERE post_id = ? ORDER BY id DESC LIMIT 1
  # ...
end
```

ORM склоняет программиста к мысли, что он работает просто с объектами в памяти.
Но он работает с доступным по сети сервисом, а на установление соединений и передачу данных
требуется время. Даже если запрос выполняется 50ms, то 20 запросов будут выполняться секунду.

## Дополнительные данные

Скажем, чтобы избежать описанной выше проблемы N+1, вы пишете такой
[запрос](https://www.db-fiddle.com/f/6m5FACAHWCeRSmKrTXriVH/2):

```sql
SELECT * FROM posts JOIN LATERAL (
  SELECT * FROM likes WHERE post_id = posts.id ORDER BY likes.id DESC LIMIT 1
) as last_like ON true;
```

Т.е. кроме атрибутов поста выбираются еще и все атрибуты последнего лайка.
На какую сущность отобразить эти данные?
В этом случае можно вернуть пару из поста и лайка, т.к. результат содержит все нужные атрибуты.

А что, если бы мы выбирали только часть полей, или выбрали поля, которые отсутствуют в модели,
например, количество лайков публикации?
Нужно ли вообще их отображение на сущности? Может быть, оставить их просто данными?

## State & identity

Рассмотрим код на js:

```js
const alice = { id: 0, name: 'Alice' };
```

Здесь ссылке на объект дали имя `alice`.
Т.к. это константа, то нет возможности назвать Алисой другой объект.
При этом сам объект остался мутабельным.

Например, мы можем присвоить существующий идентификатор:

```js
const bob = { id: 1, name: 'Bob' };
alice.id = bob.id;
```

Напомню, что у сущности есть 2 идентичности: ссылка и первичный ключ в БД.
И константы не могут помешать сделать Алису Бобом, пусть даже после сохранения.

Объект, ссылку на который мы назвали `alice`, выполняет 2 обязанности:
одновременно моделирует идентичность и состояние.
Состояние - это значение, описывающие сущность в заданный момент времени.

А что, если разделить эти 2 обязанности и использовать для состояния
[неизменяемые структуры](http://facebook.github.io/immutable-js/)?

```js
function Ref(initialState, validator) {
  let state = initialState;

  this.deref = () => state;
  this.swap = (updater) => {
    const newState = updater(state);
    if (! validator(state, newState) ) throw "Invalid state";
    state = newState;
    return newState;
  };
}

const UserState = Immutable.Record({ id: null, name: '' });

const aliceState = new UserState({id: 0, name: 'Alice'});
const alice = new Ref( aliceState, (oldS, newS) => oldS.id === newS.id );

alice.swap( oldS => oldS.set('name', 'Queen Alice') );
alice.swap( oldS => oldS.set('id', 1) ); // BOOM!
```

`Ref` - контейнер для неизменяемого состояния, допускающий его контролируемую замену.
`Ref` моделирует идентичность подобно тому, как мы даем названия предметам.
Мы называем реку "Волга", но в каждый момент времени она имеет различное неизменяемое состояние.

## Storage

Рассмотрим следующий API:

```js
storage.tx( t => {
  const alice = t.get(0);
  const bobState = new UserState({id: 1, name: 'Bob'});
  const bob = t.create(bobState);
  alice.swap( oldS => oldS.update('friends', old => old.push(bob.deref.id)) );
});
```

`t.get` и `t.create` возвращают экземпляр `Ref`.

Мы открываем бизнес транзакцию `t`, находим Алису по ее идентификатору, создаем Боба и указываем,
что Алиса считает Боба своим другом.

Объект `t` контролирует создание  `ref`.

`t` может хранить внутри себя отображение идентификаторов сущностей на содержащие их состояние `ref`.
Т.е. может реализовывать Identity Map. В этом случае `t` выступает кэшем, при повторном запросе Алисы запроса в БД не будет.

`t` может запоминать начальное состояние сущностей,
чтобы в конце транзакции отследить, какие изменения нужно записать в БД.
Т.е. может реализовывать [Unit of Work](https://martinfowler.com/eaaCatalog/unitOfWork.html).
Или, если добавить к `Ref` поддержку наблюдателей, то становится возможным
при каждом изменении `ref` сбрасывать изменения в БД.
Это оптимистический и пессимистический подходы к фиксации изменений.

При оптимистическом подходе нужно отслеживать версии состояний сущностей.
При изменении из БД мы должны запомнить версию, а при фиксации изменений проверить, что версия сущности
в БД не отличается от начальной. В противном случае нужно повторить бизнес транзакцию.
Такой подход позволяет использовать групповые операции вставки и удаления и очень короткие транзакции БД,
что экономит ресурсы.

При пессимистическом подходе транзакция БД полностью соответствует бизнес транзакции.
Т.е. мы вынуждены забирать соединение из пула все на время исполнения бизнес транзакции.

API позволяет извлекать сущности по одной, что не очень оптимально.
Т.к. у нас реализован паттерн
[Identity Map](https://martinfowler.com/eaaCatalog/identityMap.html),
то мы можем ввести в API метод `preload`:

```js
storage.tx( t => {
  t.preload([0, 1, 2, 3]);
  const alice = t.get(0); // from cache
});
```

## Queries

Если мы не хотим длинных транзакций, то мы не можем делать выборки по произвольному ключу,
т.к. память может содержать "грязные" объекты и выборка вернет неожиданный результат.

Мы можем воспользоваться Запросами (Query) и извлекать любые данные (состояние) вне транзакции
и перечитать данные, находясь в транзакции.

```js
const aliceId = userQuery.findByEmail('alice@mail.com');
storage.tx( t => {
  const alice = t.getOne(aliceId);
});
```

При этом происходит разделение ответственности.
Для запросов мы можем использовать поисковые движки, масштабировать чтение с помощью реплик.
А API storage всегда работает с основным хранилищем (мастер).
Естественно, что реплики будут содержать устаревшие данные, перечитывание данных в транзакции
решает эту проблему.

## Commands

Бывают ситуации, когда операцию можно выполнить без чтения данных.
Например, списать месячную плату со счетов всех клиентов.
Или вставить и обновить при конфликте данные (upsert).

В случае проблем с производительностью связку из Storage и Query можно заменить такой командой.

## Связи

Если сущности беспорядочно ссылаются друг на друга, сложно обеспечить согласованность при их изменении.
Связи стараются максимально упростить, упорядочить, отказаться от лишних.

[Агрегаты](https://martinfowler.com/bliki/DDD_Aggregate.html)- способ упорядочивания связей.
Каждый агрегат имеет корневую сущность и вложенные сущности.
Любая внешняя сущность может ссылаться только на корень агрегата.
Корень обеспечивает целостность всего агрегата.
Транзакция не может пересекать границу агрегата, иными словами в транзакции участвует агрегат целиком.

Агрегат может, например, состоять из Поста (корень) и его переводов. Или Заказа и его Позиций.

Наш API работает с целыми агрегатами. При этом обеспечение ссылочной целостности между агрегатами
ложится на приложение. API не поддерживает ленивую загрузку связей.
Но мы можем выбирать направление связей.
Рассмотрим связь один ко многим Пользователь - Пост. Мы можем хранить идентификатор пользователя в посте,
но будет ли это удобно? Гораздо больше информации мы получим, если будем хранить массив идентификаторов
постов в пользователе.

## Заключение

Я сделал акценты на проблемы при работе с БД, показал вариант применения иммутабельности.
Формат статьи не позволяет детально раскрыть тему.

Если вас заинтересовал такой подход, то обратите внимание на мою книгу
[app from scratch](https://app-from-scratch.darkleaf.ru/),
в которой описывается создание веб приложения с нуля с упором на архитектуру.
В ней разбираются SOLID, Clean Architecture, паттерны работы с БД.
Примеры кода в книге и само [приложение](https://github.com/darkleaf/publicator)
написаны на языке Clojure, который пропитан идеями иммутабельности и удобством обработки данных.
