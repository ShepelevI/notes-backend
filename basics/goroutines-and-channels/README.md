#### Примитивы синхронизации

#### Mutex

`Mutex` означает **MUTual EXclusion** (взаимное исключение), и обеспечивает безопасный доступ к общим ресурсам.

Под капотом мьютекса используются функции из пакета `atomic` (`atomic.CompareAndSwapInt32` и `atomic.AddInt32`),
можно считать `Mutex` надстройкой над `atomic`. `Mutex` медленнее чем `atomic`,
потому что он блокирует другие горутины на всё время действия блокировки.
В свою очередь `atomic` быстрее, потому что использует атомарные инструкции процессора.

В момент, когда нужно обеспечить защиту доступа — вызываем метод `Lock()`,
по завершению операции изменения/чтения данных — метод `Unlock()`.

**Отличие** `sync.Mutex` от `sync.RWMutex`:

Кроме `Lock()` и `Unlock()` у `sync.Mutex`, у `sync.RWMutex` есть отдельные аналогичные методы только для чтения
— `RLock()` и `RUnlock()`.
Если участок в памяти нуждается только в чтении — он использует `RLock()`,
который не заблокирует другие операции чтения, но заблокирует операцию записи и наоборот.

#### sync.WaitGroup

Из стандартной библиотеки доступно - `sync.WaitGroup`, используется для координации в случае,
когда программе приходится ждать окончания работы нескольких горутин
(конструкция похожа на `CountDownLatch` в Java).
Отличный способ дождаться завершения набора одновременных операций.

**Принцип работы**:

        var wg sync.WaitGroup

        wg.Add(1) // увеличиваем счётчик на 1
        go func() {
            fmt.Println("task 1")
            <-time.After(time.Second)
            fmt.Println("task 1 done")

            wg.Done() // уменьшаем счётчик на 1
        }()

        wg.Add(1) // увеличиваем счётчик на 1
        go func() {
            fmt.Println("task 2")
            <-time.After(time.Second)
            fmt.Println("task 2 done")

            wg.Done() // уменьшаем счётчик на 1
        }()

        wg.Wait() // блокируемся, пока счётчик не будет == 0
        // task 2
        // task 1
        // task 2 done
        // task 1 done
        // Total time: 1.00s

#### sync.Cond

Условная переменная (**CONDition variable**) полезна, например,
если хотим разблокировать сразу несколько горутин (`Broadcast`),
что не получится сделать с помощью канала.
Метод `Signal` отправляет сообщение самой долго-ожидающей горутине.

**Пример**:

        var (
            c  = sync.NewCond(&sync.Mutex{})
            wg sync.WaitGroup // нужна только для примера

            free = true
        )

        wg.Add(1)
        go func() {
            defer wg.Done()
            c.L.Lock()

            for !free { // проверяем, что ресурс свободен
                c.Wait()
            }
            fmt.Println("work")

            c.L.Unlock()
        }()

        free = false                  // забрали ресурс, чтобы выполнить с ним работу
        <-time.After(1 * time.Second) // эмуляция работы
        free = true                   // освободили ресурс
        c.Signal()                    // оповестили горутину

        wg.Wait()

#### sync.Once

Позволяет определить задачу для однократного выполнения за всё время работы программы. 
Содержит единственную функцию `Do`, позволяющую передавать другую функцию для однократного применения:

        var once sync.Once

        for i := 0; i < 10; i++ {
            once.Do(func() {
                fmt.Println("Hell yeah!")
            })
        }

        // Hell yeah! (выводится 1 раз вместо 10)

#### sync.Pool

Используется для уменьшения давления на `GC` путём повторного использования выделенной памяти (**потокобезопасно**).
Пул необязательно освободит данные при первом пробуждении `GC`, но он может освободить их в любой момент.
У пула нет возможности определить и установить размер, и нет необходимости заботиться о его переполнении.

#### Channel (канал)

Объект связи, с помощью которого (чаще всего) горутины обмениваются данными.

Потокобезопасен, передаётся "по указателю".

Технически можно представить как конвейер (или трубу), откуда можно считывать и помещать данные.

Для создания канала использовать ключевое слово `chan`.

Бывают синхронные - **не буферизированные** и асинхронные - **буферизированные**,
работают по принципу **FIFO (first in, first out)** очереди.

**Буферизованные** каналы работают как **асинхронные** пока не заполнится буфер,
после чего они превращаются в **синхронные** пока в буфере не появится свободное место (если появится).

Создание **не буферизированного** канала `c := make(chan int)`.

Чтение из канала — `data := <-c`.

Запись в канал — `c <- 123`.

Закрытие канала - `close(c)`.

Запись данных в **закрытый** канал или повторное закрытие канала вызовет `panic`.

Чтение или запись данных в **не буферизированный** канал блокирует горутину, контроль передается свободной горутине.

Через закрытый канал **невозможно передать** данные, **можно вычитать** из **буферизованного** канала данные,
отправленные в него **до закрытия**.

Проверить открытость канала и прочитать можно используя
`val, isOpened := <- channel`, где `isOpened == true` в том случае, если канал открыт,
в противном случае вернётся `false` и нулевое значение `val` исходя из типа данных для канала,
`isOpened == false`, если канал закрыт и отсутствуют данные для чтения из него.

Если канал **закрытый**, но **буферизованный** и **содержит значения** - эта проверка не сработает пока,
не будут вычитаны все значения из буфера.

**Буферизированный** канал создается указанием второго аргумента для `make` — `c := make(chan int, 5)`,
в этом случае горутина не блокируется до тех пор, пока буфер не будет заполнен.
Подобно слайсам, буферизированный канал имеет длину (`len`, количество сообщений в очереди, не считанных)
и емкость (`cap`, размер самого буфера канала):

        c := make(chan string, 5)

        c <- "foo"
        c <- "bar"
        close(c)

        println(len(c), cap(c)) // 2 5

        for {
            val, ok := <-c // обрати внимание - читаем из уже закрытого канала

            if !ok {
                break
            }

            println(val)
        }
        // "foo"
        // "bar"

`ok == true` до того момента, пока в канале есть сообщения
(независимо от того, открыт он или закрыт), в противном случае `ok == false`,
а `val` будет **нулевым значением** в зависимости от **типа данных** канала.

Используя **буферизованный** канал и цикл `for val := range c { ... }` можно читать с **закрытых** каналов
(у закрытых каналов данные все еще живут в буфере).

При попытке **чтения** через `range` из **закрытого** канала, в котором **нет значений**, цикл будет **прерван**.

Из **закрытого** канала можно **читать**: 

* с помощью: `for val := range c { ... }` — вычитает все сообщения что в нём есть;
* с помощью:

      for {
          if val, ok := <-c; ok {
              println(val)
          } else {
              break
          }
      }

Синтактический сахар **однонаправленных** каналов:

* `c <-chan int` — только для чтения;
* `c chan<- int` — только для записи.

Приведение типа из обычного канала в ограниченный чтением или записью (`func write(c chan<- string) { ... }`),
при вызове функции выполняется неявное приведение типа.

Читать "одновременно" из нескольких каналов (как и писать) возможно с помощью `select` 
(оператор `select` является блокируемым, за исключением использования `default`)

        c1, c2 := make(chan string), make(chan string)
        defer func() { close(c1); close(c2) }() // не забываем закрыть

        go func(c chan<- string) { <-time.After(time.Second); c <- "foo" }(c1)
        go func(c chan<- string) { <-time.After(time.Second); c <- "bar" }(c2)

        for i := 1; ; i++ {
            select { // блокируемся, пока в один из каналов не попадёт сообщение
            case val := <-c1:
                println("channel 1", val)

            case val := <-c2:
                println("channel 2", val)
        }

            if i >= 2 { // через 2 итерации выходим (иначе будет deadlock)
                break
            }
        }
        // channel 1 foo
        // channel 2 bar
        // Total execution time: 1.00s

Если в оба канала одновременно придут сообщения (или они уже там были), то `case` будет выбран случайно, 
а не по порядку их объявления.

Если ни один из каналов недоступен для взаимодействия, и секция `default` отсутствует,
то текущая горутина переходит в состояние `waiting` до тех пор, пока какой-то из каналов не станет доступен.

Если в `select` указан `default`, то он будет выбран в том случае,
если все каналы не имеют сообщений (т. о. `select` становится **не блокируемым**).

В исходниках канал представлен структурой:

        type hchan struct {
        qcount   uint           // количество элементов в буфере
        dataqsiz uint           // размерность буфера
        buf      unsafe.Pointer // указатель на буфер для элементов канала
        elemsize uint16         // размер одного элемента в канале
        closed   uint32         // флаг, указывающий, закрыт канал или нет
        elemtype *_type         // содержит указатель на тип данных в канале
        sendx    uint           // индекс (смещение) в буфере, по которому должна производиться запись
        recvx    uint           // индекс (смещение) в буфере, по которому должно производиться чтение
        recvq    waitq          // указатель на связанный список горутин, ожидающих чтения из канала
        sendq    waitq          // указатель на связанный список горутин, ожидающих запись в канал
        lock     mutex          // мьютекс для безопасного доступа к каналу
        }

В общем случае, горутина захватывает мьютекс, когда совершает какое-либо действие с каналом,
кроме случаев `lock-free` проверок при неблокирующих вызовах.

Go не выделяет буфер для синхронных (**не буферизированных**) каналов,
поэтому указатель на буфер равен `nil` и `dataqsiz` равен `0`. 
При чтении из канала горутина произведёт некоторые проверки,
такие как: закрыт ли канал, буферизирован он или нет, содержит ли гоуртины в send-очереди.
Если ожидающих отправки горутин нет — горутина добавит сама себя в `recvq` и заблокируется. 
При записи другой горутиной все проверки повторяются снова, и когда она проверяет `recvq` очередь,
она находит ожидающую чтение горутину, удаляет её из очереди, записывает данные в её стек и снимает блокировку.
Это **единственное** место во всём `runtime Go`, когда одна горутина пишет **напрямую в стек** другой горутины.

При создании асинхронного (**буферизированного**) канала `make(chan bool, 1)` Go выделяет буфер
и устанавливает значение `dataqsiz` в единицу. 
Чтобы горутине отправить отправить значение в канал, сперва производятся несколько проверок: 
пуста ли очередь `recvq`, пуст ли буфер, достаточно ли места в буфере.
Если всё ок, то она записывает элемент в буфер, увеличивает значение `qcount` и продолжает исполнение далее.
Когда буфер полон, **буферизированный** канал будет вести себя точно так же, как синхронный (**не буферизированный**),
т. е. горутина добавит себя в очередь ожидания и заблокируется.

Проверки буфера и очереди реализованы как **атомарные** операции, и **не требуют** блокировки мьютекса.

При **закрытии** канала Go проходит по всем ожидающим на чтение или запись горутинам и разблокирует их.
Все получатели получают дефолтные значение переменных того типа данных канала, а все отправители паникуют.

#### Goroutine scheduler (планировщик горутин)

Перехватывающий задачи (work-stealing) планировщик, введен в Go 1.1 Дмитрием Вьюковым вместе с командой Go.
Основная его суть в том, что он управляет:

* `G` (**горутинами**) — горутины Go;
* `M` (машинами - **потоками** или **тредами**) — потоки ОС, которые могут выполнять что-либо или же бездействовать;
* P (**процессорами**) — можно рассматривать как ЦП (физическое ядро); представляет ресурсы,
необходимые для выполнения Go кода, такие как планировщик или состояние распределителя памяти.

Основная задача планировщика - сопоставить каждую `G` (код, который хотим выполнить) с `M` (где его выполнять)
и `P` (права и ресурсы для выполнения).

Когда `M` (поток ОС) прекращает выполнение кода, он возвращает свой `P` (ЦП) в пул свободных `P`.
Чтобы возобновить выполнение Go кода, он должен повторно заполучить его. 
Точно так же, когда горутина завершается, объект `G` (горутина) возвращается в пул свободных `G`
и позже может быть повторно использован для какой-либо другой горутины.

Go запускает по умолчанию до 10000 тредов `M`, т.к. многие из них могут быть заблокированы в каких-то системных вызовах
(или вызовах `cgo`, или из-за `runtime.LockOSThread`),
и распределяет на эти треды сколько угодно горутин которые уже запускает программист.
В один момент на одном ядре ЦП может находиться в исполнении только одна горутина,
а в очереди исполнения их может быть неограниченное количество.

Треды `M` во время выполнения могут переходить от одного процессора `P` к другому.
Например, когда тред делает системный вызов, в ответ на который ОС блокирует этот тред
(например — чтение какого-то большого файла с диска) — заблокируется сама горутина, которая спровоцировала этот вызов,
также заблокируются все остальные, что стоят в очереди для этого процессора `P`. 
Чтоб этого не происходило Go отвязывает горутины, стоящие в очереди от этого процессора `P`, и переназначает на другие.

Основные **типы многозадачности**, что используются в большинстве **ОС**:

* "**вытесняющая**" (все ресурсы делятся между всеми программами одинаково, всем выделяется одинаковое время выполнения);
* "**кооперативная**" (программы выполняются столько, сколько им нужно, и сами уступают друг другу место).

В Go используется **неявная кооперативность**:

* Горутина уступает место другим при обращении к вводу-выводу, каналам, вызовам ОС и т.д.;
* Может уступить место при вызове любой функции (с некоторой вероятностью произойдет переключение между горутинами);
* Есть явный способ переключить планировщик на другую горутину — вызвать функцию `runtime.Gosched()`
(почти никогда не нужна, но она есть).

Основные **принципы** планировщика:

* Очередь **FIFO (first in — first out)** — порядок запуска горутин обуславливается порядом их вызова;
* Необходимый минимум тредов — создается не больше тредов чем доступных ядер ЦП;
* Захват чужой работы — когда тред простаивает, он не удаляется `runtime Go`, а будет по возможности "нагружен" работой,
взятой из очередей горутин на исполнение с других тредов;
* "Неинвазивность" — работа горутин насильно не прерывается.

**Ограничения**:

* Очередь FIFO (нет приоритезации и изменения порядка исполнения);
* Отсутствие гарантий времени выполнения (времени запуска горутин);
* Горутины могут перемещаться между тредами, что снижает эффективность кэшей.

#### Goroutine (горутина)

Функция, выполняющаяся конкурентно с другими горутинами в том же адресном пространстве.

Для запуска использовать ключевое слово `go` перед именем вызываемой (или анонимной) функции.

Горутины очень легковесны (**~2,6Kb на горутину**).
Практически все расходы — создание стека, который очень невелик, хотя при необходимости может расти. 
Область их применения чаще всего:

* Когда нужна асинхронность (например, когда работаем с сетью, диском, базой данных, защищенным мьютексом ресурсом и т.п.);
* Когда время выполнения функции достаточно велико и можно получить выигрыш, нагрузив другие ядра.

Сама структура горутины занимает порядка **600 байт**, но для неё ещё выделяется и её собственный стек,
минимальный размер котого составляет **2Kb**, который увеличивается и уменьшается по мере необходимости
(максимум зависит от архитектуры и составляет **1 ГБ для 64-разрядных** систем и **250 МБ для 32-разрядных** систем).

Переключение между двумя горутинами **дешевое**, `O(1)`, т. е., не зависит от количества созданных горутин в системе.
Для переключения надо поменять 3 регистра — `Program counter`, `Stack Pointer` и `DX`.

**Отличия горутин от потоков ОС**:

* Каждый поток операционной системы имеет блок памяти фиксированного размера (зачастую **до 2 Мбайт**)
для стека — рабочей области, в которой он хранит локальные переменные вызовов функций,
находящиеся в работе или приостановленные на время вызова другой функции.
В противоположность этому go подпрограмма начинает работу с небольшим стеком, обычно **около 2 Кбайт**.
Стек горутины, подобно стеку потока операционной системы, хранит локальные переменные активных и приостановленных функций,
но, в отличие от потоков операционной системы, не является фиксированным,
при необходимости он может расти и уменьшаться;
* Потоки операционной системы планируются в ее ядре, а у Go есть собственный планировщик (`m:n`)
мультиплексирующий (раскидывающий) горутины (`m)` по потокам (`n`).
Основной плюс — отсутствие оверхеда на переключение контекста;
* Планировщик Go использует параметр с именем `GOMAXPROCS` для определения,
сколько потоков операционной системы могут одновременно активно выполнять код Go.
Его значение по умолчанию равно количеству процессоров (ядер) компьютера,
так что на машине с 8 процессорами (ядрами) планировщик будет планировать код Go для выполнения
на 8 потоках одновременно. Спящие или заблокированные в процессе коммуникации go подпрограммы потоков для себя не требуют.
Go подпрограммы, заблокированные в операции ввода-вывода или в других системных вызовах,
или при вызове функций, не являющихся функциями Go, нуждаются в потоке операционной системы,
но `GOMAXPROCS` их не учитывает;
* В большинстве операционных систем и языков программирования, поддерживающих многопоточность,
текущий поток имеет идентификацию, которая может быть легко получена как обычное значение 
(обычно — целое число или указатель). У горутин нет идентификации, доступной программисту.
Так решено во время проектирования языка, поскольку локальной памятью потока программисты злоупотребляют.

Т. к. горутины являются **stackful**, то память для них (их состояние) хранится на стеке.
Теоретически, если очень постараться и сделать миллиард вложенных вызовов,
то можно сделать себе переполнение стека.

Для самих переменных, что используются внутри горутин, память берётся с `heap`
(ограничены только размером "физического" `heap`, т.е. объемом памяти сколько есть на машине).

**Завершить много горутин**:

Работу одной гороутины в принципе нельзя принудительно остановить из другой.
Механизмы их завершения необходимо реализовывать отдельно (учить сами горутины завершаться).
Завершение `main` не обязательно приводит к завершению других горутин - если из `main` вызвать `runtime.Goexit()`,
то другие горутины продолжат работать

* Использовать контекст `context.Context`:

        import (
            "context"
            "time"
        )

        func f(ctx context.Context) {
        loop:
            for {
                select {
                case <-ctx.Done():
                    println("break f")
                    break loop

                default:
                    println("do some work")
                    <-time.After(time.Millisecond * 100)
                }
            }
        }

        func main() {
            ctx, cancel := context.WithCancel(context.Background())

            for i := 0; i < 3; i++ {
                go f(ctx) // запускаем 3 горутины
            }

            <-time.After(time.Millisecond * 50)
            cancel() // отменяем контекст, на что горутины должны среагировать выходом
            <-time.After(time.Millisecond * 60)

            // do some work
            // do some work
            // do some work
            // break f
            // break f
            // break f
        } 

* Отдельный канал для уведомлений о необходимости завершения
(часто для уведомлений используется пустая структура `struct{}`, которая ничего не весит):


        import (
            "time"
        )

        func f(c <-chan struct{}) {
        loop:
            for {
                select {
                case <-c:
                    println("break f")
                    break loop

                default:
                    println("do some work")
                    <-time.After(time.Millisecond * 100)
                }
            }
        }

        func main() {
            const workersCount = 3

            var c = make(chan struct{}, workersCount)

            for i := 0; i < workersCount; i++ {
                go f(c) // запускаем 3 горутины
            }

            <-time.After(time.Millisecond * 50)

            for i := 0; i < workersCount; i++ {
                c <- struct{}{} // отправляем 3 сообщения в канал (по одному для каждой горутины) о выходе
            }
            // цикл с отправкой сообщений не обязательный, можно просто закрыть канал
            close(c)

            <-time.After(time.Millisecond * 60)

            // do some work
            // do some work
            // do some work
            // break f
            // break f
            // break f
        } 

#### Задетектить гонку

Пишем тесты, запускаем их с флагом `-race` 
(`runtime` будет в случайном порядке переключаться между горутинами (если не ошибаюсь),
компилятор генерирует дополнительный код, который "журналирует" обращения к памяти). 
Этот флаг можно использовать как для `go test`, так и для `go run` или `go build`.

Детектор гонки основан на библиотеке времени выполнения (runtime library) C/C++ ThreadSanitizer.

> "Если `race detector` обнаруживает состояние гонки, то оно у вас наверняка есть, если же не обнаруживает — то это не означает, что его нет".

При наличии нормального покрытия тестами `go test -race` находит большинство проблем,
необходимости писать для этого специальные тесты, как правило, нет. 
Замедление тестов - серьёзная проблема, потому что медленные тесты разработчики избегают запускать,
а тесты нужно запускать часто, чтобы обнаруживать проблемы сразу же после внесения некорректных изменений.