# Итерация

Выводите ли вы TODO-лист, отображаете ли таблицу, показываете ли изображения продукта — итерация списка элементов это 
часто встречающая задача в веб-приложениях. Грамотная сверка различий между изменяющимися наборами элементов это возможно одна из самых
сложных задач для фреймворка.

Leptos поддерживает два разных паттерна для итерации элементов:

1. Для статических view: `Vec<_>`
2. Для динамических списков: `<For/>`

## Статические Views с `Vec<_>`

Иногда вам нужно отобразить элемент несколько раз, но список элементы которого вы отображаете меняется нечасто. В этом случае,
важно знать что вы можете вставить любой `Vec<IV> where IV: IntoView` в ваш view. Другими словами, если вы можете отрендерить `T`,
 то вы можете отрендерить и `Vec<T>`.

```rust
let values = vec![0, 1, 2];
view! {
    // this will just render "012"
    <p>{values.clone()}</p>
    // or we can wrap them in <li>
    <ul>
        {values.into_iter()
            .map(|n| view! { <li>{n}</li>})
            .collect::<Vec<_>>()}
    </ul>
}
```

Leptos также предоставляет вспомогательную функцию `.collect_view()`, которая позволяет вам собрать любой итератор
типа  `T: IntoView` в `Vec<View>`.

```rust
let values = vec![0, 1, 2];
view! {
    // this will just render "012"
    <p>{values.clone()}</p>
    // or we can wrap them in <li>
    <ul>
        {values.into_iter()
            .map(|n| view! { <li>{n}</li>})
            .collect_view()}
    </ul>
}
```

Тот факт, что _список_ статический, не означает, что интерфейс должен быть статическим.
Вы можете рендерить динамические элементы как часть статического списка.

```rust
// create a list of 5 signals
let length = 5;
let counters = (1..=length).map(|idx| create_signal(idx));

// each item manages a reactive view
// but the list itself will never change
let counter_buttons = counters
    .map(|(count, set_count)| {
        view! {
            <li>
                <button
                    on:click=move |_| set_count.update(|n| *n += 1)
                >
                    {count}
                </button>
            </li>
        }
    })
    .collect_view();

view! {
    <ul>{counter_buttons}</ul>
}
```

Вы _можете_ рендерить и `Fn() -> Vec<_>` реактивно. Но знайте, что при любом изменении каждый элемент списка будет отрендерен заново.
Это весьма неэкономично! К счастью, есть способ получше.

## Динамический Рендеринг с помощью компонента `<For/>`

Компонент [`<For/>`](https://docs.rs/leptos/latest/leptos/fn.For.html) это 
динамический список с ключами. Он принимает три свойства:

- `each`: функция (такая как сигнал), она возвращает элементы `T`, по которым будет итерация
- `key`: функция принимает `&T` и возвращает постоянный уникальный ключ или ID
- `children`: превращает каждый `T` во view

`key` это ключ. Вы можете добавлять, удалять, и передвигать элементы внутри списка. Пока ключ каждого элемента неизменен, 
фреймворку не нужно рендерить заново ни один из элементов, только если не были добавлены новые элементы,
и он может очень низкозатратно вставить добавлять, удалять и двигать элементы по мере их изменения. Это позволяет чрезвычайно 
низкозатратно обновлять список по мере его изменений, с минимальной дополнительной работой.

Создание хорошего значения `key` может быть немного сложно. В общем случае вам _не стоит_ использовать для этого индекс,
поскольку он не стабилен — если вы удаляете или передвигаете элементы, их индексы меняются.

А вот генерировать уникальный ID для каждого ряда во время его создания и использовать этот ID в функции `key` — отличная идея,   

Ознакомьтесь с компонентом `<DynamicList/>` (см. ниже) в качестве примера.

```admonish sandbox title="Live example" collapsible=true

[Click to open CodeSandbox.](https://codesandbox.io/p/sandbox/4-iteration-0-5-pwdn2y?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  Пожалуйста, включите Javascript для просмотра примеров.
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/4-iteration-0-5-pwdn2y?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>Код пример CodeSandbox</summary>

```rust
use leptos::*;

// Iteration is a very common task in most applications.
// So how do you take a list of data and render it in the DOM?
// This example will show you the two ways:
// 1) for mostly-static lists, using Rust iterators
// 2) for lists that grow, shrink, or move items, using <For/>

#[component]
fn App() -> impl IntoView {
    view! {
        <h1>"Iteration"</h1>
        <h2>"Static List"</h2>
        <p>"Use this pattern if the list itself is static."</p>
        <StaticList length=5/>
        <h2>"Dynamic List"</h2>
        <p>"Use this pattern if the rows in your list will change."</p>
        <DynamicList initial_length=5/>
    }
}

/// A list of counters, without the ability
/// to add or remove any.
#[component]
fn StaticList(
    /// How many counters to include in this list.
    length: usize,
) -> impl IntoView {
    // create counter signals that start at incrementing numbers
    let counters = (1..=length).map(|idx| create_signal(idx));

    // when you have a list that doesn't change, you can
    // manipulate it using ordinary Rust iterators
    // and collect it into a Vec<_> to insert it into the DOM
    let counter_buttons = counters
        .map(|(count, set_count)| {
            view! {
                <li>
                    <button
                        on:click=move |_| set_count.update(|n| *n += 1)
                    >
                        {count}
                    </button>
                </li>
            }
        })
        .collect::<Vec<_>>();

    // Note that if `counter_buttons` were a reactive list
    // and its value changed, this would be very inefficient:
    // it would rerender every row every time the list changed.
    view! {
        <ul>{counter_buttons}</ul>
    }
}

/// A list of counters that allows you to add or
/// remove counters.
#[component]
fn DynamicList(
    /// The number of counters to begin with.
    initial_length: usize,
) -> impl IntoView {
    // This dynamic list will use the <For/> component.
    // <For/> is a keyed list. This means that each row
    // has a defined key. If the key does not change, the row
    // will not be re-rendered. When the list changes, only
    // the minimum number of changes will be made to the DOM.

    // `next_counter_id` will let us generate unique IDs
    // we do this by simply incrementing the ID by one
    // each time we create a counter
    let mut next_counter_id = initial_length;

    // we generate an initial list as in <StaticList/>
    // but this time we include the ID along with the signal
    let initial_counters = (0..initial_length)
        .map(|id| (id, create_signal(id + 1)))
        .collect::<Vec<_>>();

    // now we store that initial list in a signal
    // this way, we'll be able to modify the list over time,
    // adding and removing counters, and it will change reactively
    let (counters, set_counters) = create_signal(initial_counters);

    let add_counter = move |_| {
        // create a signal for the new counter
        let sig = create_signal(next_counter_id + 1);
        // add this counter to the list of counters
        set_counters.update(move |counters| {
            // since `.update()` gives us `&mut T`
            // we can just use normal Vec methods like `push`
            counters.push((next_counter_id, sig))
        });
        // increment the ID so it's always unique
        next_counter_id += 1;
    };

    view! {
        <div>
            <button on:click=add_counter>
                "Add Counter"
            </button>
            <ul>
                // The <For/> component is central here
                // This allows for efficient, key list rendering
                <For
                    // `each` takes any function that returns an iterator
                    // this should usually be a signal or derived signal
                    // if it's not reactive, just render a Vec<_> instead of <For/>
                    each=counters
                    // the key should be unique and stable for each row
                    // using an index is usually a bad idea, unless your list
                    // can only grow, because moving items around inside the list
                    // means their indices will change and they will all rerender
                    key=|counter| counter.0
                    // `children` receives each item from your `each` iterator
                    // and returns a view
                    children=move |(id, (count, set_count))| {
                        view! {
                            <li>
                                <button
                                    on:click=move |_| set_count.update(|n| *n += 1)
                                >
                                    {count}
                                </button>
                                <button
                                    on:click=move |_| {
                                        set_counters.update(|counters| {
                                            counters.retain(|(counter_id, _)| counter_id != &id)
                                        });
                                    }
                                >
                                    "Remove"
                                </button>
                            </li>
                        }
                    }
                />
            </ul>
        </div>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
