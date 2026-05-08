---
# some information about your slides (markdown enabled)
title: Эффективная реализация анонимных функций в объектно-ориентированных языках

# apply unocss classes to the current slide
class: text-center

# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
---

# Эффективная реализация анонимных функций в объектно-ориентированных языках

### Болохононов Артем
### Научный руководитель: Трепаков Иван Сергеевич

---

# Анонимные функции
## В объектно-ориентированных языках программирования

В объектно-ориентированных языках лямбда-функции --- это абстракция над синтетическими классами.
````md magic-move
```scala
val capture = "arg = "

val lambda = { x => println(capture + x) }
lambda(42)
```
```scala
class Lambda$$1(val prefix: Str) extends (Int => Unit) {
  override def apply(x: Int) = {
    println(prefix + x)
  }
}

val capture = "arg = "
val lambda = new Lambda$$1(capture)
lambda(42)
```

````

<v-click>

Таким образом, анонимные функции в объектно-ориентированных языках --- это полноценный объект, который требует как аллокации, так и сборки мусора. Из-за чего возникают издержки по памяти и времени.

</v-click>

<SlideCurrentNo class="absolute right-40px bottom-30px"/>

---

# Постановка задачи
##

**Цель**: оптимизация изджержек представления анонимных функций в объектно-ориентированных языках программирования.

<v-click>

**Задачи**:

1. Изучить существующие подходы к уменьшению издержек на выделение объектов в куче,
2. Разработать подход к оптимизации издержек анонимных функций и функциональных замыканий,
3. Выполнить экспериментальную реализацию предложенного подхода,
4. Провести апробацию на представительном наборе тестов и приложений.

</v-click>

<SlideCurrentNo class="absolute right-40px bottom-30px"/>

---

# Оптимизации
## Анализ утеканий

```scala
class SomeObject(var a: Int, var b: Double, c: String)

val obj = SomeObject(42, 42.0, "42")
useField(obj.a)
useField(obj.b)
obj.c = "override"

if (someBoolean) {
  method(obj)
}

if (anotherBoolean) {
  return obj
}
```

---

# Оптимизации
## Анализ утеканий

<div class="grid grid-cols-2 gap-4">
<div>

````md magic-move
```scala
class SomeObject(var a: Int, var b: Double, c: String)

val obj = SomeObject(42, 42.0, "42")
useField(obj.a)
useField(obj.b)
obj.c = "override"
```
```scala
class SomeObject(var a: Int, var b: Double, c: String)

val (objA, objB, objC) = (42, 42.0, "42")
useField(objA)
useField(objB)
objC = "override"
```
````

<v-click>

* Данное преобразование может быть применено только на объектах, у которых нет использований по ссылке.

</v-click>

</div>
<div>

<!-- <svg" class="w-full h-full max-w-md mx-auto" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="arrow" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="currentColor" />
    </marker>
  </defs>

  <line x1="200" y1="75" x2="200" y2="160" stroke="currentColor" stroke-width="2" stroke-linecap="round" />
  <line x1="200" y1="220" x2="200" y2="300" stroke="currentColor" stroke-width="2" stroke-linecap="round" />

  <path d="M 120 50 Q 20 190 120 330" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" marker-end="url(#arrow)" />

  <text x="200" y="50" text-anchor="middle" fill="currentColor" font-size="8" font-style="italic" font-family="serif">скаляризованное</text>
  <text x="200" y="200" text-anchor="middle" fill="currentColor" font-size="8" font-style="italic" font-family="serif">на стеке</text>
  <text x="200" y="350" text-anchor="middle" fill="currentColor" font-size="8" font-style="italic" font-family="serif">в куче</text>
</svg> -->

<svg v-click="3" class="w-full h-full max-w-md mx-auto" xmlns="http://www.w3.org/2000/svg">
  <text x="200" y="50" text-anchor="middle" fill="currentColor" font-size="8" font-style="italic" font-family="serif">скаляризованное</text>
</svg>

</div>
</div>


<SlideCurrentNo class="absolute right-40px bottom-30px"/>

---

# Оптимизации
## Анализ утеканий

<div class="grid grid-cols-2 gap-4">
<div>

````md magic-move
```scala
class SomeObject(var a: Int, var b: Double, c: String)

val (objA, objB, objC) = (42, 42.0, "42")
useField(objA)
useField(objB)
objC = "override"
```
```scala
class SomeObject(var a: Int, var b: Double, c: String)

val (objA, objB, objC) = (42, 42.0, "42")
useField(objA)
useField(objB)
objC = "override"

if (someBoolean) {
  return obj  // ?
}
```
```scala
class SomeObject(var a: Int, var b: Double, c: String)

val (objA, objB, objC) = (42, 42.0, "42")
useField(objA)
useField(objB)
objC = "override"

if (someBoolean) {
  return rematerialize(objA, objB, objC)
}
```
````

</div>
<div>

<svg class="w-full h-full max-w-md mx-auto" xmlns="http://www.w3.org/2000/svg">
    <defs>
    <marker id="arrow" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="currentColor" />
    </marker>
  </defs>

  <path v-click="3" d="M 120 50 Q 20 190 120 330" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" marker-end="url(#arrow)" />

  <text x="200" y="50" text-anchor="middle" fill="currentColor" font-size="8" font-style="italic" font-family="serif">скаляризованное</text>
  <text v-click="3" x="200" y="350" text-anchor="middle" fill="currentColor" font-size="8" font-style="italic" font-family="serif">в куче</text>
</svg>

</div>
</div>

<SlideCurrentNo class="absolute right-40px bottom-30px"/>

---

# Оптимизации
## Анализ утеканий

<div class="grid grid-cols-2 gap-4">
<div>

````md magic-move
```scala
class SomeObject(var a: Int, var b: Double, c: String)

val (objA, objB, objC) = (42, 42.0, "42")
useField(objA)
useField(objB)
objC = "override"

if (someBoolean) {
  return rematerialize(objA, objB, objC)
}
```
```scala
class SomeObject(var a: Int, var b: Double, c: String)

val (objA, objB, objC) = (42, 42.0, "42")
useField(objA)
useField(objB)
objC = "override"

if (someBoolean) {
  return rematerialize(objA, objB, objC)
}

if (anotherBoolean) {
  method(obj)  // ?
}
```
```scala{*|*|*}
class SomeObject(var a: Int, var b: Double, c: String)

val (objA, objB, objC) = (42, 42.0, "42")
useField(objA)
useField(objB)
objC = "override"

if (someBoolean) {
  return rematerialize(objA, objB, objC)
}

if (anotherBoolean) {
  // метод не содержит утеканий
  method(reconstruct(obj))  // на стеке
}
```
```scala
class SomeObject(var a: Int, var b: Double, c: String)

val (objA, objB, objC) = (42, 42.0, "42")
useField(objA)
useField(objB)
objC = "override"

if (someBoolean) {
  return rematerialize(objA, objB, objC)
}

if (anotherBoolean) {
  // метод не содержит утеканий
  method(reconstruct(obj))  // на стеке --- некорректно
}
```
````

<v-click at="4">

```scala
def method(o: SomeObject) = {
  if (anotherBoolean2) {  // всегда false
    staticField = o
  }
}
```

</v-click>

</div>
<div>

<svg class="w-full h-full max-w-md mx-auto" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="arrow-2" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="currentColor" />
    </marker>
  </defs>

  <line v-click="3" x1="200" y1="75" x2="200" y2="160" stroke="currentColor" stroke-width="2" stroke-linecap="round" marker-end="url(#arrow-2)"/>
  <line v-click="6" x1="200" y1="220" x2="200" y2="300" stroke="currentColor" stroke-width="2" stroke-linecap="round" marker-end="url(#arrow-2)" stroke-dasharray="8 6"/>
  <path d="M 120 50 Q 20 190 120 330" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" marker-end="url(#arrow-2)" />

  <text x="200" y="50" text-anchor="middle" fill="currentColor" font-size="8" font-style="italic" font-family="serif">скаляризованное</text>
  <text v-click="3" x="200" y="200" text-anchor="middle" fill="currentColor" font-size="8" font-style="italic" font-family="serif">на стеке</text>
  <text x="200" y="350" text-anchor="middle" fill="currentColor" font-size="8" font-style="italic" font-family="serif">в куче</text>
</svg>

</div>
</div>

<SlideCurrentNo class="absolute right-40px bottom-30px"/>

---

# Межпроцедурный анализ частичных утеканий
## Идея

````md magic-move
```scala
def lambdaUse(x: (Int => Unit)) = {
  ...
  if (globalFlag) {  // Всегда == false
    globalVar = x
  }
  ...
}

val lambda = { x => println(x) }
lambdaUse(lambda)
```
```scala{4,9}
def lambdaUse(x: (Int => Unit)) = {
  ...
  if (globalFlag) {  // Всегда == false
    globalVar = evacuate(x)
  }
  ...
}

val lambda = onStack( { x => println(x) } )
lambdaUse(lambda)
```
````

<v-click>

* Будем перемещать объект на стек, если у него нет гарантированного *утекающего* использования,
* Если такие использования есть, то перед их исполнением будем вставлять операцию *эвакуация*, 
которая создаст копию объекта на куче.

</v-click>

<SlideCurrentNo class="absolute right-40px bottom-30px"/>

---

# Межпроцедурный анализ частичных утеканий
## Классификация использований

Утеканиями будем считать:

1. Возврат из метода;
2. Запись анализируемого объекта:
    - В поле другого объекта,
    - В качестве элемента массива,
    - В статическую переменную;

<v-click>

*3. Вызов функции, если для соответствующего параметра вычислилось, что он утекает.*

</v-click>

<SlideCurrentNo class="absolute right-40px bottom-30px"/>

---

# Межпроцедурный анализ частичных утеканий
## Оптимизации

1. Быстрые пути в эвакуации

````md magic-move
```scala
def evacuation[T](x: T): T = {
  ...  // системный вызов
}
```
```scala
def evacuation[T](x: T): T = {
  if (x == null) {
    return null
  } else if (isOnHeap(x)) {
    return x
  } else {
    ...  // системный вызов
  }
}
```
````

<SlideCurrentNo class="absolute right-40px bottom-30px"/>

---

# Межпроцедурный анализ частичных эвакуаций
## Оптимизации

2. Дополнительный стековый слот для сохранения результата

````md magic-move
```scala
val lambda = onStack( { x => println(x) } )
```
```scala
val cachedLambda = null 
val lambda = onStack( { x => println(x) } )
```
````

<v-click at="1">

```scala
def evacuation[T](x: T): T = {
  ...
  } else if (*(&x + 8) != null) {
    return *(&x + 8)
  } else {
    ...  // системный вызов

    *(&x + 8) = result
    return result
  }
}
```

</v-click>

<SlideCurrentNo class="absolute right-40px bottom-30px"/>

---

# Окружение
## Многоязыковая виртуальная машина Huawei

1. Оптимизирующий статический компилятор,
2. Управляемая среда исполнения,
3. Поддержка языков Java, Scala, Cangjie.

<style>
.aaa {
    width: 200px;
    object-fit: contain;

}
</style>


<div class="absolute right-41% bottom-80px">
<img src="/pics/Huawei.svg" class="aaa"/>
</div>

<SlideCurrentNo class="absolute right-40px bottom-30px"/>

---

# Результаты
## Статистика

Количество перемещений на стек; больше --- лучше

| Название теста            | Анализ утеканий | Межпроцедурный анализ частичных утеканий |
| :-----------------------: | :-------------: | :--------------------------------------: |
| scala-std                 | 103             | 397                                      |
| Виртуальная машина Huawei | 394             | 4488                                     |

<SlideCurrentNo class="absolute right-40px bottom-30px"/>

---

# Заключение
## 

В ходе работы было сделано:

1. Изучены существующие подходы к уменьшению издержек на выделение объектов в куче,
2. Разработана и реализована оптимизация «межпроцедурный анализ частичных утеканий»,
3. Полученное решение было протестировано.

<!--
1. Изучить существующие подходы к уменьшению издержек на выделение объектов в куче
2. Разработать подход к оптимизации издержек анонимных функций и функциональных замыканий
3. Выполнить экспериментальную реализацию предложенного подхода
4. Провести апробацию на представительном наборе тестов и приложений
-->

# Направления дальнейшей работы

1. Расширить анализ на другие типы, помимо лямбда-функций,
2. Добавить поддержку использования профилировочной информации.

<SlideCurrentNo class="absolute right-40px bottom-30px"/>

---
class: text-center
layout: cover

---

# Спасибо за внимание

---

# Анализ утеканий
## Алгоритм

<div class="grid grid-cols-2 gap-x-4 gap-y-0.5 w-full max-w-lg mx-auto text-sm">
  
  <!-- 1. Верхний узкий блок -->
  <div class="col-span-2 flex justify-center">
    <div class="w-1/2 h-15 flex items-center justify-center rounded">
      NoEscape
    </div>
  </div>

  <!-- Галочка V -->
  <div class="col-span-2 flex justify-center text-gray-400">
    <svg class="w-7 h-10" fill="none" stroke="currentColor" viewBox="0 0 24 24">
      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7"></path>
    </svg>
  </div>

  <!-- 2. Два блока в ряд -->
  <div class="h-15 flex items-center justify-center rounded">
    RetEscape
  </div>
  <div class="h-15 flex items-center justify-center rounded">
    RcvEscape
  </div>

  <!-- Галочка V -->
  <div class="col-span-2 flex justify-center text-gray-400">
    <svg class="w-7 h-10" fill="none" stroke="currentColor" viewBox="0 0 24 24">
      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7"></path>
    </svg>
  </div>

  <!-- 3. Широкий блок -->
  <div class="col-span-2 h-15 flex items-center justify-center rounded">
    RetRcvEscape
  </div>

  <!-- Галочка V -->
  <div class="col-span-2 flex justify-center text-gray-400">
    <svg class="w-7 h-10" fill="none" stroke="currentColor" viewBox="0 0 24 24">
      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7"></path>
    </svg>
  </div>

  <!-- 4. Широкий блок -->
  <div class="col-span-2 h-15 flex items-center justify-center rounded">
    GlobalEscape
  </div>

</div>

---


# Межпроцедурный анализ частичных эвакуаций
## Алгоритм

<div class="flex items-center justify-center gap-4 w-full max-w-4xl mx-auto text-sm">
  
  <!-- ЛЕВАЯ СХЕМА -->
  <div class="grid grid-cols-2 gap-x-2 gap-y-0 flex-1">
    <!-- 1. Верхний узкий блок -->
    <div class="col-span-2 flex justify-center">
      <div class="w-1/2 h-15 flex items-center justify-center">∅</div>
    </div>
    <!-- V -->
    <div class="col-span-2 flex justify-center text-gray-400">
      <svg class="w-7 h-10" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7"></path></svg>
    </div>
    <!-- 2. Два блока -->
    <div class="h-15 flex items-center justify-center">RetEscape</div>
    <div class="h-15 flex items-center justify-center">RcvEscape</div>
    <!-- V -->
    <div class="col-span-2 flex justify-center text-gray-400">
      <svg class="w-7 h-10" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7"></path></svg>
    </div>
    <!-- 3. Широкий блок -->
    <div class="col-span-2 h-15 flex items-center justify-center">RetRcvEscape</div>
  </div>

  <!-- КРЕСТИК (X) МЕЖДУ СХЕМАМИ -->
  <div class="text-2xl font-light pb-10">✕</div>

  <!-- ПРАВАЯ СХЕМА -->
  <div class="grid grid-cols-2 gap-x-2 gap-y-0 flex-1">
    <!-- 1. Два блока в ряд -->
    <div class="h-15 flex items-center justify-center">NoEscape</div>
    <div class="h-15 flex items-center justify-center">GlobalEscape</div>
    <!-- V -->
    <div class="col-span-2 flex justify-center text-gray-400">
      <svg class="w-7 h-10" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7"></path></svg>
    </div>
    <!-- 2. Широкий блок -->
    <div class="col-span-2 h-15 flex items-center justify-center">PartialEscape</div>
    <!-- Пустое место для выравнивания высоты с левой схемой, если нужно -->
  </div>

</div>