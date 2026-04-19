# Виджеты

В этом разделе - настройка виджетов правой боковой панели. Доступные виджеты:

- `search` - поле поиска по сайту
- `archives` - архив публикаций по годам
- `categories` - список категорий
- `tag-cloud` - облако тегов
- `toc` - оглавление статьи
- `taxonomy` - любая произвольная таксономия

## Настройка виджетов - params.toml

Виджеты задаются в `config/_default/params.toml` в секции `[widgets]`. Два отдельных списка - для главной страницы и для страниц статей:

```toml
[widgets]
    homepage = [
        { type = "search" },
        { type = "archives", params = { limit = 5 } },
        { type = "categories", params = { limit = 10 } },
        { type = "tag-cloud", params = { limit = 10 } },
        # { type = "taxonomy", params = { taxonomy = "series", limit = 5, title = "Серии" } },
    ]
    page = [
        { type = "toc" },
    ]
```

`homepage` - виджеты на главной и страницах секций. `page` - виджеты на странице отдельной статьи. Виджеты отображаются в том порядке, в котором перечислены. `toc` (оглавление) имеет смысл только на страницах статей, поэтому он в `page`. `taxonomy` закомментирован - его подключают при необходимости, если в проекте используются дополнительные таксономии.

Если виджеты не нужны совсем, оставьте пустые списки - иначе правая панель может показывать пустой блок:

```toml
[widgets]
    homepage = []
    page = []
```

## search - Поиск

```toml
{ type = "search" }
```

Поле поиска. При вводе перенаправляет на страницу с результатами.

Требует страницу поиска `content/page/search/index.md` (если ее еще нет - см. раздел [Сайдбар, меню и соцсети](03-sidebar-and-menu.md)):

```yaml
---
title: "Поиск"
layout: "search"
outputs: [html, json]
---
```

Индекс для поиска строится только для секций, указанных в `mainSections`. Тема Stack читает этот параметр из `params.toml` (не из `config.toml`), поэтому он должен быть там:

```toml
mainSections = ["posts", "projects"]
```

## archives - Архив

```toml
{ type = "archives", params = { limit = 5 } }
```

Список лет с количеством публикаций. `limit` - сколько лет показывать (по умолчанию `5`).

Требует страницу архива `content/page/archives/index.md`:

```yaml
---
title: "Архив"
layout: "archives"
---
```

## categories - Категории

```toml
{ type = "categories", params = { limit = 10 } }
```

Список категорий сайта. `limit` - максимальное количество (по умолчанию `10`).

Чтобы категории появились, нужно включить таксономию в `config.toml`:

```toml
[taxonomies]
    category = "categories"
    tag = "tags"
```

И указывать в front matter статей:

```yaml
categories:
  - DevOps
  - Go
```

## tag-cloud - Облако тегов

```toml
{ type = "tag-cloud", params = { limit = 10 } }
```

Облако тегов. Работает аналогично `categories`, только для поля `tags` в front matter.

## toc - Оглавление

```toml
{ type = "toc" }
```

Оглавление текущей статьи - список заголовков `##`, `###` и т.д. Используется только в списке `page`.

Для конкретной статьи оглавление можно отключить через front matter:

```yaml
---
title: "Моя статья"
toc: false
---
```

## taxonomy - Произвольная таксономия

```toml
{ type = "taxonomy", params = { taxonomy = "series", limit = 5, title = "Серии" } }
```

Базовый виджет - отображает любую таксономию. `categories` и `tag-cloud` - его обертки. Используйте напрямую, если добавили собственную таксономию, например `series`.

Сначала регистрируем таксономию в `config.toml`:

```toml
[taxonomies]
    category = "categories"
    tag = "tags"
    series = "series"
```

Доступные параметры:

| Параметр | По умолчанию | Описание |
|----------|-------------|----------|
| `taxonomy` | - | Название таксономии (обязательно) |
| `limit` | `10` | Максимум элементов |
| `title` | Название таксономии | Заголовок виджета |
| `icon` | - | Иконка из [Tabler Icons](https://tabler.io/icons) |
| `showLink` | `true` | Заголовок как ссылка на страницу таксономии |

## Свой виджет

Stack подхватывает кастомные виджеты автоматически. Создаем файл `layouts/partials/widget/<name>.html`:

```html
<section class="widget">
    <h2 class="widget-title section-title">Мой виджет</h2>
    <div class="widget-body">
        {{ .Params.text }}
    </div>
</section>
```

Внутри доступны `.Context` (текущая страница Hugo) и `.Params` (параметры из конфига). Подключаем в `params.toml`:

```toml
{ type = "my-widget", params = { text = "Привет!" } }
```
