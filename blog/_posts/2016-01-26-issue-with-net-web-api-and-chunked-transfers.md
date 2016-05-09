---
layout: post
author: Matt Dickinson
title:  "Issue with .NET Web API and Chunked Transfers"
date:   2016-01-26
tags:
  - code
  - nodejs
  - .net
  - web api
  - revelio
  - bug
---

Last night I finally tracked down a rather elusive bug. I had previous published my [http-client-factory](https://www.npmjs.com/package/http-client-factory) NPM package 
and was using it for a Revelio adapter for [apiDoc](http://www.apidocjs.com/) documentation. The site was being published just fine, until I added more endpoints. 
Debugging the .NET Web API endpoint, the request would come in null to the controller, but only on specific requests. If I removed the portion of the request dealing with the 
documented endpoint’s HTTP responses, it would work fine.

I went in to the controller and added another parameter of type `HttpRequestMessage`. If you aren’t aware, .NET Web API will populate this parameter with the current request information
without affecting the deserialization of other parameters.

![Including HttpRequestMessage as an action parameter]({{ site.contenturl }}chunked-transfers-include-httprequestmessage-in-action.png)


My first thought was that the request was malformed somehow. While debugging, I could call `requestMessage.Content.ReadAsStringAsync().Result`, and it would spit out the request just fine. Even passing that to `JsonConvert.DeserializeObject<T>` returned the correct object. I was flummoxed.

Previously when a request would come in as null, it was due to an error in the JSON request. I thought maybe it was something borking the Web API deserializer that wasn’t a problem for the Newtonsoft library (though I was 98% sure it was the same deserializer). But I ran the raw request through multiple JSON validators and it was all fine, as I would expect as it was being generated in JavaScript using `JSON.stringify()`.

My next step was to take the raw request that was failing and send it using an HTTP request tool. What do you know, it worked just fine! This narrowed it down to something with the transfer itself, so I dug in a little deeper. Finally, I noticed that the request was being sent with `"Transfer-Encoding": "chunked"` and showed a length of zero. I went back and re-read the Node.js guide to the `http` module and noticed that the `Content-Length` header has to be sent manually, otherwise it will sent chunked. After setting this header, the Revelio adapter worked perfectly. I realized that it wasn’t anything having to do with the content of the request, other than the total length of it. Once the request reached a certain length, it would cease to be processed by the controller.

I updated the `http-client-factory` package with the fix, as I don’t have the need for chunked transfers, but it bugged me that Web API was inconsistent about handling it. I did a brief search and noticed a lot of other people having issues with chunked transfers, though I wasn’t able to find any resolution. At this point, I was happy to have it working again and moved on.