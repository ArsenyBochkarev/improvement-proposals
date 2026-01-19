---
layout: sip
number: NN
permalink: /sips/:number.html
redirect_from:
  - /sips/:title.html
  - /sips/:number
stage: implementation
status: waiting-for-implementation
presip-thread: https://contributors.scala-lang.org/t/pre-sip-foo-bar/9999
title: Breakable Methods for Standard Collections
---

**By: Arseny Bochkarev**

## History

| Date          | Version            |
|---------------|--------------------|
| Jan 19th 2026 | Initial Draft      |

## Summary

Данный proposal расширяет API стандартных коллекций (`List`, `Vector` и др.) семейством методов (`breakableForeach`, `breakableFind` и др.), которые явно могут использовать механизм `boundary`/`break` для прерывания цикла. Это устраняет необходимость объявления `boundary` для большого количества сценариев, что снижает число boilerplate кода. Также, в данном proposal'е описываются правила пересечений различных `break`, и небольшие изменения в поведении текущих `foreach`-методов стандартных коллекций.

## Motivation

В Scala 3 существует механизм `scala.util.boundary`, который обеспечивает безопасное прерывание циклов, однако, он требует некоторого количества boilerplate кода, который:
1. Может быть неудобен для расположения. Границы могут быть расположены глубоко ниже цикла, из которого `break` выходит.
2. Менее понятен программистам, пришедшим из императивных языков, где `break` зачастую может присутствовать как ключевое слово в языке, которое выходит из текущего ближайшего цикла.

Status quo пример c досрочным выходом из foreach цикла:

```scala
import scala.util.boundary, boundary.break

boundary: {
  files.foreach { item =>
    if (shouldStop(item)) break()
    else Continue
  }
}
```

## Proposed solution

Добавление к `trait IterableOnce` следующих методов:
- `def breakableForeach(f: A => Unit): Unit`
- `def breakableFind(f: A => Boolean): Option[A]`
- `def breakableExists(f: A => Boolean): Boolean`
- `def breakableContains[A1 >: A](elem: A1): Boolean`

Которые бы использовались в случае, если лямбда, переданная в foreach цикл, могла бы досрочно прерваться.

### High-level overview

Пример с досрочным выходом из foreach цикла, с изменениями из данного proposal'а:

~~~ scala
files.breakableForeach { item =>
  if (shouldStop(item)) break()
  else Continue
}
~~~

### Specification

1. Каждый добавленный метод неявно создаёт свою собственную `boundary`. Внутри лямбды, переданной в этот метод, `break` разрешён к использованию. Возможный пример реализации `breakableForeach`:

~~~ scala
def breakableForeach(f: A => Unit): Unit = boundary {
  for (elem <- this) f(elem)
}
~~~

2. Прерывания внутри breakable-методов должны быть локальны, т.е. прерывать только текущий цикл. Это делает их семантически эквивалентыми использованию ключевого слова `break` в языках C/C++.

3. Запрет на использование `break` внутри не-breakable версий данных методов: это должно приводить к **ошибке компиляции**.

### Compatibility

1. Binary compatibility: присутствует. Данный proposal не вводит runtime-исключений на `break` внутри цикла для байткода, который уже был скомпилирован
2. TASTy compatibility: присутствует. Данный proposal не вводит runtime-исключений на `break` внутри цикла для TASTy файлов, который уже были порождены компилятором

Семантика существующих корректных скомпилированных программ не изменяется, однако файлы, ранее принимаемые компиляторами, должны быть переписаны: foreach циклы, в которых использовался `break`, должны быть заменены на их breakable-версии, иначе компилятор их не примет.

### Feature Interactions

Правила пересечений с другими `break`:
  - Вложенные breakable-методы: прерваться должен только внутренний цикл
  - Взаимодействие с `break` внешней границы: сработает только `break` из внутреннего цикла. Это делает невозможным использовать `break` изнутри цикла, чтобы прервать исполнение секции кода, превышающей границы самого цикла
  - Взаимодействие с `boundary`/`break` внутри цикла: если внутри цикла есть проставленная пользователем ещё одна `boundary` зона, то это не должно мешать компиляции. В этом случае взаимодействие интуитивно, `break` действует так же, как и вне циклов: выход из пользовательской границы и продолжение итераций
  - `break` внутри замыканий: не должен работать для breakable версий методов

### Other concerns

1. В данном варианте не предусмотрена возможность возвращать значение из `break`
2. Основное преимущество при данном изменении будет на стороне пользователей, недавно пришедших из императивных языков. Но, поскольку они новички в языке, они могут просто не знать про данные конструкции

## Alternatives

#### Warning при `break` для не-breakable методов
Плюсы:
- Полностью сохраняет совместимость. С этим старый код не обязательно будет переписывать, чтобы перекомпилировать

Минусы:
- При реализации потеря в предположении о невозможности встречи `break`, что может сказаться на производительности

## Related work
- [Deprecation обычных `break`](https://github.com/scala/scala-lang/blob/main/blog/_posts/2023-05-30-scala-3.3.0-released.md#boundary-and-break)
- [Почему `break` и `continue` появились в Scala изначально](https://softwareengineering.stackexchange.com/a/260022)

