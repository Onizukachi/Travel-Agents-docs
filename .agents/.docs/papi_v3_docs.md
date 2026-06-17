# PAPI v3 docs

Эта папка содержит исходники документации PAPI v3 для Swagger/OpenAPI.

## Как устроена структура

Для каждого контроллера есть отдельная папка:

```text
docs/papi/v3/controllers/<controller>/
  3.0.yml
  3.1.yml
  3.12.yml
```

Правила:

- `3.0.yml` обязателен для любого контроллера.
- Если в минорных версиях у контроллера ничего не менялось, в папке лежит только `3.0.yml`.
- Если контракт менялся, создаётся точный файл версии: `3.1.yml`, `3.12.yml`, `3.15.yml`.
- Диапазоны и суффиксы вроде `3.12+`, `3.0-3.11` больше не используются.

## Как работает сборка версий

Документация для версии собирается последовательно:

1. Берётся базовый файл `3.0.yml`.
2. Затем по порядку накладываются все файлы `3.x.yml` до нужной минорной версии.
3. Если операция описана повторно в более позднем файле, она заменяется целиком.
4. Если операция должна исчезнуть из документации в новой версии, она перечисляется в `removed_operations`.

Пример:

- чтобы понять документацию `v3.15` для `orders`, нужно прочитать:
  - [3.0.yml](leveltravel/docs/papi/v3/controllers/orders/3.0.yml)
  - [3.12.yml](leveltravel/docs/papi/v3/controllers/orders/3.12.yml)
  - [3.13.yml](leveltravel/docs/papi/v3/controllers/orders/3.13.yml)
  - [3.15.yml](leveltravel/docs/papi/v3/controllers/orders/3.15.yml)

## Как заполнять документацию

### 1. Новый контроллер

Создай папку `docs/papi/v3/controllers/<controller>/` и положи в неё `3.0.yml`.

### 2. Контракт изменился в минорной версии

Создай новый файл exact version, например `3.12.yml`.

В файл нужно класть только те операции, которые реально поменялись в этой версии.
Каждая изменённая операция описывается полностью, а не патчем.

То есть правильно так:

```yaml
controller: orders
operations:
  payment_methods:
    http_method: get
    path: /orders/payment_methods
    description: Список способов оплаты
    operation:
      summary: Способы оплаты
      responses:
        '200':
          description: OK
```

А так делать не надо:

- не писать частичный patch только для `description`;
- не использовать ключи вида `version >= 3.12`;
- не создавать файлы `3.12+` или `3.0-3.11.yml`.

### 3. Операция исчезла в новой версии

Используй `removed_operations`:

```yaml
removed_operations:
  - legacy_method
```

## Формат файла

Поддерживаемые корневые ключи:

- `controller`
- `description`
- `path_prefix`
- `operations`
- `removed_operations`

Поддерживаемые ключи операции:

- `http_method`
- `path`
- `path_alias`
- `description`
- `operation`

Минимальный пример базового файла:

```yaml
controller: partner
description: Партнерские выгрузки
path_prefix: /partner/
operations:
  settings:
    http_method: get
    path: /partner/settings
    description: Настройки партнера
    operation:
      summary: Настройки партнера
      responses:
        '200':
          description: OK
```

## Как проверять результат

Собрать документацию одного контроллера:

```bash
ruby script/api_docs.rb compile-controller orders 3.12
```

Собрать полный OpenAPI для партнёра:

```bash
ruby script/api_docs.rb compile-openapi PARTNER_ID 3.12
```

Провалидировать формат файлов:

```bash
ruby script/api_docs.rb validate-sequential
```

Посмотреть, у каких контроллеров есть дополнительные минорные версии:

```bash
ruby script/api_docs.rb report-sequential
```

## Практическое правило

Если разработчику непонятно, что попадёт в итоговую документацию для `v3.x`, это значит, что файл версии написан плохо.

Хороший файл версии:

- содержит только реально изменившиеся операции;
- не требует мысленно применять диапазоны и условия;
- читается сверху вниз в хронологическом порядке.
