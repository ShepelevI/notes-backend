#### Сериализация

Процесс преобразования объекта в поток байтов для сохранения или передачи.

**Минусы** - нарушение инкапсуляции, т.е. после сериализации "приватные" свойства структур могут быть доступны
для изменения.

Пример в Go - преобразование структур в `json` объекты.
Кроме `json` существуют различные кодеки типа `MessagePack`, `CBOR`.

#### Десериализация

Восстановление объекта/структуры из последовательности байтов.
Синонимом можно считать термин "маршалинг" (от англ. `marshal` — упорядочивать).

#### Context (контекст)

Пакет `context` полезен при взаимодействиях с API и медленными процессами,
особенно в "production-grade" системах. 
С его помощью можно уведомить горутины о необходимости завершить свою работу,
"пошарить" какие-то данные (например, в `middleware`), или организовать работу с таймаутом.

* `context.WithCancel()`

Эта функция создает новый контекст из переданного ей родительского, возвращая первым аргументом новый контекст,
а вторым — функцию "отмены контекста" (при её вызове родительский контекст "отменен" не будет).
Вызывать функцию отмены контекста может не только та функция, которая его создает,
передача кому-то этих cancel-функций часто оказывается полезной.

**Пример**:

Конструктор объекта запускает кучку вспомогательных для создаваемого объекта горутин,
которые необходимо остановить когда на объекте будет вызван метод вроде `Close()`,
удобнее всего это реализовать, создав cancel-функцию в конструкторе, и сохранив её внутри объекта,
для вызова из метода `Close()`.

При вызове функции отмены сам контекст и все контексты, созданные на основе него, получат в `ctx.Done()` пустую структуру
и в `ctx.Err()` ошибку `context.Canceled`:

        ctx, cancel := context.WithCancel(context.Background())
        fmt.Println(ctx.Err()) // nil

        cancel()

        fmt.Println(<-ctx.Done())      // {}
        fmt.Println(ctx.Err().Error()) // context canceled

* `context.WithDeadline()`

Так же создает контекст от родительского, который отменится самостоятельно при наступлении переданной временной отметки,
или при вызове функции отмены. Отмена/таймаут затрагивает только сам контекст и его "наследников".
`ctx.Err()` возвращает ошибку `context.DeadlineExceeded`. Полезно для реализации таймаутов:

        ctx, cancel := context.WithDeadline(
            context.Background(),
            time.Now().Add(time.Millisecond*100),
        )
        defer cancel()
        fmt.Println(ctx.Err()) // nil

        <-time.After(time.Microsecond * 110)

        fmt.Println(<-ctx.Done())      // {}
        fmt.Println(ctx.Err().Error()) // context deadline exceeded

* `context.WithTimeout()`

Работает аналогично `context.WithDeadline()` за исключением того, что принимает в качестве значения таймаута длительность
(например — `time.Second`):

        ctx, cancel := context.WithTimeout(context.Background(), time.Second*2)

* `context.WithValue()`

Позволяет "пошарить" данные через всё контекстное дерево "ниже". 
Часто используют, чтоб передать, например, логгер или HTTP запрос в цепочке `middleware` 
(**в 9 из 10 случаев так делать не надо, можно считать антипаттерном**). 
Лучше использовать функции для помещения/извлечения данных из контекста ("в нём" они хранятся как `interface{}`):

        import (
            "context"
            "log"
            "os"
        )

        const loggerCtxKey = "logger" // should be unique

        func PutLogger(ctx context.Context, logger *log.Logger) context.Context {
            return context.WithValue(ctx, loggerCtxKey, logger)
        }

        func GetLogger(ctx context.Context) *log.Logger {
            return ctx.Value(loggerCtxKey).(*log.Logger)
        }

        func f(ctx context.Context) {
            logger := GetLogger(ctx)

            logger.Print("inside f")
            println(logger)
        }

        func main() {
            var (
                logger        = log.New(os.Stdout, "", 0)
                ctxWithLogger = PutLogger(context.Background(), logger)
        )

            logger.Printf("main")
            println(logger)

            f(ctxWithLogger)

            // main
            // 0xc0000101e0
            // inside f
            // 0xc0000101e0
        } 

**НО** функции нужны не ради приведения `interface{}` к нужному типу, а для того,
чтобы можно было использовать не экспортируемое значение для ключа внутри контекста -
это защищает от конфликтов по ключу с другими пакетами и от неконтролируемого изменения значений принадлежащих
этому пакету другими.