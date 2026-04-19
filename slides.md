---
# some information about your slides (markdown enabled)
title: Эффективная реализация лямбда-функций в объектно-ориентированных языках программирования

# apply unocss classes to the current slide
class: text-center

# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
---

# Эффективная реализация лямбда-функций в объектно-ориентированных языках программирования

### Болохононов Артем
### Научный руководитель: Трепаков Иван Сергеевич

---

# Анонимные функции
## Определение

<div class="grid grid-cols-2 gap-4">

<div>

<v-click at="1">

* *Анонимные функции* (или *лямбда-функции*) --- функции, которые создаются по ходу программы и не имеют
уникального индентификатора,
<!-- концепция, пришедшая из языков функциональной парадигмы.
Она позволяет по ходу программы создавать функции, которые не имеют уникального индентификатора. -->

</v-click>

<v-click at="2">

* Анонимные функции могут замыкать окружающий контекст, в таком случае они называются *функциональными замыканиями*,

</v-click>
<br>
<v-click at="3">

* Сейчас в объектно-ориентированных языках программирования они используются повсеместно, часто используются при работе с коллекциями.

</v-click>

</div>
<div>
````md magic-move
```scala
def filterEven(x: Int): Bool = {
  x % 2 == 0
}
```
```scala
val filerEvenLambda: Int => Bool = { x: Int => x % 2 }
```
````
<br>
<br>
<br>

<v-click at="2">

```scala
val capture: Int = 5
val functionalCaputre: => Int = { capture }

...

functionalCapture() // == 5
```

</v-click>

<v-click at="3">
```scala
val collection: Seq[Int] = ...
collection.filter(x => x % 2)
collection.sort(x, y => x > y)
collection.foreach(x => println(x))
collection.map(x => x * x)
...
```
</v-click>

</div>
</div>
---

# Анонимные функции
## В объектно-ориентированных языках программирования

В объектно-ориентированных языках лямбда-функции --- это абстракция над синтетическими классами. Рассмотрим пример:
````md magic-move
```scala
val lambda = { x => println(x) }
lambda(42)
```
```scala{1-5}
class Lambda$$1 extends (Int => Unit) {
  override def apply(x: Int) = {
    println(x)
  }
}

val lambda = { x => println(x) }
lambda(42)
```
```scala{7}
class Lambda$$1 extends (Int => Unit) {
  override def apply(x: Int) = {
    println(x)
  }
}

val lambda = new Lambda$$1()
lambda(42)
```
```scala
val capture = "arg = "

</d
val lambda = { x => println(capture + x) }
lambda(42)
```
```scala{1,7,8}
class Lambda$$1(val prefix: Str) extends (Int => Unit) {
  override def apply(x: Int) = {
    println(prefix + x)
  }
}

val capture = "arg = "
val lambda = new Lambda$$1(capture)
lambda(42)
```

</d
````

---

# Постановка задачи
## Проблема

Каждая из анонимных функций в условиях *JVM* (*Java Virtual Machine*) --- это полноценный объект, который требует как аллокации, так и сборки мусора. 
Из-за чего возникают издержки по памяти и времени.

<v-click>
Безусловно, существует набор оптимизаций, которые могут помочь в избавлении от создания объектов в куче.
</v-click>

---

# Оптимизации
## Анализ утеканий

<v-click>

1. Скаляризация объектов

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

</v-click>

<v-click>

* Данное преобразование может быть применено только на объектах, у которых нет использований по ссылке

</v-click>

---

# Оптимизации
## Анализ утеканий

2. Создание объектов на стеке

````md magic-move
```scala
class SomeObject(var a: Int, var b: Double, c: String)

val obj = SomeObject(42, 42.0, "42")
useField(obj.a)
useField(obj.b)
obj.c = "override"

useObject(obj)
```
```scala
class SomeObject(var a: Int, var b: Double, c: String)

val obj = onStack(SomeObject(42, 42.0, "42"))
useField(obj.a)
useField(obj.b)
obj.c = "override"

useObject(obj)
```
````

<v-click>

* Можно применить, если доказанно, что объект существует только в пределах своего стекового кадра 

</v-click>

---

# Оптимизации
## Анализ эвакуаций

```scala
class SomeObject(var a: Int, var b: Double, c: String)

val obj = onStack(SomeObject(42, 42.0, "42"))
useField(obj.a)
useField(obj.b)
obj.c = "override"

otherObject.field = obj  // otherObject также находится на стеке
useObject(obj)

...

staticField = heapification(otherObject)
```

<v-click>

* Мощный оптимистичный статический анализ, который обходит весь граф вызовов

</v-click>


<v-click>

* Однако обладает большими издержками: увеличение времени компиляции до 300 раз

</v-click>

---

# Оптимизации
## Стратегии

1. Анализ утеканий
    - Скаляризация
        - Локальный анализ
        - У объекта отсутствуют использования по ссылке
    - Размещение на стеке
        - Локальный и межпроцедурный анализ
        - У объекта отсутствуют утекания во всем графе вызовов
2. Анализ эвакуаций
    - Большое количество перемещаемых на стек объектов
    - Требуются эвакуации --- операция глобального рекурсивного перемещения на кучу
    - Статический анализ занимает много времени, а также требуется дополнительная метаинформация о расположении объектов на стеке

---

# Тестовый стенд
## Huawei VM

<style>
.aaa {
    width: 300px;
    height: 300px;
    object-fit: contain;

}
</style>


<div class="absolute right-35% bottom-110px">
<img src="/pics/Huawei.svg" class="aaa"/>
</div>

---

# Проблематика
## Цель и задачи

**Цель**: разработать и реализовать оптимизацию лямбда-функций на основе анализа утеканий, которая уменьшит потребление
памяти за счет увеличения количества размещений анонимных функций на стек, 

<v-click>

**Задачи**:

1. Изучить существующие подходы, а также существующую реализацию в Huawei VM
2. Разработать и реализовать оптимизацию
3. Протестировать полученное решение

</v-click>

---

# Межпроцедурный анализ частичных эвакуаций
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

* Будем перемещать объект на стек, если у него нет гарантированного *утекающего* использования
* Если такие использования есть, то перед их исполнением будем вставлять операцию *эвакуация*, которая создаст копию 
объекта на куче

</v-click>

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

---

# Межпроцедурный анализ частичных эвакуаций
## Алгоритм

* Алгоритм рассматривает только *помеченные* типы, в данной работе в качестве помеченных типов рассматриваются только лямбда-функции и функциональные интерфейсы

<v-click>

*Теорема 1.* Анализ остается корректным при любой конфигурации помеченных типов

</v-click>

---

# Межпроцедурный анализ частичных эвакуаций
## Оптимизации

1. Быстрые пути в эвакуации

````md magic-move
```scala
def evacuation[T](x: T): T = {
  ...  // рантайм-вызов
}
```
```scala
def evacuation[T](x: T): T = {
  if (x == null) {
    return null
  } else if (isOnHeap(x)) {
    return x
  } else {
    ...  // рантайм-вызов
  }
}
```
````

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
    ...  // рантайм-вызов

    *(&x + 8) = result
    return result
  }
}
```

</v-click>

---

# Результаты
## Статистика

...

---

# Результаты
## Поставленные задачи

В ходе работы было сделано:

1. Изучены существующие подходы, а также существующую реализацию в Huawei VM
2. Разработано и реализовано оптимизация межпроцедурный анализ частичных эвакуаций
3. Полученное решение было протестировано