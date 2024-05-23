---
title: "Generic Concurrency в Go"
date: 2024-05-23T18:00:00+00:00
draft: false
image: generic-concurrency-in-go.jpg
imageWidth: 600
description: "Дженерики и горутины (и итераторы в будущем) -- это отличные инструменты, которые мы можем использовать для параллельной обработки данных в общем виде."
---
{{< img src="generic-concurrency-in-go.jpg" width="600" alt="Похожий на осьминога гофер задумчиво смотрит вверх" >}}

Привет, гоферы!

В этой статье я хочу поделиться своими мыслями и идеями, которые у меня накопились за время работы с дженериками в Go, и в частности о том, как шаблоны многозадачности (concurrency) могут стать намного более удобными и переиспользуемыми с помощью дженериков. 

## TL;DR

Дженерики и горутины (и итераторы в будущем) -- это отличные инструменты, которые мы можем использовать для параллельной обработки данных в общем виде.

В этой статье мы рассмотрим возможности их совместного использования.

## Вступление

Давайте сформируем контекст и рассмотрим несколько простых примеров, чтобы увидеть, какую проблему решают дженерики и как мы можем встроить их в существующую модель многозадачности в Go.

> В этой статье мы будем говорить об отображении (`map()`) коллекций или последовательностей элементов. Таким образом, _отображение_ -- это процесс, который приводит к формированию новой коллекции элементов, где каждый элемент является результатом вызова некоторой функции `f()` с соответствующим элементом из исходной коллекции.

### Эра Pre-Generics

Давайте рассмотрим простую функцию отображения целочисленных чисел (которую в коде Go мы будем называть `transform()`, чтобы избежать путанницы со встроенным типом `map`):

```go
func transform([]int, func(int) int) []int
```

{{<details summary="Пример реализации">}} 
```go
func transform(xs []int, f func(int) int) []int {
    ret := make([]int, len(xs))
    for i, x := range xs {
        ret[i] = f(x)
    }
    return ret
}
```
{{</details>}}

Пример использования такой функции будет выглядеть так:

```go
// Output: [1, 4, 9]
transform([]int{1, 2, 3}, func(n int) int {
    return n * n
})
```

Предположим, что теперь мы хотим отобразить целые числа в строки. Чтобы это сделать, мы просто можем определить `transform()` _немного_ иначе:

```go
func transform([]int, func(int) string) []string
```

Теперь мы можем использовать это следующим образом:

```go
// Output: ["1", "2", "3"]
transform([]int{1, 2, 3}, strconv.Itoa) 
```

Как насчет другой функции, которая будет возвращать признак четности числа? Просто _еще одна небольшая корректировка_:

```go
func transform([]int, func(int) bool) []bool
```

Теперь использование функции будет выглядеть так:

```go
// Output: [false, true, false]
transform([]int{1, 2, 3}, func(n int) bool {
    return n % 2 == 0
})
```

_Обобщая_ изменения в реализации `transform()`, которые мы сделали выше для каждого конкретного случая использования, можно сказать, что **независимо** от типов, с которыми работает функция, она делает ровно то же самое снова и снова. Если бы мы хотели сгенерировать код для каждого типа используя шаблоны `text/template`, мы могли бы сделать это так:

```go
func transform_{{ .A }}_{{ .B }}([]{{ .A }}, func({{ .A }}) {{ .B }}) []{{ .B }}

// transform_int_int([]int, func(int) int) []int
// transform_int_string([]int, func(int) string) []string
// transform_int_bool([]int, func(int) bool) []bool
```

> Раньше подобные шаблоны действительно использовались для генерации "generic" кода. Проект [genny][genny] -- один из примеров.

### Эра Generics

Благодаря дженерикам, теперь мы можем _параметризовать_ функции и типы с помощью _параметров типа_ и определить `transform()` следующим образом:

```go
func transform[A, B any]([]A, func(A) B) []B
```

{{<details summary="И реализация изменится совсем чуть-чуть!">}} 
```go
func transform[A, B any](xs []A, f func(A) B) []B {
    ret := make([]B, len(xs))
    for i, x := range xs {
        ret[i] = f(x)
    }
    return ret
}
```
{{</details>}}

Теперь мы можем использовать `transform()` для любых типов аргументов и результатов (предполагается, что `square(int) int` и `isEven(int) bool` определены где-то выше в пакете):

```go
transform([]int{1, 2, 3}, square)       // [1, 4, 9]
transform([]int{1, 2, 3}, strconv.Itoa) // ["1", "2", "3"]
transform([]int{1, 2, 3}, isEven)       // [false, true, false]
```

## Параллельное отображение

Давайте перейдем к основной теме этой статьи и рассмотрим шаблоны многозадачности, которые мы можем использовать совместно с дженериками, и которые станут только лучше от такого использования.

### Пакет `x/sync/errgroup`

Прежде чем погрузиться в код, давайте немного отвлечемся и взглянем на очень популярную в Go библиотеку [golang.org/x/sync/errgroup][errgroup]. Вкратце, она позволяет вам запускать различное количество горутин для выполнения разных задач и ожидать их завершения или ошибки.

Предполагается, что библиотека используется следующим образом:
```go
// Create workers group and a context which will get canceled if any of the
// tasks fails.
g, gctx := errgroup.WithContext(ctx)
g.Go(func() error {
	return doSomeFun(gctx)
})
g.Go(func() error {
	return doEvenMoreFun(gctx)
})
if err := g.Wait(); err != nil {
	// handle error
}
```

Причина, по которой я упомянул `errgroup` в том, что если посмотреть на то, что делает библиотека с немного другой и более общей точки зрения, то мы увидим, что по сути, она является тем же механизмом отображения. `errgroup` _параллельно отображает_ набор задач (функций) с результатом их выполнения, а так же предоставляет обобщенный способ обработки и "всплытия" ошибок c прерыванием уже выполняющихся задач (через отмену [`context.Context`][context]).

В этой статье мы хотим создать что-то подобное, и, как намекает частое использование слов "общий" и "обобщенный", мы собираемся сделать это в общем виде.

### Наивная реализация

Возвращаясь к функции `transform()`. Допустим, что все вызовы `f()` могут выполняться параллельно, не ломая выполнение нашей программы. Тогда мы можем начать с этой наивной _многозадачной_ реализации:

{{<func file="./src/async/async.go" name="transform_00" next="transform" lang="go">}}

Мы запускаем горутину для каждого входного элемента и вызываем `f(elem)`. Затем мы сохраняем результат по соответствующему индексу в "общем" слайсе `bs`. Никакого контекста, прерывания, или ошибок -- такая реализация не кажется полезной в чем-либо, кроме простых вычислений.

### Отмена контекста

В реальном мире большинство проблем требующих многозадачности, особенно те, которые включают работу с i/o, будут контролироваться экземпляром [`context.Context`][context]. Если есть контекст, то может быть его тайм-аут или отмена. Давайте посмотрим на это следующим образом (здесь и далее я буду выделять строчки кода, которые были добавлены по сравнению с предыдущей версией):

{{<func file="./src/async/async.go" name="transform_01" next="transform" lang="go" class="focus" hl_lines="2 4 10 12-13 20-23 28-32">}}

Теперь у нас есть еще один "общий" слайс `es` для хранения потенциальных ошибок `f()`. Если какой-либо вызов `f()` завершается ошибкой, мы отменяем контекст всего вызова `transform()` и ожидаем, что каждый другой уже выполняющийся вызов `f()` учитывает отмену контекста и, если она происходит, прерывает выполнение как можно скорее.

### Limiting concurrency

На самом деле, мы не можем слишком много предполагать о `f()`. Пользователи `transform()` могут захотеть ограничить количество одновременных вызовов `f()`. Например, `f()` может отображать url на результат http запроса. Без каких-либо ограничений мы можем перегрузить сервер или забанить самих себя.

Давайте пока не будем думать о структуре параметров и просто добавим аргумент параллелизма `int` в аргументы функции.

На этом этапе нам нужно перейти от использования `sync.WaitGroup` к [семафору][semaphore] `chan`, поскольку мы хотим контролировать (максимальное) количество одновременно выполняющихся горутин, а также обрабатывать отмену контекста, используя `select`.

{{<func file="./src/async/async.go" name="transform_02" next="transform" lang="go" class="focus" hl_lines="13-14 17-19 21-36 38-42 49-64">}}

> Для этой и последующих итераций `tranform()` мы могли бы оставить реализацию в том виде, в котором она сейчас есть, и оставить рассматриваемые проблемы на откуп реализации `f()`. Например, мы могли бы просто запустить `N` горутин вне зависимости от ограничений многозадачности и позволить пользователю `transform()` частично сериализовать их так, как он хочет. Это потребовало бы накладных расходов на запуск `N` горутин вместо `P` (где `P` -- это ограничение "параллелизма", которое может быть гораздо меньше, чем `N`). Это также подразумевало бы некоторые накладные расходы на синхронизацию горутин, в зависимости от используемого механизма. Поскольку все это излишне, мы продолжаем реализацию the hard way, но во многих случаях эти усложнения являются необязательными.
> {{<details summary="Пример реализации на стороне пользователя">}} 
```go
// Initialised x/time/rate.Limiter instance.
var lim *rate.Limiter
transform(ctx, as, func(_ context.Context, url string) (int, error) {
    if err := lim.Wait(ctx); err != nil {
        return 0, err
    }

    // Process url.

    return 42, nil
})
```
{{</details>}}

### Переиспользование горутин

В предыдущей итерации мы запускали горутины для каждой задачи, но не более `parallelism` горутин одновременно. Это выявляет ещё одну интересную деталь -- пользователи могут захотеть иметь собственный контекст выполнения для каждой горутины. Например, предположим, что у нас есть `N` задач с максимумом `P` выполняющихся одновременно (и `P` может быть значительно меньше `N`). Если каждая задача требует какой-либо подготовки ресурсов, например, большой аллокации памяти, создания сессии базы данных или, может быть, запуска однопоточной Cgo "сопрограммы", то было бы логично подготовить только `P` ресурсов и переиспользовать их через контекст.

Как и выше, давайте оставим структуру передачи параметров в стороне.

{{<func file="./src/async/async.go" name="transform_03" next="transform" lang="go" class="focus" hl_lines="3 20 33-36 51-59 64-72">}}

На этом этапе мы запускаем _до_ `P` горутин и распределяем задачи между ними с помощью канала `wrk` без буфера. Мы не используем буфер, потому что мы хотим получать обратную связь о том, есть ли в данный момент бездействующие горутины или стоит ли нам рассмотреть возможность запуска новой. Как только все задачи обработаны или любой из вызовов `f()` завершается ошибкой, мы сигнализируем всем горутинам о необходимости прекратить дальнейшее выполнение (с помощью `close(wrk)`).

> Как и в предыдущем разделе, этого можно добиться на уровне `f()`, например, используя [`sync.Pool`][sync-pool]. `f()` может взять ресурс (или создать его, если в пуле нет свободных) и вернуть его, когда он больше не нужен. Поскольку набор горутин фиксирован, вероятность того, что ресурсы будут иметь хорошую локальность CPU велика, поэтому накладные расходы могут быть минимальными.
> {{<details summary="Пример реализации на стороне пользователя">}} 
```go
// Note that this snippet assumes `transform()` can limit its concurrency.
var pool sync.Pool
transform(ctx, 8, as, func(_ context.Context, userID string) (int, error) {
    sess := pool.Get().(*db.Session)
    if sess == nil {
        // Initialise database session.
    }
    defer pool.Put(sess)

    // Process userID.

    return 42, nil
})
```
{{</details>}}

## Обобщение `transform()`

До сих пор наш фокус был на отображении слайсов, что во многих случаях достаточно. Однако, что, если мы хотим отображать типы `map`, или, возможно, даже `chan`? Можем ли мы отображать все, по чему можно итерироваться с помощью `range`? И, сравнивая с циклом `for`, действительно ли нам всегда нужно _отображать_ значения?

Это интересные вопросы, которые приводят нас к мысли о том, что мы можем обобщить наш подход к многозадачной итерации. Мы можем определить функцию более "низкого уровня", которая будет вести себя почти так же, но будет делать немного меньше предположений о входных и выходных данных. Тогда потребуется не так много, чтобы реализовать более специфичную функцию `transform()`, которая использовала бы "низкоуровневую" функцию итерации. Давайте назовем такую функцию `iterate()` и определим входные и выходные данные функциями, а не конкретными типами данных. Мы будем извлекать входные элементы с помощью `pull()` и отправлять результаты обратно пользователю с помощью `push()`. Таким образом, пользователь `iterate()` сможет контролировать способ предоставления входных элементов и обработку результатов.

Мы также должны подумать о том, какие результаты `iterate()` будет передавать пользователю. Поскольку мы планируем сделать отображение входных элементов опциональным, `(B, error)` больше не кажется единственным правильным и очевидным вариантом. На самом деле этот вопрос довольно тонкий, и, возможно, в большинстве случаев явное возвращение ошибки было бы предпочтительнее. Однако, семантически это не имеет большого смысла, так как результат `f()` просто передается в вызов `push()` без какой-либо обработки, что означает, что у `iterate()` нет никаких ожиданий или зависимости относительно результата. Другими словами, результат имеет смысл только для функции `push()`, которую предоставляет пользователь. Кроме того, единственный возвращаемый параметр будет лучше работать с итераторами Go, которые мы рассмотрим в конце этой статьи. Имея это в виду, давайте попробуем уменьшить количество возвращаемых параметров до одного. Так же, поскольку мы планируем передавать результаты через вызов функции, нам скорее всего нужно делать это последовательно -- `transform()` и `iterate()` уже имеют всю необходимую синхронизацию внутри, поэтому мы можем избавить пользователя от необходимости дополнительной синхронизации при сборе результатов.

Еще один момент, о котором следует подумать, это обработка ошибок -- выше мы не связывали ошибку с входным элементом, обработка которого её вызывала. Очевидно, что `f()` может обернуть ошибку за нас, но более правильно для `f()` было бы не иметь предположений о том, как её будут вызывать. Другими словами, `f()` не должна предполагать, что её вызывают как аргумент `iterate()`. Если она была вызвана с одним элементом `a`, внутри `f()` нет смысла оборачивать `a` в ошибку, так как очевидно, что именно этот `a` и вызвал ошибке. Этот принцип приводит нас к другому наблюдению -- любое возможное связывание входного элемента с ошибкой (или любым другим результатом) также должно происходить во время выполнения `push()`. По тем же причинам, функция `push()` должна контролировать итерацию и решать, должна ли итерация быть прервана в случае ошибки.

Такой дизайн `iterate()` естественным образом обеспечивает flow control -- если пользователь делает что-то медленное внутри функции `push()`, то другие горутины в конечном итоге приостановят обработку новых элементов. Это произойдет потому, что они будут заблокированы отправляя результаты своего вызова `f()` в канал `res`, который читает функция, вызывающая `push()`.

Поскольку код функции становится слишком большим, давайте рассмотрим его по частям.

### Интерфейс

{{<func file="./src/async/async.go" name="iterate" next="iterate" lang="go" class="focus" lo="0" hi="8" hl_lines="1 5-8">}}

Теперь аргументы функции и возвращаемые параметры больше не имеют входных и выходных слайсов, как это было для функции `transform()`. Вместо этого входные элементы извлекаются с помощью вызова функции `pull()`, а результаты возвращаются пользователю с помощью вызова функции `push()`. Обратите внимание, что функция `push()` возвращает параметр `bool`, который контролирует итерацию -- как только возвращается `false`, ни одного вызова `push()` больше не будет сделано, и контекст всех уже выполняющихся `f()` будет отменен. Функция `iterate()` возвращает только ошибку, которая может быть не `nil` только при прекращении итерации из-за отмены `ctx` переданного в аргументах `iterate()`. В противном случае не будет способа узнать, почему прекратилась итерация.

> Не смотря на то, что есть всего три случая, когда итерация может быть прервана:
> - `pull()` вернул `false`, что означает, что больше нет элементов для обработки.
> - `push()` вернул `false`, что означает, что пользователю больше не нужны никакие результаты.
> - `ctx` переданный в аргументах `iterate()` был отменён.
> 
> Без усложнений пользовательского кода трудно сказать, были ли обработаны все элементы до отмены `ctx`.
> 
> {{<details summary="Пример" raw="true">}} 
Давайте представим, что мы хотим реализовать многозадачный `forEach()`, который использует `iterate()`:
{{<func file="./src/async/async.go" name="forEach" next="forEach" lang="go">}}
{{</details>}}


### Пролог

{{<func file="./src/async/async.go" name="iterate" next="iterate" lang="go" lo="8" hi="40">}}

В предыдущих версиях `transform()` мы сохраняли результат по индексу, соответствующему входному элементу, в слайс результатов, что выглядело как `bs[i] = f(as[i])`. Теперь, при функциональном вводе и выводе такой подход невозможен. Поэтому, как только у нас есть результат обработки _любого элемента_, нам, скорее всего, нужно немедленно отправить его пользователю с помощью `push()`. Вот почему мы хотим иметь две горутины для распределения входных элементов между воркерами и отправки результатов пользователю -- пока мы распределяем входные элементы, мы можем получить результат обработки уже распределенных ранее.

### Горутина распределения элементов

{{<func file="./src/async/async.go" name="iterate" next="iterate" lang="go" lo="41" hi="71">}}

### Цикл распределения

{{<func file="./src/async/async.go" name="iterate" next="iterate" lang="go" class="focus" lo="72" hi="105" hl_lines="1 3 6-14 17 23 27-29 32-33">}}

### Горутина выполнения функции

{{<func file="./src/async/async.go" name="iterate" next="iterate" lang="go" class="focus" lo="106" hi="134" hl_lines="2 13-21">}}

### Сбор результатов

{{<func file="./src/async/async.go" name="iterate" next="iterate" lang="go" lo="134" hi="181">}}

Как вы можете видеть, результаты возвращаются пользователю в случайном порядке -- не в том порядке, в котором они были извлечены. Это ожидаемо, поскольку мы обрабатываем их параллельно.

> Субъективное замечание: совместное использование `sync.WaitGroup` и канала `sem` в данном случае является редким исключением, когда использование обоих механизмов синхронизации в одном коде оправдано. Я считаю, что в большинстве случаев, если есть канал, `sync.WaitGroup` будет излишним, и наоборот.

Фух, вот и все! Это было не просто, но это то, что мы хотели сделать. Давайте посмотрим, как мы можем использовать это.

{{<details summary="Полный листинг кода" raw="true">}} 
{{<func file="./src/async/async.go" name="iterate" next="iterate" lang="go">}}
{{</details>}}

## Использование `iterate()` для `transform()`

Чтобы посмотреть, как обобщенная функция итерации может решать проблему _отображения_, давайте реализуем `transform()` используя `iterate()`. Очевидно, что сейчас это выглядит намного компактнее, поскольку мы перенесли всю логику многозадачности в `iterate()`, а в `transform()` сосредоточились на сохранении результатов отображения.

{{<func file="./src/async/async.go" name="transform_05" next="transform" lang="go">}}

## Новый `errgroup`

Чтобы закрыть аналогию с пакетом `errgroup`, давайте попробуем реализовать что-то подобное с использованием `iterate()`.

```go
type taskFunc func(context.Context) error
```

{{<func file="./src/async/async.go" name="errgroup" next="errgroup" lang="go">}}

Таким образом, использование функции будет очень похоже на использование `errgroup`:
```go
// Create the workers group and a context which will be canceled if any of the
// tasks fails.
g, wait := errgroup(ctx)
g(func(ctx context.Context) error {
	return doSomeFun(gctx)
})
g(func(ctx context.Context) error {
	return doEvenMoreFun(gctx)
})
if err := wait(); err != nil {
	// handle error
}
```

## Go Iterators

Давайте кратко рассмотрим ближайшее будущее Go, и то, как оно может повлиять на идеи рассмотренные выше.

С недавним [экспериментом range over func][rangefunc-wiki] (начиная с Go 1.22) возможна итерация `range` по функциям, совместимыми с типами [итератора последовательностей][iter-seq], определенными в пакете `iter`. Это новая концепция Go, которая, надеюсь, станет частью стандартной библиотеки в будущих версиях. Для получения дополнительной информации, пожалуйста, прочтите [range over func proposal][rangefunc-proposal], а также предопределяющую статью о [сопрограммах в Go][coro-article] от Russ Cox, на основе которой реализован пакет `iter`.

Сделать `iterate()` совместимым с `iter` -- проще простого:
{{<func file="./src/async/async.go" name="iter_02" next="iterate" lang="go" class="focus" hl_lines="1 5 7-13">}}

Эксперимент позволяет нам делать потрясающие вещи -- итерироваться по результатам параллельно обрабатываемых элементов _последовательности_ в обычном цикле `for`!
```go
// Assuming the standard library supports iterators.
seq := slices.Sequence([]int{1, 2, 3})

// Output: [1, 4, 9]
for a, b := range iterate(ctx, nil, 0, seq, square) {
	fmt.Println(a, b)
}
```

## Выводы

Я хотел бы, чтобы это было частью стандартной библиотеки Go.

Изначально я думал, что было бы круто оставить этот раздел с одним предложением выше. Но, вероятно, все же стоит сказать несколько слов почему. Я считаю, что подобные библиотеки общего назначения могут быть гораздо лучше восприняты и применены в проектах, если большая часть сообщества согласна с их дизайном и реализацией. Конечно, у нас могут быть разные библиотеки, решающие похожие проблемы, но на мой взгляд, в некоторых случаях большое количество библиотек _может_ приводить к большому количеству разногласий о том, что, когда и как использовать. В некоторых случаях нет ничего плохого в том, чтобы иметь совершенно разные подходы и реализации, но в некоторых случаях это также может означать и вовсе отсутствие полноценного решения. Очень часто библиотеки появляются как конкретное решение какой-то конкретной проблемы, гораздо более конкретное, чем требуется для библиотеки общего применения. Чтобы получить общee решение, дизайн и концепцию было бы здорово обсуждать _задолго до_ начала реализации. Так происходит в open-source foundations или, в случае с Go -- в команде разработчиков языка. Наличие инструментов стандартной библиотеки для многозадачной обработки данных кажется естественным развитием Go после пакетов `slices`, `coro` и `iter`.

Большое спасибо за ваше внимание.

Написание этой статьи заняло достаточно много времени и усилий. Я надеюсь, она была полезной!

## Ссылки

- [Parallel Collections в Scala][scala-par]
- [When To Use Generics][when-generics]
- [Rangefunc Experiment][rangefunc-wiki]



[context]: https://pkg.go.dev/context#Context
[coro-article]: https://research.swtch.com/coro
[coro-source]: https://github.com/golang/go/blob/20f052c83c380b8b3700b7aca93017178a692d78/src/runtime/coro.go
[errgroup]: https://pkg.go.dev/golang.org/x/sync/errgroup
[genny]: https://github.com/cheekybits/genny 
[interfaces]: https://research.swtch.com/interfaces
[iter-proposal]: https://github.com/golang/go/issues/61897
[iter-seq]: https://github.com/golang/go/blob/20f052c83c380b8b3700b7aca93017178a692d78/src/iter/iter.go#L19-L27
[method-expressions]: https://go.dev/ref/spec#Method_expressions
[rangefunc-proposal]: https://github.com/golang/go/discussions/56413
[rangefunc-wiki]: https://go.dev/wiki/RangefuncExperiment
[scala-par]: https://docs.scala-lang.org/overviews/parallel-collections/overview.html
[semaphore]: https://ru.wikipedia.org/wiki/%D0%A1%D0%B5%D0%BC%D0%B0%D1%84%D0%BE%D1%80_(%D0%BF%D1%80%D0%BE%D0%B3%D1%80%D0%B0%D0%BC%D0%BC%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5)
[sync-pool]: https://pkg.go.dev/sync#Pool
[when-generics]: https://go.dev/blog/when-generics