# API-CI [![Build status](https://ci.appveyor.com/api/projects/status/1ua40i539lq1poyt?svg=true)](https://ci.appveyor.com/project/mzrivan/api-ci)

# Домашнее задание к занятию «1.2. Тестирование API, CI»
## Задача №1: настройка CI

Напоминаем, CI — это чаще всего отдельная система — сервер, набор серверов, облако — в которой ваш код и ваши автотесты собираются в автоматическом режиме без вашего непосредственного участия. Вы лишь настраиваете CI для того, чтобы при возникновении определённых событий, например, push в репозиторий, стартовал процесс сборки и прогона тестов.

Детальнее про CI вы можете узнать, погуглив «Continuous integration», «Jenkins», «GitLab CI», «AppVeyor», «Travis», «CircleCI», «GitHub Actions».

Что надо сделать: берёте проект [с лекции](https://github.com/netology-code/aqa-code/tree/master/api-ci/rest), настраиваете для него CI (см. инструкцию ниже). Удостоверяетесь, что CI показывает, что сборка падает. В процессе сборки будут автоматически прогоняться все автотесты.

**Важно**: иногда можно настроить CI так, что там **всегда будет success** 😈! Не забывайте убедиться, что в CI сборка действительно падает, если вы запушите в GitHub падающий тест.

<details>
  <summary>Подсказка</summary>
  
  Возможно, это как-то связано с файлом gradlew и правами доступа на него. Для добавления прав на запуск файла gradlew, добавьте в CI исполнение команды `chmod +x gradlew` перед тем как использовать этот файл как команду для работы с гредлом. 
</details>

Общая схема работы выглядит следующим образом: CI должен запустить целевой сервис, который вы и тестируете, в фоновом режиме и ваши автотесты. Для этого мы будем на этот раз использовать возможности Bash.

Для того чтобы запустить целевой сервис, есть несколько вариантов, самый простой из которых — положить JAR-файл прямо в ваш репозиторий. Когда AppVeyor будет выкачивать исходники автотестов, он выкачает и ваш сервис.

Конечно, вы должны понимать, что в реальной жизни артефакты (собранный целевой сервис) хранятся в специальных системах, и процесс выкачивания будет зависеть от того, где и как хранится артефакт.

Ваш целевой сервис (SUT — System under test), расположен в файле [app-mbank.jar](app-mbank.jar), этот же файл используется в примерах на лекции. Вам нужно его положить в каталог `artifacts` вашего проекта, который необходимо создать.

Поскольку файлы с расширением `.jar` находятся в списках `.gitignore`, вам нужно принудительно заставить Git следить за ними: `git add -f artifacts/app-mbank.jar`.

После чего сделать `git push`. Обязательно удостоверьтесь, что файл попал в репозиторий.

### AppVeyor

[AppVeyor](https://www.appveyor.com) — одна из платформ, предоставляющих функциональность Continuous integration. В базовом варианте она бесплатна.

## Задача №2: JSON Schema

JSON Schema предлагает нам инструмент валидации JSON-документов. С описанием вы можете познакомиться по [адресу](https://json-schema.org/understanding-json-schema/index.html).

Как строится схема: 
```js
{
  "$schema": "http://json-schema.org/draft-07/schema", // версия схемы: https://json-schema.org/understanding-json-schema/reference/schema.html
  "type": "array", // тип корневого элемента: https://json-schema.org/understanding-json-schema/reference/type.html
  "items": { // какие элементы допустимы внутри массива: https://json-schema.org/understanding-json-schema/reference/array.html#items
    "type": "object", // должны быть объектами: https://json-schema.org/understanding-json-schema/reference/object.html
    "required": [ // должны содержать следующие поля: https://json-schema.org/understanding-json-schema/reference/object.html#required-properties
      "id",
      "name",
      "number",
      "balance",
      "currency"
    ],
    "additionalProperties": false, // дополнительных полей быть не должно 
    "properties": { // описание полей: https://json-schema.org/understanding-json-schema/reference/object.html#properties
      "id": {
        "type": "integer" // целое число: https://json-schema.org/understanding-json-schema/reference/numeric.html#integer
      },
      "name": {
        "type": "string", // строка: https://json-schema.org/understanding-json-schema/reference/string.html
        "minLength": 1 // минимальная длина — 1: https://json-schema.org/understanding-json-schema/reference/string.html#length
      },
      "number": {
        "type": "string", // строка: https://json-schema.org/understanding-json-schema/reference/string.html
        "pattern": "^•• \\d{4}$" // соответствует регулярному выражению: https://json-schema.org/understanding-json-schema/reference/string.html#regular-expressions
      },
      "balance": {
        "type": "integer" // целое число: https://json-schema.org/understanding-json-schema/reference/numeric.html#integer
      },
      "currency": {
        "type": "string" // строка: https://json-schema.org/understanding-json-schema/reference/string.html
      }
    }
  }
}
```

Что нужно сделать:

#### Шаг 1. Добавить зависимость

```groovy
dependencies {
    testImplementation 'io.rest-assured:rest-assured:4.3.0'
    testImplementation 'io.rest-assured:json-schema-validator:4.3.0'
    testImplementation 'org.junit.jupiter:junit-jupiter:5.6.1'
}
```

#### Шаг 2. Сохранить схему в ресурсах

Создайте каталог `resources` в `src/test` и поместите туда схему. Не забудьте удалить комментарии:

![](pic/schema.png)

#### Шаг 3. Включить проверку схемы

Модифицируйте существующий тест так, чтобы он проверял соответствие схеме. Для этого:

```java
      // код теста
      .then()
          .statusCode(200)
          // static import для JsonSchemaValidator.matchesJsonSchemaInClasspath
          .body(matchesJsonSchemaInClasspath("accounts.schema.json"))
      ;
```

Удостоверьтесь, что тесты проходят при соответствии ответа схеме и падают, если вы поменяете что-то в схеме, например, тип для `id`.

#### Шаг 4. Доработать схему

Изучите документацию на тип [`object`](https://json-schema.org/understanding-json-schema/reference/object.html) и найдите способ валидации значения поля на два из возможных значений: «RUB» или «USD».

Доработайте схему соответствующим образом, удостоверьтесь, что тесты проходят, в том числе в CI.

Поменяйте «RUB» на «RUR» и удостоверьтесь, что тесты падают, в том числе в CI.

Пришлите на проверку ссылку на ваш репозиторий. Удостоверьтесь, что в истории сборки были как success, так и fail, иначе будет не видно, как вы проверяли, что сборка падает в CI.
