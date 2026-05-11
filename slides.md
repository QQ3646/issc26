---
# some information about your slides (markdown enabled)
title: Эффективная реализация анонимных функций в объектно-ориентированных языках

# apply unocss classes to the current slide
class: text-center

# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
---

# Эффективная реализация анонимных функций в объектно-ориентированных языках

### Болохононов Артем Владимирович, гр. 24152
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
val capture = "arg = "
val lambda = new Lambda$$1(capture)
lambda(42)

class Lambda$$1(val prefix: Str) extends (Int => Unit) {
  override def apply(x: Int) = {
    println(prefix + x)
  }
}
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

<SlideCurrentNo class="absolute right-40px bottom-30px"/>

---

# Оптимизации
## Анализ утеканий

<div class="grid grid-cols-2 gap-4" style="height: 80%;">
<div>

````md magic-move
```scala
class SomeObject(var a: Int, var b: Double, c: String)

val obj = SomeObject(42, 42.0, "42")
useField(obj.a)
useField(obj.b)
obj.c = "override"
```
```scala{*|*}
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
```scala{*|*}
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

<v-click at="9">

```scala
def method(o: SomeObject) = {
  if (anotherBoolean2) {  // очень редко
    staticField = o
  }
}
```

</v-click>

</div>
<div>

<svg class="w-full h-full max-w-md mx-auto" xmlns="http://www.w3.org/2000/svg" style="height: 100%; min-height: 100%; display: block;">
    <defs>
    <marker viewBox="0 0 10 10" id="arrow-2" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse" >
      <path d="M 0 0 L 10 5 L 0 10 z" fill="currentColor" />
    </marker>
  </defs>

  <line v-click="8" x1="200" y1="75" x2="200" y2="165" stroke="currentColor" stroke-width="2" stroke-linecap="round" marker-end="url(#arrow-2)"/>
  <line v-click="11" x1="200" y1="225" x2="200" y2="315" stroke="currentColor" stroke-width="2" stroke-linecap="round" marker-end="url(#arrow-2)" stroke-dasharray="8 6"/>
  <path v-click="5" d="M 120 75 Q 20 190 120 315" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" marker-end="url(#arrow-2)" />

  <text v-click="2" x="200" y="50" text-anchor="middle" fill="currentColor" font-size="8" font-style="italic" font-family="serif">скаляризованное</text>
  <text v-click="8" x="200" y="200" text-anchor="middle" fill="currentColor" font-size="8" font-style="italic" font-family="serif">на стеке</text>
  <text v-click="5" x="200" y="350" text-anchor="middle" fill="currentColor" font-size="8" font-style="italic" font-family="serif">в куче</text>
</svg>

</div>
</div>

<SlideCurrentNo class="absolute right-40px bottom-30px"/>

---

# Межпроцедурный анализ частичных утеканий
## Идея

````md magic-move
```scala
val lambda = { x => println(x) }
lambdaUse(lambda)

def lambdaUse(x: (Int => Unit)) = {
  ...
  if (globalFlag) {  // Часто false
    globalVar = x
  }
  ...
}
```
```scala{1,7}
val lambda = onStack( { x => println(x) } )
lambdaUse(lambda)

def lambdaUse(x: (Int => Unit)) = {
  ...
  if (globalFlag) {  // Часто false
    globalVar = evacuate(x)
  }
  ...
}
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
## Алгоритм

Данный анализ является расширением анализа утеканий и является потоковым анализом.

## Межпроцедурная часть

<style>
    .box {
        margin-top: 30px; /* отступ сверху 30px */
        padding: 10px;
    }
</style>

<div class="box">
<img src="/pics/drawing_old.svg" class="aaa"/>
</div>

<div class="box" v-click>
<img src="/pics/drawing_new_upd.svg" class="aaa"/>
</div>

<SlideCurrentNo class="absolute right-40px bottom-30px"/>

---

# Межпроцедурный анализ частичных утеканий
## Локальная часть

<style>
    .box-2 {
        width: 300px;
        object-fit: contain;
        margin-top: 30px; /* отступ сверху 30px */
        padding: 10px;
    }
</style>

<div class="absolute right-40% bottom-40px">
<img src="/pics/cfg_upd.svg" class="box-2"/>
</div>


<SlideCurrentNo class="absolute right-40px bottom-30px"/>

---

# Межпроцедурный анализ частичных утеканий
## Локальная часть

<style>
    .box-3 {
        width: 300px;
        object-fit: contain;
        margin-top: 30px; /* отступ сверху 30px */
        padding: 10px;
    }
</style>

<div class="absolute right-40% bottom-40px">
<img src="/pics/cfg_2_upd.svg" class="box-3"/>
</div>


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
```asm
    call "EVACUATION"
```
```asm
    cmp   object, 0        ; если объект == null
    jne   else
    mov   ecx, 0           ; возвращаем null
    jmp   continue
else:
    mov   eax, [object]
    test  eax, eax         ; если объект уже на куче
    jne   else_if
    mov   ecx, object      ; возвращаем его же
    jmp   continue
else_if:
    ALLOCATE_AND_COPY      ; иначе копируем
continue:
    ...
```
````

<SlideCurrentNo class="absolute right-40px bottom-30px"/>

---

# Межпроцедурный анализ частичных эвакуаций
## Оптимизации

2. Дополнительный стековый слот для сохранения результата

<div class="grid grid-cols-2 gap-4" style="height: 80%;">
<div>

````md magic-move
```asm{*|*}{at:0}
    cmp   object, 0        ; если объект == null
    jne   else
    mov   ecx, 0           ; возвращаем null
    jmp   continue
else:
    mov   eax, [object]
    test  eax, eax         ; если объект уже на куче
    jne   else_if
    mov   ecx, object      ; возвращаем его же
    jmp   continue
else_if:
    ALLOCATE_AND_COPY      ; иначе копируем
continue:
    ...
```
```asm{8,11-16}
    cmp   object, 0        ; если объект == null
    jne   else
    mov   ecx, 0           ; возвращаем null
    jmp   continue
else:
    mov   eax, [object]
    test  eax, eax         ; если объект уже на куче
    jne   get_from_cache
    mov   ecx, object      ; возвращаем его же
    jmp   continue
get_from_cache:
    mov   eax, [object-8]  ; получаем значение из кэша
    test  eax, eax         ; проверяем наличие объекта
    jne   else_if
    mov   ecx, eax         ; возвращаем из кэша
    jmp   continue 
else_if:
    ALLOCATE_AND_COPY      ; иначе копируем
continue:
    ...
```
````

</div>
<div>


````md magic-move{at:1}
```asm
-------------- 
->  HEADER
    FIELD_1
    FIELD_2
    FIELD_3
    ...
--------------
```
```asm
-------------- 
    ALLOCATED_SPACE (== nullptr)
->  HEADER
    FIELD_1
    FIELD_2
    FIELD_3
    ...
--------------
```
````

<v-click at="2">

```scala
def evacuation[T](x: T): T = {
  val result = ...
  (&x - 8) = result
  return result 
}
```

</v-click>


</div>
</div>


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
| Статический компилятор    | 394             | 4488                                     |

---

# Результаты
## Статистика

Количество выделений объектов помеченных типов; меньше --- лучше

| Название теста            | Анализ утеканий | Межпроцедурный анализ частичных утеканий |
| :-----------------------: | :-------------: | :--------------------------------------: |
| Компиляция Hello World    | 5'058'176       | 3'717'185                                |


<SlideCurrentNo class="absolute right-40px bottom-30px"/>

---

# Заключение
## 

В ходе работы было сделано:

1. Изучены существующие подходы к уменьшению издержек на выделение объектов в куче,
2. Разработано и реализовано расширение анализа утеканий «межпроцедурный анализ частичных утеканий»,
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