---
layout: post
author: Matt Dickinson
title:  "Cleaner Javascript Assertions"
date:   2016-01-19
tags:
  - code
  - tdd
  - javascript
---
I just published my first package to npm, [assertchain-jasmine](https://www.npmjs.com/package/assertchain-jasmine). It’s a library to assist with javascript assertions 
in the [jasmine](http://jasmine.github.io/) framework. It adds a more fluent-like syntax and allows for extensibility. 
I’m always a proponent of making unit tests easier to write and understand, so I appreciate being able to reduce repeated keystrokes and to group related information together.

I’ve already started using it for [Revelio](https://getrevelio.com) code. Here are a couple looks at how it can clean things up.

![Before]({{ site.contenturl }}assertchain-before.png)

![After]({{ site.contenturl }}assertchain-after.png)

When dealing with hierarchical data, the nesting of assertions really improves readability.

Also, I’ve found that I end up writing similar assertions over and over, so this library allows you to add extensions and call them inline with the other assertions.