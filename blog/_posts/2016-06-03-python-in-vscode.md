---
layout: post
author: Matt Dickinson
title:  "Using Python in Visual Studio Code"
date:   2016-06-03
tags:
  - code
  - python
---

I just started on a project written in Python, which I've never used until now. I decided to use Visual Studio Code, as that's been a great experience so far.

There's a pretty great extension in the marketplace, simply called [Python](https://marketplace.visualstudio.com/items?itemName=donjayamanne.python). 
It gives you intellisense, autocompletion, everything you'd expect from IDE integration. So far it's been pretty great...other than filling my
screen with all sorts of squiggles. Of course, that's not the fault of the extension, but of the linter.

## Configuring the Linter

The extension allows you to use a few different linters. In my case, I'm using the default, [pylint](https://www.pylint.org). It can be very helpful at
tracing issues, but I felt the signal-to-noise ratio was a bit off. With pylint, you can customize which errors/warnings/messages you want to see.
Each item has a code associated with it, like the useful message below (E1111).

![A useful message]({{ site.contenturl }}python-linter-message.png)

Pylint supports disabling certain messages in code, or via a configuration file. In my case, I wanted to disable the messages once, rather than in every single file.
I poked around in the user settings that the Python extension gives you, but there was nothing about defining the configuration file. There was, however,
a setting for the pylint path. I added the command-line parameter, held my breath, and ran pylint. Sure enough, it worked!

I put the configuration file in a central location (as opposed to local to the project) and then configured my user settings to use that file.

![Configuring the path]({{ site.contenturl}}python-linter-config-path.png)

Here's my configuration file (at the time of this blog post)

```
[MESSAGES CONTROL]
#C0111 Missing docstring
#C0103 Invalid constant name
#C0301 Line too long
#C0303 trailing whitespace
disable=C0111,C0103,C0303,C0301,
```

The `[MESSAGES CONTROL]` header is required, and the `disable=` line does the actual disabling of messages. I picked up the suggestion of adding comments
with the human readable names for each of the disabled codes, but it's not required.

## Running unit tests

I also wanted to be able to run unit tests inside Code. I'm not completely sold on my approach here, but it's worked so far. The existing project was
using `nose`, so I stuck with that.

I first added a test task (`.vscode/tasks.json`) to run tests in the active test file.

```json
{
    "version": "0.1.0",
    "command": "nosetests",
    "isShellCommand": true,
    "args": [],
    "showOutput": "always",
    "tasks": [
        {
            "taskName": "${file}",
            "isTestCommand": true,
            "args": []
        }
    ]
}
```

I feel like I have to be doing this wrong, as it's setting `command` to `nosetests` for all tasks. It seems like you'd want to define `command`
at the task level, but Code yells at me when I try that. In any case, `taskName` gets passed as a parameter to `nosetests`, so we use the 
token `${file}` to get the filename of the active (focused) file. I primarily did this because the project had integration tests
that I didn't want to have to run every time. [Here](https://code.visualstudio.com/Docs/editor/tasks#_variable-substitution) are some other options
if you don't want to limit yourself to the current file.

The next step was to bind a keyboard shortcut to run the tests. I used F6, as it wasn't bound to anything else.

```json
[
    {
        "key": "f6",
        "command": "workbench.action.tasks.test"
    }
]
```

Running the task will print to the Tasks output window.

![Good deal]({{ site.contenturl}}python-unit-test-output.png)