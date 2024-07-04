# Тестирование компонентов

Тестирование пользовательских интерфейсов может быть сравнительно замысловатым, но очень важным делом.
В этой статье мы обсудим пару принципов и подходов к тестированию Leptos-приложений.

## 1. Тестирование бизнес-логики с помощью обычных тестов Rust

Во многих случаях, имеет смысл вынести логику за пределы ваших компонентов и тестировать её отдельно.
Какие-то простые компоненты может не содержать логику, подлежащую тестированию, но для остальных стоит использовать
тестируемый тип-обёртку и использовать обычные Rust блоки `impl`.

К примеру, вместо встраивания логики в компонент напрямую как здесь:

```rust
#[component]
pub fn TodoApp() -> impl IntoView {
    let (todos, set_todos) = create_signal(vec![Todo { /* ... */ }]);
    // ⚠️ this is hard to test because it's embedded in the component
    let num_remaining = move || todos.with(|todos| {
        todos.iter().filter(|todo| !todo.completed).sum()
    });
}
```

Можно внести логику в отдельную структуру и тестировать её:

```rust
pub struct Todos(Vec<Todo>);

impl Todos {
    pub fn num_remaining(&self) -> usize {
        self.0.iter().filter(|todo| !todo.completed).sum()
    }
}

#[cfg(test)]
mod tests {
    #[test]
    fn test_remaining() {
        // ...
    }
}

#[component]
pub fn TodoApp() -> impl IntoView {
    let (todos, set_todos) = create_signal(Todos(vec![Todo { /* ... */ }]));
    // ✅ this has a test associated with it
    let num_remaining = move || todos.with(Todos::num_remaining);
}
```

В целом, чем меньше логики завернуто в сами компоненты, тем более естественным ощущается код и тем проще его тестировать.

## 2. Сквозное (`e2e`) тестирование компонентов

В директории [`examples`](https://github.com/leptos-rs/leptos/tree/main/examples)) есть несколько подробных примеров сквозного тестирования с использованием различных инструментов.

Проще всего разобраться как использовать эти инструменты это посмотреть на сами примеров тестов:

### `wasm-bindgen-test` с [`counter`](https://github.com/leptos-rs/leptos/blob/main/examples/counter/tests/web.rs)

Вот достаточно простой сетап для ручного тестирования, использующий команду  [`wasm-pack test`](https://rustwasm.github.io/wasm-pack/book/commands/test.html).

#### Образец Теста

````rust
#[wasm_bindgen_test]
fn clear() {
    let document = leptos::document();
    let test_wrapper = document.create_element("section").unwrap();
    let _ = document.body().unwrap().append_child(&test_wrapper);

    mount_to(
        test_wrapper.clone().unchecked_into(),
        || view! { <SimpleCounter initial_value=10 step=1/> },
    );

    let div = test_wrapper.query_selector("div").unwrap().unwrap();
    let clear = test_wrapper
        .query_selector("button")
        .unwrap()
        .unwrap()
        .unchecked_into::<web_sys::HtmlElement>();

    clear.click();

assert_eq!(
    div.outer_html(),
    // here we spawn a mini reactive system to render the test case
    run_scope(create_runtime(), || {
        // it's as if we're creating it with a value of 0, right?
        let (value, set_value) = create_signal(0);

        // we can remove the event listeners because they're not rendered to HTML
        view! {
            <div>
                <button>"Clear"</button>
                <button>"-1"</button>
                <span>"Value: " {value} "!"</span>
                <button>"+1"</button>
            </div>
        }
        // the view returned an HtmlElement<Div>, which is a smart pointer for
        // a DOM element. So we can still just call .outer_html()
        .outer_html()
    })
);
}
````

### [`wasm-bindgen-test` с `counters_stable`](https://github.com/leptos-rs/leptos/tree/main/examples/counters_stable/tests/web)

А вот более развернутый набор тестов, использующий систему фикстур, чтобы заменить ручную манипуляцию DOM в тестах `counter` и легко тестировать широкий диапазон кейсов.

#### Sample Test

```rust
use super::*;
use crate::counters_page as ui;
use pretty_assertions::assert_eq;

#[wasm_bindgen_test]
fn should_increase_the_total_count() {
    // Given
    ui::view_counters();
    ui::add_counter();

    // When
    ui::increment_counter(1);
    ui::increment_counter(1);
    ui::increment_counter(1);

    // Then
    assert_eq!(ui::total(), 3);
}
```

### [Playwright и `counters_stable`](https://github.com/leptos-rs/leptos/tree/main/examples/counters_stable/e2e)

Эти тесты используют обычный инструмент тестирования JavaScript — Playwright для выполнения сквозных тестов в том же примере,
используя библиотеку и подход к тестированию, знакомые многим, кто ранее занимался разработкой frontend.

#### Образец Теста

```js
import { test, expect } from "@playwright/test";
import { CountersPage } from "./fixtures/counters_page";

test.describe("Increment Count", () => {
  test("should increase the total count", async ({ page }) => {
    const ui = new CountersPage(page);
    await ui.goto();
    await ui.addCounter();

    await ui.incrementCount();
    await ui.incrementCount();
    await ui.incrementCount();

    await expect(ui.total).toHaveText("3");
  });
});
```

### [Gherkin/Cucumber тесты с `todo_app_sqlite`](https://github.com/leptos-rs/leptos/blob/main/examples/todo_app_sqlite/e2e/README.md)

Можно интегрировать какой угодно инструмент тестирования. Вот пример использования Cucumber, тестового фреймворка основанного
на естественном языке.

```
@add_todo
Feature: Add Todo

    Background:
        Given I see the app

    @add_todo-see
    Scenario: Should see the todo
        Given I set the todo as Buy Bread
        When I click the Add button
        Then I see the todo named Buy Bread

    # @allow.skipped
    @add_todo-style
    Scenario: Should see the pending todo
        When I add a todo as Buy Oranges
        Then I see the pending todo
```

Определения для этих действий даны в Rust-коде.

```rust
use crate::fixtures::{action, world::AppWorld};
use anyhow::{Ok, Result};
use cucumber::{given, when};

#[given("I see the app")]
#[when("I open the app")]
async fn i_open_the_app(world: &mut AppWorld) -> Result<()> {
    let client = &world.client;
    action::goto_path(client, "").await?;

    Ok(())
}

#[given(regex = "^I add a todo as (.*)$")]
#[when(regex = "^I add a todo as (.*)$")]
async fn i_add_a_todo_titled(world: &mut AppWorld, text: String) -> Result<()> {
    let client = &world.client;
    action::add_todo(client, text.as_str()).await?;

    Ok(())
}

// etc.
```

### Узнать Больше

Вы можете ознакомиться с настройками CI в репозитории Leptos, чтобы узнать больше о том, как использовать эти инструменты
в вашем собственном приложении. Примеры приложений Leptos регулярно тестируются всеми этими способами.
