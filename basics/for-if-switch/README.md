#### Прерывание for/switch или for/select

    for {
        switch f() {
        case true:
            break
        case false:
            // Do something
        }
    }

В примере, если `f()` вернёт `true`, будет вызван `break`, но прерван будет `switch`, а не цикл `for`. 
Решение проблемы – использовать именованный (labeled) цикл и вызывать `break` c этой меткой:

    loop:
        for {
            switch f() {
            case true:
                break loop
            case false:
                // Do something
            }
        }