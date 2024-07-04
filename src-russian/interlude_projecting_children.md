# Проброс дочерних элементов

При написании компонентов можно время от времени ловить себя на желании "пробросить" через несколько уровней компонентов.

## Задача

Рассмотрим следующий пример:

```rust
pub fn LoggedIn<F, IV>(fallback: F, children: ChildrenFn) -> impl IntoView
where
    F: Fn() -> IV + 'static,
    IV: IntoView,
{
    view! {
        <Suspense
            fallback=|| ()
        >
            <Show
				// check whether user is verified
				// by reading from the resource
                when=move || todo!()
                fallback=fallback
            >
				{children()}
			</Show>
        </Suspense>
    }
}
```

Он достаточно прост: когда пользователь авторизован, выводится `children`. Когда нет — выводится `fallback`. 
А пока информация не пришла, выводится `()`, т.е. ничего.

Другими словами, дочерние элементы  `<LoggedIn/>` хочется передать _через_ компонент `<Suspense/>`,
чтобы они стали дочерними элементами `<Show/>`. Это то, что имеется в виду под "пробросом".

Это не скомпилируется.

```
error[E0507]: cannot move out of `fallback`, a captured variable in an `Fn` closure
error[E0507]: cannot move out of `children`, a captured variable in an `Fn` closure
```

Проблема в том, что компонентам `<Suspense/>` и `<Show/>` нужна возможность строить свои `children` несколько раз.
При первом построении дочерних элементов `<Suspense>` владение `fallback` и `children` требуется, 
чтобы переместить эти значения внутрь вызова `<Show/>`, но тогда они недоступны для последующих построений дочерних элементов `<Suspense/>`.

## Подробности

> Можете смело перейти сразу к решению.

Если хотите по-настоящему понять проблему, стоит посмотреть на расширенный макрос `view`. Вот подчищенный вариант:

```rust
Suspense(
    ::leptos::component_props_builder(&Suspense)
        .fallback(|| ())
        .children({
            // fallback and children are moved into this closure
            Box::new(move || {
                {
                    // fallback and children captured here
                    leptos::Fragment::lazy(|| {
                        vec![
                            (Show(
                                ::leptos::component_props_builder(&Show)
                                    .when(|| true)
									// but fallback is moved into Show here
                                    .fallback(fallback)
									// and children is moved into Show here
                                    .children(children)
                                    .build(),
                            )
                            .into_view()),
                        ]
                    })
                }
            })
        })
        .build(),
)
```

Все компоненты владеют своими свойствами; так что `<Show/>` в данном случае не может быть вызван поскольку он лишь 
захватил ссылки на `fallback` и `children`.

## Решение

Однако, и `<Suspense/>` и `<Show/>` принимают `ChildrenFn` в качестве аргумента, т.е. их `children` должна реализовывать
тип `Fn`, чтобы они могли вызваться несколько раз с лишь иммутабельной ссылкой. Это означает, что владеть 
`children` или `fallback` не нужно; нужно просто передать `'static'` ссылки на них.

Эту проблему можно решить через примитив [`store_value`](https://docs.rs/leptos/latest/leptos/fn.store_value.html).
Он по сути сохраняет значение в реактивной системе, передавая его во владение фреймворку, в обмен на ссылку, 
которая, подобно сигналу, реализует Copy, имеет время жизни 'static, и которую с помощью определённых методов можно изменить или получить к ней доступ.

В данном примере это очень просто:

```rust
pub fn LoggedIn<F, IV>(fallback: F, children: ChildrenFn) -> impl IntoView
where
    F: Fn() -> IV + 'static,
    IV: IntoView,
{
    let fallback = store_value(fallback);
    let children = store_value(children);
    view! {
        <Suspense
            fallback=|| ()
        >
            <Show
                when=|| todo!()
                fallback=move || fallback.with_value(|fallback| fallback())
            >
                {children.with_value(|children| children())}
            </Show>
        </Suspense>
    }
}
```

На верхнем уровне `fallback` и `children` сохраняется в реактивной области видимости, которой владеет `LoggedIn`.
Теперь можно просто переместить эти ссылки через другие уровни в компонент`<Show/>` и вызывать их там.

## В заключение

Учтите, что это работает потому что компонентам `<Show/>` и `<Suspense/>` нужна иммутабельная ссылка на их дочерние элементы
(которую им может дать `.with_value` ), а не владение.

В иных случаях, может понадобиться пробрасывать свойства с владением через функцию, которая принимает `ChildrenFn`
и которая таким образом должна вызываться более одного раза.
В этом случае может быть полезна вспомогательный синтаксис `clone:` в макросе `view`.

Рассмотрим пример

```rust
#[component]
pub fn App() -> impl IntoView {
    let name = "Alice".to_string();
    view! {
        <Outer>
            <Inner>
                <Inmost name=name.clone()/>
            </Inner>
        </Outer>
    }
}

#[component]
pub fn Outer(children: ChildrenFn) -> impl IntoView {
    children()
}

#[component]
pub fn Inner(children: ChildrenFn) -> impl IntoView {
    children()
}

#[component]
pub fn Inmost(name: String) -> impl IntoView {
    view! {
        <p>{name}</p>
    }
}
```

Даже с `name=name.clone()`, возникает ошибка

```
cannot move out of `name`, a captured variable in an `Fn` closure
```

Переменная захватывается на нескольких уровнях элементов-потомков, которые должны выполняться более одного раза и 
нет очевидного способа клонировать её _в них_.

Здесь пригождается синтаксис `clone:`. Вызов `clone:name` клонирует `name` _перед_ перемещением в дочерние элементы `<Inner/>`,
что решает проблему с владением.

```rust
view! {
	<Outer>
		<Inner clone:name>
			<Inmost name=name.clone()/>
		</Inner>
	</Outer>
}
```

Эти проблемы могут быть немного сложны для понимания и отладки, из-за непрозрачности макроса `view`.
Но в целом, их всегда можно решить.

