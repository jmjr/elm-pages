---
{
  "type": "blog",
  "author": "Dillon Kearns",
  "draft": true,
  "title": "Extensible Markdown Parsing in Pure Elm",
  "description": "Elm is the perfect fit for a static site generator. Learn about some of the features and philosophy behind elm-pages.",
  "image": "/images/article-covers/extensible-markdown-parsing.jpg",
  "published": "2019-10-06",
}
---

I'm excited to share a new approach to markdown parsing for the Elm ecosystem: [`dillonkearns/elm-markdown`](https://package.elm-lang.org/packages/dillonkearns/elm-markdown/latest/)!

As a matter of fact, the blog post you're reading right now is being rendered with it! If you want to jump straight to a live demo, you can check out:

- [This Ellie example showing custom HTML elements in action](https://ellie-app.com/6QH7BxdcWxKa1), or
- [This Ellie example that extracts a table of contents](https://ellie-app.com/6QtYW8pcCDna1) from the parsed markdown blocks

## Why does Elm need another markdown parser?

I built this tool was so that I could:

- Render my markdown blocks using my preferred UI library (`elm-ui`, in my case, but you could use `elm-css` or anything else!)
- Extend what can be expressed beyond the standard markdown blocks like headings, lists, etc.
- Inject data from my Elm model into my markdown

And yet, I wanted to do all of this with the benefits that come from using standard Markdown:

- Familiar syntax
- Great editor tooling (I write my blog posts in Ulysses using Markdown, and I have prettier set up to auto format markdown when I'm tweaking markdown directly in my code projects)
- Previews render in Github
- Easy for others to contribute (for example, to the [`elm-pages` docs](https://elm-pages.com/docs))

## Core Features

So how do you get the best of both worlds? There are three key features that give `dillonkearns/elm-markdown` rich extensibility without actually adding to the Markdown syntax:

- ⚙️ **Map HTML to custom Elm rendering functions** - to give you fully extensible markdown!
- 🎨 **Use [custom renderers](https://package.elm-lang.org/packages/dillonkearns/elm-markdown/latest/Markdown-Parser#Renderer)** - to allow users to do custom rendering using their preferred style and UI library
- 🌳 **Give users access to the parsed Markdown Blocks before rendering** - (i.e. abstract syntax tree) allow users to inspect, transform, or extract data from the parsed Markdown before passing it on to their custom Markdown Renderer

Let's explore these three key features in more depth.

## ⚙️ Map HTML to custom Elm rendering functions

I didn't want to add additional features that weren't already a part of Markdown syntax. Since HTML is already valid Markdown, it seemed ideal to just use declarative HTML tags to express these custom view elements. `dillonkearns/elm-markdown` leverages that to give you a declarative Elm syntax to explicitly say what kind of HTML is accepted (think JSON Decoders) and, given that accepted HTML, how to render it.

If you're familiar with [MDX](https://mdxjs.com) (it's Markdown syntax, but extended with some extra syntax from JSX, including like JS `import`s and JSX HTML tags). Guillermo Rauch, the creator of MDX even talks about the benefits that a more declarative approach, like the one `dillonkearns/elm-markdown` takes, could have over the current MDX approach of using low-level `import` statements and JSX syntax [in this talk (around 20:36 - 22:30)](https://www.youtube.com/watch?v=8oFJPVOT7FU&feature=youtu.be&t=1236).

### Avoiding low-level HTML in markdown

I like to think of the HTML tags within these markdown documents as similar to a WebComponent. It should be a very high-level way of expressing the data you need for a custom view. With standard Github-flavored markdown, you'll often see people injecting `<div>` tags with styles, or `<img>` tags, etc. To me, I consider this too low-level to be injecting into Markdown. The Markdown document should be more declarative, concerned only with _what_ to render, not _how_ to render it.

### An example of custom HTML in `dillonkearns/elm-markdown`

Let's say we want to abstract the expected data and presentation logic for team members' bios on an `about-us` page. And you'll probably want to have richer presentation logic than plain markdown gives you (for example, showing icons with the right dimensions, and displaying them in a row not column view, etc.) Also, since we're using Elm, we get pretty spoiled by explicit and precise error messages. So we'd like to get an error message if we don't provide a required attribute! Here's what that might look like:

```elm
<bio
	name="Dillon Kearns"
	photo="https://avatars2.githubusercontent.com/u/1384166"
	twitter="dillontkearns"
	github="dillonkearns"
>
Dillon really likes building things with Elm!

Here are some links:

- [Articles](https://incrementalelm.com/articles)
</bio>
```

Our Elm code to handle this type of HTML tag looks like this:

```elm
Markdown.Html.oneOf
  [ Markdown.Html.tag "bio"
    (\name photoUrl twitter github dribbble renderedChildren ->
      bioView renderedChildren name photoUrl twitter github dribbble
    )
      |> Markdown.Html.withAttribute "name"
      |> Markdown.Html.withAttribute "photo"
      |> Markdown.Html.withOptionalAttribute "twitter"
      |> Markdown.Html.withOptionalAttribute "github"
      |> Markdown.Html.withOptionalAttribute "dribbble"
  ]
```

Now, if we forget to pass in a `photo` attribute, we'll get an error message like this:

```
Problem with the given value:

<bio
	name="Dillon Kearns"
	twitter="dillontkearns"
	github="dillonkearns"
>

Expecting attribute "photo".
```

## 🎨 Use custom renderers

Many Markdown libraries just give you the rendered HTML directly. With `dillonkearns/elm-markdown`, one of the main goals was to give you full control over presentation at the initial render (rather than needing to add CSS rules to apply to your rendered output). I personally like to use `elm-ui` whenever I can, so I wanted to use that directly not just for my navbar, but to style my rendered markdown blocks.

Beyond just rendering directly to your preferred UI library, custom Renderers also open up a number of new potential uses. You can render your Markdown into `elm-ui` `Element`s, but you could also render it to any other Elm type. That could be data, or even functions. Why would you render a function? Well, that would allow you to inject dynamic data from your Elm model!

Some other use cases that custom Renderers enable:

- Regular old `Html` (using the [`defaultHtmlRenderer`](https://package.elm-lang.org/packages/dillonkearns/elm-markdown/latest/Markdown-Parser#defaultHtmlRenderer))
- Render into [`elm-ui`](https://package.elm-lang.org/packages/mdgriffith/elm-ui/latest/) `Element`s
- Render into ANSI color codes for rich formatting in terminal output
- Render into plain text with all formatting stripped out (for search functionality)

### Performing validations in Renderers

Another goal with `dillonkearns/elm-markdown` is to allow early and precise feedback. One of my favorite uses of Custom Renderers is to catch dead links (or images). `elm-pages` will stop the production build when the Renderer fails. [Here's the relevant code](https://github.com/dillonkearns/elm-pages/blob/c76e96af497406fb9acf294acebbcb0c0e391197/examples/docs/src/MarkdownRenderer.elm#L90-L93) from elm-pages.com

```elm
renderer : Markdown.Parser.Renderer (Element msg)
renderer =
	{
	link =
		\link body ->
		Pages.isValidRoute link.destination
			|> Result.map
	, -- rest of the Renderer definition
	}
```

## 🌳 Give users access to the parsed Markdown Blocks before rendering

Exposing the AST allows for a number of powerful use cases as well. And it does so without requiring you to dig into the internals. You just get access to a nice Elm custom type and you can do what you want with it before passing it on to your Custom Renderer.

Here are some use cases that this feature enables:

- Extract metadata before rendering, like building a table of contents data structure with proper links ([here's an Ellie demo of that!](https://ellie-app.com/6QtYW8pcCDna1))
- Run a validation and turn it into an `Err`, for example, if there are multiple level 1 headings (having multiple `h1`s on a page causes accessibility problems)
- Transform the blocks by applying formatting rules, for example use a title casing function on all headings
- Transform the AST before rendering it, for example dropping each heading down one level (H1s become H2s, etc.)

## The future of `dillonkearns/elm-markdown`

I've been really enjoying using this in production for several weeks. But it certainly isn't fully handling all cases in Github-flavored markdown.

I'm running all 1400 end-to-end test cases from the Marked.js test suite (which is what `elm-explorations/markdown` runs under the hood). And that test suite includes running through every example in the [Github-flavored markdown spec](https://github.github.com/gfm/). I'm working through handling more of these cases to make it more widely useful, but feel free to use it now with that caveat in mind.

Pull requests are very welcome, I would love community contributions on this project!

### Fault-Tolerance Versus Helpful Errors

That said, the goal is not to get to 100% compliance with the Github-Flavored Markdown Spec. Markdown has a goal of being Fault-Tolerant, meaning it will always try to "do the best it can" rather than giving an error message when something unexpected happens. That means there's no such thing as "invalid markdown." But there is most certainly **"markup that probably doesn't do what you expected."** For example

```
[My link](/home oh wait I forgot to close this link tag...
```

⚠️ This is technically **valid** Markdown! ⚠️

It "does the best it can" with the input and renders to a raw string rather than rendering a link. So this is an example that is squarely in the category of markup that **"probably doesn't do what you expected."**

The goal of `dillonkearns/elm-markdown` is not fault-tolerance. It prioritizes **helpful error messages** over fault-tolerance. Sound familiar? There is a very similar difference in philosophy between JavaScript and Elm.

So the rule of thumb for `dillonkearns/elm-markdown` is:

- Follow the Github-Flavored Markdown Spec whenever it doesn't cover up feedback about something that "probably doesn't do what you expected"
- Otherwise, break with the Github-Flavored Markdown Spec and instead give a helpful error message

You can follow along with the [current GFM Spec Compliance here](https://github.com/dillonkearns/elm-markdown#current-github-flavored-markdown-compliance).

Thanks for reading! If you give this library a try, let me know what you think. I'd love to hear from you!