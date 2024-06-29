# Формы и поля ввода

Формы и поля ввода — важная часть интерактивных приложений. Есть два паттерна взаимодействия
с полями ввода в Leptos. Вы, возможно, уже их знаете, если вы знакомы с React, SolidJS или похожим фреймворком:
использование **контролируемых** или **неконтролируемых** элементов форм.

## Контролируемые поля ввода

В случае с "контролируемым полем ввода" фреймворк контролирует состояние элемента. При каждом `input` событии, он
обновляет локальный сигнал, хранящий текущее состояние, который, в свою очередь, обновляет значение свойства `value`.

Есть две важные вещи, которые нужно запомнить:

1. Событие `input` срабатывает (почти) при каждом изменении элемента, когда как `change`  (обычно) срабатывает только
после того, как вы сняли фокус с поля ввода. Вам наверно подойдёт `on:input`, но мы хотим дать вам свободу выбирать.

2. Значение _атрибута_ `value` лишь задает первоначальное значение поля ввода, то есть, оно обновляет значение поля ввода
лишь до тех пор, пока вы не начали печатать. _Свойство_ `value` продолжает обновлять поле вода и после этого.
По этой причине, обычно вам стоит использовать `prop:value`. (Это также справедливо для `checked` и `prop:checked`
   в `<input type="checkbox">`.)

```rust
let (name, set_name) = create_signal("Controlled".to_string());

view! {
    <input type="text"
        on:input=move |ev| {
            // event_target_value is a Leptos helper function
            // it functions the same way as event.target.value
            // in JavaScript, but smooths out some of the typecasting
            // necessary to make this work in Rust
            set_name(event_target_value(&ev));
        }

        // the `prop:` syntax lets you update a DOM property,
        // rather than an attribute.
        prop:value=name
    />
    <p>"Name is: " {name}</p>
}
```

> #### Зачем нужно `prop:value`?
>
> Веб-браузеры это самая вездесущая и стабильная платформа рендеринга графических пользовательских интерфейсов из всех существующих.

> Они также сохранили невероятную обратную совместимость на протяжении трёх десятков лет. Это неизбежно означает, что есть и странности.
> 
> Одна из странностей состоит в том, что есть разница между HTML атрибутами и свойствами DOM элемента, то есть, 
> между так называемым "атрибутом", что берется из HTML и может быть задан DOM-элементу с помощью  `.setAttribute()`,
> и между "свойством" — полем в JavaScript-представлении этого элемента HTML.
>
> В случае с `<input value=...>`, _атрибут_ `value` задаёт первоначальное значение этого поля ввода,
> а _свойство_ `value` задаёт его текущее значение. Возможно самый простой способ разобраться с этим это открыть  `about:blank`
> и выполнить следующий код на JavaScript в консоли браузера строчка за строчкой:
>
>
> ```js
> // создадим поле ввода и добавим его в DOM
> const el = document.createElement("input");
> document.body.appendChild(el);
>
> el.setAttribute("value", "тест"); // изменяет значение поля ввода
> el.setAttribute("value", "тест тест"); // тоже изменяет значение поля ввода
>
> // теперь напечайте что-нибудь в поле ввода: удалите какие-нибудь символы и т. д.
>
> el.setAttribute("value", "ещё разок?");
> // ничего не должно было измениться. смена "первоначального значения" теперь ни на что не влияет
>
> // однако...
> el.value = "А это работает";
> ```
>
> Многие другие frontend фреймворки объединяют атрибуты и свойства или в порядке исключения для полей ввода устанавливают
> значение корректно. Может Leptos'у тоже стоит так делать; но пока что я предпочитаю давать пользователям максимальную
> степень контроля над тем что они задают — атрибут или свойство, и делаю всё, что в моих силах, чтобы рассказать людям
> о реальном поведении браузера, вместо того, чтобы скрывать его.

## Неконтролируемые поля ввода

В случае "неконтролируемого поля ввода" браузера сам управляет состояние элемента поля ввода.
Вместо того чтобы постоянно обновлять сигнал, чтобы он содержал значение поля, мы используем [`NodeRef`](https://docs.rs/leptos/latest/leptos/struct.NodeRef.html) для
получения доступа к полю ввода когда мы хотим получить его значение.

В данном примере мы уведомляем фреймворк лишь когда `<form>` вызывает событие `submit`.
Обратите внимание на использование модуля  [`leptos::html`](https://docs.rs/leptos/latest/leptos/html/index.html#), который предоставляет набор типов для каждого элемента HTML.

```rust
let (name, set_name) = create_signal("Uncontrolled".to_string());

let input_element: NodeRef<html::Input> = create_node_ref();

view! {
    <form on:submit=on_submit> // on_submit defined below
        <input type="text"
            value=name
            node_ref=input_element
        />
        <input type="submit" value="Submit"/>
    </form>
    <p>"Name is: " {name}</p>
}
```

Данный view уже должен быть достаточно понятен без объяснений. Обратите внимание на следующие вещи:

1. В отличие от примера с контролируемыми полями ввода, мы используем `value` (не `prop:value`).
   Это потому, что мы просто задаём первоначальное значение поля ввода и позволяем браузеру его состояние.
   (Мы могли бы использовать `prop:value` вместо этого.)
2. Мы используем `node_ref=...` чтобы заполнить `NodeRef`. (Более старые примеры иногда используют `_ref`.
Это то же само, но `node_ref` лучше поддерживается анализатором Rust.)

`NodeRef` это наподобие реактивного умного указателя: мы можем использовать его для доступа к узлу DOM, на который тот указывает.
Значение `NodeRef` задаётся когда соответствующий элемент отрендерен.

```rust
let on_submit = move |ev: leptos::ev::SubmitEvent| {
    // stop the page from reloading!
    ev.prevent_default();

    // here, we'll extract the value from the input
    let value = input_element()
        // event handlers can only fire after the view
        // is mounted to the DOM, so the `NodeRef` will be `Some`
        .expect("<input> should be mounted")
        // `leptos::HtmlElement<html::Input>` implements `Deref`
        // to a `web_sys::HtmlInputElement`.
        // this means we can call`HtmlInputElement::value()`
        // to get the current value of the input
        .value();
    set_name(value);
};
```

Наш обработчик `on_submit` получит доступ к значению поля ввода и использует его при вызове `set_name`.
Чтобы получить доступ к узлу DOM хранимому в `NodeRef`, мы можем просто вызвать его как функцию (или использовать `.get()`).
Она вернёт тип `Option<leptos::HtmlElement<html::Input>>`, но мы знаем, что элемент уже был примонтирован (как иначе возникло событие!),
так что вызов unwrap() здесь безопасен. 

Затем мы можем `.value()` чтобы получить значение поля ввода, поскольку `NodeRef` даёт нам доступ к корректно типизированному 
HTML элементу.

Посмотрите на [`web_sys` and `HtmlElement`](../web_sys.md) чтобы узнать больше об использовании `leptos::HtmlElement`.
Также посмотрите полный CodeSandbox пример в конце этой страницы.

## Особые случаи: `<textarea>` и `<select>`

Эти два элемента формы склонны вызывать непонимание, каждый по-своему.

### `<textarea>`

В отличие от `<input>`, элемент `<textarea>` не поддерживает атрибут `value`.
Вместо этого, он получает своё значение в качестве текстового узла в своих HTML детях.

В текущей версии Leptos (а точнее, в Leptos 0.1-0.6), создание динамического child-узла влечёт также вставку узла комментария-маркера.
Это может вызвать некорректно рендеринг `<textarea>` (и проблемы во время гидратации) если вы попробуете использовать этот элемент для
отображения какого-то динамического контента. 

Вместо этого вы можете передать нереактивное первоначальное значение внутрь `<textarea>` и использовать `prop:value` чтобы задавать
текущее значение.  (`<textarea>` не поддерживает **атрибут** `value`, но _поддерживает_
 **свойство** `value`...)

```rust
view! {
    <textarea
        prop:value=move || some_value.get()
        on:input=/* etc */
    >
        /* plain-text initial value, does not change if the signal changes */
        {some_value.get_untracked()}
    </textarea>
}
```

### `<select>`

Элемент `<select>` может таким же образом контролироваться через свойство `value` самого `<select>`, 
`<option>` с этим значением будет выбран автоматически.

```rust
let (value, set_value) = create_signal(0i32);
view! {
  <select
    on:change=move |ev| {
      let new_value = event_target_value(&ev);
      set_value(new_value.parse().unwrap());
    }
    prop:value=move || value.get().to_string()
  >
    <option value="0">"0"</option>
    <option value="1">"1"</option>
    <option value="2">"2"</option>
  </select>
  // a button that will cycle through the options
  <button on:click=move |_| set_value.update(|n| {
    if *n == 2 {
      *n = 0;
    } else {
      *n += 1;
    }
  })>
    "Next Option"
  </button>
}
```

```admonish sandbox title="Controlled vs uncontrolled forms CodeSandbox" collapsible=true

[Click to open CodeSandbox.](https://codesandbox.io/p/sandbox/5-forms-0-5-rf2t7c?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  Пожалуйста, включите Javascript для просмотра примеров.
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/5-forms-0-5-rf2t7c?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>Код примера CodeSandbox</summary>

```rust
use leptos::{ev::SubmitEvent, *};

#[component]
fn App() -> impl IntoView {
    view! {
        <h2>"Controlled Component"</h2>
        <ControlledComponent/>
        <h2>"Uncontrolled Component"</h2>
        <UncontrolledComponent/>
    }
}

#[component]
fn ControlledComponent() -> impl IntoView {
    // create a signal to hold the value
    let (name, set_name) = create_signal("Controlled".to_string());

    view! {
        <input type="text"
            // fire an event whenever the input changes
            on:input=move |ev| {
                // event_target_value is a Leptos helper function
                // it functions the same way as event.target.value
                // in JavaScript, but smooths out some of the typecasting
                // necessary to make this work in Rust
                set_name(event_target_value(&ev));
            }

            // the `prop:` syntax lets you update a DOM property,
            // rather than an attribute.
            //
            // IMPORTANT: the `value` *attribute* only sets the
            // initial value, until you have made a change.
            // The `value` *property* sets the current value.
            // This is a quirk of the DOM; I didn't invent it.
            // Other frameworks gloss this over; I think it's
            // more important to give you access to the browser
            // as it really works.
            //
            // tl;dr: use prop:value for form inputs
            prop:value=name
        />
        <p>"Name is: " {name}</p>
    }
}

#[component]
fn UncontrolledComponent() -> impl IntoView {
    // import the type for <input>
    use leptos::html::Input;

    let (name, set_name) = create_signal("Uncontrolled".to_string());

    // we'll use a NodeRef to store a reference to the input element
    // this will be filled when the element is created
    let input_element: NodeRef<Input> = create_node_ref();

    // fires when the form `submit` event happens
    // this will store the value of the <input> in our signal
    let on_submit = move |ev: SubmitEvent| {
        // stop the page from reloading!
        ev.prevent_default();

        // here, we'll extract the value from the input
        let value = input_element()
            // event handlers can only fire after the view
            // is mounted to the DOM, so the `NodeRef` will be `Some`
            .expect("<input> to exist")
            // `NodeRef` implements `Deref` for the DOM element type
            // this means we can call`HtmlInputElement::value()`
            // to get the current value of the input
            .value();
        set_name(value);
    };

    view! {
        <form on:submit=on_submit>
            <input type="text"
                // here, we use the `value` *attribute* to set only
                // the initial value, letting the browser maintain
                // the state after that
                value=name

                // store a reference to this input in `input_element`
                node_ref=input_element
            />
            <input type="submit" value="Submit"/>
        </form>
        <p>"Name is: " {name}</p>
    }
}

// This `main` function is the entry point into the app
// It just mounts our component to the <body>
// Because we defined it as `fn App`, we can now use it in a
// template as <App/>
fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
