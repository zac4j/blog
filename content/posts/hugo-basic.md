---
title: "Hugo Basic"
date: 2020-04-04T09:25:04+08:00
description: "Hugo 的简单用法"
tags: ["hugo", "pages"]
categories: ["hugo", "basic"]
draft: false
---

## Add new content

`hugo new posts/article-name.md`

## Article meta info

```md
---
title: "New Article"
date: 2020-04-04T09:25:04+08:00
description: "Hugo 的简单用法"
tags: ["hugo", "pages"]
categories: ["hugo"]
draft: true
---
```

## Start server with drafts enabled

`hugo server -D`

## Customize the theme

Open up `config.toml` in a text editor:

```yaml
baseURL = "https://example.org/"
languageCode = "en-us"
title = "My New Hugo Site"
theme = "ananke"
```

## Build static pages

`hugo -D` output will be in `./public/` directory by default (`-d/--destination` flag to change it, or set `publishdir` in the config file)

## Publish article

+ build web site: `hugo -D`
+ deploy web site: `./deploy.sh 'comments'`

## Cross reference

[deploy]:https://segmentfault.com/a/1190000012975914
[cross-ref]:https://gohugo.io/content-management/cross-references/
[about]:https://digitaldrummerj.me/hugo-add-additional-pages/
