From time immemorial, web developers have employed templates and includes in order to re-use site-wide layouts on different pages, without having to rewrite associated code.

TODO: Insert Image (of `ot-site` with real content)

Developers making Single Page Applications no longer need to re-use layouts on multiple pages--but we continue to have repeating layout structures, like content panels *within* our applications.

At OpenTable, we're using an approach we call "micro-apps"; each micro-app is a separate SPA. We quickly realized that in addition to content panels, we also still needed a way to efficiently share our top-level product layout and navigation... so we considered our options.

## Templates & Includes

Long, long ago we used backend template systems, which allowed us to separate and save portions of our HTML, then include it on different pages.

```markup
<!-- #include virtual="head.html" -->
<!-- #include virtual="menu.html" -->
I render in <!--#echo var="CONTAINER" -->.
<!-- #include virtual="foot.html" -->
```

Later, when Ajax created the need, we used JavaScript templates. These were templates we could use on the client. Even better, we could use them on both the client *and* the server.

```markup
{{> head}}
{{> menu}}
I render in {{container}}.
{{> foot}}
```

## UI Components

The most modern approach to this sort of code re-use, however, is a UI component, ideally linked to a custom tag.

```markup
<ot-site>
    I render in body.
</ot-site>
```

This what we did at OpenTable. As part of our evolving UI component library, we created an `ot-site` UI component to serve as a layout scaffold and content container.

### Native Elements 

Why a UI component? Well, it's inspired by how native HTML elements work. For example, a native range `input` brings everything it needs for its appearance and functionality along with it. The `input` tag "bootstraps" all the templates, scripts, and styles required for a range slider.


<input type="range" min="1" max="8" />
```markup
<input type="range" min="1" max="8" />
```

As a result, when we use the range `input`, we don't have to write markup to make the separate parts of the range slider: the track and handle. No additional styles are required to make either the track look like a track or the handle look like a handle. Nor do we need to write any script to move the handle along the track or set a corresponding `value` property. All of that comes built in. 

Furthermore, the `input` tag also exposes a way to control options relevant to a range slider, like min and max values. Setting these options change the slider's behavior--again, without additional scripting.

All in all, native HTML elements are powerful. What better way to share our custom functionality on the web than to emulate them?

### Content Containers

Unlike an `input`, though, OpenTable's `ot-site` layout scaffold is a content container. It needs to behave more like a `div` does and wrap content so that, when employing `ot-site` as component users, we can insert blocks of arbitrary HTML into the scaffold containers. 

More, `ot-site` needs to be able to accept content at more than one location. Clearly, for `ot-site`, we need more than attributes. We need `ot-site`'s template to have special points at which we can insert content, and we also need a way to label our content for a specific insertion point.

#### Content Insertion Points

That's a good, generic name: content insertion points.  That's exactly what we need.  And, spoiler, it's also the the name the Shadow DOM specification uses.  

In part 1 of 3 of this series, we'll implement `ot-site` and content insertion points using Web Components.

Many frameworks provide functionality of this sort in slightly different ways, however.  Parts 2 and 3  by [Kara Erickson](https://twitter.com/karaforthewin) will cover Angular 1.3 and Angular 2.0.

## `ot-site`

### Component Development

First things first, we have to write the HTML that will be built into our `ot-site` component, remembering to mark each pre-designated insertion point.

```markup
<div id="site">
    <header>
        <svg id="logo"></svg>
        <!-- insertion point 1 -->
    </header>
    <nav>
        <!-- insertion point 2 -->
    </nav>
    <main>
        <!-- insertion point 3 -->
    </main>
    <footer>
        &copy; 2015 OpenTable, Inc.
    </footer>
</div>
<style>
    /* use your imagination */
</style>
```

The anatomy of `ot-site` is a head, menu, body, and foot. The head contains the OpenTable logo and the foot contains a copyright statement. The head, menu, and body will contain content insertion points.  We'll also style `ot-site` so that it looks the way it should:

TODO: Insert Image (of empty `ot-site`)

### Component Use

Next, let's set up an example of how we'll use `ot-site` and add some sample content by nesting it inside, remembering to label where the content should end up.

```markup
<ot-site>
    <div>
        <!-- for insertion point 1 -->
        I render in head.
    </div>
    <div>
        <!-- for insertion point 2 -->
        I render in menu.
    </div>
    <div>
        <!-- for insertion point 3 -->
        I render in body.
    </div>
</ot-site>
```

Now that we've chosen our mission, outlined what we need, and written our markup, we'll have to:

1. Replace our comments with something real
1. Match the content with its insertion point
2. Combine and render the result


## Web Components

Web Components aim to provide content insertion point functionality natively with the Shadow DOM specification.

### Native Elements

```markup
<input type="range" />
```

An interesting thing about the Shadow DOM is that it formally describes how native elements already work. A shadow DOM is a large part of how the range `input` does all the things it does.

TODO: Insert Image (of chrome dev tools setting)

If we turn on "Show user agent Shadow DOM" in Chrome developer tools, we can see a secret DOM that wasn't visible to us before. 

```markup
<input type="range" />

#shadow-root (user-agent)
  <div id="track">
    <div id="thumb">
    </div>
  </div>
```

The native element has a hidden shadow DOM associated with it. As you can see above, the shadow DOM is why we get a track and a handle when we use the range `input`.

To keep things straight, from here on we'll call the regular, non-shadow DOM in which the `input` element lives the "**light DOM**".

### Shadow DOM

And what exactly is a shadow DOM? It's a bit like a document fragment, however, a shadow DOM is a complete, separate DOM. Even when it's being rendered, a shadow DOM actually remains separate in many practical ways: 

- Scripts in a shadow DOM are encapsulated, 
- Events in a shadow DOM are re-targeted,
- And styles in a shadow DOM are scoped.

### Shadow Host

With web components support, we're able to create our own shadow DOM. But we  still start with the light DOM. We need an element in the light DOM to play host to the shadow DOM we'll create. This is where the `ot-site` tag comes in.

```markup
<ot-site>
    <div>
        <!-- for insertion point 1 -->
        I render in head.
    </div>
    <div>
        <!-- for insertion point 2 -->
        I render in menu.
    </div>
    <div>
        <!-- for insertion point 3 -->
        I render in body.
    </div>
</ot-site>
```

Before making `ot-site` a shadow host, this markup gives us:

TODO: Insert image  (of plain unstyled "I render in...")

### Shadow Root

Now, in order to make `ot-site` a shadow host, we have to create a shadow root on the element. The Shadow DOM specification defines a `createShadowRoot` method on elements which will do precisely that.

```javascript
var otSite = document.querySelector("ot-site");
otSite.createShadowRoot()
```

Creating and hosting a shadow DOM is as simple as that.  The result we'll see in developer tools is something like this:

```markup
<ot-site>
    #shadow-root
        <!-- shadow root is currently empty -->
    <div>
        <!-- for insertion point 1 -->
        I render in head.
    </div>
    <div>
        <!-- for insertion point 2 -->
        I render in menu.
    </div>
    <div>
        <!-- for insertion point 3 -->
        I render in body.
    </div>
</ot-site>
```

And, from this point on, any light DOM descendants of our gracious shadow host won't render. So our sample content is all missing now, which leaves our page looking like this:

TODO: Insert Image (of blank white page)

You may be wondering how that could possibly be useful. Well, now that we have a shadow DOM, we can add elements to it--and those will render.

### Filling the Shadow DOM

That means we can add the built-in parts of `ot-site`--its head, logo, menu, body, foot, copyright, and styles--to our shadow root. 

We could dress this up with more web component goodies like custom element registration and a template tag. But for this tutorial, we'll stick with the barest essentials and we'll just set `innerHTML` to add `ot-site`'s markup to the shadow root.

```javascript
otSite.shadowRoot.innerHTML = `
    <div id="site">
        <header>
            <svg id="logo"></svg>
            <!-- insertion point 1 -->
        </header>
        <nav>
            <!-- insertion point 2 -->
        </nav>
        <main>
            <!-- insertion point 3 -->
        </main>
        <footer>
            &copy; 2015 OpenTable, Inc.
        </footer>
    </div>
    <style>
        /* use your imagination */
    </style>`
```

Wondering about those backticks?  That's an ECMAScript 6 feature: template strings. When we use those we don't have to escape quotes in our `ot-site` HTML and we can use line breaks; it's very convenient for inline markup.  You'll have to use something like [Traceur](https://github.com/google/traceur-compiler) if you want to use template strings, though.

### Composed DOM

Now that we've added the `ot-site` layout to our shadow DOM, the below is the composite DOM we've actually created. `ot-site` is in the light DOM,  and it hosts a shadow root that contains the markup we want built into `ot-site`.

```markup
<ot-site>
    #shadow-root
        <div id="site">
            <header>
                <svg id="logo"></svg>
                <!-- insertion point 1 -->
            </header>
            <nav>
                <!-- insertion point 2 -->
            </nav>
            <main>
                <!-- insertion point 3 -->
            </main>
            <footer>
                &copy; 2015 OpenTable, Inc.
            </footer>
        </div>
        <style>
            /* use your imagination */
        </style>
    <div>
        <!-- for insertion point 1 -->
        I render in head.
    </div>
    <div>
        <!-- for insertion point 2 -->
        I render in menu.
    </div>
    <div>
        <!-- for insertion point 3 -->
        I render in body.
    </div>
</ot-site>
```

As a result, the image below is what we get when we use the `ot-site` tag. We get six tags, representing head, logo, menu, body, foot, and copyright as well as all the styles to lay them out for the price of a single tag.

TODO: Insert Image (of empty `ot-site`)

But we still need to be able to add our content, so let's do something real with those placeholder comments.

### `<content>`

The Shadow DOM specification also defines the `content` element, which creates a content insertion point. As mentioned earlier, content insertion points are how we can fill the containers in our layout scaffold with user-authored content.

```markup
<content></content>
```

When, like in `ot-site`, we want multiple content insertion points, we can use the optional `select` attribute. The value of the select attribute is a CSS selector, used to specify which parts of the light DOM can appear at each insertion point.

```markup
<content select="*"></content>
```

We'll replace the placeholder comments in `ot-site`'s layout scaffold with `content` tags using attribute selectors.  In the `header` of the code below, for example, this will match against any light DOM children of `ot-site` that have the `head` attribute name.

```markup
<div id="site">
    <header>
        <svg id="logo"></svg>
        <content select="[head]" />
    </header>
    <nav>
        <content select="[menu]" />
    </nav>
    <main>
        <content select="[body]" />
    </main>
    <footer>
        &copy; 2015 OpenTable, Inc.
    </footer>
</div>
```

We'll also need to add the attributes we defined above to our content in order to label the appropriate destination for each `div`. We just add, for example, the attribute name `head` on the div that should display in the header of `ot-site`.

```markup
<ot-site>
  <div head>
    I render in head.
  </div>
  <div menu>
    I render in menu.
  </div>
  <div body>
    I render in body.
  </div>
</ot-site>
```

### Matching

I mentioned that we'd need to match up the content and the insertion point.  But with the shadow DOM, the functionality [is native (sort of)](http://webcomponents.org/), so we actually don't have to deal with this ourselves.

By adding our `content` tags and defining selectors in the shadow DOM and adding the same to our content in the light DOM, we've enabled a browser ([or polyfill](http://webcomponents.org/polyfills/)) to match everything up... as long as we didn't make any typos.

There's an algorithm behind up the native functionality, of course, and [you can read about it if you want](http://w3c.github.io/webcomponents/spec/shadow/#distribution-algorithms).  But, mostly, keep in mind that in parts 2 and 3 of this series we'll look again at correlating content and insertion points in Angular 1.3 and Angular 2.0.

### Rendering

This leaves us with only the result and how it renders--which is a very important thing to keep in mind when using a shadow DOM. Things only *render* as if the light DOM and shadow DOM are combined. 

#### Content Projection

What is actually occurring is something people are calling content projection. The user's light DOM content is projected, like a movie onto the big screen, onto the insertion point--which makes it render in some ways as if it were there.

But it isn't. For all the purposes mentioned earlier--script execution context, events, and styles--our light DOM content remains in the light DOM and our shadow DOM content is still totally isolated in the shadow DOM. 

It's useful to think of this as if they're each processed separately before ever rendering together.

TODO: Insert Image (of empty `ot-site` and plain unstyled "I render in...")

So, say we had a style in the light DOM that made the text red. What happens?

```css
div {
    color: red;
}
```

We'll get red text, like in the image below--regardless of the rules in the shadow DOM's stylesheet, or how specific the selectors are, because the text doesn't move.

TODO: Insert Image (of red, filled `ot-site`)

The `div`s and their text contents are still in the light DOM and can, naturally, be styled by any CSS also in the light DOM.  But not by CSS in shadow DOM: `<div head>`, for example, won't be affected by any rules for `header div`.

But, by the same token, if we add the a rule to the light DOM stylesheet to make any text in `footer` red...

```css
div, footer {
    color: red;
}
```

The copyright text in our shadow DOM won't change, as the below image illustrates.

TODO: Insert Image (of correct/white, filled `ot-site`)

Content projection is about more than style scoping, of course, but styles provide a great visual!  You can see [a full demo of content projection](http://codepen.io/morewry/pen/zxajGB/) as described here on CodePen.

After we remove our educational red, content projection gives us exactly the end result we were aiming for--with the added benefit of a shadow DOM's encapsulation.  You can see [a full demo of `ot-site`](http://codepen.io/morewry/pen/azKWLQ/) as we implemented it here on CodePen.

### Conclusion

That covers the process of implementing the `ot-site` layout scaffold using web components and native functionality. Next, head over to Kara's articles on Angular 1.3 and Angular 2.0.

