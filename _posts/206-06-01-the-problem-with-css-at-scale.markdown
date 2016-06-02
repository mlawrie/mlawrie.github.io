---
layout: post
title:  "The Problem with CSS at Scale"
date:   2016-06-01 21:00:00 -0600
permalink: problem-with-css-at-scale
---

*CSS just wasn't designed to work for large, long-lived software projects.*

There's no mistake about it. Virtually every project I've seen has had a messy CSS code base. On the bigger projects, this has been particularly acute. They all go down the same road: bad ideas and good ideas alike  collect over time. It's hard to tell what CSS applies to what parts of the app. This makes deleting code dangerous so things tend to pile up. Slowly, "override hell" takes over and it becomes very difficult to make new changes. In exasperation, less-enlightened developers start to mark things as `!important`. This makes deleting code even more dangerous so it is almost never done. Eventually, you're left with a [ball of mud](https://en.wikipedia.org/wiki/Big_ball_of_mud).

If you've been there, don't be too quick to blame your team. CSS is really not a good technology for this sort of thing.

## First, some history

HTML wasn't designed for visual designers. It wasn't designed for web applications, either. Doing either thing with HTML was a total kludge. But since HTML became the lingua franca, kludge we did. Before CSS, the technique of the day was table layouts.

This wasn't what tables were intended to do, and this rankled folks like Berners-Lee and the W3C. Their vision of the web was entirely different. At the time, the intent was to build the [Semantic Web](https://en.wikipedia.org/wiki/Semantic_Web)" &mdash; a web that could be processed by computers and humans alike based on standards like HTML, XML, RDF, and so on. For this to work, table layouts were a [big problem](https://en.wikipedia.org/wiki/Tableless_web_design#Rationale). This was the basic rationale for CSS. It wasn't designed to produce beautiful web pages and it certainly wasn't designed to build interfaces for large-scale web applications. It was just designed as a way to get presentational logic out of HTML so that it could be more semantic.

As a result, CSS 1.0 has some glaring omissions. Perhaps most notably, no provisions for flexible two-column layouts were included. This was possible with float layouts, but float layouts only work as a result of unintended side-effects of the original CSS spec. This is a layout technique that was discovered, not designed. Key to this method is the [clear fix](https://css-tricks.com/snippets/css/clear-fix/), a wild kludge which was invented long after CSS became popular.

## CSS: The bad parts

Here's a brief survey of the worst parts of CSS:

*It's a global namespace:* beyond new inventions like the [CSS scoped attribute](http://www.w3schools.com/tags/att_style_scoped.asp), there's no way to scope CSS to given regions of the page. The only method to do this is making very specific CSS selectors.

*It's too easy to write very broad code:* if you define a style for `h1`, this will of course apply to every `h1` in your application. Without naming things very carefully, you will soon find that your CSS rules apply in unintended circumstances.

*It's impossible to understand what CSS does in a web app without running the app:* It's very difficult to know all of the possible permutations of HTML that your application can produce. The mapping between CSS selectors and HTML is also complex. These two factors combined make static code analysis totally impractical. 

*The cascading behavior of CSS is inconsistent:* Some properties apply to both the selected element and its children while others apply only to the element itself. Properties used for layout interact strangely with properties used for other purposes like typesetting. 

*The cascading behavior of CSS is fundamentally a bad idea:* The cascade is a sort of polymorphic inheritance and as a result suffers from the same issues as class inheritance in object-oriented programming languages.

*Overriding is too easy, effective reuse is too hard:* Indiscriminate reuse and overriding leads to deeply cross-dependent code that is very difficult to change or refactor.   

## What can we do about it?

In a future blog post, I will outline two approaches to solving the problem:

*100% inline styles:* A modern approach which does away with stylesheets and thus most of the bad parts of CSS.

*Single Responsibility Principle CSS:* An approach that for conventional CSS codebases which still use stylesheets.