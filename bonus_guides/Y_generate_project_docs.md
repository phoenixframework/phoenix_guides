The correct invocation of helper modules and functions can be intimidating because
* these are generated dynamically (e.g., when creating a new project or adding a new `resource`)
* they are not documented explicitly (e.g., `MyApp.ErrorHelpers.error_tag`) 
* the documentation does not cover all examples (e.g., `MyApp.Router.Helpers.*_path` in `Phoenix.Router`).

Although the created helpers are scattered all over your project but their location follows a solid logic. You can get used to them pretty quick and fortunately, when you generate a project with Phoenix, the code is shipped with documentation via Elixir's `@doc` and `@moduledoc` module attributes.

These docs are not limited to helpers only but you can also
* see your project broken down by submodules/functions/macros
* add your own documentation
* look up any functions that were generated under the namespace of your project (e.g., `MyApp.Repo` contains callback function implementations from `Ecto.Repo`)

To generate documentation from your source code, add `ex_doc` as dependency into your `mix.exs` file: 

```elixir
# config/mix.exs

def deps do
  [{:ex_doc, "~> 0.11", only: :dev}]
end
```

> You can use Markdown within Elixir `@doc` and `@moduledoc` attributes.

Then, run `mix deps.get` to fetch and compile the new modules and generate the project documentation with `mix docs`. 
An example output is the [official Elixir Docs](http://elixir-lang.org/docs/stable/elixir/).

To serve them immediately use `mix docs --output priv/static/doc` and navigate to `my_app_url_or_ip/doc/index.html`.

**Additional reading:**

- [`ex_doc`](https://github.com/elixir-lang/ex_doc)
- [Version requirement operators (`Elixir.Version`)](http://elixir-lang.org/docs/stable/elixir/Version.html) 

The bulk of this guide is referenced from [Elixir Recipes](http://elixir-recipes.github.io/documentation/documentation-with-exdoc/).
