Public websites typically need to have some basic search engine optimization (SEO) tags, such as `<title>` and `<meta name="description">`. These tags obviously need to have different content on different pages.

There are several ways we can achieve this in Phoenix, but the recommended way is to use [`render_existing/3`](http://hexdocs.pm/phoenix/Phoenix.View.html#render_existing/3).

## Update Layout

Update the `<head>` section of our `web/templates/layout/app.html.eex` file to look like this:

```html
<head>
  <%= render_existing(@view_module, "meta." <> @view_template, assigns) ||
      render_existing(@view_module, "meta.html", assigns) ||
      render("meta.html", assigns) %>
</head>
```

This tells the layout to look for and render "meta" templates in the following order:

- Render `templates/{view_name}/meta.{action_name}.html.eex` if present
- Render `templates/{view_name}/meta.html.eex` if present
- Render `templates/layout/meta.html.eex` as a default

We'd then need to add the `layout/meta.html.eex` file, and add some default values:

```html
<title>Default Title</title>
<meta name="description" content="Default description" />
```

## Usage

The three-stage rendering in our layout allows for fine-grained control over our titles and meta descriptions.

1. If we don't want to customize meta content for a given controller action, we don't have to do anything. The defaults will be rendered.

2. If all actions in our controller can share the same metadata, just add a top level `meta.html.eex` file to `templates/{controller_name}/`, and all the actions will render it.

3. If we need specific metadata for a given controller action, create a `meta.{action_name}.html.eex` file in `templates/{controller_name}`.

Alternatively, if we don't want to create all those extra files, we can define them as functions on our view modules instead. For example, we could define the `render` function for an index action's meta information like so:

```elixir
def render("meta.index.html", _assigns) do
  ~E{
    <title>Title Here</title>
    <meta name="description" content="..." />
  }
end
```
