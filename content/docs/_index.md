---
title: Как сделать такой же сайт
---

Перевод со страницы [Hexlet docs](https://imfing.github.io/hextra/docs/getting-started/)

## Как сделать такой же сайт

### Перед началом нужно установить

- [Git](https://git-scm.com)
- [Go](https://go.dev/doc/install)
- [Hugo (extended version)](https://gohugo.io/installation/)

### Шаги

{{% steps %}}

### Инициализация сайта Hugo

```
hugo new site my-site --format=yaml
```

### Настройка темы Hextra через модуль

```
# инициализация модуля hugo
cd my-site
hugo mod init github.com/username/my-site

# добавление темы Hextra
hugo mod get github.com/imfing/hextra
```

Настройте hugo.yaml для использования темы Hextra:

```
module:
  imports:
    - path: github.com/imfing/hextra
```

### Добавьте страницы с контентом


```
hugo new content/_index.md
hugo new content/docs/_index.md
```

### Запустите локально

```
hugo server --buildDrafts --disableFastRender
```

Превью сайта доступен по ссылке `http://localhost:1313/`

{{% /steps %}}

Подробнее см. на странице [Hexlet docs](https://imfing.github.io/hextra/docs/getting-started/)