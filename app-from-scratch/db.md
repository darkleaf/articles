# Способы работы с БД из приложения


Проблемы.

Отслеживание изменений.

```ruby
post.title += " (Editor's Choice)"
post.save
```

Дырявые абстракции.
ORM склоняет программиста думать, что он работает просто с объектами  в памяти.

Проблема идентичности.

```ruby
post1 = Post.first
post2 = Post.find post1.id
post1 != post2
```

ее решение в hibernate через identiy map.

но, что если мы делаем сложные выборки?
`flush`

```ruby
post.likes.length #=> 2
Like.create post: post, # ...
post.likes.length #=> 2
```

N+1

```ruby
posts = Post.all
posts.each do |post|
  like = post.likes.where(user_id: 1).last
  # ...
end
```

что делать с частичными или дополнительными данными?
higlighting, rank и т.п.




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
