---
author: Matt Dickinson
title:  "Writing a Weasley Clock with Alexa"
date:   2017-04-30 17:45:00
published: false
tags:
  - code
  - nodejs
  - alexa
  - pumlhorse
---

# Writing a Weasley Clock with Alexa

In the Harry Potter universe, the Weasley family owns a clock that, rather than telling the time, tracks where friends and family members are. That is all I will say about the Weasley Clock because I'm sure the Potterheads can find 17 things wrong with that sentence. For our muggle purposes, we will use [Alexa](https://developer.amazon.com/alexa) and a handful of other technologies to do the same.

At a high level, we will need to accomplish the following:

* Teach Alexa how to listen for our commands ("Where is Harry?", "Find Peter Pettigrew", ~~"What is a horcrux?"~~)
* Write an API that will handle Alexa's requests
* Write an API that will track family member locations
* Somehow send requests to the API with location updates

Let's get to it, shall we?

## Teaching Alexa

For this project, we will be creating a "skill" for Alexa. This is essentially a plugin where we can define commands. The first thing you will need is an [Amazon Developer](https://developer.amazon.com) account (don't fret, it's free). Once you've created that, you can log in to the [Alexa Developer Console](https://developer.amazon.com/edw/home.html#/) and click "Get Started" for the Alexa Skills Kit.

In the upper right corner, click the "Add a New Skill" button. There are three options, but we'll stick with "Custom Interaction Model". Pick a Name and Invocation Name, and click Save, then click Next.

![Creating a skill]({{ site.contenturl }}alexa-create-skill.png)

Recently, Amazon added a "Skill Builder" UI interface for designing skills. Feel free to play around with it, but we're just going to paste some JSON

```json
{
  "intents": [
    {
      "name": "AMAZON.CancelIntent",
      "samples": []
    },
    {
      "name": "AMAZON.HelpIntent",
      "samples": []
    },
    {
      "name": "AMAZON.StopIntent",
      "samples": []
    },
    {
      "name": "FindPerson",
      "samples": [
        "Where is {person}",
        "Find {person}"
      ],
      "slots": [
        {
          "name": "person",
          "type": "AMAZON.US_FIRST_NAME",
          "samples": []
        }
      ]
    }
  ],
  "prompts": [
    {
      "id": "Elicit.Intent-FindPerson.IntentSlot-person",
      "promptVersion": "1.0",
      "definitionVersion": "1.0",
      "variations": [
        {
          "type": "PlainText",
          "value": "Who would you like to find?"
        }
      ]
    }
  ],
  "dialog": {
    "version": "1.0",
    "intents": [
      {
        "name": "FindPerson",
        "confirmationRequired": false,
        "prompts": {},
        "slots": [
          {
            "name": "person",
            "type": "AMAZON.US_FIRST_NAME",
            "elicitationRequired": true,
            "confirmationRequired": false,
            "prompts": {
              "elicit": "Elicit.Intent-FindPerson.IntentSlot-person"
            }
          }
        ]
      }
    ]
  }
}
```

Essentially, the above defines a command `FindPerson` that accepts a name and passes that in the `person` parameter. Next, we'll go to the Configuration page.

Alexa takes our `FindPerson` command and passes it to a web API. You can host it with Amazon, or you can use any other site over HTTPS. For this article we're use the fantastic [Glitch](https://www.glitch.com) by Fog Creek, so we can add the URL there and click Next.

![Configure skill]({{ site.contenturl }}alexa-configure-skill.png)

If you're following along and using Glitch, you can choose "My development endpoint is a sub-domain of ..." and click Next.

The next page will let us test the API... that...doesn't exist yet. We should probably do something about that.

## Writing the API

If you're unaware, [Glitch](https://www.glitch.com) is Fog Creek's version of [CodePen](https://codepen.io/), [JSFiddle](https://jsfiddle.net/), et al but for backend Node projects. It allows you to quickly set up and run an [Express](https://expressjs.com/) web server. Other developers can view your site and code and "remix" (fork) it themselves.

We will also be using [Pumlhorse](http://www.pumlhorse.com), my scripting language I introduced in [that one blog post I never got around to writing](https://media.giphy.com/media/87xihBthJ1DkA/giphy.gif). You don't have to use it, but you do have to suffer through it here. Long story short, Pumlhorse is an engine written for Node that runs scripts written in YAML. I initially wrote it for API integration testing, but recently created [pumlhorse-express](https://www.npmjs.org/package/pumlhorse-express) to use it as a controller engine for Express apps.

Back to Glitch, once you've logged in you can create a new project. It will be assigned a kooky "{adjective}-{noun}" name, but you can change this in the upper left menu.

The next thing we'll want to do is install our dependencies. Open the `package.json` file. This should be familiar to anyone that has used Node before. In addition to the text editing, Glitch provides the "Add package" dialog to search [npm](https://www.npm.org) for packages. Use this to add the following packages:

* `body-parser` - Allows us to parse JSON requests
* `pumlhorse-express` - Pumlhorse extension for Express

With those items installed, we can go to `server.js`. This file is responsible for starting up the Express server. Glitch automatically adds some endpoints, but we can remove those. Or you can keep them. Whatever, you do you. You will, however, need to import the packages we just added. At the top, just below where `app` is initialized, we'll tell Express to use our extensions. After you're done, it should look something like this:

```javascript
var express = require('express');
var app = express();
var bodyParser = require('body-parser');
var pumlhorseExpress = require('pumlhorse-express');

app.use(express.static('public'));
app.use(bodyParser.json());

app.use(pumlhorseExpress.Router({
  routes: {
    "/api/findPerson": {
      post: "site/api/findPerson.puml"
    }
  }
}))

var listener = app.listen(process.env.PORT, function () {
  console.log('Your app is listening on port ' + listener.address().port);
});
```

You'll note that `pumlhorseExpress.Router()` takes a configuration object. You may prefer to move this to a `.json` or `.yaml` file for larger projects. Essentially, this configuration tells Express to use the `site/api/findPerson.puml` file (which we will create next) to run any POST requests to the URL `/api/findPerson`. Simple, right?

Next, we need to create that file. Click "New File" and type `site/api/findPerson.puml` (Glitch will parse the folder structure). This file should look like the following:

```yaml
name: Find a person
expect:
  - body
steps:
  - person = $body.request.intent.slots.person.value
  - log: Received request for $person
  - ok:
      version: 1.0
      response:
        outputSpeech:
          type: PlainText
          text: Sorry, I don't know where $person is
```

Line by line: first, we give the script a name. Then we tell Pumlhorse that we expect to be passed a variable named `body`. This step isn't strictly necessary, but it's a good practice for Pumlhorse in general.

The `steps` section contains three steps. The first gets the person's name from the request from Alexa and assigns it to the variable `person`. The next step logs this (only visible in the Glitch console, helpful for making sure we're getting good data). The last step tells Express to return a `200 OK` value with the given JSON response. This response looks like this in JSON (assuming we were passed "Harry"):

```json
{
  "version": "1",
  "response": {
    "outputSpeech": {
      "type": "PlainText",
      "text": "Sorry, I don't know where Harry is"
    }
  }
}
```

## Testing the API

What, you can't just trust me that it works? Psh, fine. Go back to your Amazon Developer Console which should still be in the Test tab. Go down to the Service Simulator, type "Find Voldemort", and submit it. The request and response boxes should be filled with the respective JSON. You can also click "Listen" under the response to hear Alexa say the forbidden name.

That's it, we're done! Weasley clock is finished! We don't know where anyone is, but hey we learned a lot in the process.

## We're not done

Ugh, fine. I suppose we should put some functionality behind this.

So, to make this happen, we need to somehow track people's locations. I'm sure the NSA has an array of options, but we'll just have those people's phones make POST requests to an API. Hey, an API! Back to Glitch!

We need an endpoint to accept location updates, which we'll call checkins. No one tell FourSquare. Wait, is FourSquare still a thing? Nevermind, put that on the back burner, let's add an endpoint to the configuration in `server.js`

```javascript
app.use(pumlhorseExpress.Router({
  routes: {
    "/api/findPerson": {
      post: "site/api/findPerson.puml"
    },
    "/api/checkins": {
      post: "site/api/checkins/create.puml"
    }
  }
}))
```

Oh, you already did it? Was it when I was talking about FourSquare? Did you ever use FourSquare? I never got into it, didn't really see the appeal in --

```yaml
name: Create checkin
expects:
  - body
steps:
  - log: Stop it with FourSquare, just tell me what to do with $body.user and $body.location
```

Fine. Yes, we're going to accept a JSON payload of `{"user": "user's name", "location": "latitude/longitude coordinates"}`. The `location` property will look something like "42.43472,-83.9871887", but we'll convert that to something nicer later. First we need a place to store this data. The default Glitch site stores its dreams in an array, but as this is in memory it gets trashed every time the site reloads (i.e. every time you edit any code). We're going to use something persistent, namely [nedb](https://www.npmjs.com/package/nedb). The API is very similar to MongoDB. Actually, we're technically going to use [nedb-promise](https://www.npmjs.com/package/nedb-promise) which wraps the `nedb` API with Promises. Pumlhorse works best with functions that return Promises, so it'll make our lives easier.

Speaking of making our lives easier, we're going to write and use a Pumlhorse module for calling `nedb`. Create a new file `puml_modules/storage.js` and paste the following:

```javascript
const Datastore = require('nedb-promise');

pumlhorse.module('storage')
    .function('create', ['in', 'data', create])
    .function('get', ['in', 'where', 'sort', list]);

function create(inName, data) {
    return getDatabase(inName).insert(data);
}

function get(inName, where, sort) {
    var cursor = getDatabase(inName).cfindOne(where);
  
    if (sort != null) {
      cursor = cursor.sort(sort);
    }
    
    return cursor.exec();
}

const databases = {};
function getDatabase(name) {
  if (databases[name] == null) {
    databases[name] = new Datastore({ filename: '.data/' + name, autoload: true});
  }
  
  return databases[name];
}
```

This code is a thin wrapper for the `nedb-promise` calls. Back to the `create.puml` script. We need to tell Pumlhorse to use the module we just wrote.

```yaml
name: Create checkin
modules:
  # Import the storage file
  - db = storage
expects:
  - body
steps:
  - log: Stop it with FourSquare, just tell me what to do with $body.user and $body.location
```

There's a little magic behind the scenes here. Since the file is in the `puml_modules` directory, we don't have to provide a path to it. Also, since the file and the module are named the same (`pumlhorse.module("storage")`), we can skip a step. This module is loaded under the `db` namespace to make function calls a little clearer.

Ok cool. Now we want to a) sanitize our input, and b) add the data.

```yaml
name: Create checkin
modules:
  - db = storage
expects:
  - body
steps:
  - user = $body.user
  - location = $body.location
  - log: Adding checkin for $user and $location
  # Sanitize input
  - if:
      value: ${ user == null || location == null }
      is true:
        - badRequest: Fields "user" and "location" are required
  # Create the record in the DB
  - record = db.create:
      in: checkin
      data:
        user: $user.toLowerCase() # Alexa sends all lowercase, easiest to normalize now
        location: $location
        dateTime: $date().getTime() # Store the milliseconds since the epoch
  # Return a 204 CREATED response with the record
  - created: $record
```

You can see that in addition to variables, we can do simple expressions like `${ user == null || location == null }` and call functions like `$date()`.

[Cool cool cool](https://media.giphy.com/media/uCZPNoeoaLjt6/giphy.gif). Now we need a way to send our location data. I have an Android phone with [Tasker](https://play.google.com/store/apps/details?id=net.dinglisch.android.taskerm&hl=en) installed. I normally just have it set up to put my phone on vibrate at work and unsilence it at home, but it's capable of a lot more. Suffice it to say, it's capable of making HTTP requests, so I've set it up to make POST requests to the weasley-clock API every ten minutes. Running this often is pretty unnecessary, and Tasker gives a lot more options (run when you connect to a certain WiFi, go to a certain location, etc), but it'll work for now. Here's what the action looks like in Tasker (retrieving the current location with `%LOC`):

![Tasker configuration]({{ site.contenturl }}alexa-tasker-config.png)

## Returning Location Data to Alexa

Now that we have location data, we can actually answer Alexa's questions. Unfortunately, most people won't be interested to hear latitude/longitude information, so we need to convert it to something useful. Fortunately, [Google's Maps API](https://developers.google.com/maps/documentation/geocoding/start) provides a [Reverse Geocoding](https://developers.google.com/maps/documentation/geocoding/intro#ReverseGeocoding) endpoint. This will translate our position to a multitude of addresses, from street number to country. We'll try to find something somewhere in between.

To get this information from Google's API, we'll use a partial script that can be called from other scripts. We'll put this in `/site/helpers/getReverseGeocode.puml`

```yaml
name: Get reverse geocode
expects:
  - latlong
steps:
  - response = http.get:
      url: https://maps.googleapis.com/maps/api/geocode/json
      data:
        latlng: $latlong
        key: $process.env.GOOGLE_MAPS_API_KEY
  - http.isOk: $response

  # Grab the best address info
  - address = getAddress: $response.json.results
  - addressName = $address.address_components[0].long_name # This variable will be used by the caller
functions:
  getAddress:
    - array
    # Get the first address with type "neighborhood" or "locality"
    - return array.filter(addr => addr.types.indexOf("neighborhood") > -1 || addr.types.indexOf("locality") > -1)[0]
```

You'll note that the GET request has a `key` parameter. This is an API key required by Google to throttle and monitor API requests. You can obtain one for free, but you'll want to keep it secure. Glitch provides the `.env` file to provide secure configuration. As it is secure, you can't see it in a project unless you have permission to edit it.

Now we can go back to `findPerson.puml` and call our partial script:

```yaml
name: Find a person
modules:
  - db = storage
expect:
  - body
steps:
  - person = $body.request.intent.slots.person.value
  - log: Received request for $person
  - latest = db.get:
      in: checkin
      where:
        user: $person.toLowerCase()
      sort:
        dateTime: -1 # Descending order
  - if:
      value: ${latest == null}
      is true:
        - ok:
            version: 1.0
            response:
              outputSpeech:
                type: PlainText
                text: I don't know where $person is
  - locationResult = run:
      script: ../helpers/getReverseGeocode.puml
      with:
        latlong: $latest.location
  - ok:
      version: 1.0
      response:
        outputSpeech:
          type: PlainText
          text: $person was last at $locationResult.addressName
```

Now we can test through the Service Simulator and even our Alexa!

## One last thing

As it stands, anyone can call our API. We need to [verify that the call is coming from Amazon](https://developer.amazon.com/public/solutions/alexa/alexa-skills-kit/docs/handling-requests-sent-by-alexa#verifying-that-the-request-is-intended-for-your-service). Amazon will pass our `ApplicationId` in the request, so we need to compare it.

```yaml
name: Find a person
modules:
  - db = storage
  - config
expect:
  - body
steps:
  - applicationId = $body.session.application.applicationId
  - myAppId = getSecureString: AMAZON_APPLICATION_ID
  - if:
      value: ${applicationId != myAppId}
      is true:
        - forbidden: You are not authorized to access this resource
  # Do everything else
```

`getSecureString` is just a helper to get values from `process.env`. Check out the [source code](https://glitch.com/edit/#!/weasley-clock) for more detail.

## Taking it further

This project has a lot of possibilities for extension. For starters, we should include some API key in the requests so we don't have unauthorized people adding location data.

Some other ideas:

* **Configure location names** - Associate a given radius as a known location, so Alexa will say "work", rather than "Brooklyn"
* **Allow alternate names** - It's one thing for my wife to say "Where is Matt", but my kids would want to say "Where is daddy?"
* **User interface** - The previous two enhancements would be best served by a UI. Did I mention Pumlhorse can server HTML too?

A final note: please let me know if you try out Pumlhorse, I'm very interested in feedback! You can contact me on Twitter ([@mdickin_dev](https://www.twitter.com/mdickin_dev) or [@pumlhorse](https://www.twitter.com/pumlhorse)), or use [Github](https://www.github.com/pumlhorse/pumlhorse).