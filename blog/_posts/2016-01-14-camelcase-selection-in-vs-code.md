---
layout: post
author: Matt Dickinson
title:  "CamelCase Selection in VS Code"
date:   2016-01-14
tags:
  - tutorial
  - vscode
---

I recently stumbled upon a [nice little extension](https://marketplace.visualstudio.com/items/ow.vscode-subword-navigation) for Visual Studio Code that allows Resharper-like selection. 
You know, when you hit ctrl+shift+right in “SomeObjectName” and it only selects “Some”. The extension calls this “subword navigation”, which is probably a much better description than the mess I just wrote. 
This is the “Use CamelHumps” option in Resharper.

While the functionality is slick, the documentation is a little sparse, so I figured I’d give a little tutorial.

To install the application, open the command prompt (default is `ctrl+p`) and type `ext install subword` and it should come up with a search result. Hit enter, and it’ll prompt you to restart Code.

The extension is now installed, but dormant. You’ll have to manually tell Code that you want to use Subword Navigation. Go to `File > Preferences > Keyboard Shortcuts`, or type `> Keyboard` in the command prompt.

On the left you’ll see the list of default keyboard shortcuts. Note that this file is readonly – you’ll make your customizations in `keybindings.json`, which is opened on the right. 
These customizations work across workspaces, so you don’t have to worry about setting this up for every project.

As the Subword Navigation documentation hints at, you can look for the cursorLeft binding in the default shortcuts for inspiration…or you can just paste this inside the [] brackets in keybindings.json

```json
{
 "key": "ctrl+left",
 "command": "subwordNavigation.cursorSubwordLeft",
 "when": "editorTextFocus"
 },
 {
 "key": "ctrl+right",
 "command": "subwordNavigation.cursorSubwordRight",
 "when": "editorTextFocus"
 },
 {
 "key": "ctrl+shift+left",
 "command": "subwordNavigation.cursorSubwordLeftSelect",
 "when": "editorTextFocus"
 },
 {
 "key": "ctrl+shift+right",
 "command": "subwordNavigation.cursorSubwordRightSelect",
 "when": "editorTextFocus"
 }
```

When you’re done, it should look something like this:

```json
// Place your key bindings in this file to overwrite the defaults
[
    //Command prompt
    {
        "key": "ctrl+t",
        "command": "workbench.action.quickOpen"
    },
    {
        "key": "ctrl+p",
        "command": "workbench.action.showAllSymbols"
    },
    //Subword navigation
    {
        "key": "ctrl+left",
        "command": "subwordNavigation.cursorSubwordLeft",
        "when": "editorTextFocus"
    },
    {
        "key": "ctrl+right",
        "command": "subwordNavigation.cursorSubwordRight",
        "when": "editorTextFocus"
    },
    {
        "key": "ctrl+shift+left",
        "command": "subwordNavigation.cursorSubwordLeftSelect",
        "when": "editorTextFocus"
    },
    {
        "key": "ctrl+shift+right",
        "command": "subwordNavigation.cursorSubwordRightSelect",
        "when": "editorTextFocus"
    }
]
```

Note that I have extra key bindings. I swapped _ctrl+t_ and _ctrl+p_ to (somewhat) replicate the Resharper Go To Everything feature.

After you’ve saved that, you can navigate those subwords like a CamelHumps pro!