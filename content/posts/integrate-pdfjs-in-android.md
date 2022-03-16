---
title: "Integrate pdf.js in Android"
date: 2021-03-29T20:03:37+08:00
description: "Introduce how to integrate pdf.js in Android project"
tags: ["pdf.js", "android pdf viewer", "pdf preview"]
categories: ["pdf", "webview"]
draft: true
---

## Quick start

### Preparetion

+ Add pdf.js dependencies to project assets folder:
  + pdf.js
  + pdf.work.js
  + pdf_viewer.js
  + pdf_viewer.css
  + cmaps(optional)

+ Also add [mobile-viewer][mv] files to assets folder to customize pdf viewer:
  + images
  + viewer.html
  + viewer.css
  + viewer.js

### Customize pdf viewer

In viewer.html file, update dependencies path in head element:

``` html
<link rel="stylesheet" href="pdf_viewer.css">

<script src="pdf.js"></script>
<script src="pdf_viewer.js"></script>
```

In viewer.js file, update CMAPS path and worker src:

``` js
...
const CMAP_URL = "../../node_modules/pdfjs-dist/cmaps/";
...
pdfjsLib.GlobalWorkerOptions.workerSrc = "./pdf.worker.js";
```

### What it can do

+ load a remote pdf url
+ load local pdf file

### How to use it

``` kotlin
val pdfUrl = "https://sample.com/privacy_policy.pdf"
webView.loadUrl("file:///android_asset/pdfviewer/index.html?pdfUrl=$pdfUrl")
```

### What's CMAPs

### How to make hyperlink text clickable

## Problems occur

[mv]:https://github.com/mozilla/pdf.js/tree/master/examples/mobile-viewer
