# Компоненты и Свойства

Пока что мы строили всё наше приложения в одном единственном компоненте. В малюсеньких примерах это нормально,
но любом реальном приложении вам нужно будет разбивать пользовательский интерфейс на компоненты, чтобы разбить его на 
более маленькие, переиспользуемые, компонуемые куски.

Взять к примеру наш индикатор выполнения. Представьте, два индикатора выполнения вместо одного: один 
пусть движется вперёд по одному тику за клик, а другой по два тика за клик.

Этого _можно_ добиться просто создав два элемента  `<progress>`:

```rust
let (count, set_count) = create_signal(0);
let double_count = move || count() * 2;

view! {
    <progress
        max="50"
        value=count
    />
    <progress
        max="50"
        value=double_count
    />
}
```

Но конечно же это не слишком хорошо масштабируется. Если хотите добавить третий индикатор выполнения, 
придётся добавить этот код ещё раз. И если захотите в нём что-либо изменить, придётся менять это в трёх местах.

Вместо этого, давайте создадим компонент `<ProgressBar/>`.

```rust
#[component]
fn ProgressBar() -> impl IntoView {
    view! {
        <progress
            max="50"
            // hmm... where will we get this from?
            value=progress
        />
    }
}
```

Есть только одна проблема: `progress` не задана. Откуда она должна появиться?
Когда мы задавали всё вручную, мы просто использовали локальные имена переменных.
Нам нужен какой-то способ передать аргумент в наш компонент.

## Свойства Компонента

Мы делаем это используя свойства компонента или "props". Если когда-нибудь использовали другой frontend фреймворк,
эта идея наверняка для вас не нова. Попросту говоря, свойства для компонентов это то же, что атрибуты для HTML элементов:
 они позволяют вам передавать дополнительную информацию в компонент.

Свойства в Leptos объявляются путём добавления дополнительных аргументов в функцию компонента.

```rust
#[component]
fn ProgressBar(
    progress: ReadSignal<i32>
) -> impl IntoView {
    view! {
        <progress
            max="50"
            // now this works
            value=progress
        />
    }
}
```

Теперь мы можем использовать наш компонент во view нашего основного компонента `<App/>`.

```rust
#[component]
fn App() -> impl IntoView {
    let (count, set_count) = create_signal(0);
    view! {
        <button on:click=move |_| { set_count.update(|n| *n += 1); }>
            "Click me"
        </button>
        // now we use our component!
        <ProgressBar progress=count/>
    }
}
```

Использование компонента во view очень похоже на использование HTML элемента. Как можно заметить,
элементы и компоненты легко отличить, поскольку компоненты всегда имеют имена вида `PascalCase`. Свойство `progress`
передаётся как если бы это был атрибут HTML элемента. Просто.

### Реактивные и Статические Свойства

Как видно из этого примера, `progress` принимает реактивный тип `ReadSignal<i32>`, а не обычный `i32`. Это **очень важно**. 

Свойства компонентов не наделены особым смыслом. Компонент это просто функция, которая выполняется единожды для установки 
пользовательского интерфейса. Единственный способ попросить интерфейс реагировать на изменения это передать ему сигнальный тип.
Так что если у вас есть свойство компонента, которое со временем будет меняться, как наше `progress`, оно должно быть сигналом. 

### Необязательные свойства (`optional`)

Сейчас настройка `max` жестко задана в коде. Давайте же и её станем принимать в качестве свойства. Но с подвохом: 
давайте сделаем это свойство необязательным, добавив аннотацию `#[prop(optional)]` к аргументу функции нашего компонента.

```rust
#[component]
fn ProgressBar(
    // mark this prop optional
    // you can specify it or not when you use <ProgressBar/>
    #[prop(optional)]
    max: u16,
    progress: ReadSignal<i32>
) -> impl IntoView {
    view! {
        <progress
            max=max
            value=progress
        />
    }
}
```

Теперь мы можем написать `<ProgressBar max=50 progress=count/>`, или же мы можем пропустить `max`
чтобы использовать значение по-умолчанию (т.е. `<ProgressBar progress=count/>`). Значением по-умолчанию у
необязательного свойства это его значение `Default::default()`, которое для `u16` будет `0`. 
В случае с индикатором выполнения, максимальное значение `0` не очень-то полезно.

Так что давайте зададим вместо него своё значение по-умолчанию.

### Свойства с `default`

Назначить значение по-умолчанию отличное от `Default::default()` можно с помощью `#[prop(default = ...)`.

```rust
#[component]
fn ProgressBar(
    #[prop(default = 100)]
    max: u16,
    progress: ReadSignal<i32>
) -> impl IntoView {
    view! {
        <progress
            max=max
            value=progress
        />
    }
}
```

### Свойства с обобщенными типами

Прекрасно. Но мы начали с двух счетчиков: один движим `count`, а другой производным сигналом `double_count`.
Давайте воссоздадим это путем передачи `double_count` в качестве свойства `progress` в ещё одном `<ProgressBar/>`.

```rust
#[component]
fn App() -> impl IntoView {
    let (count, set_count) = create_signal(0);
    let double_count = move || count() * 2;

    view! {
        <button on:click=move |_| { set_count.update(|n| *n += 1); }>
            "Click me"
        </button>
        <ProgressBar progress=count/>
        // add a second progress bar
        <ProgressBar progress=double_count/>
    }
}
```

Хм... не компилируется. Понять почему довольно просто: мы объявили что свойство `progress` принимает `ReadSignal<i32>`, 
а `double_count` имеет тип не `ReadSignal<i32>`. Как сообщит вам `rust-analyzer`, её тип   
`|| -> i32`, то есть, замыкание, которое возвращает `i32`.

Есть пара способов справиться с этим. Первый это сказать: — Ну, я знаю, что `ReadSignal` это функция и я знаю, что
замыкание это функция; может мне просто принимать любую функцию?
Подкованные в вопросе могут знать, что оба типа реализуют типаж `Fn() -> i32`. 
Так что можно использовать обобщенный компонент:

```rust
#[component]
fn ProgressBar(
    #[prop(default = 100)]
    max: u16,
    progress: impl Fn() -> i32 + 'static
) -> impl IntoView {
    view! {
        <progress
            max=max
            value=progress
        />
    }
}
```

Это вполне оправданный способ написать данный компонент: `progress` теперь принимает любое значение реализующее типаж `Fn()`.

> Обобщенные свойства могут также быть определены используя `where` или используя inline обобщения такие как `ProgressBar<F: Fn() -> i32 + 'static>`.
> Учтите, что поддержка `impl Trait` синтаксиса была выпущена в версии 0.6.12; если вы получите сообщение об ошибке, возможно вам нужно сделать `cargo update` чтобы удостовериться в том, что у вас последняя версия.

Обобщения должны быть использованы где-то в свойствах компонента. Это потому, что свойства встраиваются в структуру, 
так что все обобщенные типы должны быть использованы в структуре. Зачастую этого легко добиться используя
 необязательное свойство с типом `PhantomData`. Потом можно указать обобщенный тип во view используя
синтаксис для выражения типов: `<Component<T>/>` (но не в turbofish-стиле `<Component::<T>/>`).

```rust
#[component]
fn SizeOf<T: Sized>(#[prop(optional)] _ty: PhantomData<T>) -> impl IntoView {
    std::mem::size_of::<T>()
}

#[component]
pub fn App() -> impl IntoView {
    view! {
        <SizeOf<usize>/>
        <SizeOf<String>/>
    }
}
```

> Следует учитывать, что есть некоторые ограничения. Например, наш парсер макроса view не умеет обрабатывать 
> вложенные обобщенные типы как `<SizeOf<Vec<T>>/>`.

### Свойства с `into`

Есть ещё один способ реализовать это: использовать  `#[prop(into)]`.
Этот атрибут автоматически вызывает `.into()` у значений, которая передаются как свойства, что позволяет 
легко передавать свойства разных типов.

В этом случае, полезно будет знать о типе [`Signal`](https://docs.rs/leptos/latest/leptos/struct.Signal.html). `Signal` это перечисляемый тип, 
способный представить читаемый реактивный сигнал любого вида. Его полезно использовать во время определения API для компонентов,
которые вы захотите переиспользовать передавая в них разные виды сигналов. Тип [`MaybeSignal`](https://docs.rs/leptos/latest/leptos/enum.MaybeSignal.html) полезен когда вы
хотите иметь возможность принимать либо статическое либо реактивное значение.

```rust
#[component]
fn ProgressBar(
    #[prop(default = 100)]
    max: u16,
    #[prop(into)]
    progress: Signal<i32>
) -> impl IntoView
{
    view! {
        <progress
            max=max
            value=progress
        />
    }
}

#[component]
fn App() -> impl IntoView {
    let (count, set_count) = create_signal(0);
    let double_count = move || count() * 2;

    view! {
        <button on:click=move |_| { set_count.update(|n| *n += 1); }>
            "Click me"
        </button>
        // .into() converts `ReadSignal` to `Signal`
        <ProgressBar progress=count/>
        // use `Signal::derive()` to wrap a derived signal
        <ProgressBar progress=Signal::derive(double_count)/>
    }
}
```

### Необязательные обобщенные свойства

Учтите, что нельзя объявлять необязательные обобщенные свойства. Давайте посмотрим, что будет если всё же попробовать:

```rust,compile_fail
#[component]
fn ProgressBar<F: Fn() -> i32 + 'static>(
    #[prop(optional)] progress: Option<F>,
) -> impl IntoView {
    progress.map(|progress| {
        view! {
            <progress
                max=100
                value=progress
            />
        }
    })
}

#[component]
pub fn App() -> impl IntoView {
    view! {
        <ProgressBar/>
    }
}
```

Rust услужливо выдаст ошибку:

```
xx |         <ProgressBar/>
   |          ^^^^^^^^^^^ cannot infer type of the type parameter `F` declared on the function `ProgressBar`
   |
help: consider specifying the generic argument
   |
xx |         <ProgressBar::<F>/>
   |                     +++++
```

Обобщения в компонентах можно задать используя синтаксис `<ProgressBar<F>/>` (в макросе `view` без turbofish).
Задание корректного типа здесь невозможно; замыкания и функции в целом являются неименуемыми типами.
Компилятор может отобразить их с помощью сокращения, но их нельзя задать.

Однако, это можно обойти передавая конкретный тип `Box<dyn _>` или `&dyn _`:

```rust
#[component]
fn ProgressBar(
    #[prop(optional)] progress: Option<Box<dyn Fn() -> i32>>,
) -> impl IntoView {
    progress.map(|progress| {
        view! {
            <progress
                max=100
                value=progress
            />
        }
    })
}

#[component]
pub fn App() -> impl IntoView {
    view! {
        <ProgressBar/>
    }
}
```

Поскольку компилятор Rust теперь знает конкретный тип свойства, а следовательно и его размер в памяти даже в случае `None`, это без проблем компилируется.

> Конкретно в данном случае, `&dyn Fn() -> i32` вызовет проблемы с временем жизни ссылки, но в других случаях это может быть допустимо.

## Документирование Компонентов

Это один из наименее необходимых, но наиболее важных разделов этой книги. 
Документирование компонентов и свойств не является строго необходимым.
Оно может оказаться очень важным — зависит от размера команды и приложения.
Но это очень просто и приносит плоды незамедлительно.   

Чтобы задокументировать компонент и его свойства достаточно просто добавить doc-комментарии
перед компонентом и перед каждым свойством:

```rust
/// Shows progress toward a goal.
#[component]
fn ProgressBar(
    /// The maximum value of the progress bar.
    #[prop(default = 100)]
    max: u16,
    /// How much progress should be displayed.
    #[prop(into)]
    progress: Signal<i32>,
) -> impl IntoView {
    /* ... */
}
```

Это всё, что вам нужно. Эти комментарии ведут себя как обычные Rust doc-комментарии, за исключением того,
что в отличие от аргументов Rust функций, можно задокументировать каждое свойство в отдельности.

Документация будет автоматически сгенерирована для вашего компонента, его структуры `Props` и для каждого поля,
использованного для добавления свойства. и каждое из аргументов использованных для добавления свойств. Осознать 
насколько это мощная штука может быть сложновато, поэтому можно навести мышь на имя компонента или свойства и узреть мощь компонента
`#[component]` вкупе с `rust-analyzer`.

> #### Продвинутая Тема: `#[component(transparent)]`

>
> Все Leptos компоненты возвращают `-> impl IntoView`. Хотя некоторым нужно возвращать данные напрямую без какой-либо
> дополнительной обёртки. Они могут быть помечены с помощью `#[component(transparent)]`, в этом они будут возвращать ровно 
> то, что они возвращают без какой-либо обработки со стороны системы рендеринга
>
> Это используется по большей части в двух ситуациях:
>
> 1. Создание обёрток над `<Suspense/>` или `<Transition/>`, которые возвращают прозрачную suspense-структуру 
> для должной интеграции с SSR и гидратацией.
> 
> 2. Рефакторинг определений `<Route/>` для `leptos_router` в отдельные компоненты, поскольку `<Route/>` это прозрачный компонент,
> возвращающий структуру `RouteDefinition`, а не view.
>
> В общем случае вам не нужно использовать прозрачные компоненты, если вы не создаёте кастомные компоненты-обёртки, подпадающие 
> под одну из этих двух категорий.

```admonish sandbox title="Live example" collapsible=true

[Нажмите, чтобы открыть CodeSandbox.](https://codesandbox.io/p/sandbox/3-components-0-5-5vvl69?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  Пожалуйста, включите Javascript для просмотра примеров.
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/3-components-0-5-5vvl69?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>Код примера CodeSandbox</summary>

```rust
use leptos::*;

// Composing different components together is how we build
// user interfaces. Here, we'll define a reusable <ProgressBar/>.
// You'll see how doc comments can be used to document components
// and their properties.

/// Shows progress toward a goal.
#[component]
fn ProgressBar(
    // Marks this as an optional prop. It will default to the default
    // value of its type, i.e., 0.
    #[prop(default = 100)]
    /// The maximum value of the progress bar.
    max: u16,
    // Will run `.into()` on the value passed into the prop.
    #[prop(into)]
    // `Signal<T>` is a wrapper for several reactive types.
    // It can be helpful in component APIs like this, where we
    // might want to take any kind of reactive value
    /// How much progress should be displayed.
    progress: Signal<i32>,
) -> impl IntoView {
    view! {
        <progress
            max={max}
            value=progress
        />
        <br/>
    }
}

#[component]
fn App() -> impl IntoView {
    let (count, set_count) = create_signal(0);

    let double_count = move || count() * 2;

    view! {
        <button
            on:click=move |_| {
                set_count.update(|n| *n += 1);
            }
        >
            "Click me"
        </button>
        <br/>
        // If you have this open in CodeSandbox or an editor with
        // rust-analyzer support, try hovering over `ProgressBar`,
        // `max`, or `progress` to see the docs we defined above
        <ProgressBar max=50 progress=count/>
        // Let's use the default max value on this one
        // the default is 100, so it should move half as fast
        <ProgressBar progress=count/>
        // Signal::derive creates a Signal wrapper from our derived signal
        // using double_count means it should move twice as fast
        <ProgressBar max=50 progress=Signal::derive(double_count)/>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
