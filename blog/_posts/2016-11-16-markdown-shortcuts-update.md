---
layout: post
author: Matt Dickinson
title:  "Markdown Shortcuts Extension for VSCode"
date:   2016-11-16 17:45:00
tags:
  - code
  - markdown
  - extension
---

I recently released a new version for my [Visual Studio Code](https://code.visualstudio.com) extension, 
[Markdown Shortcuts](https://marketplace.visualstudio.com/items?itemName=mdickin.markdown-shortcuts). 
It's a neat little tool that lets you use keyboard shortcuts to generate Markdown. This blog is hosted
on [GitHub](https://www.github.com), and the posts are written in Markdown. I found myself wanting a `ctrl+B` shortcut 
as I edited my posts, making things bold, italic, etc. I would frequently have to look up the correct syntax for things
like links (do I put the URL first or last?)

### New Menus

In this most recent release, I added support for Code's new extension authoring APIs. This includes adding right-click 
context menu options and title bar buttons. These APIs still seem to be in their infancy, as I found myself wanting finer control
over things like display order, display text, and menu hierarchies. Currently, everything is sorted by name. This means that
my title bar buttons would display the order "Toggle bold", "Toggle bullet points", "Toggle italic", "Toggle number list". I ended up
removing the list buttons, as it looked bizarre having Bold and Italic separated. I've submitted an 
[enhancement request](https://github.com/Microsoft/vscode/issues/15596), so I'll wait to see what the team decides with this.
I know other things come in to play, in terms of ordering icons across multiple extensions.

**Update:** it turns out this was already a feature but had been left out of documentation.

Fortunately, the context menu allows you to group items, but only in a flat list. I ended up excluding some of the shortcuts
from the context menu, as it would have completely blown up the list. If I could have had a top-level "Heading" item, with a sub-menu
of "H1", "H2", etc, that would have been ideal. There is already an [enhancement request](https://github.com/Microsoft/vscode/issues/9827) 
to add such functionality.

### New Commands

In addition to menu integration, I added commands to add tables and
[GitHub Flavored Markdown (GFM) checkboxes](https://github.com/blog/1375-task-lists-in-gfm-issues-pulls-comments). The checkboxes
were pretty straightforward, just a slight modification of the bulleted list. My only concern was that it is not standard Markdown.
Therefore, it doesn't display as a checkbox in Code's Markdown preview window.

Adding tables was a different story. I was less certain of how users would want to interact with the command. I ended up working on
two (three) use cases:

1. User wants to create a new table and just needs some scaffolding (I forgot the syntax again!)
2. User wants to convert some tabular data into a table and...
    1. define the column headers, or
    2. interpret the first row as the header. 

The first use case is easy enough: if nothing is selected, just paste in an example table.
The second use case was not as straightforward. I considered having a prompt to ask if it should add a header row,
but instead I chose to make it two separate commands, "Add table" (use case 2.1) and "Add table with header" (use case 2.2).
I'm still not super happy with the wording, as I feel like it's ambiguous as to which is which. I'm open to suggestions for
how to make this more clear! 

There are other table-related use cases out there that I wasn't prepared to tackle yet, like "Add column to the right",
"Delete column", and all the other functionality something like Microsoft Word gives you. I wanted to get this basic functionality
out first and see what the users wanted.

### Going Forward

I have a few items in the [backlog](https://github.com/mdickin/vscode-markdown-shortcuts/issues), mostly around
improving the behavior of text selection. Please feel free to add your suggestions! I would also appreciate
your reviews on the [Visual Studio Code Marketplace](https://marketplace.visualstudio.com/items?itemName=mdickin.markdown-shortcuts).