# Пробрасывание дочерних элементов

В процессе построения компонентов, может возникнуть желание как бы "пробросить" дочерний элемент сквозь несколько слоёв компонентов.

## Суть проблемы

Рассмотрим следующее:

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

Все довольно прозаично: когда пользователь "вошёл", показываем `children`. Если же нет, то отображаем `fallback`. А пока ожидаем ответа, рендерим `()`, т.е. ничего.

Другими словами, пытаемся передать дочерний элемент `<LoggedIn/>` _сквозь_ `<Suspense/>`, чтобы тот уже стал дочерним компонента `<Show/>`. Что-то подобное можно понимать под "пробрасыванием".

Только дело в том, что этот код не скомпелируется:

```
error[E0507]: cannot move out of `fallback`, a captured variable in an `Fn` closure
error[E0507]: cannot move out of `children`, a captured variable in an `Fn` closure
```

Проблема в том, что и `<Suspense/>`, и `<Show/>` должны иметь возможность конструировать своих `children` несколько раз. При первом построении дочерних элементов `<Suspense/>`, им будет взято право владения и `fallback`, и `children`, чтобы можно было передать их для вызова в `<Show/>`. Однако, из-за этого они перестают быть доступны для построения будущих дочерних компонентов `<Suspense/>`.

## Детали

> Можете смело пропустить этот раздел и перейти к решению.

Если есть желание действительно понять, что конкретно происходит, в этом может помочь развёрнутый `view` макрос. Вот "причёсанная" версия:

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

Каждый компонент владеет своими параметрами; от чего нет возможности запустить `<Show/>` в этом сценарии, поскольку тот захватил лишь ссылки на `fallback` и `children`.

## Решение

Тем не менее и `<Suspense/>`, и `<Show/>` принимают `ChildrenFn`, что значит `children` должен реализовывать тип `Fn`, чтобы его можно было вызывать многократно, используя лишь неизменяемую ссылку. Это означает, что нет нужды владеть `children` или `fallback`; достаточно иметь возможность передавать `'static` ссылки на них.

Этого можно достичь, используя примитив [`store_value`](https://docs.rs/leptos/latest/leptos/fn.store_value.html). Суть в том, что он сохраняет значение в реактивной системе, передавая управление владением фреймворку, подобно сигналам. В обмен получаем ссылку `Copy` и `'static`, к которой можно в дальнейшем обращаться и модифицировать через определённые методы.

Для данного случая, всё довольно наглядно:

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

На верхнем уровне храним `fallback` с `children` в реактивной области видимости, принадлежащей `LoggedIn`. Теперь можно просто передать эти ссылки сквозь остальные слои прямиком в `<Show/>` и вызвать их там.

## Примечание напоследок

Заметим, что это работает благодаря тому, что `<Show/>` и `<Suspense/>` нужна только неизменяемая ссылка на их дочерние элементы (которую `.with_value` может предоставить), а не право собственности.

В других ситуациях может понадобиться пробрасывать владеемый параметр сквозь функцию, что принимает `ChildrenFn`, от чего тот будет вызван более одного раза. В этом случае может помочь `clone:` в макросе `view`.

Рассмотрим следующий пример:

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

Даже при использовании `name=name.clone()`, компилятор выдёт ошибку

```
cannot move out of `name`, a captured variable in an `Fn` closure
```

Этот параметр передаётся через несколько уровней дочерних элементов, что должны запускать более одного раза. Как итог нет очевидного способа клонировать его _в_ дочерние элементы.

Здесь и пригождается `clone:` синтакс. Вызов `clone:name` клонирует `name` _перед_ перемещением его в дочерние элементы `<Inner/>`, что решает проблему владения.

```rust
view! {
	<Outer>
		<Inner clone:name>
			<Inmost name=name.clone()/>
		</Inner>
	</Outer>
}
```

Данные проблемы могут быть несколько сложны в понимания и отладке из-за непрозрачности макроса `view`. Но в целом их всегда можно решить.
