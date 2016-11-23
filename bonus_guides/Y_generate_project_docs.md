The correct invocation of helper modules and functions can be intimidating because
* these are generated dynamically (e.g., when creating a new project or adding a new `resource`)
* they are not documented explicitly (e.g., `MyApp.ErrorHelpers.error_tag`) 
* the documentation does not cover all examples (e.g., `MyApp.Router.Helpers.*_path` in `Phoenix.Router`).

Although the created helpers are scattered all over your project but their location follows a solid logic. You can get used to them pretty quickly and fortunately, when you generate a project with Phoenix, the code is shipped with documentation via Elixir's `@doc` and `@moduledoc` module attributes.

These docs are not limited to helpers only but you can also
* see your project broken down by submodules / functions / macros
* add your own documentation
* look up any functions that were generated under the namespace of your project (e.g., `MyApp.Repo` contains callback function implementations from `Ecto.Repo`)

To generate documentation from your source code, add `ex_doc` as dependency into your `mix.exs` file: 

```elixir
# ./mix.exs

def deps do
  [{:ex_doc, "~> 0.11", only: :dev}]
end
```

> You can use Markdown within Elixir `@doc` and `@moduledoc` attributes.

Go to the console and run
```bash
 # To fetch and compile the added module:
 $ mix deps.get 
 
 # To generate documentation in the project root:
 $ mix docs 
```
If you would like to serve it in the current project:
```bash
$ mix docs --output priv/static/doc

# ... and edit ./lib/my_project/endpoint.ex:
#
#  plug Plug.Static,
#    at: "/", from: :timesheet, gzip: false,
#    only: ~w(css fonts images js favicon.ico robots.txt doc)
#                                                        ^^^
``` 
To see the result by navigating to `my_app_url_or_ip/doc/index.html`

**Additional reading:**

- [`ex_doc`](https://github.com/elixir-lang/ex_doc)
- [Version requirement operators (`Elixir.Version`)](http://elixir-lang.org/docs/stable/elixir/Version.html)
- [How to serve static assets from custom folders in Phoenix](http://stackoverflow.com/questions/38176422/how-to-serve-static-assets-from-custom-folders-in-phoenix)

The bulk of this guide is referenced from [Elixir Recipes](http://elixir-recipes.github.io/documentation/documentation-with-exdoc/).
