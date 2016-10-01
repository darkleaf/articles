Веб разработка, функциональное программирование.

# Организация роутинга в веб-приложении

Существует набор библиотек на различных языках, имеющее общие черты. 
Это compojure, sinatra, grape, express, koa и подобные.

У них схожий подход к роутингу.
Они не накладывают никаких ограничений и не предлагают структуру для огранизации url.
Разработчики в таках условиях склонны не заботиться о стуруктуре и в последствии получать полохо поддерживаемый код.

Другая общая черта - это однонаправленность. Т.е. определенному запросу соответствует определенный обработчик.
Разработчики вынуждены прописывать url в виде строк. Нет возможности указать в виде конструкции языка какой url нужно сгенеровать.
Это приводит к тому, что представлениях остаются мертвые ссылки и нет способа найти их, кроме как протыкать все страницы.

Я расскажу как улучшить поддеерживаемость кода в экосистеме Clojure и покажу как:

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

Если вы не знакомы с rails, то вот ссылка на описаниее роутинга ["Rails Routing from the Outside In"](http://guides.rubyonrails.org/routing.html). Eсли вы подзабыли что такое HTTP, REST, то я советую прочитать короткую и шутливую статью ["15 тривиальных фактов о правильной работе с протоколом HTTP"](https://habrahabr.ru/company/yandex/blog/265569/).

Итак, используйте REST для огранизации url.

