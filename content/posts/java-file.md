---
title: "Java File"
date: 2022-05-31T21:37:14+08:00
description: "Java File Processing"
tags: ["java", "file", "stream"]
categories: ["java"]
author: "杨晓峰·geektime"
draft: true
---

我们先来比较 IO 和 NIO 两种写入文件的实现：

+ 使用 IO 实现：

``` java
public void copyFile(File source, File dest) throws IOException {
    try(InputStream in = new FileInputStream(source);
    OutputStream out = new FileOutputStream(dest);) {
        byte[] buffer = new byte[1024];
        int length;
        while((length = is.read(buffer)) >0) {
            os.write(buffer, 0, length);
        }
    }
}
```

+ 利用 NIO 实现：

``` java
public void copyFile(File source, File dest) throws IOException {
    try(FileChannel inChannel = new FileInputStream(source).getChannel());
    FileChannel outChannel = new FileOutputStream(dest).getChannel();) {
        for (long count = inChannel.size; count >0; ) {
            long transferred = inChannel.transferTo(inChannel.position(), count, outChannel);
            count -= transferred;
        }
    }
}
```

不同 copy 方式，底层实现机制有什么区别？
