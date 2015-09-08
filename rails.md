# Управление сложностью в ruby on rails проектах

В серии статей я соберу довольно большую часть своего опыта разработки на Ruby on Rails. Эти методики позволяют держать сложность в рамках и оттягивают тот момент, когда проект становится сложно сопровождать. Применять эти методики можно на проектах любой сложности. Могут возникнуть вопросы зачем оверхэд на маленьких проектах? Ответ: Маленькие проекты имеют свойство становиться большими, оверхеда практически нет, и на всех проектах применяется один подход. Так же это не влияет на скорость разработки новых фич, т.е. нет особой разницы впилить костыль или сделать расширяемо.

Основная проблема проектов на RoR в том, что как правило все пытаются уместить в модели, контроллеры и представления. Тут под моделями я подразумеваю классы-потомки ActiveRecord::Base. Такой подход приводит к печальным последствиям: долго делаются фичи, появляются регрессии, у разработчиков пропадает мотивация. Кому интересно, можно посмотреть в исходники редмайна или coderwall.

Выход из данной ситуации довольно-таки очевидный. Давайте делать проекты не на ruby on rails, а с использованием ruby on rails. И возьмем на вооружение подходы из мира ООП и ФП.

Как это будет выглядеть: мы никуда не уходим от MVC и rails, просто пересмотрим эти понятия. Для начала расширим понятие модели. Модель - это не просто класс-наследник ORM. Модель - это вся бизнес логика приложения. Модель включает в себя: модели, сервисы, политики, репозитории, формы и другие элементы, которые я опишу далее. Так же расширим представления. Представления это - шаблоны, презентеры, хелперы, билдеры форм. Скоро я опишу как разбить логику контроллеров.

Кроме этих методик пригодятся знания по SOLID, ruby style guide, rails conventions, ruby object model, ruby metaprogramming, основным паттернам. Так же важно понимать от куда возникают проблемы и, что все это, по сути, обход проблем дизайна ruby, rails и библиотек.

Начнем с простого, с представлений.

Самый простой совет - используйте хэлперы. В них удобно завернуть частые операции:

```
module ModelHelper
  def han(model, attribute)
    model.to_s.classify.constantize.human_attribute_name(attribute)
  end

module ApplicationHelper
  def menu_item(model, action, name, url, link_options = {})
    return unless policy(model).send "#{action}?"
    content_tag :li do
      link_to name, url, link_options
    end
  end
end

#_nav.haml
= menu_item current_user, :show, t(:show_profile), user_path(current_user)
= menu_item current_user, :show, t(:edit_profile), edit_user_path(current_user)
= menu_item current_user, :statistics_show, t(:my_statistics), user_statistics_path(current_user)

module ApplicationHelper
 def show_attribute(model, attribute)
    value = model.send(attribute)
    return if value.blank?
    [
        content_tag(:dt, han(model.model_name, attribute)),
        content_tag(:dd, value)
    ].join.html_safe
  end
end

# show.haml
 = show_attribute user_presenter.model, :name
 = show_attribute user_presenter, :role_text
 = show_attribute user_presenter, :profile_image
 = show_attribute user_presenter, :email
 = show_attribute user_presenter, :contacts
```

Т.е. в хэлперы выносится повторяющиеся действия. Не следует в хэлперах делать верстку, для этого есть partials.

Рассмотрим формы:

```
= simple_form_for @user, builder: PunditFormBuilder do |f|
  = f.input :name
  = f.input :email
  = f.input :role
  = f.input :contacts, as: :big_textarea
  = f.input :profile_image, as: :attachment
  # some other inputs
  = f.button :submit
```

Я использую gem simple_form для рендеринга форм. Как видите, все получается компактно и понятно. Выделим следующие вещи: я не указываю какой-либо текст, использую свои inputs и builder.

Как я уже упоминал, важно знать соглашения в rails. Локали для labels, placeholders, submit подставляются автоматически - достаточно прописать в локалях правильные ключи и ваша форма будет автоматически переведена:

```
ru:
  # эти переводы для всех моделей
  attributes:
    created_at: Создано
    updated_at: Обновлено
  activerecord:
    models:
      user: Сотрудник
    attributes:
      user:
        name: Имя
        position: Должность
  helpers:
    submit:
      create: Сохранить
      invite_form:
        create: Пригласить
```

Теперь подробнее про свои inputs. SimpleForm позволяет писать свои imputs, это нужно, что бы не конфигурировать стандартные inputs. 

```
class BigTextareaInput < SimpleForm::Inputs::TextInput
  def input_html_options
    { rows: 10 }
  end
end
```

Так же SimpleForm позволяет подключать свои билдеры форм:
```
class PunditFormBuilder < SimpleForm::FormBuilder
  def input(attribute_name, options = {}, &block)
    return unless show_attribute? attribute_name
    super(attribute_name, options, &block)
  end

  def show_attribute?(attr_name)
    # some code
  end
end
```
PunditFormBuilder нужен, что бы показывать только те поля, к которым имеет доступ текущий пользователь приложения. Более подробно я расскажу об этом в главе про ACL.

Давайте теперь рассмотрим более специфическую задачу, а именно проектирование http json api. Вот наиболее простые способы:
* метод `Model#to_json`
* метод конроллера serialize_model

Все эти способы противоречат принципу единственной ответственности и паттерну MVC. Модель и конроллер не должны заниматься отображением - это обязанность представлений.

Я вижу 2 способа решения:
* шаблоны jbuilder
* serializers, причем как одноименный gem, так и просто объекты-сериализаторы

Т.е. представления это не только шаблоны и хэлперы, но и другие объекты, занимающиеся представлением данных, например - сериалайзеры.

Так мы плавно подошли к следующему подходу: использование презентеров. В rails они используются как дополнение хэлперов.

gem drapper внес хорошую путаницу: его разработчики назвали презентеры декораторами. Хотя эти паттерны похожи, но имеют значительное различие: декораторы не изменяют интерфейс. Так же с этим гемом есть много проблем, достаточно посмотреть на список issues.

Я нашел простой, элегантный и понятный способ реализовать презентеры http://nithinbekal.com/posts/rails-presenters/  


```
# app/presenters/base_presenter.rb
class BasePresenter < Delegator
  attr_reader :model, :h
  alias_method :__getobj__, :model

  def initialize(model, view_context)
    @model = model
    @h = view_context
  end

  def inspect
    "#<#{self.class} model: #{model.inspect}>"
  end
end

# app/presenters/task_presenter.rb
class TaskPresenter < BasePresenter
  def to_link
    h.link_to model.to_s, model
  end

  def description
    h.markdown model.description
  end

  def users
    model.users.map { |user| h.present user }
  end
end

# app/helpers/application_helper.rb
def present(model)
  return if model.blank?
  klass = "#{model.class}Presenter".constantize
  presenter = klass.new(model, self)
  yield(presenter) if block_given?
  presenter
end

```






* представления
  * хэлперы для часто встречающихся операций
  * simpleforms
    * inputs
    * builders
  * serializers
  * презентеры  
* контроллеры
  * тонкие контроллеры
  * пректирование по rest
  * иерархия контроллеров
  * responders
  * аутентификация
  * breadcrumbs
* модели
  * concerns
  * неймспейсы моделей
  * мультиязычность
  * репозитории
  * sequel && ransack
  * querues
  * view models (virtus, использубтся вместе с queries)
  * policies
  * forms
  * types
  * services
  * переопределние сеттеров
  * after_commit
  * ничего не удаляем
  * стейт машины и enumerize
* conventions over configurations
* денормолизация
  * counter_culture
* проектирование API
* интеграция со сторонними сервисами
  * service_locator
  * тестирование
* интеграция с другими базами данных
* ACL на примере портала
* миграции данных
* тестирование
* assets (webpack)
* monkeypatch
* человекопонятные урлы
