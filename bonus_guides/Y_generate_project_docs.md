The correct invocation of helper modules and functions can be intimidating because these are generated dynamically (for example, when creating a new project or adding a new `resource`) and they are not documented explicitly (e.g., `MyApp.ErrorHelpers.error_tag`) or the documentation does not cover all examples (e.g., `MyApp.Router.Helpers.*_path` in `Phoenix.Router`).

To generate documentation from `@doc` and `@moduledoc` attributes in your source code, add `ex_doc` and a markdown processor as dependencies into your `mix.exs` file: 

```elixir
# config/mix.exs

def deps do
  [{:ex_doc, "~> 0.11", only: :dev}]
end
```

> You can use Markdown within Elixir `@doc` and `@moduledoc` attributes.

Then, run `mix deps.get` to fetch and compile the new modules and generate the project documentation with `mix docs`. 
An example output is the [official Elixir Docs](http://elixir-lang.org/docs/stable/elixir/).



**Additional reading:**

- [`ex_doc`](https://github.com/elixir-lang/ex_doc)
- [Version requirement operators (`Elixir.Version`)](http://elixir-lang.org/docs/stable/elixir/Version.html) 

The bulk of this guide is referenced from [Elixir Recipes](http://elixir-recipes.github.io/documentation/documentation-with-exdoc/).
