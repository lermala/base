---
title: RML
weight: 30
---

## Введение
> Визуализация – это возможность преодолеть проблему ограниченности нашей краткосрочной памяти.

Полезные ссылки:
- Первоисточник RML (только на англ.) [Visual Models for Software Requirements. Joy Beatty and Anthony Chen][rml]

[rml]: https://www.oreilly.com/library/view/visual-models-for/9780735667730/

## Что это?

**Requirements Modeling Language (RML)** - язык, разработанный специально для визуального моделирования требований. Подробнее о нем можно прочитать в книге [Visual Models for Software Requirements][rml].

При этом RML **не заменяет текстовое описание требований**, а лишь дополняет.

## Категории RML

Модели RML делят на 4 категории, которые суммарно называют **OPSD** (Objectives - цели, People - люди, Systems - система, Data - данные).

Подробнее о составе категорий:

```mermaid
%%{init:{
  "theme": "base", 
  "look": "handDrawn", 
  "handDrawnSeed": 5,
  "flowchart":{"nodeSpacing":10,"rankSpacing":15}
}}%%
flowchart TD
  
  Root("Виды RML моделей")
  
  %% === МОДЕЛИ ЦЕЛЕЙ ===
  subgraph Goals[" "]
    G1(["Модель бизнес-целей"])
    G2(["Цепочка целей</br>(Objective Chain)"])
    G3(["Модель KPI"])
    G4(["Дерево функций</br>(Feature Tree)"])
    G5(["Матрица соответствия требований (Requirements Mapping Matrix)"])
  end

  %% === МОДЕЛИ ЛЮДЕЙ ===
  subgraph People[" "]
    P1(["Оргструктура (Org Chart)"])
    P2(["Бизнес-процесс</br>(Process Flow)"])
    P3(["Варианты использования</br>(Use Case)"])
    P4(["Матрица ролей и прав</br>(Roles & Permissions Matrix)"])
  end

  %% === МОДЕЛИ СИСТЕМ ===
  subgraph Systems[" "]
    S1(["Карта экосистемы</br>(Ecosystem Map)"])
    S2(["Поток системы</br>(System Flow)"])
    S3(["Поток интерфейсов</br>(User Interface Flow)"])
    S4(["Display–Action–Response"])
    S5(["Таблица решений</br>(Decision Table)"])
    S6(["Дерево решений</br>(Decision Tree)"])
    S7(["Таблица интерфейсов системы"])
  end

  %% === МОДЕЛИ ДАННЫХ ===
  subgraph Data[" "]
    D1(["Бизнес-данные</br>(Business Data Diagram)"])
    D2(["Диаграмма потоков данных</br>(Data Flow Diagram)"])
    D3(["Словарь данных</br>(Data Dictionary)"])
    D4(["Таблица состояний</br>(State Table)"])
    D5(["Диаграмма состояний</br>(State Diagram)"])
    D6(["Таблица отчетов</br>(Report Table)"])
  end

  %% === ИЕРАРХИЯ ===
  Root -- "Модели целей" --> Goals
  Root -- "Модели людей" --> People
  Root -- "Модели системы" --> Systems
  Root -- "Модели данных" --> Data

  %% === ССЫЛКИ ===
  click G1 "/models/goals/business-objectives" _blank
  click S3 "/docs/30_formalization/uml/" _blank
```

{{< callout >}}
  Если блок подчеркнут, то **это ссылка**. Кликаем на него для подробной инфо.
{{< /callout >}}