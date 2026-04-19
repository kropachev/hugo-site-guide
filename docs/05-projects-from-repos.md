# Проекты из репозиториев (README как статьи)

В этом разделе добавим пункт меню "Проекты": список проектов будет выводиться как карточки статей, а содержимое самих статей - это README из отдельных репозиториев.

## Общая идея

1. В Hugo-заготовке заводим **секцию `projects`**:
   - `content/projects/<slug>/index.md` - статья про проект.
2. В меню оставляем (или добавляем) пункт "Проекты", ведущий на страницу `/projects/` (список секции).
3. Тема Stack автоматически выводит список страниц секции на ее главной странице.
4. Отдельный скрипт (локально или в CI):
   - читает список репозиториев из конфигурации,
   - забирает их `README.md` с GitHub,
   - генерирует/обновляет `content/projects/<slug>/index.md`,
   - удаляет папки проектов, убранных из списка.

Hugo видит эти файлы как обычные статьи и строит по ним сайт.

---

## Секция `projects`

Для начала создаем папку `content/projects` и файл `content/projects/_index.md`:
```markdown
---
title: "Проекты"
---
```

## Пункт меню "Проекты"

Добавляем информацию о разделе в `config/_default/menus.toml`:

```toml
[[main]]
    identifier = "projects"
    name = "Проекты"
    url = "/projects/"
    weight = -80
    [main.params]
        icon = "code"
```

> Не забудьте скачать иконку `code` с сайта  [Tabler Icons](https://tabler.io/icons) положить в папку `assets/icons/`.

Пункт меню ведет на страницу секции `/projects/`, где тема автоматически покажет список всех проектов.

## Синхронизируем информацию о проектах с содержимым сайта

Сейчас мы можем руками создавать подпапки с проектамми и файлы index.md для каждого проекта. Но я предлагаю синхронизировать эту информацию с помощью скрипта.

При запуске скрипта (см. ниже) для каждого проекта из `data/projects.yaml` будут выполнены следующие действия:

- создается папка `content/projects/<slug>/` (если ее еще нет),
- создается или перезаписывается файл `content/projects/<slug>/index.md`,
- заполняется т front matter и вставляется содержимое из `README.md` репозитория,
- удаляются папки проектов, которых больше нет в `data/projects.yaml`.



### Настраиваем список проектов

Мы хотим чтобы **максимум данных собиралось из самого репозитория**, а в конфигурации указывались только ссылки.

Создадим файл с описанием проектов в папке `data/`, например `data/projects.yaml`:

```yaml
- repo: https://github.com/username/my-awesome-tool
  slug: mat                         # опционально, по умолчанию имя репозитория
  branch: main                      # опционально, по умолчанию main
  title: my-awesome-tool            # опционально, по умолчанию имя репозитория
  description: Краткое описание     # опционально, иначе будет взято из GitHub или останется пустым
  fetch_image: true                 # опционально, по умолчанию true
  strip_cover: true                 # опционально, по умолчанию false

- repo: https://github.com/username/go-telegram-bot
```

Как заполняются поля:

- **`repo`** - ссылка на репозиторий GitHub (единственное обязательное поле).
- **`slug`** - автоматически получается из имени репозитория (часть после последнего `/`), используется как:
  - имя папки `content/projects/<slug>/`,
  - значение `slug` в front matter.
- **`branch`** - ветка, из которой брать README (если не указана, по умолчанию `"main"`).
- **`title`**:
  - если указано в `projects.yaml` - берем его;
  - иначе используем имя репозитория (то же, что и `slug`).
- **`description`**:
  - если указано явно - берем из `projects.yaml`;
  - иначе берем `description` репозитория из GitHub API (поле About в карточке репозитория).
- **`fetch_image`** - загружать ли обложку (по умолчанию `true`). Если включено, скрипт:
  1. пробует взять **social preview** репозитория (кастомное изображение из настроек GitHub);
  2. если его нет - берет **первое изображение** из README (бейджи `shields.io` пропускаются);
  3. сохраняет файл как `cover.*` в папку проекта и прописывает `image` в front matter.
  Установите `false`, чтобы отключить для конкретного проекта.
- **`strip_cover`** - удалять ли первое изображение из тела README (по умолчанию `false`). Используйте совместно с `fetch_image: true`, если обложка берется из README и отображается в нем же - это предотвращает дублирование картинки на странице проекта.

### Скрипт синхронизации README

Ниже пример скрипта на Go для сбора информации о проектах:

- читает `data/projects.yaml`,
- для каждого проекта скачивает `README.md` с GitHub,
- загружает обложку (social preview репозитория, если нет - первое изображение из README),
- создает/обновляет `content/projects/<slug>/index.md` с front matter и телом из README,
- удаляет папки проектов, убранных из списка,

> Скрипт создает файлы `content/projects/*/index.md` в рабочей директории. **В CI они существуют только во время сборки** и не попадают в репозиторий. Для локальной разработки добавим исключение в `.gitignore`:
>
> ```text
> # Сгенерированные страницы проектов
> content/projects/*/
> ```
>
> Паттерн `content/projects/*/` исключает подпапки проектов, но сохраняет `content/projects/_index.md` (заголовок секции, создан вручную).

Создаем файлы в папке `scripts/fetch-projects/`:

- **main.go** - основной скрипт
- **sync.go** - получение README и description
- **cover.go** - загрузка обложки: social preview и первое изображение README
- **helpers.go** - парсинг URL, санитизация slug, работа со строками

<details><summary>main.go</summary>

Содержимое файла `scripts/fetch-projects/main.go`:

```go
package main

import (
    "fmt"
    "net/http"
    "os"
    "path/filepath"
    "sync"
    "time"

    "gopkg.in/yaml.v3"
)

var httpClient = &http.Client{Timeout: 30 * time.Second}

type Project struct {
    Repo        string `yaml:"repo"`
    Slug        string `yaml:"slug"`
    Branch      string `yaml:"branch"`
    Title       string `yaml:"title"`
    Description string `yaml:"description"`
    FetchImage  *bool  `yaml:"fetch_image"`
    StripCover  bool   `yaml:"strip_cover"`
}

func main() {
    projects, err := loadProjects("data/projects.yaml")
    if err != nil {
        panic(err)
    }

    for i := range projects {
        if projects[i].Slug == "" {
            _, name, err := parseRepoURL(projects[i].Repo)
            if err == nil {
                projects[i].Slug = sanitizeSlug(name)
            }
        }
        if projects[i].Branch == "" {
            projects[i].Branch = "main"
        }
    }

    if err := cleanupProjects(projects); err != nil {
        fmt.Fprintf(os.Stderr, "cleanup: %v\n", err)
    }

    const maxConcurrency = 5
    sem := make(chan struct{}, maxConcurrency)
    var wg sync.WaitGroup

    for _, p := range projects {
        wg.Add(1)
        go func(p Project) {
            defer wg.Done()
            sem <- struct{}{}
            defer func() { <-sem }()

            if err := syncProject(p); err != nil {
                fmt.Fprintf(os.Stderr, "project %s: %v\n", p.Slug, err)
            }
        }(p)
    }
    wg.Wait()
}

func loadProjects(path string) ([]Project, error) {
    f, err := os.Open(path)
    if err != nil {
        return nil, err
    }
    defer f.Close()

    var projects []Project
    dec := yaml.NewDecoder(f)
    if err := dec.Decode(&projects); err != nil {
        return nil, err
    }
    return projects, nil
}

func cleanupProjects(projects []Project) error {
    dir := filepath.Join("content", "projects")
    entries, err := os.ReadDir(dir)
    if err != nil {
        if os.IsNotExist(err) {
            return nil
        }
        return err
    }

    expected := make(map[string]bool)
    for _, p := range projects {
        expected[p.Slug] = true
    }

    for _, e := range entries {
        if !e.IsDir() {
            continue
        }
        if !expected[e.Name()] {
            path := filepath.Join(dir, e.Name())
            fmt.Println("Removing", path)
            if err := os.RemoveAll(path); err != nil {
                return err
            }
        }
    }
    return nil
}
```
</details>

<details><summary>sync.go</summary>

Содержимое файла `scripts/fetch-projects/sync.go`:

```go
package main

import (
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "os"
    "path/filepath"
    "strings"
)

type repoInfo struct {
    Description string
}

func syncProject(p Project) error {
    readme, err := fetchReadme(p)
    if err != nil {
        return err
    }

    if p.Title == "" {
        p.Title = p.Slug
    }

    body := stripFirstH1(readme)
    body = rewriteImageURLs(body, p)

    info, infoErr := fetchRepoInfo(p.Repo)

    if p.Description == "" && infoErr == nil {
        p.Description = info.Description
    }

    dir := filepath.Join("content", "projects", p.Slug)
    if err := os.MkdirAll(dir, 0o755); err != nil {
        return err
    }

    var coverFile string
    if p.FetchImage == nil || *p.FetchImage {
        if data, ext, err := fetchCoverImage(p, readme); err == nil {
            entries, _ := os.ReadDir(dir)
            for _, e := range entries {
                if strings.HasPrefix(e.Name(), "cover.") {
                    os.Remove(filepath.Join(dir, e.Name()))
                }
            }
            coverFile = "cover" + ext
            if err := os.WriteFile(filepath.Join(dir, coverFile), data, 0o644); err != nil {
                fmt.Fprintf(os.Stderr, "  cover write: %v\n", err)
                coverFile = ""
            }
        }
    }

    if p.StripCover && coverFile != "" {
        body = stripFirstImage(body)
    }

    path := filepath.Join(dir, "index.md")
    f, err := os.Create(path)
    if err != nil {
        return err
    }
    defer f.Close()

    var imageField string
    if coverFile != "" {
        imageField = fmt.Sprintf("\nimage: \"%s\"", coverFile)
    }

    frontMatter := fmt.Sprintf(`---
title: "%s"
slug: "%s"
description: "%s"%s
tags:
  - projects
repo: "%s"
---

`, escapeYAML(p.Title), escapeYAML(p.Slug), escapeYAML(p.Description), imageField, escapeYAML(p.Repo))

    if _, err := io.WriteString(f, frontMatter); err != nil {
        return err
    }
    if _, err := io.WriteString(f, body); err != nil {
        return err
    }

    fmt.Println("Updated", path)
    return nil
}

func fetchReadme(p Project) (string, error) {
    owner, name, err := parseRepoURL(p.Repo)
    if err != nil {
        return "", err
    }

    rawURL := fmt.Sprintf("https://raw.githubusercontent.com/%s/%s/%s/README.md", owner, name, p.Branch)

    resp, err := httpClient.Get(rawURL)
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return "", fmt.Errorf("fetch %s: %s", rawURL, resp.Status)
    }

    b, err := io.ReadAll(resp.Body)
    if err != nil {
        return "", err
    }
    return string(b), nil
}

func fetchRepoInfo(repoURL string) (repoInfo, error) {
    owner, name, err := parseRepoURL(repoURL)
    if err != nil {
        return repoInfo{}, err
    }

    apiURL := fmt.Sprintf("https://api.github.com/repos/%s/%s", owner, name)
    resp, err := httpClient.Get(apiURL)
    if err != nil {
        return repoInfo{}, err
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return repoInfo{}, fmt.Errorf("github api: %s", resp.Status)
    }

    b, err := io.ReadAll(resp.Body)
    if err != nil {
        return repoInfo{}, err
    }

    var r struct {
        Description string `json:"description"`
    }
    if err := json.Unmarshal(b, &r); err != nil {
        return repoInfo{}, err
    }
    return repoInfo{Description: r.Description}, nil
}
```

</details>

<details><summary>cover.go</summary>

Содержимое файла `scripts/fetch-projects/cover.go`:

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "regexp"
    "strings"
)

var ogImageRe = regexp.MustCompile(`<meta\s+property="og:image"\s+content="([^"]+)"`)
var mdImageRe = regexp.MustCompile(`!\[[^\]]*\]\(([^)\s]+)`)

func fetchCoverImage(p Project, readmeContent string) ([]byte, string, error) {
    if data, ext, err := fetchSocialPreview(p); err == nil {
        return data, ext, nil
    }

    imageURL := firstReadmeImageURL(p, readmeContent)
    if imageURL == "" {
        return nil, "", fmt.Errorf("no cover image found")
    }
    return downloadImage(imageURL)
}

func fetchSocialPreview(p Project) ([]byte, string, error) {
    owner, name, err := parseRepoURL(p.Repo)
    if err != nil {
        return nil, "", err
    }

    pageURL := fmt.Sprintf("https://github.com/%s/%s", owner, name)
    req, err := http.NewRequest(http.MethodGet, pageURL, nil)
    if err != nil {
        return nil, "", err
    }
    req.Header.Set("User-Agent", "fetch-projects/1.0")

    resp, err := httpClient.Do(req)
    if err != nil {
        return nil, "", err
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return nil, "", fmt.Errorf("fetch %s: %s", pageURL, resp.Status)
    }

    // Meta-теги находятся в <head>, достаточно первых 50 КБ
    head, err := io.ReadAll(io.LimitReader(resp.Body, 50*1024))
    if err != nil {
        return nil, "", err
    }

    m := ogImageRe.FindSubmatch(head)
    if len(m) < 2 {
        return nil, "", fmt.Errorf("og:image not found")
    }

    imageURL := string(m[1])

    // Автосгенерированные превью - не настоящие обложки
    if strings.Contains(imageURL, "opengraph.githubassets.com") {
        return nil, "", fmt.Errorf("no custom social preview")
    }

    return downloadImage(imageURL)
}

func firstReadmeImageURL(p Project, content string) string {
    for _, m := range mdImageRe.FindAllStringSubmatch(content, -1) {
        if len(m) < 2 || m[1] == "" {
            continue
        }
        url := m[1]
        if strings.Contains(url, "shields.io") {
            continue
        }
        if !isAbsoluteURL(url) {
            owner, name, err := parseRepoURL(p.Repo)
            if err != nil {
                continue
            }
            url = strings.TrimPrefix(url, "./")
            url = fmt.Sprintf("https://raw.githubusercontent.com/%s/%s/%s/%s",
                owner, name, p.Branch, url)
        }
        return url
    }
    return ""
}

func downloadImage(imageURL string) ([]byte, string, error) {
    resp, err := httpClient.Get(imageURL)
    if err != nil {
        return nil, "", err
    }
    defer resp.Body.Close()

    if resp.StatusCode != 200 {
        return nil, "", fmt.Errorf("fetch image %s: %s", imageURL, resp.Status)
    }

    data, err := io.ReadAll(io.LimitReader(resp.Body, 10*1024*1024))
    if err != nil {
        return nil, "", err
    }

    ext := extFromContentType(resp.Header.Get("Content-Type"))
    return data, ext, nil
}

func extFromContentType(ct string) string {
    switch {
    case strings.Contains(ct, "image/png"):
        return ".png"
    case strings.Contains(ct, "image/jpeg"):
        return ".jpg"
    case strings.Contains(ct, "image/gif"):
        return ".gif"
    case strings.Contains(ct, "image/webp"):
        return ".webp"
    default:
        return ".png"
    }
}
```

</details>

<details><summary>helpers.go</summary>

Содержимое файла `scripts/fetch-projects/helpers.go`:

```go
package main

import (
    "bufio"
    "fmt"
    "regexp"
    "strings"
)

var slugRe = regexp.MustCompile(`[^a-zA-Z0-9._-]`)

func parseRepoURL(repoURL string) (owner, name string, err error) {
    trimmed := strings.TrimSuffix(repoURL, "/")
    parts := strings.Split(trimmed, "/")
    if len(parts) < 2 {
        return "", "", fmt.Errorf("invalid repo url: %s", repoURL)
    }
    return parts[len(parts)-2], parts[len(parts)-1], nil
}

func escapeYAML(s string) string {
    s = strings.ReplaceAll(s, `\`, `\\`)
    s = strings.ReplaceAll(s, `"`, `\"`)
    return s
}

func sanitizeSlug(s string) string {
    s = strings.ToLower(s)
    s = slugRe.ReplaceAllString(s, "-")
    s = strings.Trim(s, "-")
    if s == "" || s == "." || s == ".." {
        return "_"
    }
    return s
}

func stripFirstH1(content string) string {
    first, rest, found := strings.Cut(content, "\n")
    if strings.HasPrefix(strings.TrimSpace(first), "# ") {
        if found {
            return rest
        }
        return ""
    }
    return content
}

func firstNonEmptyLine(content string) string {
    scanner := bufio.NewScanner(strings.NewReader(content))
    for scanner.Scan() {
        l := strings.TrimSpace(scanner.Text())
        if l == "" || strings.HasPrefix(l, "![") || strings.HasPrefix(l, "<img") {
            continue
        }
        return l
    }
    return ""
}

// rewriteImageURLs заменяет относительные ссылки на изображения в тексте README
// на абсолютные URL вида https://raw.githubusercontent.com/owner/repo/branch/...
// чтобы картинки отображались на Hugo-сайте.
func rewriteImageURLs(content string, p Project) string {
    owner, name, err := parseRepoURL(p.Repo)
    if err != nil {
        return content
    }
    baseRaw := fmt.Sprintf("https://raw.githubusercontent.com/%s/%s/%s",
        owner, name, p.Branch)

    // Markdown: ![alt](url) и ![alt](url "title")
    // Группы: 1 = "![alt](", 2 = url
    mdRe := regexp.MustCompile(`(!\[[^\]]*\]\()([^)\s]+)`)
    content = mdRe.ReplaceAllStringFunc(content, func(m string) string {
        sub := mdRe.FindStringSubmatch(m)
        if len(sub) < 3 || isAbsoluteURL(sub[2]) {
            return m
        }
        rel := strings.TrimPrefix(sub[2], "./")
        return sub[1] + baseRaw + "/" + rel
    })

    // HTML: <img src="url"> и <img src='url'>
    // Группы: 1 = все до url, 2 = кавычка, 3 = url, 4 = закрывающая кавычка
    htmlRe := regexp.MustCompile(`(<img\b[^>]*\bsrc=)(["'])([^"']+)(["'])`)
    content = htmlRe.ReplaceAllStringFunc(content, func(m string) string {
        sub := htmlRe.FindStringSubmatch(m)
        if len(sub) < 5 || isAbsoluteURL(sub[3]) {
            return m
        }
        rel := strings.TrimPrefix(sub[3], "./")
        return sub[1] + sub[2] + baseRaw + "/" + rel + sub[4]
    })

    return content
}

func isAbsoluteURL(s string) bool {
    return strings.HasPrefix(s, "http://") ||
        strings.HasPrefix(s, "https://") ||
        strings.HasPrefix(s, "//") ||
        strings.HasPrefix(s, "#") ||
        strings.HasPrefix(s, "mailto:")
}

func stripFirstImage(content string) string {
    re := regexp.MustCompile(`!\[[^\]]*\]\([^)\s]+\)`)
    done := false
    result := re.ReplaceAllStringFunc(content, func(m string) string {
        if done || strings.Contains(m, "shields.io") {
            return m
        }
        done = true
        return ""
    })
    // убираем строки, ставшие пустыми после удаления изображения
    emptyLineRe := regexp.MustCompile(`\n{3,}`)
    return emptyLineRe.ReplaceAllString(result, "\n\n")
}
```

</details>


> Для работы примера понадобится зависимость `gopkg.in/yaml.v3`. Ее можно добавить командой:
>
> ```bash
> go get gopkg.in/yaml.v3
> ```

Локальный запуск:

```bash
go run ./scripts/fetch-projects/
```

Скрипт создаст файлы в `content/projects/`. Если запущен `hugo server`, он подхватит изменения автоматически.

## Настройка красоты

Локально проверяем результат
```bash
hugo server
```

Переходим в раздел `Проекты` и видим компактный список без обложек - это дефолтный `list.html` темы. Чтобы проекты отображались в виде красивых карточек создаем `layouts/projects/list.html`:

```html
{{ define "main" }}

    <header>
        <h3 class="section-title">
                {{ .Title }}
        </h3>
    </header>

    {{- $paginator := partial "helper/paginator.html" . -}}

    <section class="article-list">
        {{ range $index, $element := $paginator.Pages }}
            {{ partial "article-list/default" . }}
        {{ end }}
    </section>

    {{- partial "pagination.html" . -}}
    {{- partial "footer/footer" . -}}
{{ end }}

{{ define "right-sidebar" }}
    {{ partial "sidebar/right.html" (dict "Context" . "Scope" "homepage") }}
{{ end }}
```

Это копия `home.html` темы с добавлением заголовка раздела. Hugo применит его к секции `projects`, заменив дефолтный компактный список.

### Убираем дублирование обложки на странице проекта

Скрипт указывает в `image:` файл обложки для миниатюры в карточке. Но та же картинка есть и в тексте README - на странице проекта она появится дважды.

Чтобы избежать этого, добавьте `strip_cover: true` в `data/projects.yaml` для нужного проекта. Скрипт найдет первое изображение в теле README и удалит его перед записью - обложка останется только в карточке списка.

## GitHub Actions: автоматическая синхронизация

Мы проверили работу скриптов - самое время автоматизировать сборку.  
В workflow добавляем шаг «Fetch project READMEs» **между шагами «Cache restore» и «Build the site»**. 
```yaml
      - name: Fetch project READMEs
        run: go run ./scripts/fetch-projects/

      - name: Build the site
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/" \
            --cacheDir "${{ runner.temp }}/hugo_cache"
```

Ниже полный файл `.github/workflows/deploy.yml`:
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

      - name: Fetch project READMEs
        run: go run ./scripts/fetch-projects/

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

Таким образом:

- список проектов контролируется в `data/projects.yaml`,
- при каждом деплое README подгружаются автоматически,
- на сайте обновляются страницы проектов и их список в секции.
