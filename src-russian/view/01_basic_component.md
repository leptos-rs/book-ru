# Простой компонент

Пример "Привет, Мир!" был _очень_ простым. Давайте перейдём к чему-то больше напоминающему настоящее приложение.

Для начала давайте изменим функцию `main` так чтобы вместо рендеринга всего приложения, она рендерила лишь
компонент `<App/>`.  Компоненты являются основной единицей композиции и дизайна в большинстве Веб-фреймворков: они 
представляют секцию DOM с независимым, обозначенным поведением. В отличие от HTML элементов, они именуются 
с помощью `PascalCase`, так что большинство Leptos приложений начинаются с компонента вроде `<App/>`.


```rust
fn main() {
    leptos::mount_to_body(|| view! { <App/> })
}
```

Теперь давайте определим сам  `<App/>` компонент. Поскольку это сравнительно просто, я сначала покажу вам его целиком,
а затем мы пройдемся по нему строка за строкой.

```rust
#[component]
fn App() -> impl IntoView {
    let (count, set_count) = create_signal(0);

    view! {
        <button
            on:click=move |_| {
                // в ветке stable, это будет выглядеть как set_count.set(3);
                set_count(3);
            }
        >
            "Click me: "
            // в ветке stable, это будет выглядеть как {move || count.get()}
            {move || count()}
        </button>
    }
}
```

## Сигнатура Компонента

```rust
#[component]
```
Как и все определения компонентов, это определение начинается с макроса [`#[component]`](https://docs.rs/leptos/latest/leptos/attr.component.html). `#[component]` добавляет
аннотации к функции чтобы она могла использоваться как компонент в вашем Leptos приложении. Мы рассмотрим некоторые
другие способности этого макроса через пару глав.

```rust
fn App() -> impl IntoView
```

Каждый компонент это функция со следующими характеристиками

1. Принимает ноль или более аргументов любого типа.
2. Возвращает `impl IntoView`, непрозрачный тип, включающий в себя всё, что может быть возвращено из блока Leptos `view`.

> Аргументы функции-компонента собираются вместе в одну структуру, которая надлежащим образом строится макросом `view`.


## Тело Компонента

Телом функции-компонента является установочная функция, которая выполняется лишь раз, в отличие от рендерных функций, 
которые могут повторно выполняться множество раз. Вы обычно будете её использовать, чтобы создать ряд реактивных переменных,
определить побочные эффекты, которые выполняются в ответ на изменения значений этих переменных, а также для описания 
пользовательского интерфейса.

```rust
let (count, set_count) = create_signal(0);
```

[`create_signal`](https://docs.rs/leptos/latest/leptos/fn.create_signal.html)
создаёт сигнал, основную единицу реактивных изменений и управления состоянием в Leptos.
Она возвращает кортеж вида `(getter, setter)`. Чтобы получить доступ к текущему значению используйте `count.get()`
(или краткий вариант `count()`, доступный в `nightly` версии Rust). Чтобы задать текущее значение вызывайте
`set_count.set(...)` (или `set_count(...)`).

> `.get()` клонирует значение, а `.set()` перезаписывает его. Зачастую более эффективным является использовать  `.with()` или `.update()`; 
> ознакомьтесь с документацией к [`ReadSignal`](https://docs.rs/leptos/latest/leptos/struct.ReadSignal.html) и 
> к [`WriteSignal`](https://docs.rs/leptos/latest/leptos/struct.WriteSignal.html) если хотите узнать больше
> об этих компромиссах уже сейчас.

## View (Представление)

Leptos описывает пользовательские интерфейсы с помощью JSX-подобного формата через макрос [`view`](https://docs.rs/leptos/latest/leptos/macro.view.html).

```rust
view! {
    <button
        // объявим слушатель события с помощью on:
        on:click=move |_| {
            set_count(3);
        }
    >
        // текстовые узлы обрачиваются в кавычки
        "Click me: "
        // блоки могут содержать Rust код
        {move || count()}
    </button>
}
```

Это должно быть по большей части просто для понимания: выглядит как HTML со специальным свойством `on:click`
задающим слушатель события `click`, текстовым узлом отформатированным в виде Rust строки, а дальше...

```rust
{move || count()}
```

что бы это ни значило.

Люди иногда шутят, что они используют больше замыканий в своём первом Leptos приложении чем 
они использовали их за всю жизнь до этого. И это подмечено справедливо. Попросту говоря,
передача функции во view говорит фреймворку: — Эй, это что-то что может измениться.   

Когда мы нажимаем на кнопку и вызываем `set_count()`, сигнал `count` обновляется. Замыкание `move || count()`
, чье значение зависит от значения `count`, выполняется повторно, и фреймворк совершает точечное изменение этого текстового узла,
 не трогая ничего больше в вашем приложении. Это то, что позволяет совершать невероятно низкозатратные изменения DOM.

Если у вас включен Clippy или просто намётан глаз, вы могли заметить, что данное замыкание избыточно, по крайней мере если
вы используете `nightly` версию Rust. Если используете `nightly` версию Rust, сигналы уже являются функциями,
так что данное замыкание не требуется. Как итог, вы можете использовать view более простого вида:

```rust
view! {
    <button /* ... */>
        "Click me: "
        // identical to {move || count()}
        {count}
    </button>
}
```

Запомните — и это очень важно — только функции обладают реактивностью. Это означает, что `{count}` и `{count()}` ведут себя
очень по-разному в макросе view. `{count}` передает функцию, говоря фреймворку обновлять view каждый раз когда `count` изменяется.
`{count()}` получает значение `count` единожды и передает `i32` во view, лишь единожды вызывая его рендеринг, без реактивности.
Разница видна в CodeSandbox примере ниже!

Давайте же внесём одно последнее изменение. `set_count(3)` весьма бесполезная вещь в обработчике нажатия. 
Давайте заменим "задать 3 в качестве значение" на "инкрементировать значение на 1":

```rust
move |_| {
    set_count.update(|n| *n += 1);
}
```

Как можете видеть, когда как `set_count` просто задает значение, `set_count.update()` дает нам мутабельную ссылку и 
мутирует значение не двигая его.  Оба варианта вызывают реактивное обновление нашего UI.


> По ходу этого руководства мы будем использовать CodeSandbox для демонстрации интерактивных примеров.
> Наведя на любую из переменных вы увидите подробности от Rust-Analyzer
> и документацию того что происходит. Не стесняйтесь форкать примеры и играть с ними сами!

```admonish sandbox title="Живой пример" collapsible=true

[Кликните чтобы открыть CodeSandbox.](https://codesandbox.io/p/sandbox/1-basic-component-3d74p3?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  Пожалуйста, включите Javascript для просмотра примеров.
</noscript>

> Чтобы увидеть браузер в песочнице вам может понадобиться нажать `Add DevTools >
Other Previews > 8080.`

<template>
  <iframe src="https://codesandbox.io/p/sandbox/1-basic-component-3d74p3?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>Код примера CodeSandbox</summary>

```rust
use leptos::*;

// The #[component] macro marks a function as a reusable component
// Components are the building blocks of your user interface
// They define a reusable unit of behavior
#[component]
fn App() -> impl IntoView {
    // here we create a reactive signal
    // and get a (getter, setter) pair
    // signals are the basic unit of change in the framework
    // we'll talk more about them later
    let (count, set_count) = create_signal(0);

    // the `view` macro is how we define the user interface
    // it uses an HTML-like format that can accept certain Rust values
    view! {
        <button
            // on:click will run whenever the `click` event fires
            // every event handler is defined as `on:{eventname}`

            // we're able to move `set_count` into the closure
            // because signals are Copy and 'static
            on:click=move |_| {
                set_count.update(|n| *n += 1);
            }
        >
            // text nodes in RSX should be wrapped in quotes,
            // like a normal Rust string
            "Click me"
        </button>
        <p>
            <strong>"Reactive: "</strong>
            // you can insert Rust expressions as values in the DOM
            // by wrapping them in curly braces
            // if you pass in a function, it will reactively update
            {move || count()}
        </p>
        <p>
            <strong>"Reactive shorthand: "</strong>
            // signals are functions, so we can remove the wrapping closure
            {count}
        </p>
        <p>
            <strong>"Not reactive: "</strong>
            // NOTE: if you write {count()}, this will *not* be reactive
            // it simply gets the value of count once
            {count()}
        </p>
    }
}

// This `main` function is the entry point into the app
// It just mounts our component to the <body>
// Because we defined it as `fn App`, we can now use it in a
// template as <App/>
fn main() {
    leptos::mount_to_body(|| view! { <App/> })
}
```
