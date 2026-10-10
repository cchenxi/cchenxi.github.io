---
title: "深入理解 Kafka 生产者：缓冲机制、双线程模型与调优实践"
date: 2026-10-11T00:24:00+08:00
draft: false
tags:
- Kafka
- 消息队列
- Java
- 性能优化
categories:
- 消息中间件
aliases:
- /posts/kafka生产者原理-从第一性原理理解消息生产机制/
---

> 消息在 `send()` 之后去了哪里？为什么缓冲区会满、线程会阻塞？本文从双线程模型出发，逐层拆解 Kafka Producer 的缓冲、批量与背压机制。

---

## 一、为什么生产者需要缓冲区

Kafka 生产者并不会在代码调用 `send()` 时立刻把消息发到网络，而是先将消息放入一块内存缓冲区，攒够一批再发。这是批量（batching）的核心思想。

为什么要这样做？核心原因是网络 I/O 的成本。如果每条消息都单独发一次网络请求，开销巨大；如果把多条消息攒成一批，用一次请求发送，网络效率大幅提升。

类比理解：去邮局寄 1000 封信，每写好一封就单独发一辆车（1000 次发车），远不如把信都放进箱子，等装满了一车送出（1 次发车）。Kafka 生产者选择的是后者。

但批量设计引入了一个关键问题：缓冲区大小有限。如果放入速度持续快于取出速度，缓冲区迟早会被填满。填满之后，新的 `send()` 调用会阻塞，直到缓冲区有空间——或者超时报错。理解这个问题，需要深入生产者的内部架构。

---

## 二、双线程模型

Kafka 生产者内部有两条并行的工作线：调用线程和 Sender 线程。

下图展示了完整的双线程架构。上半部分是调用线程的流水线，下半部分是 Sender 后台线程的数据通路，中间的 RecordAccumulator 是两者共享的缓冲桥梁：

<div align="center">

![svg](data:image/svg+xml;base64,PHN2ZyB2aWV3Qm94PSIwIDAgNjgwIDMzMCIgd2lkdGg9IjEwMCUiIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyIgcm9sZT0iaW1nIiBoZWlnaHQ9IjMzMCI+CiAgPHRpdGxlPkthZmthIHByb2R1Y2VyOiB0d28tdGhyZWFkIGFyY2hpdGVjdHVyZTwvdGl0bGU+CiAgPGRlc2M+VGhlIGNhbGxpbmcgdGhyZWFkIHNlcmlhbGl6ZXMgYW5kIHBhcnRpdGlvbnMgbWVzc2FnZXMgaW50byBhIHNoYXJlZCBidWZmZXI7IGEgc2luZ2xlIGJhY2tncm91bmQgc2VuZGVyIHRocmVhZCBkcmFpbnMgdGhlIGJ1ZmZlciBhbmQgc2VuZHMgYmF0Y2hlcyB0byBLYWZrYSBicm9rZXJzIG92ZXIgdGhlIG5ldHdvcmsuPC9kZXNjPgogIDxkZWZzPgogICAgPG1hcmtlciBpZD0iYXJyb3ciIHZpZXdCb3g9IjAgMCAxMCAxMCIgcmVmWD0iOCIgcmVmWT0iNSIKICAgICAgbWFya2VyV2lkdGg9IjYiIG1hcmtlckhlaWdodD0iNiIgb3JpZW50PSJhdXRvLXN0YXJ0LXJldmVyc2UiPgogICAgICA8cGF0aCBkPSJNMiAxTDggNUwyIDkiIGZpbGw9Im5vbmUiIHN0cm9rZT0iIzVGNUU1QSIKICAgICAgICBzdHJva2Utd2lkdGg9IjEuNSIgc3Ryb2tlLWxpbmVjYXA9InJvdW5kIiBzdHJva2UtbGluZWpvaW49InJvdW5kIi8+CiAgICA8L21hcmtlcj4KICA8L2RlZnM+CiAgPHRleHQgeD0iNTAiIHk9IjM1IiBzdHlsZT0iZm9udC1zaXplOjExcHg7ZmlsbDojODg4NzgwO2ZvbnQtZmFtaWx5OnNhbnMtc2VyaWYiPmNhbGxpbmcgdGhyZWFkICh5b3VyIGFwcGxpY2F0aW9uIGNvZGUpPC90ZXh0PgogIDxyZWN0IHg9IjUwIiB5PSI1MCIgd2lkdGg9IjEwMCIgaGVpZ2h0PSI1MCIgcng9IjgiIGZpbGw9IiNFNkYxRkIiIHN0cm9rZT0iIzE4NUZBNSIgc3Ryb2tlLXdpZHRoPSIwLjUiLz4KICA8dGV4dCB4PSIxMDAiIHk9IjcyIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBkb21pbmFudC1iYXNlbGluZT0iY2VudHJhbCIgc3R5bGU9ImZvbnQtc2l6ZToxM3B4O2ZvbnQtd2VpZ2h0OjUwMDtmaWxsOiMwQzQ0N0M7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+QXBwIHRocmVhZHM8L3RleHQ+CiAgPHRleHQgeD0iMTAwIiB5PSI4OCIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZG9taW5hbnQtYmFzZWxpbmU9ImNlbnRyYWwiIHN0eWxlPSJmb250LXNpemU6MTFweDtmaWxsOiMxODVGQTU7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+c2VuZChyZWNvcmQpPC90ZXh0PgogIDxsaW5lIHgxPSIxNTUiIHkxPSI3NSIgeDI9IjE3NSIgeTI9Ijc1IiBzdHJva2U9IiM1RjVFNUEiIHN0cm9rZS13aWR0aD0iMS41IiBtYXJrZXItZW5kPSJ1cmwoI2Fycm93KSIvPgogIDxyZWN0IHg9IjE4MCIgeT0iNTAiIHdpZHRoPSI5MCIgaGVpZ2h0PSI1MCIgcng9IjgiIGZpbGw9IiNFNkYxRkIiIHN0cm9rZT0iIzE4NUZBNSIgc3Ryb2tlLXdpZHRoPSIwLjUiLz4KICA8dGV4dCB4PSIyMjUiIHk9Ijc1IiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBkb21pbmFudC1iYXNlbGluZT0iY2VudHJhbCIgc3R5bGU9ImZvbnQtc2l6ZToxM3B4O2ZvbnQtd2VpZ2h0OjUwMDtmaWxsOiMwQzQ0N0M7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+SW50ZXJjZXB0b3JzPC90ZXh0PgogIDxsaW5lIHgxPSIyNzUiIHkxPSI3NSIgeDI9IjI5NSIgeTI9Ijc1IiBzdHJva2U9IiM1RjVFNUEiIHN0cm9rZS13aWR0aD0iMS41IiBtYXJrZXItZW5kPSJ1cmwoI2Fycm93KSIvPgogIDxyZWN0IHg9IjMwMCIgeT0iNTAiIHdpZHRoPSI4NSIgaGVpZ2h0PSI1MCIgcng9IjgiIGZpbGw9IiNFNkYxRkIiIHN0cm9rZT0iIzE4NUZBNSIgc3Ryb2tlLXdpZHRoPSIwLjUiLz4KICA8dGV4dCB4PSIzNDIiIHk9Ijc1IiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBkb21pbmFudC1iYXNlbGluZT0iY2VudHJhbCIgc3R5bGU9ImZvbnQtc2l6ZToxM3B4O2ZvbnQtd2VpZ2h0OjUwMDtmaWxsOiMwQzQ0N0M7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+U2VyaWFsaXplcjwvdGV4dD4KICA8bGluZSB4MT0iMzkwIiB5MT0iNzUiIHgyPSI0MTAiIHkyPSI3NSIgc3Ryb2tlPSIjNUY1RTVBIiBzdHJva2Utd2lkdGg9IjEuNSIgbWFya2VyLWVuZD0idXJsKCNhcnJvdykiLz4KICA8cmVjdCB4PSI0MTUiIHk9IjUwIiB3aWR0aD0iMTAwIiBoZWlnaHQ9IjUwIiByeD0iOCIgZmlsbD0iI0U2RjFGQiIgc3Ryb2tlPSIjMTg1RkE1IiBzdHJva2Utd2lkdGg9IjAuNSIvPgogIDx0ZXh0IHg9IjQ2NSIgeT0iNzUiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGRvbWluYW50LWJhc2VsaW5lPSJjZW50cmFsIiBzdHlsZT0iZm9udC1zaXplOjEzcHg7Zm9udC13ZWlnaHQ6NTAwO2ZpbGw6IzBDNDQ3Qztmb250LWZhbWlseTpzYW5zLXNlcmlmIj5QYXJ0aXRpb25lcjwvdGV4dD4KICA8bGluZSB4MT0iMjkwIiB5MT0iMTAwIiB4Mj0iMjkwIiB5Mj0iMTQ1IiBzdHJva2U9IiM1RjVFNUEiIHN0cm9rZS13aWR0aD0iMS41IiBtYXJrZXItZW5kPSJ1cmwoI2Fycm93KSIvPgogIDx0ZXh0IHg9IjMwMCIgeT0iMTI1IiBzdHlsZT0iZm9udC1zaXplOjExcHg7ZmlsbDojODg4NzgwO2ZvbnQtZmFtaWx5OnNhbnMtc2VyaWYiPmFwcGVuZCB0byBidWZmZXI8L3RleHQ+CiAgPHJlY3QgeD0iMTMwIiB5PSIxNDUiIHdpZHRoPSI0MjAiIGhlaWdodD0iNjUiIHJ4PSIxMiIgZmlsbD0iI0ZBRUVEQSIgc3Ryb2tlPSIjODU0RjBCIiBzdHJva2Utd2lkdGg9IjAuNSIvPgogIDx0ZXh0IHg9IjM0MCIgeT0iMTcwIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBkb21pbmFudC1iYXNlbGluZT0iY2VudHJhbCIgc3R5bGU9ImZvbnQtc2l6ZToxNHB4O2ZvbnQtd2VpZ2h0OjUwMDtmaWxsOiM2MzM4MDY7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+UmVjb3JkQWNjdW11bGF0b3IgKHRoZSBidWZmZXIpPC90ZXh0PgogIDx0ZXh0IHg9IjM0MCIgeT0iMTkwIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBkb21pbmFudC1iYXNlbGluZT0iY2VudHJhbCIgc3R5bGU9ImZvbnQtc2l6ZToxMnB4O2ZpbGw6Izg1NEYwQjtmb250LWZhbWlseTpzYW5zLXNlcmlmIj5idWZmZXIubWVtb3J5ID0gMzIgTUIg4oCUIHNoYXJlZCBtZW1vcnkgcG9vbDwvdGV4dD4KICA8bGluZSB4MT0iMzQwIiB5MT0iMjEwIiB4Mj0iMzQwIiB5Mj0iMjU1IiBzdHJva2U9IiM1RjVFNUEiIHN0cm9rZS13aWR0aD0iMS41IiBtYXJrZXItZW5kPSJ1cmwoI2Fycm93KSIvPgogIDx0ZXh0IHg9IjM1MCIgeT0iMjM1IiBzdHlsZT0iZm9udC1zaXplOjExcHg7ZmlsbDojODg4NzgwO2ZvbnQtZmFtaWx5OnNhbnMtc2VyaWYiPnNlbmRlciBkcmFpbnMgcmVhZHkgYmF0Y2hlczwvdGV4dD4KICA8dGV4dCB4PSI0MCIgeT0iMjgwIiBzdHlsZT0iZm9udC1zaXplOjExcHg7ZmlsbDojODg4NzgwO2ZvbnQtZmFtaWx5OnNhbnMtc2VyaWYiPnNlbmRlciB0aHJlYWQgKHNpbmdsZSBiYWNrZ3JvdW5kIHRocmVhZCk8L3RleHQ+CiAgPHJlY3QgeD0iMTIwIiB5PSIyNTUiIHdpZHRoPSIxMDAiIGhlaWdodD0iNTAiIHJ4PSI4IiBmaWxsPSIjRTFGNUVFIiBzdHJva2U9IiMwRjZFNTYiIHN0cm9rZS13aWR0aD0iMC41Ii8+CiAgPHRleHQgeD0iMTcwIiB5PSIyODAiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGRvbWluYW50LWJhc2VsaW5lPSJjZW50cmFsIiBzdHlsZT0iZm9udC1zaXplOjEzcHg7Zm9udC13ZWlnaHQ6NTAwO2ZpbGw6IzA4NTA0MTtmb250LWZhbWlseTpzYW5zLXNlcmlmIj5TZW5kZXI8L3RleHQ+CiAgPGxpbmUgeDE9IjIyNSIgeTE9IjI4MCIgeDI9IjI1NSIgeTI9IjI4MCIgc3Ryb2tlPSIjNUY1RTVBIiBzdHJva2Utd2lkdGg9IjEuNSIgbWFya2VyLWVuZD0idXJsKCNhcnJvdykiLz4KICA8cmVjdCB4PSIyNjAiIHk9IjI1NSIgd2lkdGg9IjEwNSIgaGVpZ2h0PSI1MCIgcng9IjgiIGZpbGw9IiNFMUY1RUUiIHN0cm9rZT0iIzBGNkU1NiIgc3Ryb2tlLXdpZHRoPSIwLjUiLz4KICA8dGV4dCB4PSIzMTIiIHk9IjI4MCIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZG9taW5hbnQtYmFzZWxpbmU9ImNlbnRyYWwiIHN0eWxlPSJmb250LXNpemU6MTNweDtmb250LXdlaWdodDo1MDA7ZmlsbDojMDg1MDQxO2ZvbnQtZmFtaWx5OnNhbnMtc2VyaWYiPk5ldHdvcms8L3RleHQ+CiAgPGxpbmUgeDE9IjM3MCIgeTE9IjI4MCIgeDI9IjQwMCIgeTI9IjI4MCIgc3Ryb2tlPSIjNUY1RTVBIiBzdHJva2Utd2lkdGg9IjEuNSIgbWFya2VyLWVuZD0idXJsKCNhcnJvdykiLz4KICA8cmVjdCB4PSI0MDUiIHk9IjI1NSIgd2lkdGg9IjEwNSIgaGVpZ2h0PSI1MCIgcng9IjgiIGZpbGw9IiNFMUY1RUUiIHN0cm9rZT0iIzBGNkU1NiIgc3Ryb2tlLXdpZHRoPSIwLjUiLz4KICA8dGV4dCB4PSI0NTciIHk9IjI4MCIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZG9taW5hbnQtYmFzZWxpbmU9ImNlbnRyYWwiIHN0eWxlPSJmb250LXNpemU6MTNweDtmb250LXdlaWdodDo1MDA7ZmlsbDojMDg1MDQxO2ZvbnQtZmFtaWx5OnNhbnMtc2VyaWYiPkJyb2tlcjwvdGV4dD4KPC9zdmc+)

</div>

### 调用线程（你的应用代码所在线程）

当调用 `producer.send(record)` 时，消息依次经过以下流水线：

1. **Interceptors（拦截器）**：可对消息做修改或过滤，大多数场景不使用
2. **Serializer（序列化器）**：将业务对象序列化为字节数组
3. **Partitioner（分区器）**：决定消息发往哪个 partition
4. **RecordAccumulator（缓冲区）**：将消息追加到对应 partition 的批次中

这一步完成后，`send()` 方法就返回了（异步），调用线程可以继续处理后续逻辑。

### Sender 线程（唯一的后台线程）

一个单独的后台线程不停从缓冲区取出"已就绪"的批次，通过 NetworkClient 发给 Kafka Broker。这个线程随 KafkaProducer 实例创建时启动，随 `close()` 终止。

### 关键含义

- 应用线程是"生产者"——往缓冲区放数据
- Sender 线程是"消费者"——从缓冲区取数据
- 缓冲区是两者之间的桥梁
- 当放入速度 > 取出速度，缓冲区逐渐填满

多个应用线程可以共享同一个 KafkaProducer 实例（它是线程安全的），它们都在往同一个 buffer.memory 里写。但从缓冲区取数据发给 Broker 的，只有一个 Sender 线程。这是典型的生产者-消费者模式。

---


## 三、缓冲区内部结构（RecordAccumulator）

缓冲区不是一个简单的大数组。它内部按 topic-partition 组织，每个 partition 有自己的一个双端队列（deque），每个队列里装着若干个"批次"（ProducerBatch）。

下图展示了 RecordAccumulator 的内部结构：绿色批次已就绪（等待 Sender 取走），橙色批次正在写入（接受新消息）。Sender 从头部取，调用线程向尾部追加：

<div align="center">

![svg](data:image/svg+xml;base64,PHN2ZyB2aWV3Qm94PSIwIDAgNjgwIDM5MCIgd2lkdGg9IjEwMCUiIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyIgcm9sZT0iaW1nIiBoZWlnaHQ9IjM5MCI+CiAgPHRpdGxlPlJlY29yZEFjY3VtdWxhdG9yIGludGVybmFsIHN0cnVjdHVyZTwvdGl0bGU+CiAgPGRlc2M+VGhlIGJ1ZmZlciBtZW1vcnkgcG9vbCBpcyBvcmdhbml6ZWQgYnkgdG9waWMtcGFydGl0aW9uLiBFYWNoIHBhcnRpdGlvbiBoYXMgYSBkZXF1ZSBvZiBiYXRjaGVzLiBUaGUgc2VuZGVyIHRocmVhZCB0YWtlcyByZWFkeSBiYXRjaGVzIGZyb20gdGhlIGhlYWQ7IHRoZSBjYWxsaW5nIHRocmVhZCBhcHBlbmRzIG5ldyBtZXNzYWdlcyB0byB0aGUgdGFpbCBiYXRjaC48L2Rlc2M+CiAgPGRlZnM+CiAgICA8bWFya2VyIGlkPSJhcnJvdzIiIHZpZXdCb3g9IjAgMCAxMCAxMCIgcmVmWD0iOCIgcmVmWT0iNSIKICAgICAgbWFya2VyV2lkdGg9IjYiIG1hcmtlckhlaWdodD0iNiIgb3JpZW50PSJhdXRvLXN0YXJ0LXJldmVyc2UiPgogICAgICA8cGF0aCBkPSJNMiAxTDggNUwyIDkiIGZpbGw9Im5vbmUiIHN0cm9rZT0iIzVGNUU1QSIKICAgICAgICBzdHJva2Utd2lkdGg9IjEuNSIgc3Ryb2tlLWxpbmVjYXA9InJvdW5kIiBzdHJva2UtbGluZWpvaW49InJvdW5kIi8+CiAgICA8L21hcmtlcj4KICA8L2RlZnM+CiAgPHJlY3QgeD0iNDAiIHk9IjQ1IiB3aWR0aD0iNjAwIiBoZWlnaHQ9IjMyNSIgcng9IjE2IiBmaWxsPSIjRkFFRURBIiBzdHJva2U9IiM4NTRGMEIiIHN0cm9rZS13aWR0aD0iMC41Ii8+CiAgPHRleHQgeD0iMzQwIiB5PSI2OCIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZG9taW5hbnQtYmFzZWxpbmU9ImNlbnRyYWwiIHN0eWxlPSJmb250LXNpemU6MTRweDtmb250LXdlaWdodDo1MDA7ZmlsbDojNjMzODA2O2ZvbnQtZmFtaWx5OnNhbnMtc2VyaWYiPmJ1ZmZlci5tZW1vcnkgPSAzMiBNQiB0b3RhbCBwb29sPC90ZXh0PgogIDx0ZXh0IHg9IjEyMCIgeT0iOTUiIHRleHQtYW5jaG9yPSJtaWRkbGUiIHN0eWxlPSJmb250LXNpemU6MTFweDtmaWxsOiM4ODg3ODA7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+aGVhZCDigJQgc2VuZGVyIHRha2VzPC90ZXh0PgogIDx0ZXh0IHg9IjU2MCIgeT0iOTUiIHRleHQtYW5jaG9yPSJtaWRkbGUiIHN0eWxlPSJmb250LXNpemU6MTFweDtmaWxsOiM4ODg3ODA7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+dGFpbCDigJQgYXBwZW5kcyBoZXJlPC90ZXh0PgogIDxyZWN0IHg9IjU1IiB5PSIxMDUiIHdpZHRoPSIxMjUiIGhlaWdodD0iNDQiIHJ4PSI4IiBmaWxsPSIjQ0VDQkY2IiBzdHJva2U9IiM1MzRBQjciIHN0cm9rZS13aWR0aD0iMC41Ii8+CiAgPHRleHQgeD0iMTE3IiB5PSIxMjciIHRleHQtYW5jaG9yPSJtaWRkbGUiIGRvbWluYW50LWJhc2VsaW5lPSJjZW50cmFsIiBzdHlsZT0iZm9udC1zaXplOjEycHg7Zm9udC13ZWlnaHQ6NTAwO2ZpbGw6IzNDMzQ4OTtmb250LWZhbWlseTpzYW5zLXNlcmlmIj50b3BpYy1BIC8gcGFydC0wPC90ZXh0PgogIDxsaW5lIHgxPSIxODUiIHkxPSIxMjciIHgyPSIyMDAiIHkyPSIxMjciIHN0cm9rZT0iIzVGNUU1QSIgc3Ryb2tlLXdpZHRoPSIxLjUiIG1hcmtlci1lbmQ9InVybCgjYXJyb3cyKSIvPgogIDxyZWN0IHg9IjIwNSIgeT0iMTA1IiB3aWR0aD0iODAiIGhlaWdodD0iNDQiIHJ4PSI4IiBmaWxsPSIjRTFGNUVFIiBzdHJva2U9IiMwRjZFNTYiIHN0cm9rZS13aWR0aD0iMC41Ii8+CiAgPHRleHQgeD0iMjQ1IiB5PSIxMjEiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGRvbWluYW50LWJhc2VsaW5lPSJjZW50cmFsIiBzdHlsZT0iZm9udC1zaXplOjEycHg7Zm9udC13ZWlnaHQ6NTAwO2ZpbGw6IzA4NTA0MTtmb250LWZhbWlseTpzYW5zLXNlcmlmIj4xNiBLQjwvdGV4dD4KICA8dGV4dCB4PSIyNDUiIHk9IjEzNyIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZG9taW5hbnQtYmFzZWxpbmU9ImNlbnRyYWwiIHN0eWxlPSJmb250LXNpemU6MTBweDtmaWxsOiMwRjZFNTY7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+cmVhZHk8L3RleHQ+CiAgPGxpbmUgeDE9IjI5MCIgeTE9IjEyNyIgeDI9IjMwNSIgeTI9IjEyNyIgc3Ryb2tlPSIjNUY1RTVBIiBzdHJva2Utd2lkdGg9IjEuNSIgbWFya2VyLWVuZD0idXJsKCNhcnJvdzIpIi8+CiAgPHJlY3QgeD0iMzEwIiB5PSIxMDUiIHdpZHRoPSI4MCIgaGVpZ2h0PSI0NCIgcng9IjgiIGZpbGw9IiNFMUY1RUUiIHN0cm9rZT0iIzBGNkU1NiIgc3Ryb2tlLXdpZHRoPSIwLjUiLz4KICA8dGV4dCB4PSIzNTAiIHk9IjEyMSIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZG9taW5hbnQtYmFzZWxpbmU9ImNlbnRyYWwiIHN0eWxlPSJmb250LXNpemU6MTJweDtmb250LXdlaWdodDo1MDA7ZmlsbDojMDg1MDQxO2ZvbnQtZmFtaWx5OnNhbnMtc2VyaWYiPjE2IEtCPC90ZXh0PgogIDx0ZXh0IHg9IjM1MCIgeT0iMTM3IiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBkb21pbmFudC1iYXNlbGluZT0iY2VudHJhbCIgc3R5bGU9ImZvbnQtc2l6ZToxMHB4O2ZpbGw6IzBGNkU1Njtmb250LWZhbWlseTpzYW5zLXNlcmlmIj5yZWFkeTwvdGV4dD4KICA8bGluZSB4MT0iMzk1IiB5MT0iMTI3IiB4Mj0iNDEwIiB5Mj0iMTI3IiBzdHJva2U9IiM1RjVFNUEiIHN0cm9rZS13aWR0aD0iMS41IiBtYXJrZXItZW5kPSJ1cmwoI2Fycm93MikiLz4KICA8cmVjdCB4PSI0MTUiIHk9IjEwNSIgd2lkdGg9Ijg1IiBoZWlnaHQ9IjQ0IiByeD0iOCIgZmlsbD0iI0ZBQzc3NSIgc3Ryb2tlPSIjODU0RjBCIiBzdHJva2Utd2lkdGg9IjAuNSIvPgogIDx0ZXh0IHg9IjQ1NyIgeT0iMTIxIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBkb21pbmFudC1iYXNlbGluZT0iY2VudHJhbCIgc3R5bGU9ImZvbnQtc2l6ZToxMnB4O2ZvbnQtd2VpZ2h0OjUwMDtmaWxsOiM2MzM4MDY7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+NCBLQjwvdGV4dD4KICA8dGV4dCB4PSI0NTciIHk9IjEzNyIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZG9taW5hbnQtYmFzZWxpbmU9ImNlbnRyYWwiIHN0eWxlPSJmb250LXNpemU6MTBweDtmaWxsOiM4NTRGMEI7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+d3JpdGluZzwvdGV4dD4KICA8cmVjdCB4PSI1NSIgeT0iMTcyIiB3aWR0aD0iMTI1IiBoZWlnaHQ9IjQ0IiByeD0iOCIgZmlsbD0iI0NFQ0JGNiIgc3Ryb2tlPSIjNTM0QUI3IiBzdHJva2Utd2lkdGg9IjAuNSIvPgogIDx0ZXh0IHg9IjExNyIgeT0iMTk0IiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBkb21pbmFudC1iYXNlbGluZT0iY2VudHJhbCIgc3R5bGU9ImZvbnQtc2l6ZToxMnB4O2ZvbnQtd2VpZ2h0OjUwMDtmaWxsOiMzQzM0ODk7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+dG9waWMtQSAvIHBhcnQtMTwvdGV4dD4KICA8bGluZSB4MT0iMTg1IiB5MT0iMTk0IiB4Mj0iMjAwIiB5Mj0iMTk0IiBzdHJva2U9IiM1RjVFNUEiIHN0cm9rZS13aWR0aD0iMS41IiBtYXJrZXItZW5kPSJ1cmwoI2Fycm93MikiLz4KICA8cmVjdCB4PSIyMDUiIHk9IjE3MiIgd2lkdGg9IjgwIiBoZWlnaHQ9IjQ0IiByeD0iOCIgZmlsbD0iI0UxRjVFRSIgc3Ryb2tlPSIjMEY2RTU2IiBzdHJva2Utd2lkdGg9IjAuNSIvPgogIDx0ZXh0IHg9IjI0NSIgeT0iMTg4IiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBkb21pbmFudC1iYXNlbGluZT0iY2VudHJhbCIgc3R5bGU9ImZvbnQtc2l6ZToxMnB4O2ZvbnQtd2VpZ2h0OjUwMDtmaWxsOiMwODUwNDE7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+MTYgS0I8L3RleHQ+CiAgPHRleHQgeD0iMjQ1IiB5PSIyMDQiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGRvbWluYW50LWJhc2VsaW5lPSJjZW50cmFsIiBzdHlsZT0iZm9udC1zaXplOjEwcHg7ZmlsbDojMEY2RTU2O2ZvbnQtZmFtaWx5OnNhbnMtc2VyaWYiPnJlYWR5PC90ZXh0PgogIDxsaW5lIHgxPSIyOTAiIHkxPSIxOTQiIHgyPSIzMDUiIHkyPSIxOTQiIHN0cm9rZT0iIzVGNUU1QSIgc3Ryb2tlLXdpZHRoPSIxLjUiIG1hcmtlci1lbmQ9InVybCgjYXJyb3cyKSIvPgogIDxyZWN0IHg9IjMxMCIgeT0iMTcyIiB3aWR0aD0iODUiIGhlaWdodD0iNDQiIHJ4PSI4IiBmaWxsPSIjRkFDNzc1IiBzdHJva2U9IiM4NTRGMEIiIHN0cm9rZS13aWR0aD0iMC41Ii8+CiAgPHRleHQgeD0iMzUyIiB5PSIxODgiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGRvbWluYW50LWJhc2VsaW5lPSJjZW50cmFsIiBzdHlsZT0iZm9udC1zaXplOjEycHg7Zm9udC13ZWlnaHQ6NTAwO2ZpbGw6IzYzMzgwNjtmb250LWZhbWlseTpzYW5zLXNlcmlmIj45IEtCPC90ZXh0PgogIDx0ZXh0IHg9IjM1MiIgeT0iMjA0IiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBkb21pbmFudC1iYXNlbGluZT0iY2VudHJhbCIgc3R5bGU9ImZvbnQtc2l6ZToxMHB4O2ZpbGw6Izg1NEYwQjtmb250LWZhbWlseTpzYW5zLXNlcmlmIj53cml0aW5nPC90ZXh0PgogIDxyZWN0IHg9IjU1IiB5PSIyMzkiIHdpZHRoPSIxMjUiIGhlaWdodD0iNDQiIHJ4PSI4IiBmaWxsPSIjQ0VDQkY2IiBzdHJva2U9IiM1MzRBQjciIHN0cm9rZS13aWR0aD0iMC41Ii8+CiAgPHRleHQgeD0iMTE3IiB5PSIyNjEiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGRvbWluYW50LWJhc2VsaW5lPSJjZW50cmFsIiBzdHlsZT0iZm9udC1zaXplOjEycHg7Zm9udC13ZWlnaHQ6NTAwO2ZpbGw6IzNDMzQ4OTtmb250LWZhbWlseTpzYW5zLXNlcmlmIj50b3BpYy1CIC8gcGFydC0wPC90ZXh0PgogIDxsaW5lIHgxPSIxODUiIHkxPSIyNjEiIHgyPSIyMDAiIHkyPSIyNjEiIHN0cm9rZT0iIzVGNUU1QSIgc3Ryb2tlLXdpZHRoPSIxLjUiIG1hcmtlci1lbmQ9InVybCgjYXJyb3cyKSIvPgogIDxyZWN0IHg9IjIwNSIgeT0iMjM5IiB3aWR0aD0iODUiIGhlaWdodD0iNDQiIHJ4PSI4IiBmaWxsPSIjRkFDNzc1IiBzdHJva2U9IiM4NTRGMEIiIHN0cm9rZS13aWR0aD0iMC41Ii8+CiAgPHRleHQgeD0iMjQ3IiB5PSIyNTUiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGRvbWluYW50LWJhc2VsaW5lPSJjZW50cmFsIiBzdHlsZT0iZm9udC1zaXplOjEycHg7Zm9udC13ZWlnaHQ6NTAwO2ZpbGw6IzYzMzgwNjtmb250LWZhbWlseTpzYW5zLXNlcmlmIj4zIEtCPC90ZXh0PgogIDx0ZXh0IHg9IjI0NyIgeT0iMjcxIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBkb21pbmFudC1iYXNlbGluZT0iY2VudHJhbCIgc3R5bGU9ImZvbnQtc2l6ZToxMHB4O2ZpbGw6Izg1NEYwQjtmb250LWZhbWlseTpzYW5zLXNlcmlmIj53cml0aW5nPC90ZXh0PgogIDxsaW5lIHgxPSI2MCIgeTE9IjMwOCIgeDI9IjYyMCIgeTI9IjMwOCIgc3Ryb2tlPSIjRDNEMUM3IiBzdHJva2Utd2lkdGg9IjAuNSIvPgogIDx0ZXh0IHg9IjM0MCIgeT0iMzI4IiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBkb21pbmFudC1iYXNlbGluZT0iY2VudHJhbCIgc3R5bGU9ImZvbnQtc2l6ZToxMnB4O2ZpbGw6IzVGNUU1QTtmb250LWZhbWlseTpzYW5zLXNlcmlmIj5ncmVlbiA9IHJlYWR5IChiYXRjaCBmdWxsIG9yIGxpbmdlci5tcyBlbGFwc2VkKSAgwrcgIGFtYmVyID0gY3VycmVudCBiYXRjaCwgYWNjZXB0aW5nIG5ldyBtZXNzYWdlczwvdGV4dD4KICA8dGV4dCB4PSIzNDAiIHk9IjM0OSIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZG9taW5hbnQtYmFzZWxpbmU9ImNlbnRyYWwiIHN0eWxlPSJmb250LXNpemU6MTJweDtmaWxsOiM1RjVFNUE7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+ZWFjaCBiYXRjaCBtYXggc2l6ZSA9IGJhdGNoLnNpemUgKGRlZmF1bHQgMTYgS0IpPC90ZXh0Pgo8L3N2Zz4=)

</div>

### 队列的两头

- **头部（head）**：Sender 线程从这里取走"已就绪"的批次发到网络
- **尾部（tail）**：调用线程把新消息追加到这里的"当前批次"

### 批次的状态

每个批次有两种状态：

- **writing（写入中）**：当前批次，正在接受新消息。位于队列尾部
- **ready（已就绪）**：等待发送的批次。位于队列头部

### 批次何时变为"已就绪"

满足以下任意一个条件，批次就变为 ready：

1. 批次大小达到 `batch.size`（默认 16KB）——装满了
2. 从第一条消息进入批次算起，等待时间达到 `linger.ms`（默认 0ms，即立刻发送）

### 内存为何会被耗尽

所有 partition 的所有批次（包括 ready 和 writing 的）共享 buffer.memory 总池。partition 越多，每个 partition 队列里的批次越多，总占用就越大。

更关键的是：已经发给网络但还没收到 Broker ACK 的批次，内存也不会释放。它们仍在占用 buffer.memory，直到 Broker 确认收到。这意味着如果 Broker 响应慢，in-flight 的批次会持续占用缓冲区内存。

### 粗略估算

假设有 100 个 partition，每个 partition 队列平均积压 4 个批次，每个批次 16KB：

- 总占用 = 100 × 4 × 16KB = 6400KB ≈ 6.25MB
- 再加上 in-flight 的 5 个连接 × 16KB × 若干 broker 连接
- 再加上重试中的批次
- 积压严重时，32MB 很容易被撑满

---

## 四、关键参数详解

### 缓冲与批量化参数

| 参数                 | 默认值   | 作用                              |
| ------------------ | ----- | ------------------------------- |
| `buffer.memory`    | 32 MB | 缓冲区总大小，所有 partition 的批次共享       |
| `batch.size`       | 16 KB | 单个批次最大大小                        |
| `linger.ms`        | 0 ms  | 攒批等待时间，超时后即使批次没满也发送             |
| `compression.type` | none  | 压缩算法（none/gzip/snappy/lz4/zstd） |

### 阻塞与超时参数

| 参数                    | 默认值           | 作用                                   |
| --------------------- | ------------- | ------------------------------------ |
| `max.block.ms`        | 60,000 ms     | `send()` 最大阻塞时间，超时抛 TimeoutException |
| `delivery.timeout.ms` | 120,000 ms    | 从 send 到收到 ACK 的总时间上限                |
| `request.timeout.ms`  | 30,000 ms     | 单次网络请求超时                             |
| `retries`             | 2,147,483,647 | 重试次数                                 |

### 网络参数

| 参数                                      | 默认值  | 作用           |
| --------------------------------------- | ---- | ------------ |
| `max.in.flight.requests.per.connection` | 5    | 每个连接的未确认请求上限 |
| `max.request.size`                      | 1 MB | 单次请求最大大小     |

---

## 五、参数之间的关系链

理解参数关系是理解阻塞的关键。以下是从"入水"到"出水"的完整链路：

```
应用线程 send() 速率  ──→  消息进入 buffer 速率（入水）
                                    │
                                    ▼
                        buffer.memory (32MB)  ← 总池子大小
                                    │
                              被以下因素占用 ↓
                    ┌──────────────────────────────────┐
                    │ partition数 × 每队列批次数 × batch.size
                    │ + in-flight 未确认批次占用
                    │ + 重试中的批次占用
                    └──────────────────────────────────┘
                                    │
                                    ▼
                        Sender 线程取出  ──→  网络  ──→  Broker ACK
                        （出水速率）

     入水 > 出水  ⟹  buffer 填满  ⟹  新 send() 阻塞  ⟹  max.block.ms 后超时
```

### 关键关系说明

- **buffer.memory vs batch.size**：buffer.memory 是总池子，batch.size 是每个批次的上限。总占用 ≈ partition数 × 队列批次数 × batch.size
- **linger.ms vs batch.size**：linger.ms=0 时批次攒到第一条就立刻发（除非同时到达），吞吐低但延迟低；增大 linger.ms 可以让批次更满，提高吞吐
- **compression.type**：启用后批次实际占用内存大幅缩减（通常压缩率 50%~70%），等于用相同的 buffer.memory 容纳更多消息
- **retries**：默认 Integer.MAX_VALUE，重试时批次不释放，持续占用内存
- **max.in.flight.requests.per.connection**：越大 → 同时占用的 buffer 越多（in-flight 批次等 ACK 时不释放）

---


## 六、缓冲区耗尽与线程阻塞机制

下图展示了缓冲区填满后的阻塞场景：多个应用线程的 `send()` 调用因缓冲区无空闲空间而阻塞，Sender 线程排水缓慢，`max.block.ms` 倒计时中：

<div align="center">

![svg](data:image/svg+xml;base64,PHN2ZyB2aWV3Qm94PSIwIDAgNjgwIDQxMCIgd2lkdGg9IjEwMCUiIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyIgcm9sZT0iaW1nIiBoZWlnaHQ9IjQxMCI+CiAgPHRpdGxlPkJ1ZmZlciBleGhhdXN0aW9uOiB3aHkgY2FsbGluZyB0aHJlYWRzIGJsb2NrPC90aXRsZT4KICA8ZGVzYz5XaGVuIHRoZSBidWZmZXIgcG9vbCBpcyBmdWxsIGFuZCB0aGUgc2VuZGVyIHRocmVhZCBjYW5ub3QgZHJhaW4gaXQgZmFzdCBlbm91Z2gsIG5ldyBzZW5kKCkgY2FsbHMgYmxvY2suIElmIHRoZSBidWZmZXIgZG9lcyBub3QgZnJlZSB1cCB3aXRoaW4gbWF4LmJsb2NrLm1zLCBhIFRpbWVvdXRFeGNlcHRpb24gaXMgdGhyb3duLjwvZGVzYz4KICA8ZGVmcz4KICAgIDxtYXJrZXIgaWQ9ImFycm93MyIgdmlld0JveD0iMCAwIDEwIDEwIiByZWZYPSI4IiByZWZZPSI1IgogICAgICBtYXJrZXJXaWR0aD0iNiIgbWFya2VySGVpZ2h0PSI2IiBvcmllbnQ9ImF1dG8tc3RhcnQtcmV2ZXJzZSI+CiAgICAgIDxwYXRoIGQ9Ik0yIDFMOCA1TDIgOSIgZmlsbD0ibm9uZSIgc3Ryb2tlPSIjNUY1RTVBIgogICAgICAgIHN0cm9rZS13aWR0aD0iMS41IiBzdHJva2UtbGluZWNhcD0icm91bmQiIHN0cm9rZS1saW5lam9pbj0icm91bmQiLz4KICAgIDwvbWFya2VyPgogIDwvZGVmcz4KICA8cmVjdCB4PSI4NSIgeT0iNDUiIHdpZHRoPSIxMTAiIGhlaWdodD0iNTAiIHJ4PSI4IiBmaWxsPSIjRkFFQ0U3IiBzdHJva2U9IiM5OTNDMUQiIHN0cm9rZS13aWR0aD0iMC41Ii8+CiAgPHRleHQgeD0iMTQwIiB5PSI2NSIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZG9taW5hbnQtYmFzZWxpbmU9ImNlbnRyYWwiIHN0eWxlPSJmb250LXNpemU6MTNweDtmb250LXdlaWdodDo1MDA7ZmlsbDojNzEyQjEzO2ZvbnQtZmFtaWx5OnNhbnMtc2VyaWYiPlRocmVhZC0xPC90ZXh0PgogIDx0ZXh0IHg9IjE0MCIgeT0iODIiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGRvbWluYW50LWJhc2VsaW5lPSJjZW50cmFsIiBzdHlsZT0iZm9udC1zaXplOjExcHg7ZmlsbDojOTkzQzFEO2ZvbnQtZmFtaWx5OnNhbnMtc2VyaWYiPkJMT0NLRUQ8L3RleHQ+CiAgPHJlY3QgeD0iMjg1IiB5PSI0NSIgd2lkdGg9IjExMCIgaGVpZ2h0PSI1MCIgcng9IjgiIGZpbGw9IiNGQUVDRTciIHN0cm9rZT0iIzk5M0MxRCIgc3Ryb2tlLXdpZHRoPSIwLjUiLz4KICA8dGV4dCB4PSIzNDAiIHk9IjY1IiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBkb21pbmFudC1iYXNlbGluZT0iY2VudHJhbCIgc3R5bGU9ImZvbnQtc2l6ZToxM3B4O2ZvbnQtd2VpZ2h0OjUwMDtmaWxsOiM3MTJCMTM7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+VGhyZWFkLTI8L3RleHQ+CiAgPHRleHQgeD0iMzQwIiB5PSI4MiIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZG9taW5hbnQtYmFzZWxpbmU9ImNlbnRyYWwiIHN0eWxlPSJmb250LXNpemU6MTFweDtmaWxsOiM5OTNDMUQ7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+QkxPQ0tFRDwvdGV4dD4KICA8cmVjdCB4PSI0ODUiIHk9IjQ1IiB3aWR0aD0iMTEwIiBoZWlnaHQ9IjUwIiByeD0iOCIgZmlsbD0iI0ZBRUNFNyIgc3Ryb2tlPSIjOTkzQzFEIiBzdHJva2Utd2lkdGg9IjAuNSIvPgogIDx0ZXh0IHg9IjU0MCIgeT0iNjUiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGRvbWluYW50LWJhc2VsaW5lPSJjZW50cmFsIiBzdHlsZT0iZm9udC1zaXplOjEzcHg7Zm9udC13ZWlnaHQ6NTAwO2ZpbGw6IzcxMkIxMztmb250LWZhbWlseTpzYW5zLXNlcmlmIj5UaHJlYWQtMzwvdGV4dD4KICA8dGV4dCB4PSI1NDAiIHk9IjgyIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBkb21pbmFudC1iYXNlbGluZT0iY2VudHJhbCIgc3R5bGU9ImZvbnQtc2l6ZToxMXB4O2ZpbGw6Izk5M0MxRDtmb250LWZhbWlseTpzYW5zLXNlcmlmIj5CTE9DS0VEPC90ZXh0PgogIDxsaW5lIHgxPSIxNDAiIHkxPSIxMDAiIHgyPSIxNDAiIHkyPSIxNDAiIHN0cm9rZT0iI0UyNEI0QSIgc3Ryb2tlLXdpZHRoPSIxLjUiIHN0cm9rZS1kYXNoYXJyYXk9IjQsMyIvPgogIDxsaW5lIHgxPSIzNDAiIHkxPSIxMDAiIHgyPSIzNDAiIHkyPSIxNDAiIHN0cm9rZT0iI0UyNEI0QSIgc3Ryb2tlLXdpZHRoPSIxLjUiIHN0cm9rZS1kYXNoYXJyYXk9IjQsMyIvPgogIDxsaW5lIHgxPSI1NDAiIHkxPSIxMDAiIHgyPSI1NDAiIHkyPSIxNDAiIHN0cm9rZT0iI0UyNEI0QSIgc3Ryb2tlLXdpZHRoPSIxLjUiIHN0cm9rZS1kYXNoYXJyYXk9IjQsMyIvPgogIDx0ZXh0IHg9IjM0MCIgeT0iMTIyIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBkb21pbmFudC1iYXNlbGluZT0iY2VudHJhbCIgc3R5bGU9ImZvbnQtc2l6ZToxMXB4O2ZpbGw6I0UyNEI0QTtmb250LWZhbWlseTpzYW5zLXNlcmlmIj5jYW5ub3QgYXBwZW5kIOKAlCBubyBmcmVlIGJ1ZmZlciBzcGFjZTwvdGV4dD4KICA8cmVjdCB4PSIxMDAiIHk9IjE0NSIgd2lkdGg9IjQ4MCIgaGVpZ2h0PSI3MCIgcng9IjEyIiBmaWxsPSIjRkNFQkVCIiBzdHJva2U9IiNBMzJEMkQiIHN0cm9rZS13aWR0aD0iMC41Ii8+CiAgPHRleHQgeD0iMzQwIiB5PSIxNjgiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGRvbWluYW50LWJhc2VsaW5lPSJjZW50cmFsIiBzdHlsZT0iZm9udC1zaXplOjE0cHg7Zm9udC13ZWlnaHQ6NTAwO2ZpbGw6Izc5MUYxRjtmb250LWZhbWlseTpzYW5zLXNlcmlmIj5idWZmZXIubWVtb3J5ID0gMzIgTUIg4oCUIDEwMCUgRlVMTDwvdGV4dD4KICA8cmVjdCB4PSIxMjAiIHk9IjE4MyIgd2lkdGg9IjQ0MCIgaGVpZ2h0PSIxNCIgcng9IjQiIGZpbGw9IiNGN0MxQzEiLz4KICA8cmVjdCB4PSIxMjAiIHk9IjE4MyIgd2lkdGg9IjQ0MCIgaGVpZ2h0PSIxNCIgcng9IjQiIGZpbGw9IiNFMjRCNEEiLz4KICA8dGV4dCB4PSI1NTUiIHk9IjE5MSIgdGV4dC1hbmNob3I9ImVuZCIgZG9taW5hbnQtYmFzZWxpbmU9ImNlbnRyYWwiIHN0eWxlPSJmb250LXNpemU6MTBweDtmaWxsOiM3OTFGMUY7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+MzIgLyAzMiBNQjwvdGV4dD4KICA8bGluZSB4MT0iMzQwIiB5MT0iMjIwIiB4Mj0iMzQwIiB5Mj0iMjU1IiBzdHJva2U9IiM1RjVFNUEiIHN0cm9rZS13aWR0aD0iMS41IiBtYXJrZXItZW5kPSJ1cmwoI2Fycm93MykiLz4KICA8dGV4dCB4PSIzNTAiIHk9IjI0MCIgc3R5bGU9ImZvbnQtc2l6ZToxMXB4O2ZpbGw6Izg4ODc4MDtmb250LWZhbWlseTpzYW5zLXNlcmlmIj5zZW5kZXIgZHJhaW5zIOKAlCBidXQgc2xvd2x5PC90ZXh0PgogIDxyZWN0IHg9IjEyMCIgeT0iMjYwIiB3aWR0aD0iMTAwIiBoZWlnaHQ9IjUwIiByeD0iOCIgZmlsbD0iI0UxRjVFRSIgc3Ryb2tlPSIjMEY2RTU2IiBzdHJva2Utd2lkdGg9IjAuNSIvPgogIDx0ZXh0IHg9IjE3MCIgeT0iMjg1IiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBkb21pbmFudC1iYXNlbGluZT0iY2VudHJhbCIgc3R5bGU9ImZvbnQtc2l6ZToxM3B4O2ZvbnQtd2VpZ2h0OjUwMDtmaWxsOiMwODUwNDE7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+U2VuZGVyPC90ZXh0PgogIDxsaW5lIHgxPSIyMjUiIHkxPSIyODUiIHgyPSIyNTUiIHkyPSIyODUiIHN0cm9rZT0iIzVGNUU1QSIgc3Ryb2tlLXdpZHRoPSIxLjUiIG1hcmtlci1lbmQ9InVybCgjYXJyb3czKSIvPgogIDxyZWN0IHg9IjI2MCIgeT0iMjYwIiB3aWR0aD0iOTUiIGhlaWdodD0iNTAiIHJ4PSI4IiBmaWxsPSIjRTFGNUVFIiBzdHJva2U9IiMwRjZFNTYiIHN0cm9rZS13aWR0aD0iMC41Ii8+CiAgPHRleHQgeD0iMzA3IiB5PSIyODUiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGRvbWluYW50LWJhc2VsaW5lPSJjZW50cmFsIiBzdHlsZT0iZm9udC1zaXplOjEzcHg7Zm9udC13ZWlnaHQ6NTAwO2ZpbGw6IzA4NTA0MTtmb250LWZhbWlseTpzYW5zLXNlcmlmIj5OZXR3b3JrPC90ZXh0PgogIDxsaW5lIHgxPSIzNjAiIHkxPSIyODUiIHgyPSIzOTAiIHkyPSIyODUiIHN0cm9rZT0iIzVGNUU1QSIgc3Ryb2tlLXdpZHRoPSIxLjUiIG1hcmtlci1lbmQ9InVybCgjYXJyb3czKSIvPgogIDxyZWN0IHg9IjM5NSIgeT0iMjYwIiB3aWR0aD0iOTAiIGhlaWdodD0iNTAiIHJ4PSI4IiBmaWxsPSIjRTFGNUVFIiBzdHJva2U9IiMwRjZFNTYiIHN0cm9rZS13aWR0aD0iMC41Ii8+CiAgPHRleHQgeD0iNDQwIiB5PSIyODUiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGRvbWluYW50LWJhc2VsaW5lPSJjZW50cmFsIiBzdHlsZT0iZm9udC1zaXplOjEzcHg7Zm9udC13ZWlnaHQ6NTAwO2ZpbGw6IzA4NTA0MTtmb250LWZhbWlseTpzYW5zLXNlcmlmIj5Ccm9rZXI8L3RleHQ+CiAgPHRleHQgeD0iMzA3IiB5PSIzMzAiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGRvbWluYW50LWJhc2VsaW5lPSJjZW50cmFsIiBzdHlsZT0iZm9udC1zaXplOjExcHg7ZmlsbDojOTkzQzFEO2ZvbnQtZmFtaWx5OnNhbnMtc2VyaWYiPnNsb3c6IGhpZ2ggbGF0ZW5jeSAvIGJyb2tlciBvdmVybG9hZCAvIHJldHJpZXMgd2l0aCBiYWNrb2ZmPC90ZXh0PgogIDxyZWN0IHg9IjEzMCIgeT0iMzQ4IiB3aWR0aD0iNDIwIiBoZWlnaHQ9IjQ0IiByeD0iOCIgZmlsbD0iI0ZBRUNFNyIgc3Ryb2tlPSIjOTkzQzFEIiBzdHJva2Utd2lkdGg9IjAuNSIvPgogIDx0ZXh0IHg9IjM0MCIgeT0iMzY0IiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBkb21pbmFudC1iYXNlbGluZT0iY2VudHJhbCIgc3R5bGU9ImZvbnQtc2l6ZToxM3B4O2ZvbnQtd2VpZ2h0OjUwMDtmaWxsOiM3MTJCMTM7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+bWF4LmJsb2NrLm1zID0gNjAsMDAwIG1zIChkZWZhdWx0KTwvdGV4dD4KICA8dGV4dCB4PSIzNDAiIHk9IjM4MSIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZG9taW5hbnQtYmFzZWxpbmU9ImNlbnRyYWwiIHN0eWxlPSJmb250LXNpemU6MTFweDtmaWxsOiM5OTNDMUQ7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+aWYgYnVmZmVyIGRvZXMgbm90IGZyZWUgdXAgd2l0aGluIDYwcywgc2VuZCgpIHRocm93cyBUaW1lb3V0RXhjZXB0aW9uPC90ZXh0Pgo8L3N2Zz4=)

</div>

### 水箱模型

把 buffer.memory 想象成水箱：

- 应用线程 = 进水管（多个水龙头往水箱里灌水）
- Sender 线程 = 出水管（一根管子从水箱排水）
- 当进水 > 出水 → 水箱逐渐填满 → 满了之后新的水龙头放不进水 → 阻塞

### 阻塞发生的具体位置

当调用 `send()` 且缓冲区没有空闲空间时，阻塞发生在 `RecordAccumulator.append()` 方法内部。该方法尝试从 `BufferPool` 分配内存，如果池子没有空闲空间，线程进入等待状态。

### max.block.ms：阻塞的"熔断器"

`max.block.ms`（默认 60 秒）是阻塞的熔断参数。线程最多等待这么长时间，如果缓冲区仍然没有空间，`send()` 抛出 `TimeoutException`。

需要注意，`max.block.ms` 对应两种阻塞场景：

1. 等待 buffer 有空闲内存（buffer 满了）
2. 等待获取元数据（metadata，如 partition leader 信息）

两种都会计入这个超时。

### Sender 线程排水慢的常见原因

| 原因                           | 机制                  | 占用 buffer 的方式                  |
| ---------------------------- | ------------------- | ------------------------------ |
| Broker 负载高/慢                 | ACK 迟迟不来            | in-flight 批次内存不释放              |
| 网络延迟高                        | 请求-响应慢              | 同上                             |
| 重试风暴（retries 默认 Integer.MAX） | 不断重试失败请求            | 重试批次持续占用内存                     |
| partition 数量多                | 每个 partition 都有队列   | 每个 partition 至少占一个 batch（16KB） |
| linger.ms=0 + batch.size 小   | 批次太小 → 请求数多 → 网络效率低 | Sender 线程处理不过来                 |

---


## 七、每个 KafkaProducer 独享 Sender 线程

多个 KafkaProducer 实例之间**不共享** Sender 线程。每个 KafkaProducer 在构造时各自创建自己的 Sender 线程、RecordAccumulator、NetworkClient 和 TCP 连接池，完全独立，无共享状态。

下图展示了三个 KafkaProducer 实例各自独享线程和缓冲区的结构：

<div align="center">

![svg](data:image/svg+xml;base64,PHN2ZyB2aWV3Qm94PSIwIDAgNjgwIDM0MCIgd2lkdGg9IjEwMCUiIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyIgcm9sZT0iaW1nIiBoZWlnaHQ9IjM0MCI+CiAgPHRpdGxlPkVhY2ggS2Fma2FQcm9kdWNlciBpbnN0YW5jZSBoYXMgaXRzIG93biBTZW5kZXIgdGhyZWFkIGFuZCBidWZmZXI8L3RpdGxlPgogIDxkZXNjPk11bHRpcGxlIEthZmthUHJvZHVjZXIgaW5zdGFuY2VzIGRvIG5vdCBzaGFyZSBhIFNlbmRlciB0aHJlYWQuIEVhY2ggaW5zdGFuY2UgY3JlYXRlcyBpdHMgb3duIFNlbmRlciB0aHJlYWQsIFJlY29yZEFjY3VtdWxhdG9yIChidWZmZXIubWVtb3J5KSwgYW5kIE5ldHdvcmtDbGllbnQgaW5kZXBlbmRlbnRseS48L2Rlc2M+CiAgPGRlZnM+CiAgICA8bWFya2VyIGlkPSJhcjQiIHZpZXdCb3g9IjAgMCAxMCAxMCIgcmVmWD0iOCIgcmVmWT0iNSIKICAgICAgbWFya2VyV2lkdGg9IjYiIG1hcmtlckhlaWdodD0iNiIgb3JpZW50PSJhdXRvLXN0YXJ0LXJldmVyc2UiPgogICAgICA8cGF0aCBkPSJNMiAxTDggNUwyIDkiIGZpbGw9Im5vbmUiIHN0cm9rZT0iIzVGNUU1QSIKICAgICAgICBzdHJva2Utd2lkdGg9IjEuNSIgc3Ryb2tlLWxpbmVjYXA9InJvdW5kIiBzdHJva2UtbGluZWpvaW49InJvdW5kIi8+CiAgICA8L21hcmtlcj4KICA8L2RlZnM+CiAgPHJlY3QgeD0iNDAiIHk9IjQwIiB3aWR0aD0iMTgwIiBoZWlnaHQ9IjIyMCIgcng9IjEyIiBmaWxsPSIjRTZGMUZCIiBzdHJva2U9IiMxODVGQTUiIHN0cm9rZS13aWR0aD0iMC41Ii8+CiAgPHRleHQgeD0iMTMwIiB5PSI2MiIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZG9taW5hbnQtYmFzZWxpbmU9ImNlbnRyYWwiIHN0eWxlPSJmb250LXNpemU6MTNweDtmb250LXdlaWdodDo1MDA7ZmlsbDojMEM0NDdDO2ZvbnQtZmFtaWx5OnNhbnMtc2VyaWYiPlByb2R1Y2VyLTE8L3RleHQ+CiAgPHJlY3QgeD0iNTUiIHk9Ijc4IiB3aWR0aD0iMTUwIiBoZWlnaHQ9IjQwIiByeD0iOCIgZmlsbD0iI0I1RDRGNCIgc3Ryb2tlPSIjMTg1RkE1IiBzdHJva2Utd2lkdGg9IjAuNSIvPgogIDx0ZXh0IHg9IjEzMCIgeT0iOTgiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGRvbWluYW50LWJhc2VsaW5lPSJjZW50cmFsIiBzdHlsZT0iZm9udC1zaXplOjEycHg7Zm9udC13ZWlnaHQ6NTAwO2ZpbGw6IzBDNDQ3Qztmb250LWZhbWlseTpzYW5zLXNlcmlmIj5BcHAgdGhyZWFkczwvdGV4dD4KICA8bGluZSB4MT0iMTMwIiB5MT0iMTIzIiB4Mj0iMTMwIiB5Mj0iMTQwIiBzdHJva2U9IiM1RjVFNUEiIHN0cm9rZS13aWR0aD0iMS41IiBtYXJrZXItZW5kPSJ1cmwoI2FyNCkiLz4KICA8cmVjdCB4PSI1NSIgeT0iMTQ1IiB3aWR0aD0iMTUwIiBoZWlnaHQ9IjQwIiByeD0iOCIgZmlsbD0iI0ZBRUVEQSIgc3Ryb2tlPSIjODU0RjBCIiBzdHJva2Utd2lkdGg9IjAuNSIvPgogIDx0ZXh0IHg9IjEzMCIgeT0iMTY1IiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBkb21pbmFudC1iYXNlbGluZT0iY2VudHJhbCIgc3R5bGU9ImZvbnQtc2l6ZToxMnB4O2ZvbnQtd2VpZ2h0OjUwMDtmaWxsOiM2MzM4MDY7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+YnVmZmVyIDMyIE1CPC90ZXh0PgogIDxsaW5lIHgxPSIxMzAiIHkxPSIxOTAiIHgyPSIxMzAiIHkyPSIyMDciIHN0cm9rZT0iIzVGNUU1QSIgc3Ryb2tlLXdpZHRoPSIxLjUiIG1hcmtlci1lbmQ9InVybCgjYXI0KSIvPgogIDxyZWN0IHg9IjU1IiB5PSIyMTIiIHdpZHRoPSIxNTAiIGhlaWdodD0iNDAiIHJ4PSI4IiBmaWxsPSIjRTFGNUVFIiBzdHJva2U9IiMwRjZFNTYiIHN0cm9rZS13aWR0aD0iMC41Ii8+CiAgPHRleHQgeD0iMTMwIiB5PSIyMzIiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGRvbWluYW50LWJhc2VsaW5lPSJjZW50cmFsIiBzdHlsZT0iZm9udC1zaXplOjEycHg7Zm9udC13ZWlnaHQ6NTAwO2ZpbGw6IzA4NTA0MTtmb250LWZhbWlseTpzYW5zLXNlcmlmIj5TZW5kZXIgdGhyZWFkLTE8L3RleHQ+CiAgPHJlY3QgeD0iMjUwIiB5PSI0MCIgd2lkdGg9IjE4MCIgaGVpZ2h0PSIyMjAiIHJ4PSIxMiIgZmlsbD0iI0U2RjFGQiIgc3Ryb2tlPSIjMTg1RkE1IiBzdHJva2Utd2lkdGg9IjAuNSIvPgogIDx0ZXh0IHg9IjM0MCIgeT0iNjIiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGRvbWluYW50LWJhc2VsaW5lPSJjZW50cmFsIiBzdHlsZT0iZm9udC1zaXplOjEzcHg7Zm9udC13ZWlnaHQ6NTAwO2ZpbGw6IzBDNDQ3Qztmb250LWZhbWlseTpzYW5zLXNlcmlmIj5Qcm9kdWNlci0yPC90ZXh0PgogIDxyZWN0IHg9IjI2NSIgeT0iNzgiIHdpZHRoPSIxNTAiIGhlaWdodD0iNDAiIHJ4PSI4IiBmaWxsPSIjQjVENEY0IiBzdHJva2U9IiMxODVGQTUiIHN0cm9rZS13aWR0aD0iMC41Ii8+CiAgPHRleHQgeD0iMzQwIiB5PSI5OCIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZG9taW5hbnQtYmFzZWxpbmU9ImNlbnRyYWwiIHN0eWxlPSJmb250LXNpemU6MTJweDtmb250LXdlaWdodDo1MDA7ZmlsbDojMEM0NDdDO2ZvbnQtZmFtaWx5OnNhbnMtc2VyaWYiPkFwcCB0aHJlYWRzPC90ZXh0PgogIDxsaW5lIHgxPSIzNDAiIHkxPSIxMjMiIHgyPSIzNDAiIHkyPSIxNDAiIHN0cm9rZT0iIzVGNUU1QSIgc3Ryb2tlLXdpZHRoPSIxLjUiIG1hcmtlci1lbmQ9InVybCgjYXI0KSIvPgogIDxyZWN0IHg9IjI2NSIgeT0iMTQ1IiB3aWR0aD0iMTUwIiBoZWlnaHQ9IjQwIiByeD0iOCIgZmlsbD0iI0ZBRUVEQSIgc3Ryb2tlPSIjODU0RjBCIiBzdHJva2Utd2lkdGg9IjAuNSIvPgogIDx0ZXh0IHg9IjM0MCIgeT0iMTY1IiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBkb21pbmFudC1iYXNlbGluZT0iY2VudHJhbCIgc3R5bGU9ImZvbnQtc2l6ZToxMnB4O2ZvbnQtd2VpZ2h0OjUwMDtmaWxsOiM2MzM4MDY7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+YnVmZmVyIDMyIE1CPC90ZXh0PgogIDxsaW5lIHgxPSIzNDAiIHkxPSIxOTAiIHgyPSIzNDAiIHkyPSIyMDciIHN0cm9rZT0iIzVGNUU1QSIgc3Ryb2tlLXdpZHRoPSIxLjUiIG1hcmtlci1lbmQ9InVybCgjYXI0KSIvPgogIDxyZWN0IHg9IjI2NSIgeT0iMjEyIiB3aWR0aD0iMTUwIiBoZWlnaHQ9IjQwIiByeD0iOCIgZmlsbD0iI0UxRjVFRSIgc3Ryb2tlPSIjMEY2RTU2IiBzdHJva2Utd2lkdGg9IjAuNSIvPgogIDx0ZXh0IHg9IjM0MCIgeT0iMjMyIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBkb21pbmFudC1iYXNlbGluZT0iY2VudHJhbCIgc3R5bGU9ImZvbnQtc2l6ZToxMnB4O2ZvbnQtd2VpZ2h0OjUwMDtmaWxsOiMwODUwNDE7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+U2VuZGVyIHRocmVhZC0yPC90ZXh0PgogIDxyZWN0IHg9IjQ2MCIgeT0iNDAiIHdpZHRoPSIxODAiIGhlaWdodD0iMjIwIiByeD0iMTIiIGZpbGw9IiNFNkYxRkIiIHN0cm9rZT0iIzE4NUZBNSIgc3Ryb2tlLXdpZHRoPSIwLjUiLz4KICA8dGV4dCB4PSI1NTAiIHk9IjYyIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBkb21pbmFudC1iYXNlbGluZT0iY2VudHJhbCIgc3R5bGU9ImZvbnQtc2l6ZToxM3B4O2ZvbnQtd2VpZ2h0OjUwMDtmaWxsOiMwQzQ0N0M7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+UHJvZHVjZXItMzwvdGV4dD4KICA8cmVjdCB4PSI0NzUiIHk9Ijc4IiB3aWR0aD0iMTUwIiBoZWlnaHQ9IjQwIiByeD0iOCIgZmlsbD0iI0I1RDRGNCIgc3Ryb2tlPSIjMTg1RkE1IiBzdHJva2Utd2lkdGg9IjAuNSIvPgogIDx0ZXh0IHg9IjU1MCIgeT0iOTgiIHRleHQtYW5jaG9yPSJtaWRkbGUiIGRvbWluYW50LWJhc2VsaW5lPSJjZW50cmFsIiBzdHlsZT0iZm9udC1zaXplOjEycHg7Zm9udC13ZWlnaHQ6NTAwO2ZpbGw6IzBDNDQ3Qztmb250LWZhbWlseTpzYW5zLXNlcmlmIj5BcHAgdGhyZWFkczwvdGV4dD4KICA8bGluZSB4MT0iNTUwIiB5MT0iMTIzIiB4Mj0iNTUwIiB5Mj0iMTQwIiBzdHJva2U9IiM1RjVFNUEiIHN0cm9rZS13aWR0aD0iMS41IiBtYXJrZXItZW5kPSJ1cmwoI2FyNCkiLz4KICA8cmVjdCB4PSI0NzUiIHk9IjE0NSIgd2lkdGg9IjE1MCIgaGVpZ2h0PSI0MCIgcng9IjgiIGZpbGw9IiNGQUVFREEiIHN0cm9rZT0iIzg1NEYwQiIgc3Ryb2tlLXdpZHRoPSIwLjUiLz4KICA8dGV4dCB4PSI1NTAiIHk9IjE2NSIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZG9taW5hbnQtYmFzZWxpbmU9ImNlbnRyYWwiIHN0eWxlPSJmb250LXNpemU6MTJweDtmb250LXdlaWdodDo1MDA7ZmlsbDojNjMzODA2O2ZvbnQtZmFtaWx5OnNhbnMtc2VyaWYiPmJ1ZmZlciAzMiBNQjwvdGV4dD4KICA8bGluZSB4MT0iNTUwIiB5MT0iMTkwIiB4Mj0iNTUwIiB5Mj0iMjA3IiBzdHJva2U9IiM1RjVFNUEiIHN0cm9rZS13aWR0aD0iMS41IiBtYXJrZXItZW5kPSJ1cmwoI2FyNCkiLz4KICA8cmVjdCB4PSI0NzUiIHk9IjIxMiIgd2lkdGg9IjE1MCIgaGVpZ2h0PSI0MCIgcng9IjgiIGZpbGw9IiNFMUY1RUUiIHN0cm9rZT0iIzBGNkU1NiIgc3Ryb2tlLXdpZHRoPSIwLjUiLz4KICA8dGV4dCB4PSI1NTAiIHk9IjIzMiIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZG9taW5hbnQtYmFzZWxpbmU9ImNlbnRyYWwiIHN0eWxlPSJmb250LXNpemU6MTJweDtmb250LXdlaWdodDo1MDA7ZmlsbDojMDg1MDQxO2ZvbnQtZmFtaWx5OnNhbnMtc2VyaWYiPlNlbmRlciB0aHJlYWQtMzwvdGV4dD4KICA8dGV4dCB4PSIzNDAiIHk9IjI5NSIgdGV4dC1hbmNob3I9Im1pZGRsZSIgZG9taW5hbnQtYmFzZWxpbmU9ImNlbnRyYWwiIHN0eWxlPSJmb250LXNpemU6MTJweDtmaWxsOiM1RjVFNUE7Zm9udC1mYW1pbHk6c2Fucy1zZXJpZiI+MyBwcm9kdWNlcnMgPSAzIFNlbmRlciB0aHJlYWRzID0gMyB4IDMyIE1CIGJ1ZmZlciA9IDk2IE1CIHRvdGFsPC90ZXh0PgogIDx0ZXh0IHg9IjM0MCIgeT0iMzE1IiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiBkb21pbmFudC1iYXNlbGluZT0iY2VudHJhbCIgc3R5bGU9ImZvbnQtc2l6ZToxMnB4O2ZpbGw6Izg4ODc4MDtmb250LWZhbWlseTpzYW5zLXNlcmlmIj5jb21wbGV0ZWx5IGluZGVwZW5kZW50LCBubyBzaGFyZWQgc3RhdGU8L3RleHQ+Cjwvc3ZnPg==)

</div>

| 组件                               | 是否共享 |
| -------------------------------- | ---- |
| Sender 线程                        | 独享   |
| RecordAccumulator（buffer.memory） | 独享   |
| NetworkClient                    | 独享   |
| TCP 连接池                          | 独享   |

Sender 线程名是 `kafka-producer-network-thread | <clientId>`，在 JVM 线程 dump 中可以看到。3 个 KafkaProducer 实例 = 3 条 Sender 线程 = 3 × 32MB buffer = 96MB 总内存，完全独立，无共享状态。

---

## 八、优化方案（按优先级排列）

### 1. 启用压缩（效果最显著，成本最低）

```properties
compression.type=lz4
```

压缩后批次实际占用的 buffer 内存大幅缩减，等于用相同的 32MB 容纳 2~3 倍的消息。lz4 压缩/解压速度极快，几乎不增加 CPU 开销。这是性价比最高的优化。

### 2. 增大 buffer.memory（缓解，不根治）

```properties
buffer.memory=67108864    # 64 MB，或更大
```

从 32MB 提升到 64MB 甚至 128MB，给更多缓冲余量。但如果出水速度始终跟不上，只是推迟被填满的时间，不是根本解法。

### 3. 调大 batch.size + 设置合理的 linger.ms（提升吞吐）

```properties
batch.size=65536       # 64 KB，更大批次 → 更少请求 → 更高网络效率
linger.ms=5            # 最多等 5ms 攒批 → 批次更满 → 吞吐更高
```

更大的批次意味着 Sender 每次网络请求携带更多数据，提高了出水效率。但注意 batch.size 增大后每个 partition 队列的内存占用也变大。

### 4. 限制重试时间和次数（防止内存被重试占死）

```properties
delivery.timeout.ms=120000   # 从 send 到收到 ACK 总上限 120s
retries=3                    # 限制重试次数（替代默认的 Integer.MAX_VALUE）
```

默认 retries 是 Integer.MAX_VALUE，如果 Broker 持续不可用，重试的批次会一直占用 buffer 内存不释放。设置合理的上限可以避免重试风暴。

### 5. 多 Producer 实例（当单个 Sender 线程成为瓶颈时）

如果单个 Sender 线程确实处理不过来，可以创建多个 KafkaProducer 实例，每个实例有自己的 buffer.memory 和 Sender 线程。应用线程按 topic 或 hash 分配到不同 Producer 上。代价是更多的内存占用和更多的 TCP 连接。

### 6. 排查根因：Broker/网络瓶颈

最终极的解法是让出水管快起来。检查方向：

- Broker 端 CPU/磁盘/网络是否达到瓶颈
- GC 考察（Broker 端 STW 停顿）
- 生产者到 Broker 的网络延迟和带宽
- Broker 端配置（num.replica.fetchers、queued.max.requests 等）

---

## 九、核心心智模型总结

把整个 Kafka Producer 想象成一个水箱系统：

```
你的代码 (多个水龙头)  →  buffer.memory (水箱, 32MB)  →  Sender (1根出水管)  →  Kafka Broker (下水道)

水箱满了 → 新龙头放不进水 → 等待 (max.block.ms) → 超时报错
```

所有调参手段都在围绕进水/出水均衡做文章：

- 压缩 → 减小进水单次体积
- 增大 buffer.memory → 增大水箱
- 调大 batch.size + linger.ms → 提升出水效率
- 限制 retries → 防止水卡在管子里
- 多 Producer → 增加出水管数量
- 排查 Broker → 让下水道通畅

参数调整的难易程度排序：

**compression > buffer.memory > batch.size + linger.ms > retries > 多 Producer > 排查 Broker**
