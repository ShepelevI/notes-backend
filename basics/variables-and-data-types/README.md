`type switch` - проверка типа переменной, а не её значения. 

Может быть в виде одного `switch` и множеством `case`:

    package main

    func checkType(i interface{}) {
        switch i.(type) {
        case int:
            println("is integer")

        case string:
            println("is string")

        default:
            println("has unknown type")
        }
    }

Может быть в виде `if` конструкции:

    package main

    func main() {
        var any interface{}

        any = "foobar"

        if s, ok := any.(string); ok {
            println("this is a string:", s)
        }

        // так можно проверить наличие функций у структуры
        if closable, ok := any.(interface{ Close() }); ok {
            closable.Close()
        }
    }

**Дополнительный блок фигурных скобок в функции** можно использовать, означает **отдельный скоуп** для всех переменных
объявленных в нём (возможен и "захват переменных" объявленных вне скоупа ранее). 
Иногда используется, например, для декомпозиции какого-то отдельного куска функции.

    var i, s1 = 1, "foo"

    {
        var j, s2 = 2, "bar"

        println(i, s1) // 1 foo
        println(j, s2) // 2 bar

        s1 = "baz"
    }

    println(i, s1) // 1 baz
    //println(j, s2) // ERROR: undefined: j and s2

Так же может быть связано с AST (Abstract Syntax Tree) — когда оно строится и происходят SSA (Static Single Assignment)
оптимизации, SSA не работает на всю длину дерева. 
Если слишком длинная функция и по каким-то причинам не можем её декомпозировать,
но можем изолировать какие-то скоупы то, т. о. помогаем SSA произвести оптимизации (если они возможны).

#### Захват переменной

Во вложенном скоупе есть возможность обращаться к переменным, объявленным в скоупе выше (но не наоборот).

Обращение к переменным из вышестоящего скоупа и есть их захват.

Ошибка - использовать значение итератора в цикле:

    var out []*int

    for i := 0; i < 3; i++ {
        out = append(out, &i)
    }

    println(*out[0], *out[1], *out[2]) // 3 3 3

Испраляется путём создания локальной (для скоупа цикла) переменной с копией знаяения итератора:

    var out []*int

    for i := 0; i < 3; i++ {
        i := i // Copy i into a new variable.
        out = append(out, &i)
    }

    println(*out[0], *out[1], *out[2]) // 0 1 2

Теоретически, можно возвращать неограниченное количество значений из функции. 
Есть правила "де-факто", которых следует придерживаться:

* **Последним** значением возвращать ошибку, если её возврат подразумевается;
* **Первым** значением возвращать контекст, если он подразумевается;
* Хорошим тоном является не возвращать **более четырёх** значений;
* Если функция что-то проверяет и возвращает значение + булевый результат проверки —
то результат проверки возвращать последним (пример — `os.LookupEnv(key string) (string, bool)`);
* Если возвращается **ошибка**, то остальные значения возвращать **нулевыми** или `nil`.
