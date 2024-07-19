# Управление глобальным состоянием

Пока что мы работали только с локальным состоянием в компонентах и мы узнали как координировать состояние
между родительным и дочерними компонентами. Время от времени требуется более общее решение для управление глобальным
состоянием, которое может работать по всему приложению.

Вообще, **вам не требуется эта глава**. Обычно приложение строят из компонентов, каждый из которых управляет собственным
локальным состоянием, чтобы не хранить всё состояние в одной глобальной структуре.
Однако, бывают такие случаи (темы интерфейсы, сохранение настроек пользователя или общий доступ к данным между разными частями UI),
когда может захотеться хранить какое-то глобальное состояние.

Вот три лучших подход к глобальному состоянию:

1. Использование Маршрутизатора для управления глобальным состоянием через URL
2. Передача сигналов через контексты
3. Создание глобальной структуры состояния и создания направленных на её линз через `create_slice`

## Вариант №1: URL как глобальное состояние

Во многом URL это действительно лучший способ хранить глобальное состояние. К нему можно получить доступ из любого компонента,
из любой части дерева. Есть такие нативные элементы как `<form>` и `<a>`, существующие исключительно для обновления URL.
И он сохраняется при перезагрузке страницы и при смене устройства; можно поделиться ссылкой с другом или отправить
её с телефона на свой же ноутбук и сохраненное в ней состояние будет восстановлено.

Несколько следующих разделов руководства будут о маршрутизаторе, в них эти темы будут разобраны куда более глубоко.

А пока мы лишь посмотрим на варианты №2 и №3.

## Вариант №2: Передача Сигналов через Контекст

В разделе о [коммуникации Родитель-Потомок](view/08_parent_child.md) мы рассмотрели, что с помощью `provide_context` передать сигнал
из родительского компонента потомку, а с помощью `use_context` получить его в потомке.
Но `provide_context` работает на любом расстоянии. Можно создать глобальный сигнал, хранящий кусочек состояния, вызвать
`provide_context` и иметь к нему доступ отовсюду в потомках компонента, в котором была вызвана функция `provide_context`.

Сигнал предоставляемый с помощью контекста вызывает реактивные обновление лишь там, где он из него читают, а не во всех
компонентах между родителем и читающщим потомком, так что сила мелкозернистых реактивных обновлений сохраняется,
даже на расстоянии.

Начнём с создания сигнала в корне приложения и предоставления его всем потомкам через `provide_context`.

```rust
#[component]
fn App() -> impl IntoView {
    // here we create a signal in the root that can be consumed
    // anywhere in the app.
    let (count, set_count) = create_signal(0);
    // we'll pass the setter to specific components,
    // but provide the count itself to the whole app via context
    provide_context(count);

    view! {
        // SetterButton is allowed to modify the count
        <SetterButton set_count/>
        // These consumers can only read from it
        // But we could give them write access by passing `set_count` if we wanted
        <FancyMath/>
        <ListItems/>
    }
}
```

`<SetterButton/>` это счетчик, который мы уже несколько раз писали.
(См. песочницу ниже если не понимаете о чём я)

`<FancyMath/>` и `<ListItems/>` оба получают предоставляемый сигнал с помощью `use_context` и что-то с ним делают.

```rust
/// A component that does some "fancy" math with the global count
#[component]
fn FancyMath() -> impl IntoView {
    // here we consume the global count signal with `use_context`
    let count = use_context::<ReadSignal<u32>>()
        // we know we just provided this in the parent component
        .expect("there to be a `count` signal provided");
    let is_even = move || count() & 1 == 0;

    view! {
        <div class="consumer blue">
            "The number "
            <strong>{count}</strong>
            {move || if is_even() {
                " is"
            } else {
                " is not"
            }}
            " even."
        </div>
    }
}
```

Отметим, что этот же паттерн можно применить и к более сложным состояниям. Если хочется независимо обновлять сразу несколько полей, то
можно передавать через `provide_context` структуру с сигналами:

```rust
#[derive(Copy, Clone, Debug)]
struct GlobalState {
    count: RwSignal<i32>,
    name: RwSignal<String>
}

impl GlobalState {
    pub fn new() -> Self {
        Self {
            count: create_rw_signal(0),
            name: create_rw_signal("Bob".to_string())
        }
    }
}

#[component]
fn App() -> impl IntoView {
    provide_context(GlobalState::new());

    // etc.
}
```

## Варианта №3: Создание Глобальной Структуры Состояния и Срезы

Оборачивание каждого поля структуры в отдельный сигнал может показаться громоздким.
В некоторых случаях полезно создать обычную структуру с нереактивными полями, а затем обернуть её в сигнал. 

```rust
#[derive(Copy, Clone, Debug, Default)]
struct GlobalState {
    count: i32,
    name: String
}

#[component]
fn App() -> impl IntoView {
    provide_context(create_rw_signal(GlobalState::default()));

    // etc.
}
```

Но с этим есть проблема: поскольку всё наше состояние обёрнуто в единственный сигнал, обновление значения одного поля
приведёт к реактивным обновлениям в частях UI, которые зависят лишь от других полей.

```rust
let state = expect_context::<RwSignal<GlobalState>>();
view! {
    <button on:click=move |_| state.update(|state| state.count += 1)>"+1"</button>
    <p>{move || state.with(|state| state.name.clone())}</p>
}
```

В данном примере нажатие на кнопку вызовет обновление текста внутри  `<p>`, с повторным клонированием `state.name`!
Поскольку сигналы это мельчайшие составные части реактивности, обновление любого поля данного сигнала вызывает обновление
всего, что от него зависит.

Есть способ получше. Можно брать мелкозернистые, реактивные срезы используя [`create_memo`](https://docs.rs/leptos/latest/leptos/fn.create_memo.html) или [`create_slice`](https://docs.rs/leptos/latest/leptos/fn.create_slice.html) (которая использует `create_memo`, но также представляет сеттер). 
“Мемоизация” значения означает создание нового реактивного значения, которое обновляется лишь тогда, когда исходное меняется.
“Мемоизация среза” означает создание новое реактивного значения, которое обновляется лишь тогда, когда определенное поле структуры обновляется.

Вот, вместо чтения из сигнала состояния напрямую мы берём "срезы" этого состояния с мелкозернистыми обновлениями через  `create_slice`.
Каждый сигнал-срез обновляется лишь когда определенная часть структуры меняется. Это значит можно сделать единый корневой сигнал,
а затем брать от него независимые мелкозернистые срезы в различных компонентах, каждый из которых обновляется
не уведомляя других об изменениях. 

```rust
/// A component that updates the count in the global state.
#[component]
fn GlobalStateCounter() -> impl IntoView {
    let state = expect_context::<RwSignal<GlobalState>>();

    // `create_slice` lets us create a "lens" into the data
    let (count, set_count) = create_slice(

        // we take a slice *from* `state`
        state,
        // our getter returns a "slice" of the data
        |state| state.count,
        // our setter describes how to mutate that slice, given a new value
        |state, n| state.count = n,
    );

    view! {
        <div class="consumer blue">
            <button
                on:click=move |_| {
                    set_count(count() + 1);
                }
            >
                "Increment Global Count"
            </button>
            <br/>
            <span>"Count is: " {count}</span>
        </div>
    }
}
```

Нажатие на эту кнопку обновляет лишь  `state.count`, так что если создать где-то ещё срез, берущий лишь `state.name`,
нажатие на кнопку не приведет к обновление этого дополнительного среза. Это позволяет объединить преимущества 
течения данных сверху вниз и мелкозернистых реактивных обновлений.

> **Примечание**: У этого подхода есть существенные недостатки. И сигналам и мемоизированным значениям нужно владеть их значениями,
> так что мемоизированное значение будет клонировать значение поля при каждом изменении. 
> Самый естественный путь управлять состоянием в фреймворке как Leptos это всегда предоставлять сигналы
> настолько локальные и мелкозернистые насколько это возможно, а не поднимать всё что ни попадя в глобальное состояние.
> Но когда _нужно_ какое-то глобальное состояние, `create_slice` может быть полезна.

```admonish sandbox title="Живой пример" collapsible=true

[Нажмите, чтобы открыть CodeSandbox.](https://codesandbox.io/p/sandbox/15-global-state-0-5-8c2ff6?file=%2Fsrc%2Fmain.rs%3A1%2C2)

<noscript>
  Пожалуйста, включите Javascript для просмотра примеров.
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/15-global-state-0-5-8c2ff6?file=%2Fsrc%2Fmain.rs%3A1%2C2" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandbox Source</summary>

```rust
use leptos::*;

// So far, we've only been working with local state in components
// We've only seen how to communicate between parent and child components
// But there are also more general ways to manage global state
//
// The three best approaches to global state are
// 1. Using the router to drive global state via the URL
// 2. Passing signals through context
// 3. Creating a global state struct and creating lenses into it with `create_slice`
//
// Option #1: URL as Global State
// The next few sections of the tutorial will be about the router.
// So for now, we'll just look at options #2 and #3.

// Option #2: Pass Signals through Context
//
// In virtual DOM libraries like React, using the Context API to manage global
// state is a bad idea: because the entire app exists in a tree, changing
// some value provided high up in the tree can cause the whole app to render.
//
// In fine-grained reactive libraries like Leptos, this is simply not the case.
// You can create a signal in the root of your app and pass it down to other
// components using provide_context(). Changing it will only cause rerendering
// in the specific places it is actually used, not the whole app.
#[component]
fn Option2() -> impl IntoView {
    // here we create a signal in the root that can be consumed
    // anywhere in the app.
    let (count, set_count) = create_signal(0);
    // we'll pass the setter to specific components,
    // but provide the count itself to the whole app via context
    provide_context(count);

    view! {
        <h1>"Option 2: Passing Signals"</h1>
        // SetterButton is allowed to modify the count
        <SetterButton set_count/>
        // These consumers can only read from it
        // But we could give them write access by passing `set_count` if we wanted
        <div style="display: flex">
            <FancyMath/>
            <ListItems/>
        </div>
    }
}

/// A button that increments our global counter.
#[component]
fn SetterButton(set_count: WriteSignal<u32>) -> impl IntoView {
    view! {
        <div class="provider red">
            <button on:click=move |_| set_count.update(|count| *count += 1)>
                "Increment Global Count"
            </button>
        </div>
    }
}

/// A component that does some "fancy" math with the global count
#[component]
fn FancyMath() -> impl IntoView {
    // here we consume the global count signal with `use_context`
    let count = use_context::<ReadSignal<u32>>()
        // we know we just provided this in the parent component
        .expect("there to be a `count` signal provided");
    let is_even = move || count() & 1 == 0;

    view! {
        <div class="consumer blue">
            "The number "
            <strong>{count}</strong>
            {move || if is_even() {
                " is"
            } else {
                " is not"
            }}
            " even."
        </div>
    }
}

/// A component that shows a list of items generated from the global count.
#[component]
fn ListItems() -> impl IntoView {
    // again, consume the global count signal with `use_context`
    let count = use_context::<ReadSignal<u32>>().expect("there to be a `count` signal provided");

    let squares = move || {
        (0..count())
            .map(|n| view! { <li>{n}<sup>"2"</sup> " is " {n * n}</li> })
            .collect::<Vec<_>>()
    };

    view! {
        <div class="consumer green">
            <ul>{squares}</ul>
        </div>
    }
}

// Option #3: Create a Global State Struct
//
// You can use this approach to build a single global data structure
// that holds the state for your whole app, and then access it by
// taking fine-grained slices using `create_slice` or `create_memo`,
// so that changing one part of the state doesn't cause parts of your
// app that depend on other parts of the state to change.

#[derive(Default, Clone, Debug)]
struct GlobalState {
    count: u32,
    name: String,
}

#[component]
fn Option3() -> impl IntoView {
    // we'll provide a single signal that holds the whole state
    // each component will be responsible for creating its own "lens" into it
    let state = create_rw_signal(GlobalState::default());
    provide_context(state);

    view! {
        <h1>"Option 3: Passing Signals"</h1>
        <div class="red consumer" style="width: 100%">
            <h2>"Current Global State"</h2>
            <pre>
                {move || {
                    format!("{:#?}", state.get())
                }}
            </pre>
        </div>
        <div style="display: flex">
            <GlobalStateCounter/>
            <GlobalStateInput/>
        </div>
    }
}

/// A component that updates the count in the global state.
#[component]
fn GlobalStateCounter() -> impl IntoView {
    let state = use_context::<RwSignal<GlobalState>>().expect("state to have been provided");

    // `create_slice` lets us create a "lens" into the data
    let (count, set_count) = create_slice(

        // we take a slice *from* `state`
        state,
        // our getter returns a "slice" of the data
        |state| state.count,
        // our setter describes how to mutate that slice, given a new value
        |state, n| state.count = n,
    );

    view! {
        <div class="consumer blue">
            <button
                on:click=move |_| {
                    set_count(count() + 1);
                }
            >
                "Increment Global Count"
            </button>
            <br/>
            <span>"Count is: " {count}</span>
        </div>
    }
}

/// A component that updates the count in the global state.
#[component]
fn GlobalStateInput() -> impl IntoView {
    let state = use_context::<RwSignal<GlobalState>>().expect("state to have been provided");

    // this slice is completely independent of the `count` slice
    // that we created in the other component
    // neither of them will cause the other to rerun
    let (name, set_name) = create_slice(
        // we take a slice *from* `state`
        state,
        // our getter returns a "slice" of the data
        |state| state.name.clone(),
        // our setter describes how to mutate that slice, given a new value
        |state, n| state.name = n,
    );

    view! {
        <div class="consumer green">
            <input
                type="text"
                prop:value=name
                on:input=move |ev| {
                    set_name(event_target_value(&ev));
                }
            />
            <br/>
            <span>"Name is: " {name}</span>
        </div>
    }
}
// This `main` function is the entry point into the app
// It just mounts our component to the <body>
// Because we defined it as `fn App`, we can now use it in a
// template as <App/>
fn main() {
    leptos::mount_to_body(|| view! { <Option2/><Option3/> })
}
```

</details>
</preview>
