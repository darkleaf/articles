Веб разработка, функциональное программирование.

# Организация роутинга в clojure веб-приложении

Существует набор библиотек на различных языках, имеющее общие черты. 
Это compojure, sinatra, grape, express, koa и подобные.

У них схожий подход к роутингу.
Они не накладывают никаких ограничений и не предлагают структуру для огранизации url.
Разработчики в таках условиях склонны не заботиться о стуруктуре и в последствии получают полохо поддерживаемый код.

Другая общая черта - это однонаправленность. Т.е. определенному запросу соответствует определенный обработчик.
Разработчики вынуждены прописывать url строками в шаблонах. Нет возможности указать в виде конструкции языка, какой url сгенеровать.
Это приводит к тому, что представлениях остаются мертвые ссылки и нет способа найти их, кроме как протыкать все страницы.

Я расскажу как улучшить поддерживаемость кода в экосистеме Clojure и покажу как:

1. организовать url'ы
2. структурировать код обработчиков
3. использовать языковые конструкции для генерации url

***

Важно понимать, что перечисленные выше библиотеки имеют средства для организации обработчиков в модули, как-то структурировать их.
Но об этом, как правило, задумываются слишком поздно.

Я ruby разработчик, и в других экосистемах(clojure, js, erlang, go), мне не хватает огранизации роутинга, подобного rails.
Мне не хватает REST и понятия "ресурс".
Мне не хватает контроллеров ресурсов для структурирования кода моего приложения.
Мне не хватает хелперов, для генерации uri, вроде `admin_page_path(@page)`.

Если вы не знакомы с rails, то вот ссылка на описаниее роутинга ["Rails Routing from the Outside In"](http://guides.rubyonrails.org/routing.html). 
Eсли вы подзабыли, что такое HTTP, REST, то я советую прочитать короткую и шутливую статью ["15 тривиальных фактов о правильной работе с протоколом HTTP"](https://habrahabr.ru/company/yandex/blog/265569/).

Итак, используйте REST для огранизации url.

Для того, что бы разобраться с оставшимися двумя пунктами, я покажу примеры кода с помощью своей библиотеки [darkleaf/router](https://github.com/darkleaf/router/). Т.к. это экосистема clojure, то, разумеется, это ring-совместимый роутинг.

```clojure
(ns hello-world.core
  (:require [darkleaf.router :refer :all]))
  
(def pages-controller
  {:middleware (fn [handler] (fn [req] req))
   :member-middleware some-other-middleware
   :index (fn [req] some-ring-response)
   :show (fn [req] some-ring-response)})

(def routes
  (build-routes
   (resources :pages 'page-id pages-controller)))

(def handler (build-handler routes))
(def request-for (build-request-for routes))

(handler {:uri "/pages", :request-method :get}) ;; call index action from pages-controller
(request-for :index [:pages]) ;; returns {:uri "/pages", :request-method :get}

(handler {:uri "/pages/1", :request-method :get}) ;; call show action from pages-controller
(request-for :show [:pages] {:page-id "1"}) ;; returns {:uri "/pages/1", :request-method :get}
```

Здесь, подобно rails, объявляется контролер. В данном случае с двумя экшенами: index и show.
Контроллер это всего лишь map, и вы можете, например генерировать похожие контроллеры с помощью вашей функции.
Котнтроллер может содержать только следующие ключи:

* :middleware - middleware которая оборачивает все экшены контроллера, включая обработчики вложенных роутов
* :member-middleware - оборачивает только member actions и обработчики вложенных роутов, помеченные как member
* collection actions: :index, :new, :create
* member actions: :show, :edit, :update, :destroy

Вот полный пример для ресурса:

```clojure
(resources :pages 'page-id {:index identity
                            :new identity
                            :create identity
                            :show identity
                            :edit identity
                            :update identity
                            :destroy identity}
           :collection
           [(action :archived identity)]
           :member
           [(resources :comments 'comment-id {:index identity})])
```
Первым аргуменом указывается имя ресурсов, им же задается сегмент в url. Вторым параметром задается название идентификатора ресурса.

Как я упоминал выше, ресурсы могут включать в себя вложенные роуты двух типов: collection и member.

```clojure
;; pages collection routes
(request-for :archived [:pages] {}) ;; #=> {:uri "/pages/archived", :request-method :get}

;; pages member routes
(request-for :index [:pages :comments] {:page-id "some-id"}) ;; #=> {:uri "/pages/some-id/comments", :request-method :get}
```

Кроме функции-генератора роутов resources, есть так же: root, action, wildcard, not-found, scope, guard, resource. 
Я не буду на них останавливаться, подробные примеры их использования вы найдете в [тестах](https://github.com/darkleaf/router/blob/master/test/darkleaf/router_test.clj).

Нет способа добавить в контроллер новые экшены. Это осознанное ограничение, подталкивающее к REST и вложенным ресурсам.
Если вам все-таки нужны дополнительные экшены и вы не хотите использовать вложенные ресурсы, 
то вы можете использовать вложенный роут, сгенерированный функцией action, который используется в примере выше для роута :archived.
Но это будет отдельная функция, вне контроллера.

Как вы уже заметили `request-for` возвращает структуру запроса целиком, а не только uri, в отличие от рельсовых хэлперов.
Это полезно, когда для определения обработчика используются заголовки или другие параметры запроса, например, host.
В следующих релизах планирутеся поддержка clojurescript и вы можете использовать тот же request-for для посроения запросов к бэкенду.
     
Библиотека поделена на 2 неймспейса: darkleaf.router и darkleaf.router.low-level.
Если у вас какие-то специфические требования к роутингу или вы поддерживаете старую схему url, 
то вы можете написать свои функции поверх darkleaf.router.low-level, точно так же как это сделано в darkleaf.router.

Полные примеры использования вы найдете в тестах 
[darkleaf.router](https://github.com/darkleaf/router/blob/master/test/darkleaf/router_test.clj)
и
[darkleaf.router.low-level](https://github.com/darkleaf/router/blob/master/test/darkleaf/router/low_level_test.clj).

Библиотка использует внутри `core.match` и довольно занятные макросы. Но это уже тема отдельной статьи. Пишите в комментариях, если вам интересно почитать про то, как это работает внутри. 

