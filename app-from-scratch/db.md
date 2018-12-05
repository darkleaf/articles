# Как приложению работать с БД?

Мы рассмотрим некоторые проблемы при работе с БД
как с ORM так и без.
Увидим дыры в абстракциях.
И разберем более простую абстракцию, основанную на иммутабельности.

Предполагается, что читатель знаком с паттерном Active Record.
Я хорошо знаком с его реализаций из Rails: ActiveRecord.
Также я буду ссылаться на концепции релизации паттерна Data Mapper - Hibernate.

Контекст статьи: достаточно сложные проекты, которые нельзя выкинуть и быстро переписать.

Рассмотрим следующий код на ruby:

```ruby
post.title += " (Editor's Choice)"
post.save
```
Напомню, что в ruby по умолчанию строки мутабельны.
Из-за этого [Dirty](https://api.rubyonrails.org/classes/ActiveModel/Dirty.html)
не сможет отследить это in-place изменение.

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



Следующая проблема - проблема сохранения идентичности.

```ruby
post_a = Post.find 1
post_b = Post.find 1

post_a.object_id != post_b.object_id # true

post_a.title = "foo"
post_b.title != "foo" # true
```

Т.е. мы получаем 2 ссылки на 2 разных объекта в памяти.

Таким образом, мы можем потерять  изменения,
если по невнимательности мы начнем работать с одной и той же сущностью, но одновременно представленной двумя объектами.

Hibernate имеет сессию, фактически кэш первого уровня, который хранит сопоставление идентификатора сущности
на объект в памяти. Если мы повторно запросим ту же сущность, то получим существующий объект.

Т.е. Hibernate реализует паттерн [Identity Map](https://martinfowler.com/eaaCatalog/identityMap.html).

Но, что если мы делаем выборки не по идентификатору?
Чтобы не допустить рассинхронизации состояния объектов и состояния бд,
Hibernate перед запросом выборки делает
[flush](https://docs.jboss.org/hibernate/stable/core.old/reference/en/html/objectstate-flushing.html),
т.е. сохраняет "грязные" объекты, чтобы запрос прочитал согласованные данные.


Пожалуй самая большая "дыра" в абстракции ORM - проблема N+1 запроса.


```ruby
posts = Post.all # select * from posts
posts.each do |post|
  like = post.likes.where(user_id: 1).last # select * from likes where user_id = ?
  # ...
end
```

ORM склоняет программиста к мысли, что он работает просто с объектами в памяти.
Но он работает с доступным по сети сервисом, а на установление соединений и передачу данных
тербеуется время (latency). Даже если запрос выполняется 10ms, то при 100 запросов будут выполняться секунду.






что делать с частичными или дополнительными данными?
higlighting, rank и т.п.

```ruby
post.likes.length #=> 2
Like.create post: post, # ...
post.likes.length #=> 2
```



Дырявые абстракции.




показать разные подходы. указать на слабые и сильный стороны.
показать решение.

мутабельность, дырявые абстракции

## Active Record

проблемы паттерна на примере реализации

кэширование (запросов, страниц) как костыль


```ruby
class AbstractPost
  def initialize(id, title)
    @id = id
    @title = title
  end

  def self.find(id)
    fail "abstract method"
  end

  def save
    fail "abstract method"
  end
end

class PostImpl < AbstractPost
  # impl
end
```

## Hibernate






## Table Data Gateway

https://www.martinfowler.com/eaaCatalog/tableDataGateway.html

## Прочие

Datomic On-Perm
Datomic Cloud, Tarantool, Stored Procedures


##

ref + immutable
