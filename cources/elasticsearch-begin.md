Это мой урок на Hexlet.io: [Введение в Elasticsearch](http://hexlet.io/lessons/elasticsearch_intro/).


Elasticsearch - поисковый движок с json rest api, использующий Lucene и написанный на Java. Описание всех преимуществ этого движка доступно на [официальном сайте](http://www.elasticsearch.org/overview/elasticsearch). Далее по тексту будем называть Elasticsearch как ES.

Подобные движки используются при сложном поиске по базе документов. Например, поиск с учетом морфологии языка или поиск по geo координатам.

В этом уроке я расскажу про основы ES на примере индексации постов блога. Покажу как фильтровать, сортировать и искать документы.

Что бы урок был максимально кроссплатформенным все запросы к ES я буду делать с помощью CURL. Так же есть [плагин для google chrome](https://chrome.google.com/webstore/detail/sense-beta/lhjgkmllcaadmopgmanpapmpjgmfcfig?hl=ru).

По тексту урока расставлены ссылки на другие источники. В конце урока размещены ссылки для быстрого доступа к документации. Определения незнакомых терминов можно прочитать в [глоссарии](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/glossary.html).

# Установка ES

Для этого нам сначала потребуется Java. Разработчики [рекомендуют](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/setup.html#jvm-version) установить версии Java, новее чем Java 8 update 20 или Java 7 update 55. 

Дистрибутив ES доступен на [сайте разработчика](http://www.elasticsearch.org/overview/elkdownloads/). После распаковки архива нужно запустить `bin/elasticsearch`. Так же доступны [пакеты для apt и yum](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/setup-repositories.html). [Подробнее об установке](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/_installation.html).

После установки и запуска проверим работоспособность:

```bash
curl -X GET http://localhost:9200/
```

Нам придет приблизительно такой ответ:

```json
{
  "status" : 200,
  "name" : "Thunderball",
  "version" : {
    "number" : "1.3.2",
    "build_hash" : "dee175dbe2f254f3f26992f5d7591939aaefd12f",
    "build_timestamp" : "2014-08-13T14:29:30Z",
    "build_snapshot" : false,
    "lucene_version" : "4.9"
  },
  "tagline" : "You Know, for Search"
}
```

# Индексация

Добавим пост в ES:

```bash
# Добавим документ c id 1 типа post в индекс blog
# curl -XPUT "http://localhost:9200/INDEX/TYPE/ID" -d'

curl -XPUT "http://localhost:9200/blog/post/1" -d'
{
   "title": "Веселые котята",
   "content": "<p>Смешная история про котят<p>",
   "tags": [
      "котята",
      "смешная история"
   ],
   "published_at": "2014-09-12T20:44:42+00:00"
}'
```

ответ сервера:

```json
{
   "_index": "blog",
   "_type": "post",
   "_id": "1",
   "_version": 1,
   "created": true
}
```

ES автоматически создал [индекс](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/glossary.html#glossary-index) blog и [тип](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/glossary.html#glossary-type) post. Индекс можно понимать как базу данных в реляционной СУБД, а тип - как таблицу в этой базе. Каждый тип имеет свою схему - [mapping](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/glossary.html#glossary-mapping), так же как и реляционная таблица. Mapping генерируется автоматически при индексации документа:

```bash
# Получим mapping всех типов индекса blog
curl -XGET "http://localhost:9200/blog/_mapping"
```

В ответе сервера я добавил в комментариях значения полей документа:

```json
{
   "blog": {
      "mappings": {
         "post": {
            "properties": {
            
               /* "content": "<p>Смешная история про котят<p>", */
               "content": {
                  "type": "string"
               },
               
               /* "published_at": "2014-09-12T20:44:42+00:00" */
               "published_at": {
                  "type": "date",
                  "format": "dateOptionalTime"
               },
               
			   /* "tags": ["котята", "смешная история"] */
               "tags": {
                  "type": "string"
               },
               
               /*  "title": "Веселые котята" */
               "title": {
                  "type": "string"
               }
            }
         }
      }
   }
}
```

Стоит отметить, что ES не делает различий между одиночным значением и массивом значений. Например, поле title содержит просто заголовок, а поле tags - массив строк, хотя они представлены в маппинге одинаково.
Позднее мы поговорим о маппинге более подобно.

# Запросы

## Извлечение документа по его id:

```bash
# извлечем документ с id 1 типа post из индекса blog
curl -XGET "http://localhost:9200/blog/post/1"
```
```json
{
   "_index": "blog",
   "_type": "post",
   "_id": "1",
   "_version": 1,
   "found": true,
   "_source": {
      "title": "Веселые котята",
      "content": "<p>Смешная история про котят<p>",
      "tags": [
         "котята",
         "смешная история"
      ],
      "published_at": "2014-09-12T20:44:42+00:00"
   }
}
```

В ответе появились новые ключи: `_version` и `_source`. Вообще, все ключи, начинающиеся с `_`относятся к служебным.

Ключ `_version` показывает версию документа. С ее помощью работает механизм оптимистических блокировок. Например,  мы хотим изменить документ, имеющего версию 1. Мы отправляем измененный документ и указываем, что это правка документа с  версией 1. Если кто-то другой тоже редактировал документ с версией 1 и отправил изменения раньше нас, то ES не примет наши изменения, т.к. он хранит документ с версией 2. 

Ключ `_source` содержит тот документ, который мы индексировали. ES не использует это значение для поисковых операций, т.к. для поиска используются индексы. Для экономии места ES хранит сжатый исходный документ. Если нам нужен только id, а не весь исходный документ, то можно отказаться от хранения исходника.

Если нам не нужна дополнительная информация можно получить только содержимое _source:

```bash
curl -XGET "http://localhost:9200/blog/post/1/_source"
```
```json
{
   "title": "Веселые котята",
   "content": "<p>Смешная история про котят<p>",
   "tags": [
      "котята",
      "смешная история"
   ],
   "published_at": "2014-09-12T20:44:42+00:00"
}
```

Так же можно выбрать только определенные поля:

```bash
# извлечем только поле title
curl -XGET "http://localhost:9200/blog/post/1?_source=title"
```
```json
{
   "_index": "blog",
   "_type": "post",
   "_id": "1",
   "_version": 1,
   "found": true,
   "_source": {
      "title": "Веселые котята"
   }
}
```

Давайте проиндексируем еще несколько постов и выполним более сложные запросы.

```bash
curl -XPUT "http://localhost:9200/blog/post/2" -d'
{
   "title": "Веселые щенки",
   "content": "<p>Смешная история про щенков<p>",
   "tags": [
      "щенки",
      "смешная история"
   ],
   "published_at": "2014-08-12T20:44:42+00:00"
}'
```

```bash
curl -XPUT "http://localhost:9200/blog/post/3" -d'
{
   "title": "Как у меня появился котенок",
   "content": "<p>душераздирающая история про бедного котенка с улицы<p>",
   "tags": [
      "котята"
   ],
   "published_at": "2014-07-21T20:44:42+00:00"
}'
```

## Сортировка

```bash
# найдем последний пост по дате публикации и извлечем поля title и published_at
curl -XGET "http://localhost:9200/blog/post/_search" -d'
{
    "size": 1,
    "_source": ["title", "published_at"],
    "sort": [{"published_at": "desc"}]
}'
```
```json
{
   "took": 2,
   "timed_out": false,
   "_shards": {
      "total": 5,
      "successful": 5,
      "failed": 0
   },
   "hits": {
      "total": 3,
      "max_score": null,
      "hits": [
         {
            "_index": "blog",
            "_type": "post",
            "_id": "1",
            "_score": null,
            "_source": {
               "title": "Веселые котята",
               "published_at": "2014-09-12T20:44:42+00:00"
            },
            "sort": [
               1410554682000
            ]
         }
      ]
   }
}
```

Мы выбрали последний пост. `size` ограничивает кол-во документов в выдаче. `total` показывает общее число документов, подходящих под запрос.  `sort` в выдаче содержит массив целых чисел по которым производится сортировка. Т.е. дата преобразовалась в целое число. Подробнее о сортировке можно прочитать в [документации](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/search-request-sort.html#search-request-sort).

## Фильтрация по дате

Используем фильтр [range](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-range-filter.html):

```bash
# получим посты опубликованные 1ого сентября или позже
curl -XGET "http://localhost:9200/blog/post/_search" -d'
{
   "query": {
      "filtered": {
         "filter": {
            "range": {
               "published_at": { "gte": "2014-09-01" }
            }
         }
      }
   }
}'
```

## Фильтрация по тегам

Используем [term filter](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-term-filter.html) для поиска id документов, содержащих заданное слово:

```bash
# найдем все документы, в поле tags которых есть элемент 'котята'
curl -XGET "http://localhost:9200/blog/post/_search" -d'
{
   "_source": [
      "title",
      "tags"
   ],
   "query": {
      "filtered": {
         "filter": {
            "term": {
               "tags": "котята"
            }
         }
      }
   }
}'
```

```json
{
   "took": 4,
   "timed_out": false,
   "_shards": {
      "total": 5,
      "successful": 5,
      "failed": 0
   },
   "hits": {
      "total": 2,
      "max_score": 1,
      "hits": [
         {
            "_index": "blog",
            "_type": "post",
            "_id": "3",
            "_score": 1,
            "_source": {
               "tags": [
                  "котята"
               ],
               "title": "Как у меня появился котенок"
            }
         },
         {
            "_index": "blog",
            "_type": "post",
            "_id": "1",
            "_score": 1,
            "_source": {
               "tags": [
                  "котята",
                  "смешная история"
               ],
               "title": "Веселые котята"
            }
         }
      ]
   }
}
```

## Полнотекстовый поиск

Три наших документа содержат в поле content следующее:

* `<p>Смешная история про котят<p>`
* `<p>Смешная история про щенков<p>`
* `<p>душераздирающая история про бедного котенка с улицы<p>`

Используем [query match](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-match-query.html) для поиска id документов, содержащих заданное слово:

```bash
# source: false означает, что не нужно извлекать _source найденных документов
curl -XGET "http://localhost:9200/blog/post/_search" -d'
{
   "_source": false,
   "query": {
      "match": {
         "content": "история"
      }
   }
}'
``` 
```json
{
   "took": 7,
   "timed_out": false,
   "_shards": {
      "total": 5,
      "successful": 5,
      "failed": 0
   },
   "hits": {
      "total": 3,
      "max_score": 0.11506981,
      "hits": [
         {
            "_index": "blog",
            "_type": "post",
            "_id": "1",
            "_score": 0.11506981
         },
         {
            "_index": "blog",
            "_type": "post",
            "_id": "2",
            "_score": 0.11506981
         },
         {
            "_index": "blog",
            "_type": "post",
            "_id": "3",
            "_score": 0.095891505
         }
      ]
   }
}
```

Однако, если искать "истории" в поле контент, то мы ничего не найдем, т.к. в индексе содержатся только оригинальные слова, а не их основы. Для того, что бы сделать качественный поиск нужно настроить анализатор.

# Анализаторы

[Анализаторы](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/analysis.html) нужны, что бы преобразовать исходный текст в набор токенов.
Анализаторы состоят из одного [Tokenizer](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/analysis-tokenizers.html) и нескольких необязательных [TokenFilters](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/analysis-tokenfilters.html). Tokenizer может предшествовать нескольким [CharFilters](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/analysis-charfilters.html). Tokenizer разбивают исходную строку на токены, например по пробелам и символам пунктуации. TokenFilter может изменять токены, удалять или добавлять новые, например оставлять только основу слова, убирать предлоги, добавлять синонимы. CharFilter - изменяет исходную строку целиком, например вырезает html теги.

В ES есть несколько [стандартных анализаторов](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/analysis-analyzers.html). Например анализатор [russian](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/analysis-lang-analyzer.html#russian-analyzer)

Воспользуемся [api](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/indices-analyze.html) и посмотрим как анализаторы standard и russian преобразуют строку "Веселые истории про котят": 

```bash
# используем анализатор standard
curl -XGET "http://localhost:9200/_analyze?analyzer=standard&text=%D0%92%D0%B5%D1%81%D0%B5%D0%BB%D1%8B%D0%B5%20%D0%B8%D1%81%D1%82%D0%BE%D1%80%D0%B8%D0%B8%20%D0%BF%D1%80%D0%BE%20%D0%BA%D0%BE%D1%82%D1%8F%D1%82"
```
```json
{
   "tokens": [
      {
         "token": "веселые",
         "start_offset": 0,
         "end_offset": 7,
         "type": "<ALPHANUM>",
         "position": 1
      },
      {
         "token": "истории",
         "start_offset": 8,
         "end_offset": 15,
         "type": "<ALPHANUM>",
         "position": 2
      },
      {
         "token": "про",
         "start_offset": 16,
         "end_offset": 19,
         "type": "<ALPHANUM>",
         "position": 3
      },
      {
         "token": "котят",
         "start_offset": 20,
         "end_offset": 25,
         "type": "<ALPHANUM>",
         "position": 4
      }
   ]
}
```

```bash
# используем анализатор russian
curl -XGET "http://localhost:9200/_analyze?analyzer=russian&text=%D0%92%D0%B5%D1%81%D0%B5%D0%BB%D1%8B%D0%B5%20%D0%B8%D1%81%D1%82%D0%BE%D1%80%D0%B8%D0%B8%20%D0%BF%D1%80%D0%BE%20%D0%BA%D0%BE%D1%82%D1%8F%D1%82"
```
```json
{
   "tokens": [
      {
         "token": "весел",
         "start_offset": 0,
         "end_offset": 7,
         "type": "<ALPHANUM>",
         "position": 1
      },
      {
         "token": "истор",
         "start_offset": 8,
         "end_offset": 15,
         "type": "<ALPHANUM>",
         "position": 2
      },
      {
         "token": "кот",
         "start_offset": 20,
         "end_offset": 25,
         "type": "<ALPHANUM>",
         "position": 4
      }
   ]
}
```

Стандартный анализатор разбил строку по пробелам и перевел все в нижний регистр, анализатор russian - убрал не значимые слова, перевел в нижний регистр и оставил основу слов.

Посмотрим какие Tokenizer, TokenFilters, CharFilters использует анализатор russian:

```json
{
  "filter": {
    "russian_stop": {
      "type":       "stop",
      "stopwords":  "_russian_"
    },
    "russian_keywords": {
      "type":       "keyword_marker",
      "keywords":   []
    },
    "russian_stemmer": {
      "type":       "stemmer",
      "language":   "russian"
    }
  },
  "analyzer": {
    "russian": {
      "tokenizer":  "standard",
      /* TokenFilters */
      "filter": [
        "lowercase",
        "russian_stop",
        "russian_keywords",
        "russian_stemmer"
      ]
      /* CharFilters отсутствуют */
    }
  }
}
```

Опишем свой анализатор на основе russian, который будет вырезать html теги. Назовем его default, т.к. анализатор с таким именем будет использоваться по умолчанию.

```json
{
  "filter": {
    "ru_stop": {
      "type":       "stop",
      "stopwords":  "_russian_"
    },
    "ru_stemmer": {
      "type":       "stemmer",
      "language":   "russian"
    }
  },
  "analyzer": {
    "default": {
      /* добавляем удаление html тегов */
      "char_filter": ["html_strip"],
      "tokenizer":  "standard",
      "filter": [
        "lowercase",
        "ru_stop",
        "ru_stemmer"
      ]
    }
  }
}
```

Сначала из исходной строки удалятся все html теги, потом ее разобьет на токены tokenizer standard, полученные токены перейдут в нижний регистр, удалятся незначимые слова и от оставшихся токенов останется основа слова.

# Создание индекса

Выше мы описали dafault анализатор. Он будет применяться ко всем строковым полям. Наш пост содержит массив тегов, соответственно теги тоже будут обработаны анализатором. Т.к. мы ищем посты по точному соответствию тегу, то необходимо отключить анализ для поля tags.

Создадим индекс blog2 с анализатором и маппингом, в котором отключен анализ поля tags:

```bash
curl -XPOST "http://localhost:9200/blog2" -d'
{
   "settings": {
      "analysis": {
         "filter": {
            "ru_stop": {
               "type": "stop",
               "stopwords": "_russian_"
            },
            "ru_stemmer": {
               "type": "stemmer",
               "language": "russian"
            }
         },
         "analyzer": {
            "default": {
               "char_filter": [
                  "html_strip"
               ],
               "tokenizer": "standard",
               "filter": [
                  "lowercase",
                  "ru_stop",
                  "ru_stemmer"
               ]
            }
         }
      }
   },
   "mappings": {
      "post": {
         "properties": {
            "content": {
               "type": "string"
            },
            "published_at": {
               "type": "date",
               "format": "dateOptionalTime"
            },
            "tags": {
               "type": "string",
               "index": "not_analyzed"
            },
            "title": {
               "type": "string"
            }
         }
      }
   }
}'
```

Добавим те же 3 поста в этот индекс(blog2). Я опущу этот процесс, т.к. он аналогичен добавлению документов в индекс blog.

# Полнотекстовый поиск с поддержкой выражений

Познакомимся с еще одним типом запросов: 

```bash
# найдем документы в которых встречается слово 'истории'
# query -> simple_query_string -> query содержит поисковый запрос
# поле title имеет приоритет 3
# поле tags имеет приоритет 2
# поле content имеет приоритет 1
# приоритет используется при ранжировании результатов
curl -XPOST "http://localhost:9200/blog2/post/_search" -d'
{
   "query": {
      "simple_query_string": {
         "query": "истории",
         "fields": [
            "title^3",
            "tags^2",
            "content"
         ]
      }
   }
}'
```

Т.к. мы используем анализатор с русским стеммингом, то этот запрос вернет все документы, хотя в них встречается только слово 'история'.

Запрос может содержать специальные символы, например:

```
"\"fried eggs\" +(eggplant | potato) -frittata"
```

Синтаксис запроса:

```
+ signifies AND operation
| signifies OR operation
- negates a single token
" wraps a number of tokens to signify a phrase for searching
* at the end of a term signifies a prefix query
( and ) signify precedence
~N after a word signifies edit distance (fuzziness)
~N after a phrase signifies slop amount
```

```bash
# найдем документы без слова 'щенки'
curl -XPOST "http://localhost:9200/blog2/post/_search" -d'
{
   "query": {
      "simple_query_string": {
         "query": "-щенки",
         "fields": [
            "title^3",
            "tags^2",
            "content"
         ]
      }
   }
}'

# получим 2 поста про котиков
```

# Ссылки

* [Elasticsearch](http://www.elasticsearch.org/)
* [документация](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/index.html)
* [глоссарий](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/glossary.html)
* [установка](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/_installation.html)
* [манипуляции с документами](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/docs.html)
* [операции с индексами](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/indices.html)
* [список запросов](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-queries.html)
* [список фильтров](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-filters.html)
