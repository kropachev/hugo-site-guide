# Быстрый старт

От `README.md` до работающего сайта за 15 минут.

## Выбираем репозиторий для сайта

Для начала нам необходимо выбрать репозиторий, который станет основной для нашего сайта. Это может быть репозиторий рабочего проекта, ИЛИ (!) репозиторий профиля. Если на GitHub создать репозиторий с именем, совпадающим с именем профиля (например, `kropachev/kropachev`), то его `README.md` будет отображаться на странице профиля. Это удобная визитка, и мы можем пойти дальше - сделать из этого репозитория полноценный сайт.

Далее по тексту используем два плейсхолдера:

- `PROFILE` - имя профиля на GitHub
- `REPOSITORY` - имя репозитория

Например, для репозитория профиля это может быть `kropachev` и `kropachev`, а для обычного проекта - `kropachev` и `my-site`.

## Инициализируем Hugo

Переходим в локальную папку с нашим репозиторием. Нам необходимо инициализировать Hugo

```bash
hugo new site . --force
```

Флаг `--force` нужен, потому что директория не пустая (в ней уже есть `README.md`, как минимум). Hugo создаст стандартную структуру папок и файл конфигурации `hugo.toml`.

Сразу создаем `.gitignore`, чтобы артефакты сборки не попадали в репозиторий:

```text
# Артефакты сборки Hugo
public/

# Кэш обработанных ассетов Hugo (минификация, fingerprint, SASS)
resources/

# Статистика Hugo Pipes для очистки CSS - пересоздается при билде
hugo_stats.json
```

Инициализируем Go-модуль (нужен для подключения темы):

```bash
hugo mod init github.com/PROFILE/REPOSITORY
```

## Подключаем тему Stack

Открываем `hugo.toml` в корне проекта (файл был создан автоматически командой `hugo new site`) и добавляем минимальное содержимое:

```toml
baseURL = "https://PROFILE.github.io/REPOSITORY/"
title = "Мой новый сайт"
languageCode = "ru"
defaultContentLanguage = "ru"
mainSections = ["posts", "projects"]

[module]
  [[module.imports]]
    path = "github.com/CaiJimmy/hugo-theme-stack/v4"

[params.sidebar]
    emoji = ""
    subtitle = "ИТ-менеджер, разработчик, филантроп"
    avatar = "https://github.com/PROFILE.png?size=150"

[menu]
[[menu.main]]
    identifier = "about"
    name = "Обо мне"
    url = "/page/about/"
    weight = -90
    [menu.main.params]
        icon = "user"
```

Что здесь указываем:

- `baseURL` - адрес, по которому сайт будет доступен после публикации. Hugo использует его при генерации ссылок, RSS и метаданных. На старте рекомендую указывать адрес GitHub Pages, а переход на собственный домен разберем отдельно позже.
- `title` - заголовок сайта. Он появится в `<title>` страницы, в шапке темы и в превью ссылок.
- `languageCode` - язык сайта по стандарту IETF, например `ru` или `en-us`. Нужен для корректных метаданных и локализации.
- `defaultContentLanguage` - основной язык контента в Hugo. Для одноязычного сайта обычно совпадает с `languageCode`.
- `[module]` и `[[module.imports]]` - подключение темы Stack как Hugo Module. Благодаря этому тему не нужно копировать в проект вручную.
- `path = "github.com/CaiJimmy/hugo-theme-stack/v4"` - адрес темы, которую Hugo скачает при первом запуске.
- `mainSections = ["posts", "projects"]` - указываем секции контента, с которых начинаем. Это стандартный параметр Hugo (не параметр темы), поэтому он задается на верхнем уровне конфига, а не под `[params]`. Сначала постов нет - сайт живет одной страницей "Обо мне".
- `[params]` - раздел с параметрами темы Stack.
- `[params.sidebar]` - настройки боковой панели.
- `emoji` - эмодзи рядом с заголовком сайта. Оставляем пустым.
- `subtitle` - короткое описание, которое будет видно в боковой панели под именем сайта.
- `avatar` - путь к файлу аватара или URL. Локально: `"img/avatar.png"` (файл в `assets/img/avatar.png`).
В качестве альтернативы можно использовать ссылку на аватар профиля GitHub: `"https://github.com/PROFILE.png?size=150"` (подставьте свой логин вместо `PROFILE`).

> Сейчас все настройки в одном файле `hugo.toml`. В следующих разделах мы разделим файлы настроек и перенесем их в папку `config/_default/`.

## Создаем страницу "Обо мне"

У нас уже есть текст о себе в README. Лучше не дублировать: пусть страница "Обо мне" подхватывает контент из README. Альтернатива - вести отдельную страницу со своим текстом.

Страницу "Обо мне" добавляем в меню навигации (боковая панель, иконка-человечек). В Варианте 1 меню прописано в `hugo.toml`, в Варианте 2 - прямо в front matter страницы. Пока это и есть наша первая и главная страница - по сути, контент из README.

### Вариант 1: Контент из README (рекомендую)

Страница "Обо мне" показывает содержимое `README.md` из корня проекта. Обновляем файл README - обновляется и профиль на GitHub, и текст сайте.

**Шаг 1.** Создаем shortcode, который выводит README. Файл `layouts/shortcodes/readme_content.html`:

```html
{{ readFile "README.md" | markdownify }}
```

**Шаг 2.** Создаем страницу только с front matter и вызовом shortcode:

```bash
hugo new content page/about/index.md
```

В `content/page/about/index.md` оставляем только метаданные и подключение README:

```markdown
---
title: "Обо мне"
---

{{< readme_content >}}
```

### Вариант 2: Отдельная страница с собственным текстом

Если "Обо мне" на сайте и README в репозитории должны отличаться, ведем контент вручную. Создаем страницу и вставляем в нее текст:

```bash
hugo new content page/about/index.md
```

Открываем `content/page/about/index.md` заполняем метаданные и текст, который должен выводиться на странице:

```markdown
---
title: "Обо мне"
menu:
    main:
        weight: 1
        params:
            icon: user
---

Привет! Я Имя Фамилия

Работаю работу, изучаю знания  и отдыхаю.

## Чем занимаюсь

- Работой
- Не работой

## Контакты

- [Telegram: @MyName](https://t.me/MyName)
```

## Локальный запуск

Перед отправкой изменений в гитхаб можно посмотреть как будет выглядеть сайт.

```bash
hugo server
```

Hugo скачает тему через Go Modules (первый запуск может занять 20–30 секунд) и запустит локальный сервер:

```
Web Server is available at http://localhost:1313/REPOSITORY/
```

Открываем ссылку в браузере. Мы увидим сайт с боковой панелью и ссылкой "Обо мне" в меню. На главной пока пусто. По умолчанию на главной воводятся последние опубликованные статьи и посты. Сделаем небольшие настройки, чтобы на главной странице сразу отображался текст README.

## Выводим "Обо мне" на главной странице

Чтобы при открытии сайта на главной сразу был тот же контент, что и на странице "Обо мне", создаем файл `content/_index.md`. Далее у нас опять два варианта, в зависимости от способа, который был выбран для вывода раздела "Обо мне".

- **Если был выбран правильный вариант 1 (контент из README):** в `content/_index.md` аналогично подключаем shortcode - главная и "Обо мне" будут брать текст из одного источника:

```markdown
---
title: "Добро пожаловать"
layout: single
---

{{< readme_content >}}
```

* **Для варианта 2 (свой текст в "Обо мне"):** в `content/_index.md` вручную вставляем тот же или краткий текст для главной. 

```markdown
---
title: "Добро пожаловать"
layout: single
---


Привет! Я Имя Фамилия

Далее произвольный текст для главной.
```

Еще раз локально проверяем результат:

```bash
hugo server
```

Базовые настройки завершены.  
В следующих разделах рассмотрим добавление постов и статей.  
А пока займемся публикацией сайта в GitHub.

## Создаем workflow

При коммитах изменений в GitHub будет запускаться автоматическая сборка.

Создаем файл `.github/workflows/deploy.yml`:

```yaml
name: Build and deploy

on:
  push:
    branches:
      - main
      - master
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

defaults:
  run:
    shell: bash

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      DART_SASS_VERSION: 1.97.1
      GO_VERSION: 1.25.5
      HUGO_VERSION: 0.160.1
      NODE_VERSION: 24.12.0
    steps:
      - name: Checkout
        uses: actions/checkout@v5
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: false

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5

      - name: Create directory for user-specific executable files
        run: mkdir -p "${HOME}/.local"

      - name: Install Dart Sass
        run: |
          curl -sLJO "https://github.com/sass/dart-sass/releases/download/${DART_SASS_VERSION}/dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
          tar -C "${HOME}/.local" -xf "dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
          rm "dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
          echo "${HOME}/.local/dart-sass" >> "${GITHUB_PATH}"

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: 'latest'
          extended: true

      - name: Cache restore
        id: cache-restore
        uses: actions/cache/restore@v4
        with:
          path: ${{ runner.temp }}/hugo_cache
          key: hugo-${{ github.run_id }}
          restore-keys: hugo-

      - name: Build the site
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/" \
            --cacheDir "${{ runner.temp }}/hugo_cache"

      - name: Cache save
        uses: actions/cache/save@v4
        with:
          path: ${{ runner.temp }}/hugo_cache
          key: ${{ steps.cache-restore.outputs.cache-primary-key }}

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

> Оригинал шаблона можно взять [поссылке](https://github.com/CaiJimmy/hugo-theme-stack-starter/blob/master/.github/workflows/deploy.yml) из репозитория темы Stack.


## Включаем GitHub Pages

GitHub Pages - это бесплатный хостинг статических сайтов от GitHub.

Переходим в наш проект на GitHub.

1. Settings - Pages
2. Source: **GitHub Actions**

Этого достаточно для первого запуска.

## Публикуем

При отправке изменений в GitHub автоматически запустится сборка.
Коммитим изменения из интерфейса vscode или выполняем команду в терминале:

```bash
git add .
git commit -m "Initial site setup"
git push
```

Переходим на вкладку **Actions** нашего репозитория - там будет запущенная сборка. Через пару минут сайт будет доступен по адресу `https://PROFILE.github.io/REPOSITORY/`.

