---
title: Брокеры сообщений
weight: 55
---

## Введение

Producer -> Exchange -> Queue -> Consumer

- Producer — отправляет сообщение.
- Exchange — принимает сообщение и определяет, в какие очереди его направить.
- Queue — хранит сообщения.
- Consumer — читает сообщения.
- Binding — связь между exchange и queue.
- Routing key — используется для маршрутизации.

Exchange types
- direct — точное совпадение routing key.
- topic — маршрутизация по шаблону:

### Вопросы

Что должен описать системный аналитик при проектировании API?
- endpoint;
- HTTP method;
- назначение операции;
- path/query parameters;
- request headers;
- request body;
- типы и обязательность полей;
- валидации;
- response;
- HTTP status codes;
- ошибки;
- авторизацию;
- примеры;
- ограничения;
- требования к идемпотентности.