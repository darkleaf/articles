# Как приложению работать с БД?

В начале я обозначу некоторые проблемы при работе с БД, покажу дыры в абстракциях.
Далее мы разберем более простую абстракцию, основанную на иммутабельности.

Предполагается, что читатель знаком с паттерном Active Record.
Я хорошо знаком с его реализаций из Rails: ActiveRecord.
Также я буду ссылаться на концепции релизации паттерна Data Mapper - Hibernate.

Это не критика ActiveRecord и Rails. Они хороши для своей ниши.

Контекст статьи: достаточно большие проекты, которые нельзя выкинуть и быстро переписать.

## Dirty checking

Рассмотрим следующий код на ruby:

```ruby
post.title += " (Editor's Choice)"
post.save
```
Напомню, что в ruby по умолчанию строки мутабельны.
Из-за этого [Dirty](https://api.rubyonrails.org/classes/ActiveModel/Dirty.html)
не может отследить это in-place изменение.

"Правильное" решение выглядит так:

```ruby
post.title_will_change!
post.title += " (Editor's Choice)"
post.save
```

Одна из стратегий
[dirty checking](https://vladmihalcea.com/the-anatomy-of-hibernate-dirty-checking/)
в Hibernate - это сохранение снимка сущности после ее восстановления из БД
и сравнение этого снимка с состоянием сущности.

## Identity map

Следующая проблема - проблема сохранения идентичности.
Идентичность - нечто, что однозначно задает сущность.
В базе данных - это первичный ключ, а в памяти - ссылка (указатель).
Хорошо, когда ссылки указывают только на один объект.

В Active Record это не так:

```ruby
post_a = Post.find 1
post_b = Post.find 1

post_a.object_id != post_b.object_id # true

post_a.title = "foo"
post_b.title != "foo" # true
```

Т.е. мы получаем 2 ссылки на 2 разных объекта в памяти.

Таким образом, мы можем потерять  изменения,
если по невнимательности начнем работать с одной и той же сущностью,
но представленной разными объектами.

Hibernate имеет сессию, фактически кэш первого уровня, который хранит сопоставление идентификатора сущности
на объект в памяти. Если мы повторно запросим ту же сущность, то получим существующий объект.
Т.е. Hibernate реализует паттерн [Identity Map](https://martinfowler.com/eaaCatalog/identityMap.html).



```ruby
post.likes.length #=> 2
Like.create post: post, # ...
post.likes.length #=> 2
```



## Долгие транзакции

Но, что если мы делаем выборки не по идентификатору?
Чтобы не допустить рассинхронизации состояния объектов и состояния бд,
Hibernate перед запросом выборки делает
[flush](https://docs.jboss.org/hibernate/stable/core.old/reference/en/html/objectstate-flushing.html),
т.е. сохраняет "грязные" объекты, чтобы запрос прочитал согласованные данные.

Это вынуждает держать открытой транзакцию БД, пока идет бизнес транзакция.
Если бизнес тразакция долгая, например запрашивает данные по сети или выполняет расчеты,
то процесс в БД, отвечающий за соедиенени простаивает.



удержание соединения при длинных транзакциях.

https://postgrespro.ru/docs/postgrespro/10/pgbouncer
держит коннект, пока активна транзакция.

## N+1

Пожалуй самая большая "дыра" в абстракции ORM - проблема N+1 запроса.

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
тербеуется время (latency). Даже если запрос выполняется 10ms, то при 100 запросов будут выполняться секунду.

## Дополнительные данные

Скажем, чтобы избежать проблемы N+1 вы пишете такой
[запрос](https://www.db-fiddle.com/f/6m5FACAHWCeRSmKrTXriVH/0):

```sql
SELECT * FROM posts JOIN LATERAL (
  SELECT * FROM likes WHERE post_id = posts.id ORDER BY likes.id DESC LIMIT 1
) as last_like ON true;
```



Т.е. кроме атрибутов поста, выбирается еще и идентификатор лайка. На какой объект отобразить эти данные?

Как вариант, можно создать представление (view) и настроить модель для работы с этим представлением вместо таблицы,
а код разделить с момощью примесей или наследования.

Это не единственный случай. Вы можете выполнять полнотекстовый поиск и хотите отображать ранк и подсветить данные:

```sql
```

## State & identity

Рассмотрим следующий код на js:

```js
let foo = { bar: 1 };
foo.bar = 2;
foo = 1;
```

Как видите, в js у нас есть  2 "степени свободы": изменяющиеся переменные и изменяющиеся данные.

Можно воспользоваться константами и убрать одну степень:

```js
const foo = { bar: 1 };
foo.bar = 2;
foo = 1; // BOOM!
```

Т.е. объект, ссылку на который мы назвали `foo`, выполняет 2 обязанности:
одновременно моделирует идентичность и состояние.

Напомню, что у сущности есть 2 идентичности: ссылка и первичный ключ в БД.
И константы не могут помешать сделать Алису Бобом, пусть даже после сохранения.

```js
const alice = { id: 0, name: 'Alice' };
const bob =   { id: 1, name: 'Bob' };

alice.id = bob.id;
```

А что если разделить эти 2 обязанности? Отдельный объект для ссылки, а состояние сделать неизменяемым.

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

class UserState extends Immutable.Record({ id: null, name: '' }) {
}

const aliceState = new UserState({id: 0, name: 'Alice'});
const alice = new Ref( alice, (old, new) => old.id === new.id );

alice.swap( old => old.set('name', 'Queen Alice') );
alice.swap( old => old.set('id', 1) ); // BOOM!
```

## Storage

## Queries

## Comands
