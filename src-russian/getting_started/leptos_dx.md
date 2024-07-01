# Улучшения опыта разработчика на Leptos

Есть пара вещей, которые можно сделать, чтобы улучшить ваш процесс разработки веб-сайтов и приложения на Leptos.
Возможно, вам стоит потратить несколько минут и настроить окружение, чтобы оптимизировать ваш процесс разработки,
особенно если вы хотите программировать попутно с примерами из этой книги.

## 1) Настройте `console_error_panic_hook`

По-умолчанию паники, которые происходят во время выполнения вашего WASM кода в браузере просто выбросят ошибку в браузере
с неинформативным сообщением вроде `Unreachable executed` и трассировкой стека, указывающей на WASM бинарник.

Используя `console_error_panic_hook` вы получите настоящую трассировку стека Rust с номерами строк в Rust коде.

Настроить это очень просто:

1. Выполните `cargo add console_error_panic_hook` в своём проекте
2. В функции main добавьте вызов `console_error_panic_hook::set_once();`

> Если это не понятно, [вот пример](https://github.com/leptos-rs/leptos/blob/main/examples/counter/src/main.rs#L4-L15).

Теперь сообщения о паниках в консоле браузере будут намного лучше!

## 2) Автоподстановка в редакторе внутри `#[component]` и `#[server]`

Из-за природы макросов (они могут разворачивать что угодно из чего угодно, но только если ввода вполне корректен в тот 
момент) rust-analyzer'у может быть сложно делать должную автоподстановку и другую поддержку.

При проблемах с использований этих макросов в вашем редакторе, можно настроить `rust-analyzer` так, чтобы он  
игнорировал определенные процедурные макросы. Особенно для макроса `#[server]`, который добавляет аннотации к телам функций,
но в действительно ничего не трансформирует в теле функции, это может быть очень полезно.

Начиная с версии Leptos 0.5.3, поддержка rust-analyzer была добавлена для `#[component]`, но если вы столкнулись с проблемами,
вы можете также добавить `#[component]` в игнор лист (см. ниже).
Учтите что это будет означать, что rust-analyzer ничего не будет знать о свойствах вашего компонента, что может породить 
собственный набор ошибок или предупреждений в IDE>

VSCode `settings.json`:

```json
"rust-analyzer.procMacro.ignored": {
	"leptos_macro": [
        // optional:
		// "component",
		"server"
	],
}
```

`VSCode` с `cargo-leptos` `settings.json`:
```json
"rust-analyzer.procMacro.ignored": {
	"leptos_macro": [
        // optional:
		// "component",
		"server"
	],
},
// if code that is cfg-gated for the `ssr` feature is shown as inactive,
// you may want to tell rust-analyzer to enable the `ssr` feature by default
//
// you can also use `rust-analyzer.cargo.allFeatures` to enable all features
"rust-analyzer.cargo.features": ["ssr"]
```

`neovim` с `lspconfig`:

```lua
require('lspconfig').rust_analyzer.setup {
  -- Other Configs ...
  settings = {
    ["rust-analyzer"] = {
      -- Other Settings ...
      procMacro = {
        ignored = {
            leptos_macro = {
                -- optional: --
                -- "component",
                "server",
            },
        },
      },
    },
  }
}
```

Helix, в `.helix/languages.toml`:

```toml
[[language]]
name = "rust"

[language-server.rust-analyzer]
config = { procMacro = { ignored = { leptos_macro = [
	# Optional:
	# "component",
	"server"
] } } }
```

Zed, в `settings.json`:

```json
{
  -- Other Settings ...
  "lsp": {
    "rust-analyzer": {
      "procMacro": {
        "ignored": [
          // optional:
          // "component",
          "server"
        ]
      }
    }
  }
}
```

SublimeText 3, `LSP-rust-analyzer.sublime-settings` в `Goto Anything...` меню:

```json
// Settings in here override those in "LSP-rust-analyzer/LSP-rust-analyzer.sublime-settings"
{
  "rust-analyzer.procMacro.ignored": {
    "leptos_macro": [
      // optional:
      // "component",
      "server"
    ],
  },
}
```


## 3) Настройка `leptosfmt` с `rust-analyzer` (необязательно)

`leptosfmt` это форматер для Leptos макроса `view!`  (внутри которого вы обычно пишете UI код).
Поскольку макрос `view!` включает 'RSX' (как JSX) стиль написания UI, cargo-fmt сложнее авто-форматировать код внутри макроса `view!`. `leptosfmt` это крейт, который решает проблемы с форматированием и поддерживает чистоту и красоту UI кода в стиле RSX.

`leptosfmt` может быть установлен и использован через командную строку или из редактора кода:

Для начала установите его командой  `cargo install leptosfmt`.

Если хотите использовать настройки по-умолчанию из командной строки, просто запустите `leptosfmt ./**/*.rs` из корня вашего проекта, чтобы отформатировать все Rust файлы используя `leptosfmt`.

Если хотите настроить ваш редактор для работы с `leptosfmt` или хотите кастомизировать настройки `leptosfmt`, пожалуйста обратитесь к инструкциям доступным в [`leptosfmt` github repo's README.md](https://github.com/bram209/leptosfmt).

Только учтите, что для наилучших результатов рекомендуется настраивать работу редактора c `leptosfmt` на уровне workspace.
