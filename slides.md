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

<v-click at="1">

```scala
val even = { x: Int => x % 2 == 0 }
```

</v-click>

<br>
<br>
<br>

<v-click at="2">

```scala
val m: Int = 5
val leq = { x => x < m }

...

leq(x) // == { x < 5 }
```

</v-click>

<v-click at="3">
```scala
val collection: Seq[Int] = ...
collection.filter (x    => x % 2)
collection.sort   (x, y => x > y)
collection.foreach(x    => println(x))
collection.map    (x    => x * x)
...
```
</v-click>

</div>
</div>

<SlideCurrentNo class="absolute right-40px bottom-30px"/>

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

**Цель**: оптимизация изджержек представления анонимных функций в объектно-ориентированных языках программирования

<v-click>

**Задачи**:

1. Изучить существующие подходы к уменьшению издержек на выделение объектов в куче
2. Разработать подход к оптимизации издержек анонимных функций и функциональных замыканий
3. Выполнить экспериментальную реализацию предложенного подхода
4. Провести апробацию на представительном наборе тестов и приложений

</v-click>

<SlideCurrentNo class="absolute right-40px bottom-30px"/>

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

<SlideCurrentNo class="absolute right-40px bottom-30px"/>

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

<SlideCurrentNo class="absolute right-40px bottom-30px"/>

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

* Оптимистичный статический анализ

</v-click>


<v-click>

* Однако обладает издержками на дополнительную метаинформацию

</v-click>

<SlideCurrentNo class="absolute right-40px bottom-30px"/>

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

<SlideCurrentNo class="absolute right-40px bottom-30px"/>

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

<SlideCurrentNo class="absolute right-40px bottom-30px"/>

---

# Межпроцедурный анализ частичных эвакуаций
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

1. Оптимизирующий статический компилятор
2. Управляемая среда исполнения
3. Поддержка языков Java, Scala, Cangjie

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

| Название теста | Анализ утеканий | МАЧУ |
| :------------: | :-------------: | :--: |
| scala-std      | 463 + 103         | 397  |
| Виртуальная машина Huawei | 799 + 394 | 4488 |

<SlideCurrentNo class="absolute right-40px bottom-30px"/>

---

# Заключение
## 

В ходе работы было сделано:

1. Изучены существующие подходы, а также существующую реализацию в Huawei VM
2. Разработано и реализовано оптимизация межпроцедурный анализ частичных эвакуаций
3. Полученное решение было протестировано

# Направления дальнейшей работы

1. Расширить анализ на другие типы, помимо лямбда-функций
2. Добавить поддержку использования профилировочной информации

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