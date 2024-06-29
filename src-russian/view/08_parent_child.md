# Коммуникация Родитель-Ребёнок

Вы можете представлять ваше приложение как вложенное дерево компонентов. Каждый компонент 
управляется со собственным локальным состоянием и управляет разделом пользовательского интерфейса,
так что компоненты склонны быть относительно самодостаточными.

Хотя иногда вам захочется наладить коммуникацию между родительным компонентом и его дочерними компонентами.
Представьте, например, что вы объявили компонент `<FancyButton/>`, добавляющий какие-то стили, логирование или что-то ещё к `<button/>`. 
Вы хотите использовать `<FancyButton/>` в вашем компоненте `<App/>`. Но как вам наладить коммуникацию между ими двумя?

Передать состояние из родителя в дочерний компонент легко. Мы частично рассматривали это в материале про [компоненты и свойства](./03_components.md).
Попросту говоря если вы хотите чтобы родитель общался с дочерним компонентом, вы можете передать
[`ReadSignal`](https://docs.rs/leptos/latest/leptos/struct.ReadSignal.html), 
[`Signal`](https://docs.rs/leptos/latest/leptos/struct.Signal.html), или
[`MaybeSignal`](https://docs.rs/leptos/latest/leptos/enum.MaybeSignal.html) в качестве свойства.

Но как быть с обратным направлением? Как дочерний элемент может уведомлять родителя о событиях или изменениях состояния?

Есть четыре простых паттерна коммуникации Родитель-Ребёнок в Leptos.

## 1. Передавать [`WriteSignal`](https://docs.rs/leptos/latest/leptos/struct.WriteSignal.html)

Один из подходов это просто передавать `WriteSignal` из родителя в дочерний компонент и в нём обновлять сигнал.
Это позволяет вам манипулировать состоянием родителя из дочернего компонента.

```rust
#[component]
pub fn App() -> impl IntoView {
    let (toggled, set_toggled) = create_signal(false);
    view! {
        <p>"Toggled? " {toggled}</p>
        <ButtonA setter=set_toggled/>
    }
}

#[component]
pub fn ButtonA(setter: WriteSignal<bool>) -> impl IntoView {
    view! {
        <button
            on:click=move |_| setter.update(|value| *value = !*value)
        >
            "Toggle"
        </button>
    }
}
```

Этот паттерн прост, но будьте с ним осторожны: передача `WriteSignal` может усложить понимание вашего кода.
Читая `<App/>` из этого примера,  вполне ясно, что даёте возможность изменять значение `toggled`,
но совершенно неясно когда и как это происходит. В этом простом, локальном примере это легко понять, но если вы
видите, что передаёте сигналы `WriteSignal` как этот по всему вашему коду, вам следует всерьёз задуматься о том не
упрощает ли этот подход донельзя написание спагетти-кода.

## 2. Использовать `Callback` (функцию обратного вызова)

Ещё один подход это передавать `Callback` в дочерний компонент: скажем, `on_click`.

```rust
#[component]
pub fn App() -> impl IntoView {
    let (toggled, set_toggled) = create_signal(false);
    view! {
        <p>"Toggled? " {toggled}</p>
        <ButtonB on_click=move |_| set_toggled.update(|value| *value = !*value)/>
    }
}


#[component]
pub fn ButtonB(#[prop(into)] on_click: Callback<MouseEvent>) -> impl IntoView
{
    view! {
        <button on:click=on_click>
            "Toggle"
        </button>
    }
}
```

Вы заметите что когда как `<ButtonA/>` получил `WriteSignal` и решает как его мутировать,
`<ButtonB/>` просто порождает событие: мутация происходит в `<App/>`. Преимущество этого в том, что мы локальное состояние
остаётся локальным, предотвращая проблему спагетти мутации. Но это также означает, что логика мутации сигнала должна находиться в 
`<App/>`, а не в `<ButtonB/>`. Это настоящие компромиссы, а не простой выбор между правильно/неправильно.

> Обратите внимание на способ, которым мы используем тип `Callback<In, Out>`. Это просто обёртка вокруг замыкания 
> `Fn(In) -> Out`, которая реализует `Copy`, что упрощает её передачу.
>
> Мы также использовали атрибут `#[prop(into)]` чтобы мы могли передавать обычное замыкание в  `on_click`.
> Пожалуйста, посмотрите [главу "Свойства с `into`"](./03_components.md#into-props) для более подробной информации.

### 2.1 Использование замыкание вместо `Callback`

Вы можете использовать Rust замыкание `Fn(MouseEvent)` напрямую вместо `Callback`:

```rust
#[component]
pub fn App() -> impl IntoView {
    let (toggled, set_toggled) = create_signal(false);
    view! {
        <p>"Toggled? " {toggled}</p>
        <ButtonB on_click=move |_| set_toggled.update(|value| *value = !*value)/>
    }
}


#[component]
pub fn ButtonB<F>(on_click: F) -> impl IntoView
where
    F: Fn(MouseEvent) + 'static
{
    view! {
        <button on:click=on_click>
            "Toggle"
        </button>
    }
}
```

В данном случае код почти не изменился. В более продвинутых случаях использование замыкания может потребовать клонирования,
в отличие от варианта с использованием `Callback`.

> Обратите внимание, что мы объявили обобщенный тип `F` ради функции обратного вызова. Если вас это смутило,
> вернитесь к разделу [Свойства с обобщенными типами](./03_components.html#свойства-с-обобщенными-типами)
> главы о компонентах.

## 3. Использование слушателя событий _(англ. Event Listener)_

По правде говоря, вы можете Вариант 2 немного другим способом. Если функция обратного вызова напрямую накладывается на нативный DOM элементом,
вы можете добавить `on:` слушатель прямо в место где вы используете этот компонент в макрос `view` в `<App/>`.

```rust
#[component]
pub fn App() -> impl IntoView {
    let (toggled, set_toggled) = create_signal(false);
    view! {
        <p>"Toggled? " {toggled}</p>
        // note the on:click instead of on_click
        // this is the same syntax as an HTML element event listener
        <ButtonC on:click=move |_| set_toggled.update(|value| *value = !*value)/>
    }
}


#[component]
pub fn ButtonC() -> impl IntoView {
    view! {
        <button>"Toggle"</button>
    }
}
```

Это позволяет вам написать намного меньше кода для `<ButtonC/>`, чем вы написали для `<ButtonB/>`, и всё даёт слушателю 
 правильно типизированное событие. Это работает так, что  слушатель событий `on:` добавляется к каждому возвращаемому `<ButtonC/>`:
 в данном случае только `<button>`.

Конечно, это работает только для настоящих событий DOM которые вы передаёте напрямую в элементы, которые вы рендерите в компоненте.
Для более сложной логики, которая не накладывается напрямую на элемент (скажем вы создали `<ValidatedForm/>`
и хотите функцию обратного вызова `on_valid_form_submit`), вам следует использовать Вариант 2.

## 4. Предоставление Контекста

Эта версия по факту является разновидностью Варинта 1. Скажем у вас дерево компонентов глубокой вложенности:

```rust
#[component]
pub fn App() -> impl IntoView {
    let (toggled, set_toggled) = create_signal(false);
    view! {
        <p>"Toggled? " {toggled}</p>
        <Layout/>
    }
}

#[component]
pub fn Layout() -> impl IntoView {
    view! {
        <header>
            <h1>"My Page"</h1>
        </header>
        <main>
            <Content/>
        </main>
    }
}

#[component]
pub fn Content() -> impl IntoView {
    view! {
        <div class="content">
            <ButtonD/>
        </div>
    }
}

#[component]
pub fn ButtonD<F>() -> impl IntoView {
    todo!()
}
```

`<ButtonD/>` теперь уже не дочерний элемент `<App/>`, так что вы не можете просто передавать `WriteSignal` в него через свойства.
Вы можете сделать то, что иногда называют “пробурить свойство”, добавив свойство в каждый слой между ими двумя:

```rust
#[component]
pub fn App() -> impl IntoView {
    let (toggled, set_toggled) = create_signal(false);
    view! {
        <p>"Toggled? " {toggled}</p>
        <Layout set_toggled/>
    }
}

#[component]
pub fn Layout(set_toggled: WriteSignal<bool>) -> impl IntoView {
    view! {
        <header>
            <h1>"My Page"</h1>
        </header>
        <main>
            <Content set_toggled/>
        </main>
    }
}

#[component]
pub fn Content(set_toggled: WriteSignal<bool>) -> impl IntoView {
    view! {
        <div class="content">
            <ButtonD set_toggled/>
        </div>
    }
}

#[component]
pub fn ButtonD<F>(set_toggled: WriteSignal<bool>) -> impl IntoView {
    todo!()
}
```

Ну и бардак. `set_toggled` не нужен ни`<Layout/>`, ни `<Content/>`; они просто передают его в `<ButtonD/>`. Но мне приходится 
объявлять свойство трижды. Помимо того, что это раздражает, такое ещё и сложно поддерживать: только вообразите, если мы добавим
"среднее положение" переключателя, тип `set_toggled` придётся поменять на `enum`. Наам придётся менять его в трёх местах!  

Нет ли какого-то способа перепрыгнуть эти уровни?

Есть!

### 4.1 API контекстов

Вы можете представлять данные, перепрыгивающие через уровни, используя [`provide_context`](https://docs.rs/leptos/latest/leptos/fn.provide_context.html) и [`use_context`](https://docs.rs/leptos/latest/leptos/fn.use_context.html). 
Контексты идентифицируются по типу данных, которые вы предоставляете (в этом примере,  `WriteSignal<bool>`), 
и они существуют в нисходящем дереве, которое следует контурам дерева вашего UI. В этом примере мы можем использовать 
контекст, чтобы избежать ненужного бурения свойств.

```rust
#[component]
pub fn App() -> impl IntoView {
    let (toggled, set_toggled) = create_signal(false);

    // share `set_toggled` with all children of this component
    provide_context(set_toggled);

    view! {
        <p>"Toggled? " {toggled}</p>
        <Layout/>
    }
}

// <Layout/> and <Content/> omitted
// To work in this version, drop their references to set_toggled

#[component]
pub fn ButtonD() -> impl IntoView {
    // use_context searches up the context tree, hoping to
    // find a `WriteSignal<bool>`
    // in this case, I .expect() because I know I provided it
    let setter = use_context::<WriteSignal<bool>>()
        .expect("to have found the setter provided");

    view! {
        <button
            on:click=move |_| setter.update(|value| *value = !*value)
        >
            "Toggle"
        </button>
    }
}
```

Эти же предостережения относятся к данному коду в той же мере, что и к `<ButtonA/>`: передавать `WriteSignal` нужно с осторожностью,
так как это позволяет мутировать состояние из произвольных мест вашего кода. Но при осторожном обращении, это одна из 
самых эффективных техник управления глобальным состоянием в Leptos: просто предоставьте состояние на самом высоком из
уровней, на которых вам оно требуется и используйте где нужно на уровнях ниже.

Заметим, что у этого подхода нет недостатков в части производительности. Поскольку вы передаете мелкозернистый реактивный сигнал,
_ничего не происходит_ в промежуточных компонентах  (`<Layout/>` и `<Content/>`) когда вы его обновляете.
Вы можете наладить прямую коммуникацию между  `<ButtonD/>` и `<App/>`. 
Фактически —и в этом состоит сила мелкозернистой реактивности— коммуникация идёт напрямую между нажатием на кнопку `<ButtonD/>`
и единственным текстовым узлом в `<App/>`. Это как если бы самих компонентов вовсе не существовало. И, ну... в рантайме они не существуют.
Лишь сигналы и эффекты до самого низа.

```admonish sandbox title="Live example" collapsible=true

[Click to open CodeSandbox.](https://codesandbox.io/p/sandbox/8-parent-child-0-5-7rz7qd?file=%2Fsrc%2Fmain.rs%3A1%2C2)

<noscript>
  Пожалуйста, включите Javascript для просмотра примеров.
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/8-parent-child-0-5-7rz7qd?file=%2Fsrc%2Fmain.rs%3A1%2C2" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>Код примера CodeSandbox</summary>

```rust
use leptos::{ev::MouseEvent, *};

// This highlights four different ways that child components can communicate
// with their parent:
// 1) <ButtonA/>: passing a WriteSignal as one of the child component props,
//    for the child component to write into and the parent to read
// 2) <ButtonB/>: passing a closure as one of the child component props, for
//    the child component to call
// 3) <ButtonC/>: adding an `on:` event listener to a component
// 4) <ButtonD/>: providing a context that is used in the component (rather than prop drilling)

#[derive(Copy, Clone)]
struct SmallcapsContext(WriteSignal<bool>);

#[component]
pub fn App() -> impl IntoView {
    // just some signals to toggle three classes on our <p>
    let (red, set_red) = create_signal(false);
    let (right, set_right) = create_signal(false);
    let (italics, set_italics) = create_signal(false);
    let (smallcaps, set_smallcaps) = create_signal(false);

    // the newtype pattern isn't *necessary* here but is a good practice
    // it avoids confusion with other possible future `WriteSignal<bool>` contexts
    // and makes it easier to refer to it in ButtonC
    provide_context(SmallcapsContext(set_smallcaps));

    view! {
        <main>
            <p
                // class: attributes take F: Fn() => bool, and these signals all implement Fn()
                class:red=red
                class:right=right
                class:italics=italics
                class:smallcaps=smallcaps
            >
                "Lorem ipsum sit dolor amet."
            </p>

            // Button A: pass the signal setter
            <ButtonA setter=set_red/>

            // Button B: pass a closure
            <ButtonB on_click=move |_| set_right.update(|value| *value = !*value)/>

            // Button B: use a regular event listener
            // setting an event listener on a component like this applies it
            // to each of the top-level elements the component returns
            <ButtonC on:click=move |_| set_italics.update(|value| *value = !*value)/>

            // Button D gets its setter from context rather than props
            <ButtonD/>
        </main>
    }
}

/// Button A receives a signal setter and updates the signal itself
#[component]
pub fn ButtonA(
    /// Signal that will be toggled when the button is clicked.
    setter: WriteSignal<bool>,
) -> impl IntoView {
    view! {
        <button
            on:click=move |_| setter.update(|value| *value = !*value)
        >
            "Toggle Red"
        </button>
    }
}

/// Button B receives a closure
#[component]
pub fn ButtonB<F>(
    /// Callback that will be invoked when the button is clicked.
    on_click: F,
) -> impl IntoView
where
    F: Fn(MouseEvent) + 'static,
{
    view! {
        <button
            on:click=on_click
        >
            "Toggle Right"
        </button>
    }

    // just a note: in an ordinary function ButtonB could take on_click: impl Fn(MouseEvent) + 'static
    // and save you from typing out the generic
    // the component macro actually expands to define a
    //
    // struct ButtonBProps<F> where F: Fn(MouseEvent) + 'static {
    //   on_click: F
    // }
    //
    // this is what allows us to have named props in our component invocation,
    // instead of an ordered list of function arguments
    // if Rust ever had named function arguments we could drop this requirement
}

/// Button C is a dummy: it renders a button but doesn't handle
/// its click. Instead, the parent component adds an event listener.
#[component]
pub fn ButtonC() -> impl IntoView {
    view! {
        <button>
            "Toggle Italics"
        </button>
    }
}

/// Button D is very similar to Button A, but instead of passing the setter as a prop
/// we get it from the context
#[component]
pub fn ButtonD() -> impl IntoView {
    let setter = use_context::<SmallcapsContext>().unwrap().0;

    view! {
        <button
            on:click=move |_| setter.update(|value| *value = !*value)
        >
            "Toggle Small Caps"
        </button>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
