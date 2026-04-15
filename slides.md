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

<div class="grid grid-cols-2 gap-4">
<div class="col-span-2 rows-span-2">

````md magic-move
```scala
class SomeObject(var a: Int, var b: Double, c: String)

val obj = SomeObject(42, 42.0, "42")
```
```scala
class SomeObject(var a: Int, var b: Double, c: String)

val obj = SomeObject(42, 42.0, "42")
useField(obj.a)
useField(obj.b)
useField(obj.c)
```
```scala
class SomeObject(var a: Int, var b: Double, c: String)

val (objA, objB, objC) = (42, 42.0, "42")
useField(objA)
useField(objB)
useField(objC)
```
```scala
class SomeObject(var a: Int, var b: Double, c: String)

val (objA, objB, objC) = (42, 42.0, "42")
useField(objA)
useField(objB)
useField(objC)

useObject(obj)
```
```scala{*|*|*}
class SomeObject(var a: Int, var b: Double, c: String)

val (objA, objB, objC) = (42, 42.0, "42")
useField(objA)
useField(objB)
useField(objC)

useObject(rematerialize(objA, objB, objC))
```
```scala
class SomeObject(var a: Int, var b: Double, c: String)

{
  val (objA, objB, objC) = (42, 42.0, "42")
  useField(objA)
  useField(objB)
  useField(objC)

  useObject(rematerialize(objA, objB, objC))
  return obj  // ?
}
```
````

</div>

<div class="rows-span-1">

<v-click at="5">

```scala
def useObject(x: SomeObject) = {
  globalField = x
  otherObject.x = x
  array.add(x)
}
```

</v-click>
</div>

<div class="rows-span-1">
<v-click at="6">

```scala
def useObject(x: SomeObject)  // ???
⠀
⠀
⠀
⠀
```

</v-click>
</div>

</div>

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

1. Изучить существующие подходы
2. Изучить текущую реализацию в компиляторе **Huawei VM**
3. Разработать и реализовать оптимизацию
4. Протестировать и замерить результаты

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

# Анализ утеканий
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