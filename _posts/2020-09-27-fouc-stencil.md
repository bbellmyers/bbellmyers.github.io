---
layout: post
title: 'Removing FOUC in Stencil.'
date: 2020-09-27
---

## That Pesky Flash of Unstyled Content

Our team is building a library of components for our design system, using
Stencil. We're big fans of the tool, and love the reusability of web
components.

However, some component library users have pointed out individual instances of a
[flash of unstyled
content](https://en.wikipedia.org/wiki/Flash_of_unstyled_content) (FOUC)
when using the component library.  This means a short time during the
load of a web page when raw HTML is shown to the user.  Then, the page
is quickly re-styled using the CSS embedded in our components.  In
addition, users have pointed out a new measure of performance in [Chrome
Lighthouse](https://developers.google.com/web/tools/lighthouse), the
\"Cumulative Layout Shift\", and wanted to understand how this may or
may not relate to the FOUC they were experiencing.

So, I did a spike to:

1.  understand the underlying causes of this FOUC
2.  determine best practices for avoiding it
3.  optimize lighthouse scores
4.  optimize user perceived performance

### Example project

I'll use [this example project](https://github.com/bbellmyers/fouc-stencil){:target="\_blank"}
to demonstrate.  Here\'s how it looks when it loads with no adjustment. 
Note the FOUC at the start.

<video controls>
    <source src="/media/fouc-example1.mp4"
            type="video/mp4">
    Sorry, your browser doesn't support embedded videos.
</video>

This example app has one component, so it's very fast. Our library has close
to 100 components.

### Hydrated Components

[Several steps take
place](https://stenciljs.com/docs/component-lifecycle) when Stencil web
components are rendered on a page.  Each of these steps takes a very
short, but measurable, amount of time:

1.  The custom tag is registered with the browser (first time only)

2.  Tag instance is initialized + connected to the DOM

3.  CSS for this component is appended to the head of the document as a
    \<style\> tag.

4.  Tag instance is rendered

5.  The tag is marked with the class \"hydrated\".

So, a brief period of time elapses between the time that the document
loads and when the component is hydrated.  If the component contains a
\<slot /\> for child components, those children won\'t be styled
immediately, creating a FOUC.  This time can vary considerably,
depending on how many children are displayed, and how deeply nested the
document may be.  However, it is usually less than 20 milliseconds.

### Stencil\'s Attempt

So, how to hide this unstyled content?  Stencil does already
automatically render a \<style\> block to the header which renders the
component \`visibility: hidden\` if the .hydrated class is not applied. 
However, it does not seem to prevent FOUC.  Why?

![](/media/stencil-hydrated.png)

The answer is because **_HTML is always faster than JavaScript_**.  The
HTML content loads first, and then the JavaScript inserts the CSS
dynamically, several milliseconds later.  It\'s fast, but not faster
than the human eye.

And, it doesn't matter if your component is using shadow DOM or not, you
still get a flash, FYI. Yeah, I thought so too. No, doesn't help.

Also, Stencil generates a style rule for EVERY component in your library. In this
one component example, thats no biggie. For our library of 100 components,
it's one long css rule.

### Previous attempts

A number of our components do something similar, by applying
:not(.hydrated){ visibility: hidden } rules to components with a \<slot
/\>.  This helps a little, but fundamentally has the same problem as
Stencil\'s attempt above.  Well, we got an \"E\" for effort.

### Getting the FOUC out

My first attempt then, was to load the exact same set of CSS rules,
extracted to a static CSS file, and load that into the page.  This
worked!  That\'s because CSS files linked in the head of the document
are applied immediately, before DOM content is shown.

Great!  We\'re done right?  Wrong.  Lighthouse HATES it when you delay
first content paint.

### Google Lighthouse Called, It Hates You

So here\'s what our Lighthouse score for our large library was before I added
the static CSS file.  Not great, but it\'s a baseline to start from.  I ran my tests on
the development build of the app, and it\'s not optimized, etc. 
Nevertheless:

![](/media/lighthouse1-no-mask.png)

And here\'s what it was AFTER putting in that static file:

![](/media/lighthouse2-all-mask.png)

- First Contentful Paint is slightly better
- Speed Index has turned orange, but the value is the same

Interestingly, Largest Contentful Paint and the Speed Index are slightly
better, but not better enough.  Even more interesting, Cumulative Layout
Shift has NOT CHANGED AT ALL.

Let's see if we can do better.

### Mask what Needs Masking

So, hiding components before they are hydrated makes the Flash of
Unstyled Content go away.  But it impacts our Lighthouse score.  So what to do?

Well, it turns out MOST components don\'t need to be hidden, they don\'t
contain static DOM elements anyway.  Only those with \<slots /\> are a
problem here.  And not even all of those are normally an issue.  So we
can focus our efforts here, and create a file that only hides slotted
components.

This style is much smaller, and it only hides what needs to be hidden.

So, how does the app perform with this new, smaller, css file?

![](/media/lighthouse3-some-mask.png)

BINGO!  Not only is the FOUC gone, but we\'ve even tweaked our speed index
and time to interactive.  Not bad.
