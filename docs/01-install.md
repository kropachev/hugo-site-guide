# Первоначальная установка

Нам понадобится три инструмента: **Git**, **Go** и **Hugo Extended**.

## Git

Скорее всего, Git у уже стоит. 

Проверка:
```bash
git --version
```

Если нет - можно скачать с [git-scm.com](https://git-scm.com/downloads).  
Процесс установки: запустить дистрибутив и все время нажимать Далее.

## Go

Тема Stack подключается через Hugo Modules, а они работают поверх Go Modules. Go нужен для локального запуска `hugo server`.

Для установки Go можно скачать дистрибутив с сайта [go.dev/dl](https://go.dev/dl/), но я предлагаю использовать пакетный менеджер:

```bash
# Windows (winget)
winget install GoLang.Go

# macOS
brew install go

# Linux (snap)
sudo snap install go --classic
```

Проверка (возможно потребуется перезапустить терминал):

```bash
go version
```

## Hugo Extended

**Extended** версия нужна для темы Stack.


```bash
# Windows (winget)
winget install Hugo.Hugo.Extended

# macOS
brew install hugo # Homebrew ставит Extended по умолчанию

# Linux (snap)
sudo snap install hugo
```

Проверка (возможно потребуется перезапустить терминал):
```bash
hugo version
```

В выводе должно быть слово **extended**:

```
hugo v0.157.0+extended linux/amd64 ...
```

> Тема Stack требует Hugo **v0.157.0** или новее.

## Текстовый редактор

Подойдет любой редактор с поддержкой Markdown. Мы, естественно, возьмем  **[VS Code](https://code.visualstudio.com/)** - самый популярный и бесплатный
