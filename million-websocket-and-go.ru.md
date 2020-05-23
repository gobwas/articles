---
title: "Миллион WebSocket и Go"
date: 2017-06-28T13:03:00+03:00
draft: false
image: wsgo.jpeg
description: "Это статья о том, как мы разрабатывали высоконагруженный WebSocket сервер на Go."
---
{{< img src="wsgo.jpeg" alt="Летающий Гофер с логотипом MailRu" >}}

Привет всем!

Меня зовут Сергей Камардин, я программист в команде почты MailRu Group.

Это статья о том, как мы разрабатывали высоконагруженный WebSocket сервер на Go.

В случае, если тема WebSocket вам близка, но Go не совсем, не стоит огорчаться – статья все равно может оказаться вам полезной с точки зрения идей и приемов оптимизации.

## Предисловие

Чтобы обозначить контекст рассказа, стоит сказать пару слов о том, для чего нам понадобился такой сервер.

В mail.ru есть множество систем, состояние которых может меняться. Очевидно, что такой системой является и хранилище писем пользователей. Об изменении состояния системы – событиях – можно узнавать несколькими способами. В основном это либо периодический опрос системы (polling) либо, в обратном направлении – уведомления со стороны системы об измении ее состояния. У обоих способов есть свои плюсы и минусы, однако, в случае с почтой, чем быстрее пользователь получит письмо – тем лучше.

Polling в почте – это около 50 тысяч HTTP запросов в секунду, 60% из которых – возвращают 304, что означает отсутствие изменений в ящике.

Поэтому, чтобы сократить нагрузку на серверы и ускорить доставку писем пользователям, было решено ~~изобрести велосипед~~ написать publisher-subscriber сервер (bus, message-broker, event-channel), который с одной стороны получает сообщения об изменении состояний, а с другой – подписки на такие сообщения:

Было:
```TeX
+-----------+           +-----------+           +-----------+
|           | ◄-------+ |           | ◄-------+ |           |
|  Storage  |           |    API    |    HTTP   |  Browser  |
|           | +-------► |           | +-------► |           |
+-----------+           +-----------+           +-----------+
```

Стало:
```TeX
 +-------------+     +---------+   WebSocket   +-----------+
 |   Storage   |     |   API * | +-----------► |  Browser  |
 +-------------+     +---------+          3)   +-----------+
        +              2) ▲
        |                 |
     1) ▼                 +     
+---------------------------------+
|               Bus               |
+---------------------------------+
```

На первой схеме – отображено то, как было раньше. Браузер с каким-то интервалом ходил в API и спрашивал об изменениях на Storage (хранилище писем).

На второй – новый вариант архитектуры. Браузер устанавливает WebSocket соединение с новым API, внутри которого посылает подписку на интересующие его события. API эту информацию сохраняет и передает ее в Bus – сервер обмена событиями (об этом сервере речи сегодня идти не будет, возможно расскажу об этом в следующих публикациях). В момент получения нового письма для пользователя, Storage посылает об этом уведомление в Bus, а та в свою очередь всем своим подписчикам, в том числе, на на сервер API. API определяет какому соединению отправить это уведомление и посылает его через WebSocket. Как вы могли догадаться, речь сегодня будет идти именно об этом API, или WebSocket сервере.

Забегая вперед, скажу, что API будет держать около 3 миллионов живых соединений. Эта цифра еще не раз всплывет в последующем рассказе об оптимизациях.

Теперь, понимая область в которой нужно работать, мы готовы отправиться дальше. Go!

## Idiomatic way

Посмотрим на то, как бы мы реализовали некоторые части нашего сервера.

Прежде, чем рассматривать работу с `net/http`, поговорим об отправке и получении данных. Давайте представим пользовательское соединение в виде структуры `Channel`, которая будет оберткой для WebSocket соединения.

### Channel struct

```go
// Packet represents application level data.
type Packet struct {
	...
}

// Channel wraps user connection.
type Channel struct {
	// WebSocket connection.
	conn net.Conn

	// Outgoing packets queue.
	send chan Packet
}

func NewChannel(conn net.Conn) *Channel {
	c := &Channel{
		conn: conn,
		send: make(chan Packet, N),
	}

	go c.reader()
	go c.writer()

	return c
}
```

Хочу обратить ваше внимание на запуск двух горутин чтения и записи. Для каждой горутины нужен свой stack, который в [зависимости от операционной системы][runtime:stack] и версии Go может иметь начальный размер от 2 до 8 килобайт. Если учесть цифру, названную выше – 3 миллиона живых соединений, то на каждое из них нам потребуется **24GB памяти** (при стеке в 4KB). И это без учета памяти, выделяемой на структуру `Channel`, очередь исходящих пакетов `ch.send` и другие внутренние поля.

### Горутины I/O 

Посмотрим на реализацию "читателя" из соединения:

```go
func (c *Channel) reader() {
	// We make buffered read to reduce read syscalls.
	buf := bufio.NewReader(c.conn)

	for {
		pkt, _ := readPacket(buf)
		c.handle(pkt)
	}
}
```

Достаточно просто, верно? Мы используем буфер, чтобы сократить количество syscall'ов на чтение и вычитывать сразу столько, сколько позволит нам размер `buf`. В бесконечном цикле мы ожидаем поступления новых данных в соединение и читаем следующий пакет. Попрошу запомнить фразу *ожидаем поступления новых данных* – к ее сути мы еще вернемся. 

Парсинг и обработка входящих пакетов останутся в стороне, поскольку это не важно для тех оптимизаций, о которых будет идти речь. А вот на `buf` все же стоит сейчас обратить внимание – по умолчанию, это 4KB, и это значит, что на наши 3 миллиона соединений это еще **12GB** памяти.

Аналогичная ситуация с "писателем":

```go
func (c *Channel) writer() {
	// We make buffered write to reduce write syscalls. 
	buf := bufio.NewWriter(c.conn)

	for pkt := range c.send {
		_ := writePacket(buf, pkt)
		buf.Flush()
	}
}
```

Мы итерируемся по каналу исходящих пакетов `c.send` и пишем их в буфер. Это, как внимательный читатель уже мог догадаться, еще 4KB памяти и еще **12GB** на наши 3 миллиона соединений.

### HTTP

Простая реализация `Channel` у нас есть, теперь нам нужно раздобыть WebSocket соединенине, с которым мы будем работать. 
Поскольку мы все еще находимся под заголовком **Idiomatic way**, то сделаем это в соответствующем ключе.

> Если вы не знакомы с тем, как работает WebSocket, то стоит сказать, что клиент переходит на протокол WebSocket с помощью специального механизма в HTTP, который называется Upgrade. После успешной обработки Upgrade запроса сервер и клиент используют TCP соединение для обмена бинарными WebSocket фреймами.
>
> Вот [тут][rfc6455:frame] описана структура фрейма внутри соединения.

```go
import (
	"net/http"
	"some/websocket"
)

http.HandleFunc("/v1/ws", func(w http.ResponseWriter, r *http.Request) {
	conn, _ := websocket.Upgrade(r, w)
	ch := NewChannel(conn)
	...
})
```

Обратим внимание на то, что `http.ResponseWriter` внутри себя так же содержит буфер записи `bufio.Writer` на 4KB, а для инициализации `*http.Request` происходит выделение буфера чтения `bufio.Reader` тоже на 4KB. 

Независимо от используемой библиотеки WebSocket, после успешного ответа на Upgrade запрос сервер [получает][net/http:buffers] I/O буферы вместе с TCP соединением при вызове `responseWriter.Hijack()`.

> В некоторых случаях, при помощи `go:linkname` можно вернуть буферы в пул `net/http` через вызов `net/http.putBufio{Reader,Writer}`.

Таким образом, нам нужно еще **24GB** памяти на 3 миллиона соединений. 

Итого уже **72GB** памяти на приложение, которое еще ничего не делает!

## Оптимизации

Стоит освежить в памяти контекст, о котором мы говорили в предисловии, и вспомнить как ведет себя пользовательское соединение?

После перехода на WebSocket клиент посылает пакет с интересующими его событиями – то есть подписывается на события. После этого (не считая технических сообщений типа `ping/pong`) клиент может ничего не отправить за все время жизни соединения.

> Время жизни соединения может быть от нескольких секунд до нескольких дней.

Зная этот факт кажется, что мы можем сделать некоторые вещи лучше.

### netpoll

Помните реализацию `Channel.reader()`, который *ожидал поступления новых данных*, блокируясь на вызове `conn.Read()` внутри `bufio.Reader`? Когда данные появлялись в соединении, runtime go "будил" нашу горутину и позволял прочитать очередной пакет. Давайте посмотрим, как runtime go понимает, что горутину нужно "разбудить".

Заглянув в [реализацию `conn.Read()`][net:conn.Read()], мы увидим, что внутри, помимо проверок ошибок, [происходит вызов `net.netFD.Read()`][net:fd.Read()]. Что там?

```go
// net/fd_unix.go

func (fd *netFD) Read(p []byte) (n int, err error) {
	//...
	for {
		n, err = syscall.Read(fd.sysfd, p)
		if err != nil {
			n = 0
			if err == syscall.EAGAIN {
				if err = fd.pd.waitRead(); err == nil {
					continue
				}
			}
		}
		//...
		break
	}
	// ...
}
```

> Сокеты в go неблокирующие. EAGAIN говорит о том, что данных в сокете нет, и чтобы не блокироваться на чтении из пустого сокета, ОС возвращает нам управление.

Мы видим, что происходит системный вызов `read()` из файлового дескриптора соединения. В случае, если чтение возвращает [ошибку `EAGAIN`][man:EAGAIN], runtime делает [вызов `pollDesc.waitRead()`][net:pollDesc.waitRead()]. Давайте посмотрим, что там:

 ```go
// net/fd_poll_runtime.go

func (pd *pollDesc) waitRead() error {
	return pd.wait('r')
}

func (pd *pollDesc) wait(mode int) error {
	res := runtime_pollWait(pd.runtimeCtx, mode)
	//...
}
 ```

 Если [покопать глубже][runtime:netpoll], то мы [увидим][runtime:netpoll_epoll], что на Linux `netpoll` реализован с помощью [`epoll`][man:epoll]. 

 Почему бы нам не использовать такой же подход для своих соединений? Мы могли бы выделять буфер на чтение и запускать горутину только тогда, когда это действительно нужно – тогда, когда в сокете точно есть данные.

 > На github/golang/go есть [issue][go:issue], которая позволит использовать внутреннюю реализацию epoll.

### Избавляемся от горутин

Предположим, что у нас есть какая-то [реализация][easygo:netpoll] netpoll для Go. Теперь мы можем не запускать горутину `Channel.reader()`, и вместо этого "подписаться" на событие наличия входящих данных с помощью netpoll:

```go
ch := NewChannel(conn)

// Make conn to be observed by netpoll instance.
// Note that IN and ONESHOT is an aliases for 
// EPOLLIN and EPOLLONESHOT on Linux.
poller.Add(conn, netpoll.IN | netpoll.ONESHOT, func() {
	// We spawn goroutine here to prevent poller wait loop
	// to become locked during receiving packet from ch.
	go func() {
		pkt, err := Receive(ch)
		if err == nil {
			poller.Resume(conn)
		}
	}()
})

// Receive reads a packet from conn and handles it somehow.
func (ch *Channel) Receive() error {
	buf := bufio.NewReader(ch.conn)
	pkt, _ := readPacket(buf)
	c.handle(pkt)
}
```

Мы подписываемся с флагом `ONESHOT`, чтобы переданная функция была вызввана только один раз. При поступлении данных в соединение, мы обрабатываем полученный пакет и возобновляем "подписку" на входящие данные с помощью `poller.Resume()`.

С `Channel.writer()` дело обстоит проще – мы можем запускать горутину и аллоцировать буфер только тогда, когда мы собираемся отправить пакет:

```go
func (ch *Channel) Send(p Packet) {
	if c.noWriterYet() {
		go ch.writer()
	}
	ch.send <- p
}
```

После чтения одного (или нескольких) исходящих пакетов из `ch.send` writer завершит свою работу и освободит память буфера.

Отлично! Мы сэкономили **48GB** – избавились от stack'а и I/O буферов внутри двух постоянно "работающих" горутин.

### Контроль ресурсов

Большое количество соединений это не только большое потребление памяти. В процессе разработки всей системы у нас не раз случались race condition'ы и deadlock'и, которые очень часто сопровождались так называемым self ddos – ситуацией, когда клиенты приложения безудержно пытались подсоединиться к серверу и еще больше ~~доламывали~~ ломали его. 

Например, если вдруг по какой-то причине мы не могли обрабатывать `ping/pong` сообщения, но обработчик "idle" соединений продолжал такие соединения отключать (предоплагая, что от этих соединений нет данных из-за того, что они были закрыты некорректно), то получалось, что клиент, вместо того, чтобы после подключения ожидать событий, терял соединение каждые N секунд и пробовал подключиться снова.

Безусловно, такие проблемы должны решаться и на стороне клиентов. Но как сервер мог бы предотвратить такие ситуации?

Если вдруг все клиенты захотят отправить нам пакет по какой-либо причине, сэкономленные ранее **48GB** будут снова в деле – по сути мы вернемся к изначальному состоянию горутины и буфера на каждое соединение.

#### Goroutine pool

Мы можем ограничить количество одновременно обрабатываемых пакетов с помощью пула горутин.

Вот как выглядит наивная реализация такого пула:

```go
package gopool

func New(size int) *Pool {
	return &Pool{
		work: make(chan func()),
		sem:  make(chan struct{}, size),
	}
}

func (p *Pool) Schedule(task func()) error {
	select {
	case p.work <- task:
	case p.sem <- struct{}{}:
		go p.worker(task)
	}
}

func (p *Pool) worker(task func()) {
	defer func() { <-p.sem }
	for {
		task()
		task = <-p.work
	}
}
```

И теперь, наш код с epoll принимает следующий вид:

```go
pool := gopool.New(128)

poller.Add(conn, func() {
	// We will block poller wait loop when
	// all pool workers are busy.
	pool.Schedule(func() {
		Receive(ch)
		epoll.Resume(conn)
	})
})
```

То есть теперь мы выполняем чтение пакета не сразу при обнаружении данных в сокете, но при первой возможности занять свободного worker'а в пуле горутин.

Аналогично мы поменяем `Send()`:

```go
p := pool.New(128)

func (ch *Channel) Send(p Packet) {
	if c.noWriterYet() {
		p.Schedule(ch.writer)
	}
	ch.send <- p
}
```

Вместо `go ch.writer()` мы хотим делать запись в горутине из пула.

Таким образом, в случае пула из `N` горутин, мы гарантируем то, что при `N` одновременно обрабатываемых запросах и пришедшем `N+1`, мы не будем аллоцировать `N+1` буфер на чтение.

Пул горутин так же позволяет лимировать `Accept()` и `Upgrade()` новых соединений и избегать большинства ситуаций с DDOS. Об этом еще пойдет речь чуть позже.

### Zero-copy upgrade

Давайте уйдем немного в сторону протокола WebSocket. 

Как уже говорилось выше, клиент переходит на протокол WebSocket с помощью HTTP запроса Upgrade. Вот как это выглядит:

```http
GET /ws HTTP/1.1
Host: mail.ru
Connection: Upgrade
Sec-Websocket-Key: A3xNe7sEB9HixkmBhVrYaA==
Sec-Websocket-Version: 13
Upgrade: websocket

HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Sec-Websocket-Accept: ksu0wXWG+YmkVx+KQR2agP0cQn4=
Upgrade: websocket
```

То есть HTTP запрос и его заголовки в нашем случае нужны только для того, чтобы перейти на WebSocket протокол. Это знание, а так же то, [что хранится][net/http:Request] внутри `http.Request`, наводит нас на мысль, что, возможно, в целях оптимизации, мы могли бы отказаться от ненужных нам аллокаций и копирований при разборе HTTP запроса и.. уйти от стандартного сервера `net/http`.

> `http.Request` содержит, например, [поле с одноименным типом `Header`][net/http:Header], которое безусловно заполняется всеми заголовками запроса путем копирования данных из соединения в строки. Можно представить сколько лишних данных можно держать внутри этого поля, например, при большом размере заголовка `Cookie`.

Но что взять взамен?

#### Реализации WebSocket

К сожалению, все существовавшие на момент оптимизации нашего сервера библиотеки позволяли делать upgrade только при использовании стандартного `net/http` сервера.

Более того, ни одна (из двух) библиотек не позволяли применить все оптимизации чтения и записи описанные выше. Для того, чтобы эти оптимизации работали, нам нужно иметь достаточно низкоуровневый API для работы с WebSocket. При переиспользовании буферов, нам нужно, чтобы функции по работе с соединением выглядели примерно так:

```go
// Somewhere inside WebSocket library.
func ReadFrame(io.Reader) Frame
func WriteFrame(io.Writer, Frame) error
```

Имея библиотеку с таким API, мы могли бы читать пакеты из соединения следующим образом (с записью все аналогично):

```go
// getReadBuf, putReadBuf are intended to reuse *bufio.Reader 
// instances (with sync.Pool for example).
func getReadBuf(io.Reader) *bufio.Reader
func putReadBuf(*bufio.Reader)

// Called when data could be read from conn.
func readPacket(conn io.Reader) error {
	buf := getReadBuf(conn)
	defer putReadBuf(buf)
	
	frame := ReadFrame(buf)
	parsePacket(frame.Payload)
	//...
}
```

Короче говоря, настало время запилить свою либу.

#### github.com/gobwas/ws

Идеологически, библиотека `ws` была написана с мыслью, что она не должна навязывать пользователю логику работы с протоколом. Все методы чтения/записи принимают стандартные интерфейсы `io.Reader` и `io.Writer`, что позволяет использовать или не использовать буферизацию, ровно как и любые другие обертки вокруг I/O.

Конечно, `ws`, кроме стандартного `net/http`, поддерживает **zero-copy upgrade** – обработку HTTP Upgrade запроса и переход на WebSocket без выделений памяти и копирований. `Upgrade()` при этом принимает `io.ReadWriter` (`net.Conn` реализует этот интерфейс). То есть мы могли бы использовать стандартный `net.Listen()` и передавать результат `ln.Accept()` сразу в `ws.Upgrade()`. Конечно, библиотека дает возможность пользователю скопировать любые данные запроса для будущего использования в приложении (например `Cookie` для проверки сессии).

Ниже сравнение upgrade в рамках стандартного `net/http` и в режиме zero-copy upgrade:
```bash
BenchmarkUpgradeHTTP    5156 ns/op    8576 B/op    9 allocs/op
BenchmarkUpgradeTCP     973 ns/op     0 B/op       0 allocs/op
```

Переход на `ws` и **zero-copy upgrade** позволил сэкономить еще **24GB** – тех самых, которые выделялись на I/O буферы при обработке запроса в хэндлере `net/http`.

### Всё вместе

Давайте структурируем оптимизации о которых я попытался рассказать:

- Горутина на чтение с буфером внутри – дорого.
  
  **Решение**: netpoll (epoll, kqueue); переиспользование буферов.

- Горутина на запись с буфером внутри – дорого.

  **Решение**: стартовать горутину тогда, когда нужно; переиспользование буферов.

- При лавине подключений netpoll не сработает.

  **Решение**: переиспользовать горутины с лимитом на их количество.

- `net/http` не самый быстрый способ обработать Upgrade на WebSocket.

  **Решение**: использовать zero-copy upgrade на "голом" tcp соединении.

Как мог бы выглядеть код сервера:

```go
import (
	"net"
	"github.com/gobwas/ws"
)

ln, _ := net.Listen("tcp", ":8080")

for {
	// Try to accept incoming connection inside free pool worker.
	// If there no free workers for 1ms, do not accept anything and try later.
	// This will help us to prevent many self-ddos or out of resource limit cases.
	err := pool.ScheduleTimeout(time.Millisecond, func() {
		conn := ln.Accept()
		_ = ws.UpgradeConn(conn)
		
		// Wrap WebSocket connection with our Channel struct.
		// This will help us to handle/send our app's packets.
		ch := NewChannel(conn)

		// Wait for incoming bytes from connection.
		poller.Add(conn, func() {
			// Do not cross the resource limits.
			pool.Schedule(func() {
				// Read and handle incoming packet(s).
				ch.Recevie()

				// Resume waiting for incoming bytes.
				pool.Resume()
			})
		})
	})
	if err != nil {   
		time.Sleep(time.Millisecond)
	}
}
```

## Заключение

> "Premature optimization is the root of all evil (or at least most of it) in programming." (c) Donald Knuth. 

Безусловно, все оптимизации выше актуальны далеко не во всех случаях. Например, если соотношение свободных ресурсов (память, cpu) к количеству живых соединений значительно велико, наверное, не имеет смысла их применять. Однако, знание тех мест, где и что можно улучшить, надеюсь, будет полезным.

## Ссылки

- https://github.com/mailru/easygo
- https://github.com/gobwas/ws
- https://github.com/gobwas/httphead
- https://github.com/gobwas/ws-examples

[net/http:buffers]: https://github.com/golang/go/blob/143bdc27932451200f3c8f4b304fe92ee8bba9be/src/net/http/server.go#L1862-L1868
[net:conn.Read()]: https://github.com/golang/go/blob/release-branch.go1.8/src/net/net.go#L176-L186
[net:fd.Read()]: https://github.com/golang/go/blob/release-branch.go1.8/src/net/fd_unix.go#L245-L257
[net:pollDesc.waitRead()]: https://github.com/golang/go/blob/release-branch.go1.8/src/net/fd_poll_runtime.go#L74-L81
[man:EAGAIN]: http://man7.org/linux/man-pages/man2/read.2.html#ERRORS
[runtime:netpoll]: https://github.com/golang/go/blob/143bdc27932451200f3c8f4b304fe92ee8bba9be/src/runtime/netpoll.go#L14-L20
[runtime:netpoll_epoll]: https://github.com/golang/go/blob/release-branch.go1.8/src/runtime/netpoll_epoll.go
[man:epoll]: http://man7.org/linux/man-pages/man7/epoll.7.html
[go:issue]: https://github.com/golang/go/issues/15735#issuecomment-266574151
[rfc6455:frame]: https://tools.ietf.org/html/rfc6455#section-5.2
[net/http:Request]: https://github.com/golang/go/blob/release-branch.go1.8/src/net/http/request.go#L100-L305
[net/http:Header]: https://github.com/golang/go/blob/release-branch.go1.8/src/net/http/header.go#L19
[runtime:stack]: https://github.com/golang/go/blob/release-branch.go1.8/src/runtime/stack.go#L64-L82
