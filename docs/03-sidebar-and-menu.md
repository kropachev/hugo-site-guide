# Сайдбар, меню и соцсети

В этом разделе - настройка боковой панели темы Stack: аватар, описание, меню и ссылки на соцсети.

По умолчанию Hugo создает один конфигурационный файл в корне проекта (`hugo.toml`). Для удобства мы разделим настройки на несколько файлов и перенесем их в папку `config/_default/`.

## Основные настройки - config.toml

Создаем папку `config/_default/` и в ней файл `config.toml`. Переносим сюда из корневого `hugo.toml` основные настройки.

```toml
baseURL = "https://PROFILE.github.io/REPOSITORY/"
title = "Мой новый сайт"
languageCode = "ru"
defaultContentLanguage = "ru"

[module]
  [[module.imports]]
    path = "github.com/CaiJimmy/hugo-theme-stack/v4"
```

После переноса корневой `hugo.toml` можно удалить.

## Настройка сайдбара и params - params.toml

Создаем файл `config/_default/params.toml`. В нем будет содержимое `[params]` - настройки сайдбара и параметры темы. Имя файла задает корневой ключ, поэтому обертку `[params]` в самом файле не пишем.

```toml
mainSections = ["posts", "projects"]

[sidebar]
  emoji = ""
  subtitle = "ИТ-менеджер, разработчик, филантроп"
  avatar = "https://github.com/PROFILE.png?size=150"

[comments]
  enabled = false
```

`mainSections` - список секций, которые тема Stack использует для главной страницы и поискового индекса. Несмотря на то что это стандартный параметр Hugo, тема читает его через `.Site.Params.mainSections`, поэтому он должен быть в `params.toml`, а не в `config.toml`.

`[comments]` - Stack v4 рендерит блок комментариев по умолчанию, даже без настроенного провайдера. Без этой строки внизу каждой страницы будет пустой блок. Когда понадобятся комментарии, поменяем на `enabled = true` и добавим провайдера (`disqus`, `utterances` и др.).

## Главное меню

Теперь новые настройки.  
Создаем `config/_default/menus.toml`:

```toml
[[main]]
    identifier = "home"
    name = "Главная"
    url = "/"
    weight = -100
    [main.params]
        icon = "home"

[[main]]
    identifier = "about"
    name = "Обо мне"
    url = "/page/about/"
    weight = -90
    [main.params]
        icon = "user"

[[main]]
    identifier = "archives"
    name = "Архив"
    url = "/page/archives/"
    weight = -70
    [main.params]
        icon = "archives"

[[main]]
    identifier = "search"
    name = "Поиск"
    url = "/page/search/"
    weight = -60
    [main.params]
        icon = "search"
```

Мы добавили ссылки разделы Архив и Поиск. Страницы для этих разделов нам нужно создать.

- **Архив:** создаем `content/page/archives/index.md`:
  ```yaml
  ---
  title: "Архив"
  layout: "archives"
  ---
  ```
- **Поиск:** создаем `content/page/search/index.md` с содержимым:
  ```yaml
  ---
  title: "Поиск"
  layout: "search"
  outputs: [html, json]
  ---
  ```

Если пока не нужны архив и поиск, можно удалить соответствующие блоки `[[main]]` из `menus.toml`.

### Что означают параметры пунктов меню

| Параметр | Описание |
|----------|----------|
| `identifier` | Уникальный ID пункта |
| `name` | Отображаемый текст |
| `url` | Ссылка |
| `weight` | Порядок (меньше = выше) |
| `params.icon` | Имя SVG-иконки из Tabler Icons |
| `params.newTab` | `true` - открывать в новой вкладке |

### Про работу поиска

Следует обратить внимание, что индекс для поиска строится только для страниц указанных в `mainSections` (в нашем примере - `["posts", "projects"]` в `config.toml`).

## Ссылки на социальные сети

Ссылки на профили социальных сетей отображаются иконками в сайдбаре. В самой теме из коробки доступен ограниченный выбор иконок. Свои иконки можно добавлять в папку `assets/icons/`.  
Сами иконки можно брать с сайта [Tabler Icons](https://tabler.io/icons), они идеально подходят для нашей темы.
Для примера скачиваем иконки `mail`, `brand-telegram`, `brand-linkedin`, ниже в примере файла будем на них ссылаться. Добавляем список ссылок на соцсети в `menus.toml`.  
В `url` указывайте свои ссылки.

```toml
[[social]]
    identifier = "github"
    name = "GitHub"
    url = "https://github.com/username"
    [social.params]
        icon = "brand-github"

[[social]]
    identifier = "telegram"
    name = "Telegram"
    url = "https://t.me/username"
    [social.params]
        icon = "brand-telegram"

[[social]]
    identifier = "linkedin"
    name = "LinkedIn"
    url = "https://linkedin.com/in/username"
    [social.params]
        icon = "brand-linkedin"

[[social]]
    identifier = "email"
    name = "Email"
    url = "mailto:user@example.com"
    [social.params]
        icon = "mail"
```
