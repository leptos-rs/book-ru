# Реагирование на изменения с помощью `create_effect`

Мы добрались до сюда, не упомянув половины реактивной системы: эффектов.

У реактивности две половины: обновление отдельных реактивных значений ("сигналов") уведомляет куски кода,
зависящие от них ("эффекты") о том, что они должны запуститься снова. 
Эти две половины (сигналы и эффекты) взаимозависимы. Без эффектов сигналы могут меняться в рамках реактивной системы,
но за ними нельзя наблюдать, взаимодействуя с внешним миром. Без сигналов эффекты выполняются только раз, поскольку нет наблюдаемого
значения, на которое можно подписаться. Эффекты это вполне буквально "побочные эффекты" реактивной системы: они
существуют, что синхронизировать реактивному систему с нереактивным миром вокруг неё.

За всем реактивным рендерингом DOM, что мы на данный момент видели, скрывается функция под названием `create_effect`.

[`create_effect`](https://docs.rs/leptos_reactive/latest/leptos_reactive/fn.create_effect.html) принимает в качестве аргумента функцию. Она сразу же вызывает эту функцию.
Если в этой функции вы обратились к какому-либо реактивному сигналу, она запоминает в реактивной среде выполнения, 
что данный эффект зависит от этого сигнала. Какой бы сигнал ни изменился, среди тех, от которых зависит наш эффект,
он будет выполнен снова.

```rust
let (a, set_a) = create_signal(0);
let (b, set_b) = create_signal(0);

create_effect(move |_| {
  // immediately prints "Value: 0" and subscribes to `a`
  log::debug!("Value: {}", a());
});
```

Функция эффекта вызывается с аргументом, содержащим значение, возвращённое ей при предыдущем вызове. При первом вызове оно `None`.

По-умолчанию эффекты **не выполняются на сервере**. Это значит, что вы можете без проблем обращаться к браузерные API внутри функции эффекта.
Если вам нужно запускать эффект на сервере, используйте [`create_isomorphic_effect`](https://docs.rs/leptos_reactive/latest/leptos_reactive/fn.create_isomorphic_effect.html).

## Авто-отслеживание и Динамические Зависимости

Если вы уже знакомы с фреймворком, таким как React, вы можете заметить одно ключевое отличие. React и схожие фреймврки
обычно требуют от вас передавать "массив зависимостей", явный набор переменных, определяющих когда эффекту выполняться снова. 

Поскольку Leptos происходит из традиции синхронного реактивного программирования, нам не нужен этот явный список зависимостей.
Вместо этого, мы автоматически отслеживаем зависимости, в зависимости от того какие сигналы вызываются внутри эффекта.

У этого есть два следствия. Зависимости являются:

1. **Автоматическими**: Вам не нужно поддерживать список зависимостей или беспокоиться о том что он должен включать, а что не должен. 
Фреймворк просто отслеживает какие сигналы могут повлечь перезапуск эффекта и делает всё за вас.
2. **Динамическими**: Список зависимостей очищается и заполняется каждый раз когда эффект выполняется. 
Если ваш эффект содержит условие (как пример), только сигналы, использованные в текущей ветке отслеживаются.
Это значит что эффекты перезапускаются самое минимальное количество раз.

> Если это звучит как магия, и если вы хотите глубоко погрузиться в то, как устроено автоматическое отслеживание зависимостей,
> [посмотрите это видео](https://www.youtube.com/watch?v=GWB3vTWeLd4). (Простите за тихий звук!)

## Эффекты как почти бесплатная абстракция

Несмотря на то, что не являясь "бесплатной абстракцией" в строгом техническом смысле — они требуют памяти,
существуют во среде выполнения, и т.д. — на более высоком уровне, с точки зрения каких бы то ни было дорогих вызовов API или
иной работы, которую вы делаете с их помощью, эффекты это бесплатная абстракция. Они перезапускаются самое минимальное количество раз,
зависящее от того, как вы их описали.

Представьте, что я пишу ПО для чата и хочу, чтобы люди могли отображать свои ФИО или просто имя, и уведомлять сервер всякий раз, когда 
их имя меняется.

```rust
let (first, set_first) = create_signal(String::new());
let (last, set_last) = create_signal(String::new());
let (use_last, set_use_last) = create_signal(true);

// this will add the name to the log
// any time one of the source signals changes
create_effect(move |_| {
    log(
        if use_last() {
            format!("{} {}", first(), last())
        } else {
            first()
        },
    )
});
```

Если `use_last` равно `true`, эффект будет перезапускаться всегда когда `first`, `last`, или `use_last` меняется. 
Но если переключу `use_last` в `false`, изменение  `last` никогда не повлечет изменение ФИО.
Фактически, `last` будет удалён из списка зависимостей пока  `use_last` не переключится вновь. 
Это избавляет нас от необходимости отправлять ненужные запросы к API если меняю `last` несколько раз пока `use_last` всё ещё `false`.

## Быть `create_effect` или не быть?

Эффекты предназначены для синхронизации реактивной системы с нереактивным внешним миром, а для не синхронизации  
реактивных значений между собой. Другими словами: использование эффекта, чтобы прочесть значение одного сигнала,
и записать его в другой — всегда неоптимально.

Если вам нужно объявить сигнал, зависящий от значения другого сигнала, используйте производный сигнал
или [`create_memo`](https://docs.rs/leptos_reactive/latest/leptos_reactive/fn.create_memo.html).
Запись в сигнал изнутри эффекта это не конец света и ваш компьютер от этого не загорится огнём, но 
производный сигнал или мемоизированное значение всегда лучше — не только потому что движение данных будет ясным, но и потому что 
производительность будет лучше.

```rust
let (a, set_a) = create_signal(0);

// ⚠️ not great
let (b, set_b) = create_signal(0);
create_effect(move |_| {
    set_b(a() * 2);
});

// ✅ woo-hoo!
let b = move || a() * 2;
```

If you need to synchronize some reactive value with the non-reactive world outside—like a web API, the console, the filesystem, or the DOM—writing to a signal in an effect is a fine way to do that. In many cases, though, you’ll find that you’re really writing to a signal inside an event listener or something else, not inside an effect. In these cases, you should check out [`leptos-use`](https://leptos-use.rs/) to see if it already provides a reactive wrapping primitive to do that!

> If you’re curious for more information about when you should and shouldn’t use `create_effect`, [check out this video](https://www.youtube.com/watch?v=aQOFJQ2JkvQ) for a more in-depth consideration!

## Эффекты и Рендеринг

Мы смогли добраться до сюда не упоминая эффекты, поскольку они встроены в Leptos DOM рендерер.
Мы уже видели, что можно создавать сигнал и передавать его в макрос `view` и он будет обновлять связанный узел DOM всякий раз, когда сигнал меняется:

```rust
let (count, set_count) = create_signal(0);

view! {
    <p>{count}</p>
}
```

Это работает потому, что фреймворк в сущности создает эффект оборачивающий это обновление. Вы можете представлять,
что Leptos переводит этот view в нечто вроде этого:

```rust
let (count, set_count) = create_signal(0);

// create a DOM element
let document = leptos::document();
let p = document.create_element("p").unwrap();

// create an effect to reactively update the text
create_effect(move |prev_value| {
    // first, access the signal’s value and convert it to a string
    let text = count().to_string();

    // if this is different from the previous value, update the node
    if prev_value != Some(text) {
        p.set_text_content(&text);
    }

    // return this value so we can memoize the next update
    text
});
```

Всякий раз когда `count` меняется, этот эффект повторно выполняется. Это делает возможными реактивные мелкозернистые обновления DOM.

## Явное, Отменяемое отслеживание через `watch`

Вдобавок к `create_effect`, Leptos предлагает функцию [`watch`](https://docs.rs/leptos_reactive/latest/leptos_reactive/fn.watch.html), которая может быть использована для двух основных вещей:

1. Отделение отслеживания от реагирования на изменения путём явной передачи набора значений для отслеживания.
2. Отмена отслеживания путём вызова stop-функции.

Как и `create_resource`, `watch` принимает первый аргумент (`deps`), отслеживаемый реактивно, и второй (`callback`), который не отслеживается.

Всякий раз, когда реактивное значение в аргументе `deps` меняется, вызывается `callback`. `watch` возвращает функцию, которую
можно вызвать, чтобы прекратить отслеживание зависимостей.

```rust
let (num, set_num) = create_signal(0);

let stop = watch(
    move || num.get(),
    move |num, prev_num, _| {
        log::debug!("Number: {}; Prev: {:?}", num, prev_num);
    },
    false,
);

set_num.set(1); // > "Number: 1; Prev: Some(0)"

stop(); // stop watching

set_num.set(2); // (nothing happens)
```

```admonish sandbox title="Live example" collapsible=true

[Кликните чтобы открыть CodeSandbox.](https://codesandbox.io/p/sandbox/14-effect-0-5-d6hkch?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  Пожалуйста, включите Javascript для просмотра примеров.
</noscript>


<template>
  <iframe src="https://codesandbox.io/p/sandbox/14-effect-0-5-d6hkch?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>Код примера CodeSandbox</summary>

```rust
use leptos::html::Input;
use leptos::*;

#[derive(Copy, Clone)]
struct LogContext(RwSignal<Vec<String>>);

#[component]
fn App() -> impl IntoView {
    // Just making a visible log here
    // You can ignore this...
    let log = create_rw_signal::<Vec<String>>(vec![]);
    let logged = move || log().join("\n");

    // the newtype pattern isn't *necessary* here but is a good practice
    // it avoids confusion with other possible future `RwSignal<Vec<String>>` contexts
    // and makes it easier to refer to it
    provide_context(LogContext(log));

    view! {
        <CreateAnEffect/>
        <pre>{logged}</pre>
    }
}

#[component]
fn CreateAnEffect() -> impl IntoView {
    let (first, set_first) = create_signal(String::new());
    let (last, set_last) = create_signal(String::new());
    let (use_last, set_use_last) = create_signal(true);

    // this will add the name to the log
    // any time one of the source signals changes
    create_effect(move |_| {
        log(if use_last() {
            with!(|first, last| format!("{first} {last}"))
        } else {
            first()
        })
    });

    view! {
        <h1>
            <code>"create_effect"</code>
            " Version"
        </h1>
        <form>
            <label>
                "First Name"
                <input
                    type="text"
                    name="first"
                    prop:value=first
                    on:change=move |ev| set_first(event_target_value(&ev))
                />
            </label>
            <label>
                "Last Name"
                <input
                    type="text"
                    name="last"
                    prop:value=last
                    on:change=move |ev| set_last(event_target_value(&ev))
                />
            </label>
            <label>
                "Show Last Name"
                <input
                    type="checkbox"
                    name="use_last"
                    prop:checked=use_last
                    on:change=move |ev| set_use_last(event_target_checked(&ev))
                />
            </label>
        </form>
    }
}

#[component]
fn ManualVersion() -> impl IntoView {
    let first = create_node_ref::<Input>();
    let last = create_node_ref::<Input>();
    let use_last = create_node_ref::<Input>();

    let mut prev_name = String::new();
    let on_change = move |_| {
        log("      listener");
        let first = first.get().unwrap();
        let last = last.get().unwrap();
        let use_last = use_last.get().unwrap();
        let this_one = if use_last.checked() {
            format!("{} {}", first.value(), last.value())
        } else {
            first.value()
        };

        if this_one != prev_name {
            log(&this_one);
            prev_name = this_one;
        }
    };

    view! {
        <h1>"Manual Version"</h1>
        <form on:change=on_change>
            <label>"First Name" <input type="text" name="first" node_ref=first/></label>
            <label>"Last Name" <input type="text" name="last" node_ref=last/></label>
            <label>
                "Show Last Name" <input type="checkbox" name="use_last" checked node_ref=use_last/>
            </label>
        </form>
    }
}

#[component]
fn EffectVsDerivedSignal() -> impl IntoView {
    let (my_value, set_my_value) = create_signal(String::new());
    // Don't do this.
    /*let (my_optional_value, set_optional_my_value) = create_signal(Option::<String>::None);

    create_effect(move |_| {
        if !my_value.get().is_empty() {
            set_optional_my_value(Some(my_value.get()));
        } else {
            set_optional_my_value(None);
        }
    });*/

    // Do this
    let my_optional_value =
        move || (!my_value.with(String::is_empty)).then(|| Some(my_value.get()));

    view! {
        <input prop:value=my_value on:input=move |ev| set_my_value(event_target_value(&ev))/>

        <p>
            <code>"my_optional_value"</code>
            " is "
            <code>
                <Show when=move || my_optional_value().is_some() fallback=|| view! { "None" }>
                    "Some(\""
                    {my_optional_value().unwrap()}
                    "\")"
                </Show>
            </code>
        </p>
    }
}

#[component]
pub fn Show<F, W, IV>(
    /// The components Show wraps
    children: Box<dyn Fn() -> Fragment>,
    /// A closure that returns a bool that determines whether this thing runs
    when: W,
    /// A closure that returns what gets rendered if the when statement is false
    fallback: F,
) -> impl IntoView
where
    W: Fn() -> bool + 'static,
    F: Fn() -> IV + 'static,
    IV: IntoView,
{
    let memoized_when = create_memo(move |_| when());

    move || match memoized_when.get() {
        true => children().into_view(),
        false => fallback().into_view(),
    }
}

fn log(msg: impl std::fmt::Display) {
    let log = use_context::<LogContext>().unwrap().0;
    log.update(|log| log.push(msg.to_string()));
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
