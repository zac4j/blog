---
title: "WebSocket 和长轮询"
date: 2020-07-21
description: "Websocket"
tags: ["network", "websocket", "polling"]
categories: ["network", "websocket"]
draft: false
---

客户端的网络请求大部分都建立在请求/响应模式的 HTTP/HTTPS 协议之下，AJAX 技术使页面看起来更加动态。尽管如此，所有的 HTTP 连接仍由客户端控制，需要用户交互或定时轮询（periodic polling）才能从服务端加载新数据。

通过长轮询，客户端打开与服务端的 HTTP 连接，并保持连接直到服务端返回响应，只要服务端有新的数据，它就会发送响应。然而这些技术有共同的问题：它们带有 HTTP 的开销，并不适合低延时应用的需求。

<!--more-->

## WebSocket 简介

WebSocket 规范定义了一个 API：在客户端和服务端之间建立 *socket* 连接。简而言之，客户端和服务端之间建立持久连接，双方都可以随时发送数据。

## 使用 OkHttp 实现 WebSocket

### 添加依赖

``` groovy
// network
implementation("com.squareup.okhttp3:okhttp:4.8.0")
implementation("com.squareup.okhttp3:mockwebserver:4.8.0")
```

### 客户端创建 WebSocket 连接

``` kotlin
/**
 * 连接 WebSocket 服务
 *
 * @param hostname 服务端域名地址
 * @param port WebSocket 服务端口号
 */
private fun connectWebSocket(hostname: String,port: Int) {
  val httpClient = OkHttpClient.Builder()
      .pingInterval(PING_INTERVAL, TimeUnit.SECONDS)
      .build()

  // Android 9 已禁止明文传输网络数据，此处使用 ws，可以参考：https://stackoverflow.com/a/50834600
  val url = "ws://${hostname}:${port}"
  Logger.i(TAG, "connectWebSocket", "connect url -> $url")

  val request = Request.Builder()
      .url(url)
      .build()

  httpClient.newWebSocket(request, object : WebSocketListener() {
    override fun onOpen(
      webSocket: WebSocket,
      response: Response
    ) {
      super.onOpen(webSocket, response)
      Logger.i(TAG, "onOpen", "response -> ${response.message}")
      // While web socket is opened create our interval work
      createIntervalWork(webSocket)
    }

    override fun onMessage(
      webSocket: WebSocket,
      text: String
    ) {
      super.onMessage(webSocket, text)
      Logger.i(TAG, "onMessage", "text -> $text")
      if (text.isNotEmpty()) {
        mHandler.post {
          mAdapter.addMessage("pong -> $text")
        }
      }
    }

    override fun onMessage(
      webSocket: WebSocket,
      bytes: ByteString
    ) {
      super.onMessage(webSocket, bytes)
      Logger.i(TAG, "onMessage", "bytes -> $bytes")
      mHandler.post {
        mAdapter.addMessage(bytes.toString())
      }
    }

    override fun onClosed(
      webSocket: WebSocket,
      code: Int,
      reason: String
    ) {
      super.onClosed(webSocket, code, reason)
      Logger.i(TAG, "onClosed", "reason -> $reason")
    }

    override fun onClosing(
      webSocket: WebSocket,
      code: Int,
      reason: String
    ) {
      super.onClosing(webSocket, code, reason)
      Logger.i(TAG, "onClosing", "reason -> $reason")
    }

    override fun onFailure(
      webSocket: WebSocket,
      t: Throwable,
      response: Response?
    ) {
      super.onFailure(webSocket, t, response)
      Logger.i(TAG, "onFailure", "${t.message}")
    }
  })
}
```

## 客户端 Mock WebSocket 服务

``` kotlin
/**
 * Mock websocket server
 */
fun mockWebServer(): MockWebServer? {
  val mockWebServer: MockWebServer? = MockWebServer()

  mockWebServer?.enqueue(MockResponse().withWebSocketUpgrade(object : WebSocketListener() {
    override fun onOpen(
      webSocket: WebSocket,
      response: Response
    ) {
      super.onOpen(webSocket, response)
      Logger.i("MockWebServer", "onOpen", "response -> ${response.message}")
    }

    override fun onMessage(
      webSocket: WebSocket,
      text: String
    ) {
      super.onMessage(webSocket, text)
      Logger.i("MockWebServer", "onMessage", "text -> $text")
      // send back message to client
      webSocket.send(text)
    }

    override fun onMessage(
      webSocket: WebSocket,
      bytes: ByteString
    ) {
      super.onMessage(webSocket, bytes)
      Logger.i("MockWebServer", "onMessage", "bytes -> $bytes")
      // send back message to client
      webSocket.send(bytes)
    }

    override fun onClosed(
      webSocket: WebSocket,
      code: Int,
      reason: String
    ) {
      super.onClosed(webSocket, code, reason)
      Logger.i("MockWebServer", "onClosed", "reason -> $reason")
    }

    override fun onClosing(
      webSocket: WebSocket,
      code: Int,
      reason: String
    ) {
      super.onClosing(webSocket, code, reason)
      Logger.i("MockWebServer", "onClosing", "reason -> $reason")
    }

    override fun onFailure(
      webSocket: WebSocket,
      t: Throwable,
      response: Response?
    ) {
      super.onFailure(webSocket, t, response)
      Logger.i("MockWebServer", "onFailure", "exception -> ${t.message}")
    }
  }))
  return mockWebServer
}
```

[cleartextƒ]:https:/stackoverflow.com/questions/45940861/android-8-cleartext-http-traffic-not-permitted
[https]:https://square.github.io/okhttp/https/
[rocks]:https://www.html5rocks.com/en/tutorials/websockets/basics/
