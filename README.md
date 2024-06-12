## what is for?
this page is my knowledge base, i want use it for:
- my personal things, progs and hotkeys that helps me in work and personal life
- some sites for time and finance managment
- knowledge about system anal.

## general plan
1. [x] remove border-radius for blocks (now it's TOO rounded) 
2. [x] create menu
2. [ ] fill base
3. [x] create page in github pages
4. [x] add portfolio
4. [x] add info abt me and resume
5. [x] add my arts to the new block (mb no?)
6. [ ] add sql and groovy examples

## content plan
1. [ ] add info + difgram with artefacts from [link](https://gb.ru/blog/rabota-s-proektom-ehtapy-osobennosti/)

## how to start in local?
before start u shoud install:
- [Go](https://go.dev/doc/install)
- [Hugo (extended version)](https://gohugo.io/installation/)

for start enter this command in cmd

```
hugo server --buildDrafts --disableFastRender
```

### useful links

- [docs for this template (named **hextra**)](https://imfing.github.io/hextra/)
- [how to add shortcodes like steps, tabs, icons etc.](https://imfing.github.io/hextra/docs/guide/shortcodes/)
- [example with changing assets (it can be useful for changing border-radius or for colors)](https://github.com/CleverCloud/documentation/blob/main/assets/css/custom.css)
- [diagrams via mermaid](https://imfing.github.io/hextra/docs/guide/diagrams/)
- [custom CSS](https://imfing.github.io/hextra/docs/advanced/customization/)
- [icons codes with preview](https://v1.heroicons.com) and [list of all icons in hextra](https://github.com/imfing/hextra/blob/main/data/icons.yaml)
- [the best example hextra site](https://github.com/CleverCloud/documentation/tree/main)


## Как сделать такой же сайт

Перевод со страницы [Hexlet docs](https://imfing.github.io/hextra/docs/getting-started/)

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
