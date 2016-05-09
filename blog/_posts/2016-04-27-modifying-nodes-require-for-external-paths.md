---
layout: post
author: Matt Dickinson
title:  "Loading Node modules from an external path"
date:   2016-04-30
tags:
  - nodejs
  - pumlhorse
---
This is kind of a niche use case, but with [Pumlhorse](http://pumlhorse.com) I wanted to be able to have scripts use node modules. 
For those not familiar, Pumlhorse reads YAML formatted `.puml` files and runs them through the engine.

The problem with this is that the require method acts under the current file’s path, thus it would always be relative to the Pumlhorse installation rather than the `.puml` file.

If I forced users to specify a relative or absolute path, this would be a simple operation, but Pumlhorse is designed to be readable, 
and adding such a restriction would detract from that experience.

So basically, I wanted to replicate the behavior of the `require` method, but from a path external to the JavaScript file. 
For awhile, I did just that, but it kept nagging at me. Finally, I went back and dug into the Node code.

I didn’t have much hope that there would be anything exposed for me to work with, figuring i would end up pasting the whole `module.js` and modifying it from there. 
I was pleasantly surprised, however, when I debugged through the code and discovered that each instance of Module has a `filename` variable. 
“Oh good”, I thought. “I can just set to the `.puml` filename and it will run its resolution steps from there!”

No such luck, however. I traced through again and discovered that even though I was changing the filename, the resolution paths 
(“node_modules”, “../node_modules”, “../../node_modules”, ad rootdireum) had already been generated. Interestingly, the paths were exposed alongside the filename. 
Thinking that I really didn’t want to write code to overwrite those paths manually, I crossed my fingers that the function generating those paths was likewise exposed.

![_nodeModulePaths function]({{ site.contenturl }}node-module-paths.png)

Well, what do you know? It is! I don’t even have to muck about with the file name, I can just store off the pre-generated paths (to assign back later, just in case…) 
and generate the new paths.

I did consider making this into an NPM package, but I figure in this post-leftpad world if your `package.json` is longer than your code file, it’s best left as a snippet.

```javascript
module.exports = requireFromPath

function requireFromPath(moduleName, directory) {
    var oldPaths = module.paths
    
    if (directory) module.paths = module.constructor._nodeModulePaths(directory)
    
    try {
        return require(moduleName)
    }
    finally {
        module.paths = oldPaths
    }
}
```

So now, regardless of where Pumlhorse is installed, `requireFromPath` will start looking for `moduleName` in `directory`.