---
layout: post
title: "How to Exclude Code Line Numbers from Selection in Jekyll"
date: 2015-06-30 21:42:12
categories: "jekyll css scss"
author: "Paul Faria"
---

I ran into an issue when trying to use the `linenos` option for highlighting code with Jekyll and pygments. I didn't like that the default version allowed you to select the line numbers when copying and pasting. The table version also has many issues (see [this post](http://thanpol.as/jekyll/jekyll-code-highlight-and-line-numbers-problem-solved/)). I didn't like the idea of having to write javascript to get around this. The problem seemed to me like it could be solved in a much simpler fashion. My solution was to add the following scss to my site:

{% highlight scss linenos %}
code {
    counter-reset: linenumber; // reset the counter every time we start a new code block
}

.lineno {
    visibility: hidden; // I want to hide the selectable line number
    font-size: 0;       // this makes it so the hidden content takes up no space

    &::before {
        font-size: 1rem;                 // use rem since the em would be relative to 0 font-size above
        content: "" counter(linenumber); // display the current line number count (
        counter-increment: linenumber;   // increment the line number each time it's rendered
        visibility: visible;             // override parent's hidden
        display: inline-block;           // so we can specify a width
        width: 1rem;                     // give us enough width to pad the line number
                                         // should work unless I decide to blog over 1000 lines
                                         // ... which I don't plan on doing anyways
    }
}

// notice that copying and pasting from this code block does not include the line numbers
{% endhighlight %}

This allowed me to avoid having to fix this ugly monstrosity (it's the same code as above, just using `linenos=table`):

{% highlight scss linenos=table %}
code {
    counter-reset: linenumber; // reset the counter every time we start a new code block
}

.lineno {
    visibility: hidden; // I want to hide the selectable line number
    font-size: 0;       // this makes it so the hidden content takes up no space

    &::before {
        font-size: 1rem;                 // use rem since the em would be relative to 0 font-size above
        content: "" counter(linenumber); // display the current line number count (
        counter-increment: linenumber;   // increment the line number each time it's rendered
        visibility: visible;             // override parent's hidden
        display: inline-block;           // so we can specify a width
        width: 1rem;                     // give us enough width to pad the line number
                                         // should work unless I decide to blog over 1000 lines
                                         // ... which I don't plan on doing anyways
    }
}

// notice that copying and pasting from this code block does not include the line numbers
{% endhighlight %}
