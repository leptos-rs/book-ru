# Дочерние элементы компонентов

Достаточно часто люди хотят передавать дочерние элементы в компонент как в обычный элемент HTML. 
Представьте, к примеру, компонент `<FancyForm/>`, усовершенствующий `<form>`. Нужен какой-то способ
передать в него все поля ввода.

```rust
view! {
    <FancyForm>
        <fieldset>
            <label>
                "Some Input"
                <input type="text" name="something"/>
            </label>
        </fieldset>
        <button>"Submit"</button>
    </FancyForm>
}
```

Как это сделать в Leptos? Есть два способа передать компоненты в другие компоненты:

1. **render-свойства**: свойства-функции, возвращающие `view`
2. свойство **`children`**: специальное свойство компонента, включающее всё, что вы передаёте в качестве дочерних элементов компонента.

Фактически вы уже видели оба этих способа в действии в описании компонента [`<Show/>`](/view/06_control_flow.html#show):

```rust
view! {
  <Show
    // `when` is a normal prop
    when=move || value() > 5
    // `fallback` is a "render prop": a function that returns a view
    fallback=|| view! { <Small/> }
  >
    // `<Big/>` (and anything else here)
    // will be given to the `children` prop
    <Big/>
  </Show>
}
```

Давайте объявим компонент, который принимает дочерние элементы и render-свойство.

```rust
#[component]
pub fn TakesChildren<F, IV>(
    /// Takes a function (type F) that returns anything that can be
    /// converted into a View (type IV)
    render_prop: F,
    /// `children` takes the `Children` type
    children: Children,
) -> impl IntoView
where
    F: Fn() -> IV,
    IV: IntoView,
{
    view! {
        <h2>"Render Prop"</h2>
        {render_prop()}

        <h2>"Children"</h2>
        {children()}
    }
}
```

И `render_prop` и `children` это функции, так что они могут быть вызываться, чтобы сгенерировать подходящие `view`
`children`, в частности, это алиас для `Box<dyn FnOnce() -> Fragment>`. (Разве вы не рады, что мы назвали его `Children` вместо этого?)

> Если вам тут понадобится `Fn` или `FnMut`, чтобы вызывать `children` больше одного раза,
> мы также добавили алиасы `ChildrenFn` и `ChildrenMut`.

Использовать этот компонент мы можем вот так:

```rust
view! {
    <TakesChildren render_prop=|| view! { <p>"Hi, there!"</p> }>
        // these get passed to `children`
        "Some text"
        <span>"A span"</span>
    </TakesChildren>
}
```

## Воздействие на дочерние элементы

Тип [`Fragment`](https://docs.rs/leptos/latest/leptos/struct.Fragment.html) это просто способ обернуть `Vec<View>` Его можно вставлять куда угодно внутри `view`.

Но мы можем также получить доступ к этим внутренним `view` чтобы воздействовать на них.
К примеру, вот компонент, принимающий дочерние элементы и превращающий их в неупорядоченный список.

```rust
#[component]
pub fn WrapsChildren(children: Children) -> impl IntoView {
    // Fragment has `nodes` field that contains a Vec<View>
    let children = children()
        .nodes
        .into_iter()
        .map(|child| view! { <li>{child}</li> })
        .collect_view();

    view! {
        <ul>{children}</ul>
    }
}
```

Вызов его вот так создаст список:

```rust
view! {
    <WrapsChildren>
        "A"
        "B"
        "C"
    </WrapsChildren>
}
```

```admonish sandbox title="Live example" collapsible=true

[Click to open CodeSandbox.](https://codesandbox.io/p/sandbox/9-component-children-0-5-m4jwhp?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  Пожалуйста, включите Javascript для просмотра примеров.
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/9-component-children-0-5-m4jwhp?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>Код примера CodeSandbox</summary>

```rust
use leptos::*;

// Often, you want to pass some kind of child view to another
// component. There are two basic patterns for doing this:
// - "render props": creating a component prop that takes a function
//   that creates a view
// - the `children` prop: a special property that contains content
//   passed as the children of a component in your view, not as a
//   property

#[component]
pub fn App() -> impl IntoView {
    let (items, set_items) = create_signal(vec![0, 1, 2]);
    let render_prop = move || {
        // items.with(...) reacts to the value without cloning
        // by applying a function. Here, we pass the `len` method
        // on a `Vec<_>` directly
        let len = move || items.with(Vec::len);
        view! {
            <p>"Length: " {len}</p>
        }
    };

    view! {
        // This component just displays the two kinds of children,
        // embedding them in some other markup
        <TakesChildren
            // for component props, you can shorthand
            // `render_prop=render_prop` => `render_prop`
            // (this doesn't work for HTML element attributes)
            render_prop
        >
            // these look just like the children of an HTML element
            <p>"Here's a child."</p>
            <p>"Here's another child."</p>
        </TakesChildren>
        <hr/>
        // This component actually iterates over and wraps the children
        <WrapsChildren>
            <p>"Here's a child."</p>
            <p>"Here's another child."</p>
        </WrapsChildren>
    }
}

/// Displays a `render_prop` and some children within markup.
#[component]
pub fn TakesChildren<F, IV>(
    /// Takes a function (type F) that returns anything that can be
    /// converted into a View (type IV)
    render_prop: F,
    /// `children` takes the `Children` type
    /// this is an alias for `Box<dyn FnOnce() -> Fragment>`
    /// ... aren't you glad we named it `Children` instead?
    children: Children,
) -> impl IntoView
where
    F: Fn() -> IV,
    IV: IntoView,
{
    view! {
        <h1><code>"<TakesChildren/>"</code></h1>
        <h2>"Render Prop"</h2>
        {render_prop()}
        <hr/>
        <h2>"Children"</h2>
        {children()}
    }
}

/// Wraps each child in an `<li>` and embeds them in a `<ul>`.
#[component]
pub fn WrapsChildren(children: Children) -> impl IntoView {
    // children() returns a `Fragment`, which has a
    // `nodes` field that contains a Vec<View>
    // this means we can iterate over the children
    // to create something new!
    let children = children()
        .nodes
        .into_iter()
        .map(|child| view! { <li>{child}</li> })
        .collect::<Vec<_>>();

    view! {
        <h1><code>"<WrapsChildren/>"</code></h1>
        // wrap our wrapped children in a UL
        <ul>{children}</ul>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
