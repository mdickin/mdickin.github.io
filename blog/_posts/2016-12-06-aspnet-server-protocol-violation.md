---
layout: post
author: Matt Dickinson
title:  "Server Protocol Violation in ASP.NET"
date:   2016-11-16 17:45:00
tags:
  - code
  - asp.net
  - error
---

Recently, the team at my new job ran into a strange issue with their website and I was asked if I'd ever seen it before.
For some reason, a certain page was getting an error when making a request to the API (i.e. `ASP.NET MVC` to `ASP.NET Web API`).
The call would fail with the following exception: 

```
The server committed a protocol violation. Section=ResponseStatusLine
```

## Finding answers

Searching for this error returned plenty of results to stop getting this error. The gist of it is that for some reason,
.NET didn't like the response that came back from the request. The primary fix I found was to enable the 
`useUnsafeHeaderParsing` flag. This sounded more like "let's ignore the error" than a solution. Similarly,
disabling `KeepAlive` on the connection would fix it, but what kind of solution is that?

I decided to dig deeper to find the cause of the bug. It was strange, because the web client that was throwing the error
was used all over the place. This was the only path that caused the error. Knowing this, I looked closer to find
any deviations from the standard operations. Nothing appeared out of the ordinary in the code, neither on the MVC nor Web API side.

## Debugging

I started local instances of the sites and put breakpoints on:
 * the web client writing the request and reading the response, and
 * the API controller endpoint
 
The call in question was something like the **sixth** REST call made on this page, so I saw all the first five calls go out 
and return successfully. On the sixth call, the request was written, and then reading the response threw the exception in question.
The strange thing about it was that I never hit the breakpoint in the API endpoint. _A call that was never made was failing._

Positive I was missing something, I fired up [Fiddler](http://www.telerik.com/fiddler) to get a closer look at the network calls.
Of course, this came with its own set of hardships. After enabling the proxy, I had to change the configuration to point to
`http://ipv4.fiddler/myApi` rather than `http://localhost/myApi`. Fiddler will not capture loopback addresses, but it recognizes `ipv4.fiddler`
and reroutes it to `127.0.0.1`. Additionally, the web client this project was using had a one-liner tucked away that explicitly 
set the client proxy to `null`...I still haven't tracked down why that is.

Once I got Fiddler up and running, I ran through the scenario and...it passed. The website hit the endpoint successfully.
Inspecting the call revealed nothing to me. No strange or malformed headers, which some of the results had hinted at. Going back
and rereading some of the answers, others had mentioned that sometimes Fiddler will correct errors in requests. So Fiddler was out.

## The Power of Open Source

Desperate to find an answer, I decided to track down whatever source code was available. Fortunately, this exception seemed pretty
distict, so maybe if I could find where it was getting thrown I could get a better feel for things. I found the general area and 
the multiple places where it was thrown; the code was fairly complex, so I just tried to get a general feel for what was happening.
Something I found gave me the impression of previous calls impacting the current call. That would then explain why disabling `KeepAlive`
would fix the problem. I had never thought of paying attention to the previous calls; they were succeeding, why would I care? 

## MORE Debugging

I went back to my breakpoints in the web client and added a few watch statements around the request and response. I watched the URI,
method, response status code, and the `ContentLength` (which is a nice property that reads the `Content-Length` header). I went through
and didn't see anything out of the ordinary but still I got the same exception. I ran through it again (and possibly again and again)
and finally realized that the call just before the error call was returning a `204 - No Content` (which was expected) and a `ContentLength`
of -1. Well, that's a strange length for content! I inspected a little closer to check the actual `Content-Length` header and...well, it wasn't there.
It looked like the `ContentLength` property was just showing a -1 for whatever cute reason it had.

I hopped over to the API side to find the endpoint that was returning a 204 and yep, it's returning 
`Request.CreateResponse(HttpStatusCode.NoContent, dto)`.

_Wait._ What's that `dto` doing in there? Why would we be returning an object in a 204? I checked Fiddler and nope, nothing being serialized
in the response. Not trusting Fiddler at this point, I used [DHC](https://chrome.google.com/webstore/detail/dhc-rest-client/aejoelaoggembcahagimdiliamlcdmfm)
and my breakpoint to check the response content in the web client. Nope, nothing there either. It seemed like .NET was smart enough not to serialize `dto` 
for 204 responses. At this point I was pulling out my hair, until that elusive debugging muse whispered "check the headers again". So I went back and looked;
not much there, just the standard HTTP stuff and the cruft that non-hardened IIS servers add. Except wait, why is there a `Content-Type` header?
The response had `Content-Type: application/json` tucked away in there. Surely that couldn't cause this to blow up, it's not really an error 
just more of a _validation problem_ **OH SHIT**

I went back to the API and removed the `dto` from `CreateResponse()` (which I was planning on doing anyway, gotta leave code nicer than you found it)
and reran the scenario. It worked! Just as a sanity check, I looked at the headers in Fiddler and DHC and sure enough, the `Content-Type` header was
gone.

## Wrapping up

In the end, this was a tricky bug because of two factors. The first was that for the longest time, I was focused on the call that was blowing up, 
when in fact it was the call just before that was causing the error. Thus, I never looked at the API endpoint and noticed the content being returned
in a `No Content` response. The second factor was that the `Content-Type` header was _so_ easy to overlook. I was looking for invalid values, malformed
headers, something interesting. But this was a header that was returned for almost **every single call**.

In preparation for this blog post, I googled the error again to find the name of the flag that would ignore this error. The first result was one
I had already seen, but this time I was immediately drawn to [this answer](http://stackoverflow.com/a/23374724/706551). I remembered seeing the issue with
204s (and I think I heard echoes in it when I was scanning through the network traces), but I had ultimately dismissed it due to the first factor.
In an effort to be a better code steward, I left a comment on that answer in the hopes that the next person would not make the same mistake.