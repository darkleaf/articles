# Управление сложностью в ruby on rails проектах

В серии статей я соберу довольно большую часть своего опыта разработки на Ruby on Rails. Львиную долю методик придумал не я и по возможности приведу ссылки на источники.

Данные методики позволяют держать сложность в рамках и оттягивают тот момент, когда проект невозможно сопровождать. Применять эти методики можно на проектах любой сложности. Могут возникнуть вопросы зачем такой оверхэд на маленьких проектах? Ответ: Маленькие проекты имеют свойство становиться большими, оверхеда практически нет, и на всех проктах применяется один подход. Так же это не влияет на скорость разработки новых фич, т.е. нет особой разницы впилить костыль или сделать нормально.

Кроме этих методик пригодятся знания по SOLID, ruby style guide, rails conventions, ruby object model, ruby metaprogramming, основным паттернам. Так же важно понимать, что все это по сути обход проблем дизайна ruby, rails, библиотек и ООП.

Основная проблема проектов на RoR в том, что как правило все пытаются уместить в модели, контроллеры и представления. Тут под моделями я подразумеваю классы-потомки ActiveRecord::Base. Такой подход приводит к печальным последствиям: долго делаются фичи, появляются регрессии, у разработчиков пропадает мотивация. Кому интересно, можно посмотреть в исходники редмайна или coderwall.

Выход из данной ситуации довольно-таки очевидный. Давайте делать проекты не на ruby on rails, а с использованием ruby on rails[можно сноску поставить]. И возьмем на вооружение подходы из мира OOP и FP.

Как это будет выглядеть: мы никуда не уходим от MVC и rails, просто пересмотрим эти понятия. Для начала расширим понятие модели. Модель - это не просто класс-наследник ORM. Модель - это вся бизнес логика приложения. Модель включает в себя: модели, сервисы, политики, репозитории, формы и другие элементы, которые я опишу далее. Так же расширим представления. Представления это - шаблоны, презентеры, хелперы, билдеры форм. Скоро я опишу как разбить логику контроллеров.


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
