---
title: "Средства измерения программ на Go"
date: 2020-05-31T21:23:00+03:00
draft: false
image: instrumentation-in-go.jpg
imageWidth: 600
description: "В этой статье я хотел бы поделиться гибким способом профилирования программ на Go."
---
{{< img src="instrumentation-in-go.jpg" width="600" alt="Гофер смотрит на читателя через лупу" >}}

## Вступление

В этой статье я хотел бы поделиться способом профилирования и трассировки
программ на Go. Я расскажу, как можно это делать, сохраняя код гибким и чистым.

## TL;DR

Логирование, сбор метрик и все, что не связано с основной функциональностью
какого-либо кода, не должно находиться внутри этого кода. Вместо этого
нужно определить _точки трассировки_, которые могут быть использованы для
измерения кода пользователем.

Другими словами, логирование и сбор метрик – это подмножества **трассировки**.

Шаблонный код трассировки может быть сгенерирован с помощью [gtrace][gtrace].

## Проблема

Предположим, что у нас есть пакет `lib` и некая структура `lib.Client`. Перед
выполнением какого-либо запроса `lib.Client` проверяет соединение:

```go
package lib

type Client struct {
	conn net.Conn
}

func (c *Client) Request(ctx context.Context) error {
	if err := c.ping(ctx); err != nil {
		return err
	}
	// Some logic here.
}

func (c *Client) ping(ctx context.Context) error {
	return doPing(ctx, c.conn)
}
```

Что делать, если мы хотим делать запись в лог прямо перед и сразу после того,
как отправляется ping-сообщение? Первый вариант – это внедрить логгер (или его
интерфейс) в `Client`:

```go
package lib

type Client struct {
	Logger Logger

	conn net.Conn
}

func (c *Client) ping(ctx context.Context) (err error) {
	c.Logger.Info("ping started")
	err = doPing(ctx, c.conn)
	c.Logger.Info("ping done (err is %v)", err)
	return
}
```

Если мы захотим собирать какие-либо метрики, мы можем сделать то же самое:

```go
package lib

type Client struct {
	Logger  Logger
	Metrics Registry

	conn net.Conn
}

func (c *Client) ping(ctx context.Context) (err error) {
	start := time.Now()
	c.Logger.Info("ping started")

	err = doPing(ctx, c.conn)

	c.Logger.Info("ping done (err is %v)", err)
	metric := c.Metrics.Get("ping_latency")
	metric.Send(time.Since(start))

	return err
}
```

И логирование, и сбор метрик – это методы трассировки компонента. Если мы
продолжим увеличивать их количество внутри `Client`, то скоро обнаружим, что
бóльшая часть его кода будет содержать код трассировки, а не код его основной
функциональности (который заключался в одной строчке с `doPing()`).

Количество несвязных (не связанных с основной функциональностью `Client`)
строчек кода это только первая проблема такого подхода.

Что, если в ходе эксплуатации программы вы поймете, например, что имя метрики
нужно поменять? Или, например, вы решите поменять логгер или сообщения в логах?

С таким подходом как выше, вам придется править код `Client`.

Это означает, что вы будете править код каждый раз, когда меняется что-то не
связанное с _основной функциональностью_ компонента. Другими словами, такой
подход нарушает [принцип единственной ответственности (SRP)][wikipedia:srp].

Что, если вы переиспользуете код `Client` между разными программами? Или, более
того, выложили ее в общий доступ? Если честно, то я советую рассматривать
_каждый_ пакет в Go, как библиотеку, даже если в реальности используете её
только вы.

Все эти вопросы указывают на ошибку, которую мы совершили:

> Мы не должны предполагать, какие методы трассировки захотят применять
> пользователи нашего кода.

## Решение

На мой взгляд, правильно было бы определить _точки трассировки_ (или _хуки_), в
которых могут быть установлены пользовательские функции (или _пробы_).

Безусловно, дополнительный код останется, но при этом мы дадим пользователям
измерять работу нашего компонента любым способом.

Такой подход используется, например, в пакете [`httptrace`][go:httptrace] из
стандартной библиотеки Go.

Давайте предоставим такой же интерфейс, но с одним исключением: вместо хуков
`OnPingStart()` и `OnPingDone()`, мы определим только `OnPing()`, который будет
возвращать callback. `OnPing()` будет вызван непосредственно перед отправкой
ping-сообщения, а callback – сразу после. Таким образом мы сможем сохранять
некоторые переменные в замыкании (например, чтобы посчитать время выполнения
`doPing()`).

`Client` теперь будет выглядеть так:

```go
package lib

type Client struct {
	OnPing func() func(error)
	conn net.Conn
}

func (c *Client) ping(ctx context.Context) (err error) {
	done := c.OnPing()
	err = doPing(ctx, c.conn)
	done(err)
	return
}
```

Выглядит аккуратненько, но только если не проверять хук `OnPing` и его
результат на `nil`. Правильнее было бы сделать следующее:

```go
func (c *Client) ping(ctx context.Context) (err error) {
	var done func(error)
	if fn := c.OnPing; fn != nil {
		done = fn()
	}
	err = doPing(ctx, c.conn)
	if done != nil {
		done(err)
	}
	return
}
```

Теперь наш код выглядит хорошо в плане SRP принципа и гибкости, но не так
хорошо в плане читаемости.

Прежде чем это исправить, давайте решим еще одну проблему трассировки.

### Объединение хуков

Как пользователи могут установить _несколько_ проб для одного хука? Пакет
`httptrace` содержит метод [`ClientTrace.compose()`][github:httptrace], который
объединяет две структуры трассировки в одну. В результате каждая функция
полученной структуры делает вызовы соответствующих функций в паре родительских
структур (если они были установлены).

Давайте попробуем сделать то же самое вручную (и без использования пакета
`reflect`). Для этого мы перенесем хук `OnPing` из `Client` в отдельную
структуру `ClientTrace`:

```go
package lib

type Client struct {
	Trace ClientTrace
	conn net.Conn
}

type ClientTrace struct {
	OnPing func() func(error)
}
```

Объединение двух таких структур в одну будет выглядеть следующим образом:

```go
func (a ClientTrace) Compose(b ClientTrace) (c ClientTrace) {
	switch {
	case a.OnPing == nil:
		c.OnPing = b.OnPing
	case b.OnPing == nil:
		c.OnPing = a.OnPing
	default:
		c.OnPing = func() func(error) {
			doneA := a.OnPing()
			doneB := b.OnPing() 
			switch {
			case doneA == nil:
				return doneB
			case doneB == nil:
				return doneA
			default:
				return func(err error) {
					doneA(err)
					doneB(err)
				}
			}
		}
	}
	return c
}
```

Достаточно много кода для одного хука, верно? Но давайте двигаться дальше, чуть
позже мы вернемся к этому.

Теперь пользователь может менять методы трассировки независимо от нашего
компонента:

```go
package main

import (
	"log"
	
	"some/path/to/lib"
)

func main() {
	var trace lib.ClientTrace

	// Logging hooks.
	trace = trace.Compose(lib.ClientTrace{
		OnPing: func() func(error) {
			log.Println("ping start")
			return func(err error) {
				log.Println("ping done", err)
			}
		},
	})

	// Some metrics hooks.
	trace = trace.Compose(lib.ClientTrace{
		OnPing: func() func(error) {
			start := time.Now()
			return func(err error) {
				metric := stats.Get("ping_latency")
				metric.Send(time.Since(start))
			}
		},
	})

	c := lib.Client{
		Trace: trace,
	}
}
```

### Трассировка и контекст

Трассировка кода так же может происходить в зависимости от контекста. Давайте
предоставим пользователю возможность связать `ClientTrace` с экземпляром
`context.Context`, который потом может быть передан в `Client.Request()`:

```go
package lib

type clientTraceContextKey struct{}

func ClientTrace(ctx context.Context) ClientTrace {
	t, _ := ctx.Value(clientTraceContextKey{})
	return t
}

func WithClientTrace(ctx context.Context, t ClientTrace) context.Context {
	prev := ContextClientTrace(ctx)
	return context.WithValue(ctx,
		clientTraceContextKey{},
		prev.Compose(t),
	)
}
```

Фух. Кажется, мы почти закончили!

Но не кажется ли утомительным писать весь этот код для всех компонентов?
Конечно, вы можете определить макросы Vim для этого (и на самом деле я делал
так какое-то время), но давайте посмотрим на другие варианты.

Хорошая новость состоит в том, что весь код для объединения хуков, проверок на
`nil` и функций для работы с контекстом весьма шаблонный, и мы можем его
[сгенерировать][go:generate] без использования макросов или пакета
`reflection`.

## github.com/gobwas/gtrace

**gtrace** это инструмент командной строки для генерации кода трассировки из
примеров выше. Для описания точек трассировки используются структуры,
помеченные директивой `//gtrace:gen`. В результате вы получаете возможность
вызова хуков без каких-либо проверок на `nil`, а так же функции для
их объединения и функции для работы с контекстом.

Пример сгенерированного кода [находится здесь][github:gtrace:example].

Теперь мы можем удалить весь рукописный код и оставить только это:

```go
package lib

//go:generate gtrace

//gtrace:gen
//gtrace:set context
type ClientTrace struct {
	OnPing func() func(error)
}

type Client struct {
	Trace ClientTrace
	conn net.Conn
}

func (c *Client) ping(ctx context.Context) (err error) {
	done := c.Trace.onPing(ctx)
	err = doPing(ctx, c.conn)
	done(err)
	return
}
```

После выполнения команды `go generate` мы можем использовать сгенерированные
_не экспортированные_ версии хуков из `ClientTrace`.

Вот и все! **gtrace** берет весь шаблонный код на себя и позволяет вам
сфокусироваться на _точках трассировки_, которые вы хотели бы предоставить
пользователям для измерения вашего кода.

Спасибо за внимание!

## References

- [github.com/gobwas/gtrace][gtrace]
- [Minimalistic C libraries](https://nullprogram.com/blog/2018/06/10/) by Chris
  Wellons. Когда-то я прочитал эту статью и был вдохновлен идеями по
  организации кода библиотек. 

[gtrace]:                https://github.com/gobwas/gtrace
[wikipedia:srp]:         https://ru.wikipedia.org/wiki/%D0%9F%D1%80%D0%B8%D0%BD%D1%86%D0%B8%D0%BF_%D0%B5%D0%B4%D0%B8%D0%BD%D1%81%D1%82%D0%B2%D0%B5%D0%BD%D0%BD%D0%BE%D0%B9_%D0%BE%D1%82%D0%B2%D0%B5%D1%82%D1%81%D1%82%D0%B2%D0%B5%D0%BD%D0%BD%D0%BE%D1%81%D1%82%D0%B8
[go:httptrace]:          https://golang.org/pkg/net/http/httptrace
[go:generate]:           https://blog.golang.org/generate
[github:httptrace]:      https://github.com/golang/go/blob/b68fa57c599720d33a2d735782969ce95eabf794/src/net/http/httptrace/trace.go#L175
[github:gtrace:example]: https://github.com/gobwas/gtrace/blob/216866e1680bb30fb108f26ee926c471f0920239/examples/pinger/main_gtrace.go
