## Немного в общих чертах про **Golang** 

Компилируемый многопоточный.

Разработан Google (Роберт Гризмер, Роб Пайк, Кен Томпсон).

Официально представлен в ноябре 2009 года (разработка началась в 2007 году).

### Особенности

* создавался по принципу "что еще можно выкинуть"
* **императивный** (конкретно описываем необходимые действия для достижения определенного результата)
* строгая типизация и отказ от иерархии типов
* сборка мусора (GC)
* простые и эффективные средства для распараллеливания вычислений
* четкое разделение интерфейса и реализации
* наличие системы пакетов и возможность импортировать внешние зависимости (пакеты)
* богатый тулинг "из коробки" (тесты, бенчмарки, быстрая компиляция, генерация кода и документации)
* нет классического ООП, вместо этого предлагается несколько отдельных подходов:
  * встраивание объектов
  * интерфейсы как контракты реализаций
  * пакеты
  * возможность создания методов к типам

===========================================================================================================

**ООП** - методология (подход) программирования, основанная на том,
что программа представляет собой некоторую совокупность объектов-классов,
которые образуют иерархию наследования.

**Ключевые фишки** - минимализация повторяемости кода (**принцип DRY**) и удобство понимания/управления.

**Фундамент ООП** - описание объектов в программировании подобно объектам из реального мира - у них есть свойства,
поведение, они могут взаимодействовать.

**Основные принципы в ООП**:

* **Абстракция** - фокусирование на тех свойствах системы, которые важны в рамках текущей задачи,
а менее существенные отбрасываем (абстракция данных и методов).
* **Инкапсуляция** - заключение данных и функциональности в оболочку,
в ее роли - классы - собирают переменные и методы в одном месте, защищают от вмешательства извне.
* **Наследование** - родительские классы лежат в основе других - дочерних,
при этом, дочерние перенимают свойства и поведение своего родителя.
* **Полиморфизм** - возможность использовать одни и те же методы для объектов разных классов.

==========================================================================================================

**В Golang**:

**Инкапсуляция** представлена возможностью использовать пакеты и видимость полей/методов
для инкапсуляции. Поля и методы, начинающиеся с заглавной буквы, являются публичными,
а с маленькой - приватными:

    package main

    type person struct {
        name string
        age  int
    }

    func (p *person) getName() string {
        return p.name
    }

    func (p *person) setName(name string) {
        p.name = name
    }

    func main() {
        p := &person{name: "John", age: 30}
        fmt.Println(p.getName()) // Выведет "John"
        p.setName("Jane")
        fmt.Println(p.getName()) // Выведет "Jane"
    }

**Полиморфизм** представлен через интерфейсы. Интерфейс определяет набор методов, которые должны быть реализованы типом. 
Тип, реализующий интерфейс, может быть использован везде, где ожидается интерфейс:

    package main

    type speaker interface {
        speak()
    }

    type dog struct {
        name string
    }

    func (d *dog) speak() {
        fmt.Printf("%s гавкает\n", d.name)
    }

    type cat struct {
        name string
    }

    func (c *cat) speak() {
        fmt.Printf("%s мяукает\n", c.name)
    }

    func main() {
        var s speaker
        s = &dog{name: "Buddy"}
        s.speak() // Выведет "Buddy гавкает"

        s = &cat{name: "Kitty"}
        s.speak() // Выведет "Kitty мяукает"
    }

**Наследование** представлено **Композицией** - встариванием одних структур в другие для расширения функциональности:

    package main

    type animal struct {
        name string
    }

    func (a *animal) speak() {
        fmt.Printf("%s говорит\n", a.name)
    }

    type dog struct {
        animal
        breed string
    }

    func main() {
        d := &dog{animal: animal{name: "Buddy"}, breed: "Labrador"}
        d.speak() // Выведет "Buddy говорит"
    }

**Абстракция** представлена использованием интерфейсов для определения абстрактных типов и их реализаций.



