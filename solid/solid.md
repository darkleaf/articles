https://habrahabr.ru/post/348286/


> SOLID критикует тот, кто думает, что действительно понимает ООП
> (c) Куряшкин Виктор

Я знаком с принципами SOLID уже 6 лет, но только в последний год осознал, что они означают. В этой статье я дам простое объяснение этим принципам. Расскажу о минимальных требованиях к языку программирования для их реализации. Дам ссылки на материалы, которые помогли мне разобраться.
<cut />

## Первоисточники
Придумал принципы SOLID Роберт Мартин (Uncle Bob). Естественно, что в своих работах он освещает эту тему.

Книга “Принципы, паттерны и методики гибкой разработки на языке C#” 2011 года. Большинство статей, которые я видел, основываются именно на этой книге. К сожалению, она дает расплывчатое описание принципов, что сильно ударило по их популярности.

Видео сайта [cleancoders.com](https://cleancoders.com/videos/clean-code/solid-principles). Дядюшка Боб в шутливой форме на пальцах рассказывает, что же именно означают принципы и как их применять.

Книга “Clean Architecture” 2017 года. Описывает архитектуру, построенную из кирпичиков, удовлетворяющих SOLID принципам. Дает определение структурному, объектно-ориентированному, функциональному программированию. Содержит лучшее описание SOLID принципов, которое я когда-либо видел.

## Требования
SOLID всегда упоминают в контексте ООП. Так получилось, что именно в ООП языках появилась удобная и безопасная поддержка динамического полиморфизма. Фактически, в контексте SOLID под ООП понимается именно динамический полиморфизм.

Полиморфизм дает возможность для <u>разных</u> типов использовать <u>один</u> код. 
Полиморфизм можно грубо разделить на динамический и статический. 
-  Динамический полиморфизм - это про абстрактные классы, интерфейсы, утиную типизацию, т.е. только в рантайме будет понятно,с каким типом будет работать наш код.
- Статический полиморфизм - это в основном про шаблоны (genererics). Когда уже на этапе компиляции из одного шаблонного кода генерируется код специфичный для каждого используемого типа.

Кроме привычных языков вроде Java, C#, Ruby, JavaScript, динамический полиморфизм реализован, например в 
- Golang, с помощью интерфейсов
- Clojure, с помощью протоколов и мультиметодов
- в прочих, совсем не “ООП” языках   

## Принципы
SOLID принципы советуют, как проектировать модули, т.е. кирпичикам, из которых строится приложение. Цель принципов - проектировать модули, которые:
- способствуют изменениям
- легко понимаемы
- повторно используемы

## SRP: The Single Responsibility Principle
*A module should be responsible to one, and only one, actor.*
Старая формулировка: *A module should have one, and only one, reason to change*.

Часто ее трактовали следующим образом: *Модуль должен иметь только одну обязанность*. И это главное заблуждение при знакомстве с принципами. Все несколько хитрее.

На каждом проекте люди играют разные роли (actor): Аналитик, Проектировщик интерфейсов, Администратор баз данных. Естественно, один человек может играть сразу несколько ролей. В этом принципе речь идет о том, что изменения в модуле может запрашивать одна и только одна роль. Например, есть модуль, реализующий некую бизнес-логику, запросить изменения в этом модуле может только Аналитик, но никак не DBA или UX.

## OCP: The Open Closed Principle
*A software artifact should be open for extension but closed for modification.*
Старая формулировка: *You should be able to extend a classes behavior, without modifying it*.

Это определенно может ввести в ступор. Как можно расширить поведение класса без его модификации? В текущей формулировке Роберт Мартин оперирует понятием артефакт, т.е. jar, dll, gem, npm package. Чтобы расширить поведение, нужно воспользоваться динамическим полиморфизмом.

Например, наше приложение должно отправлять уведомления. Используя dependency inversion, наш модуль объявляет только интерфейс отправки уведомлений, но не реализацию. Таким образом, логика нашего приложения содержится в одном dll файле, а класс отправки уведомлений, реализующий интерфейс - в другом. Таким образом, мы можем без изменения (перекомпиляции) модуля с логикой использовать различные способы отправки уведомлений.

Этот принцип тесно связан с LSP и DIP, которые мы рассмотрим далее.

## LSP: The Liskov Substitution Principle	
Имеет сложное математическое определение, которое можно заменить на: *Функции, которые используют базовый тип, должны иметь возможность использовать подтипы базового типа, не зная об этом*.

Классический пример нарушения. Есть базовый класс Stack, реализующий следующий интерфейс: length, push, pop. И есть потомок DoubleStack, который дублирует добавляемые элементы. Естественно, класс DoubleStack нельзя использовать вместо Stack.

У этого принципа есть забавное следствие: *Объекты, моделирующие сущности, не обязаны реализовывать отношения этих сущностей*. Например, у нас есть целые и вещественные числа, причем целые числа - подмножество вещественных. Однако, double состоит из двух int: мантисы и экспоненты. Если бы int наследовал от double, то получилась бы забавная картина: родитель содержит 2-х своих детей.

В качестве второго примера можно привести Generics. Допустим, есть базовый класс `Shape` и его потомки `Circle` и `Rectangle`. И есть некая функция `Foo(List<Shape> list)`. Мы считаем, что `List<Circle>` можно привести к `List<Shape>`. Однако, это не так. Допустим, это приведение возможно, но тогда в `list` можно добавить любую фигуру, например `rectangle`. А изначально `list` должен содержать только объекты класса `Circle`.

## ISP: The Interface Segregation Principle	
*Make fine grained interfaces that are client specific.*

Под интерфейсом здесь понимается именно Java, C# интерфейс. Разделение интерфейса облегчает использование и тестирование модулей. 

## DIP: The Dependency Inversion Principle
*Depend on abstractions, not on concretions.*

- Модули верхних уровней не должны зависеть от модулей нижних уровней. Оба типа модулей должны зависеть от абстракций.
- Абстракции не должны зависеть от деталей. Детали должны зависеть от абстракций.

Что такое модули верхних уровней? Как определить этот уровень? Как оказалось, все очень просто. Чем ближе модуль к вводу/выводу, тем ниже уровень модуля. Т.е. модули, работающие с BD, интерфейсом пользователя, низкого уровня. А модули, реализующие бизнес-логику - высокого уровня.

Что такое зависимость модулей? Это ссылка на модуль в исходном коде, т.е. import, require и т.п. С помощью динамического полиморфизма в runtime можно обратить эту зависимость.

Есть модуль Logic, реализующий логику, который должен отсылать уведомления. В этом же пакете объявляется интерфейс ISender, который используется Logic. Уровнем ниже, в другом пакете объявляется ConcreteSender, реализующий ISender. Получается, что в момент компиляции Logic не зависит от ConcreteSender. В runtime, например, через конструктор в Logic устанавливается экземпляр ConcreteSender.

Отдельно стоит отметить частый вопрос *“Зачем плодить абстракции, если мы не собираемся заменять базу данных?”*.

Логика тут следующая. На старте проекта, мы знаем, что будем использовать реляционную базу данных, и это точно будет Postgresql, а для поиска - ElasticSearch. Мы даже не планируем их менять в будущем. Но мы хотим отложить принятие решений о том, какая будет схема таблиц, какие будут индексы, и т.п. до момента, пока это не станет проблемой. И на этот момент мы будем обладать достаточной информацией, чтобы принять правильное решение. Также мы можем раньше отладить логику нашего приложения, реализовать интерфейс, собрать обратную связь от заказчика, и минимизировать последующие изменения, ведь многое реализовано только в виде заглушек.

Принципы SOLID подходят для проектов, разрабатываемых по гибким методологиям, ведь Роберт Мартин - один из авторов Agile Manifesto.

Принципы SOLID стремятся свести изменение модулей к их добавлению и удалению. 

Принципы SOLID способствуют откладыванию принятия технических решений и разделению труда программистов.