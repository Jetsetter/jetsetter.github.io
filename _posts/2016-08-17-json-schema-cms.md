---
title: JSON-Schema Powered CMS
author: Mike Plis
layout: post
disqus_page_identifier: json-schema-powered-cms
published: true
---

---

A few months ago, we at [Jetsetter](https://www.jetsetter.com/) decided to revamp the tools that our editorial team used to create content for our online [Magazine](https://www.jetsetter.com/magazine). Instead of listing all of the problems with our old CMS, I’ll let it speak for itself:

![Old Jetsetter CMS](/public/img/cms1.png)

That’s pretty much all there was. Placing content other than plain text inside an article required the user to insert raw HTML, oftentimes with the help of an engineer. There was also very little support for images, which didn’t make much sense given our huge repository of awesome travel and hotel imagery.
Needless to say, this was not anyone’s idea of an ideal system. So we set out to change that.

---

We researched existing CMS solutions, but decided to build our own for two reasons.

* Easy integration with our existing platform — we wanted to allow users to make use of our large image repository and hotel data in creative ways.
* Customized workflow — building a CMS ourselves would give us full control to quickly build tools that effectively met our team’s specific needs.

The old CMS offered very few tools to augment articles beyond plain text, and the ones it did offer were too cumbersome to use. We wanted a system that was flexible and powerful, while at the same time simple and intuitive to use.

We established the concept of “widgets” — modular, structured blocks of content that could be added to an article and easily reorganized. A large selection of widgets (rich text, photos, videos, social embeds, etc.) would allow for a broad range of expression, while also abstracting as many of the details away from the user as possible.

We knew we wanted to store articles as JSON because it’s a flexible format and is easy to work with, so when we began discussing how to implement “structured blocks of content”, we felt that [JSON Schema](https://spacetelescope.github.io/understanding-json-schema/) was a natural fit in a system like this. Our friends at Oyster [had success integrating JSON Schema into their rebuilt CMS](http://tech.oyster.com/when-building-your-own-cms-is-the-right-choice/), so we felt confident that it was something we could make use of as well.


---

JSON Schema allowed us to easily define the structure of our widgets and check whether the data for a specific widget was valid. Here’s a trimmed-down version of the schema we wrote that defines a widget for a social embed:

```json
{
  "type": "object",
  "title": "Social",
  "required": [
    "socialUrl",
    "socialType"
  ],
  "properties": {
    "socialType": {
      "type": "string",
      "enum": [
        "Facebook",
        "Twitter",
        "Instagram",
        "Pinterest"
      ]
    },
    "socialUrl": {
      "type": "string",
      "title": "Social URL"
    }
  }
}
```

And the data for a Social widget would simply look like this:

```json
{
    "socialType": "Facebook",
    "socialUrl": "https://www.facebook.com/Jetsetter/posts/10157238317965361"
}
```

Not only did JSON Schema enable us to document the structure of our data, we could use that documentation to validate the data on both the client and the server.

However, we still had to create the new UI that would collect this data from the user. At Jetsetter, we use [React](https://facebook.github.io/react/) so it would have been easy to create some kind of SocialWidget component that was rendered whenever the user wanted to add a social embed to their article. The SocialWidget component could have extracted the different socialTypes and put them into a `select` element and rendered an `input` element for the socialUrl. 

However, this wasn’t going to scale that well at all. We would have had to update the React component whenever we wanted to make even a small change to the schema. Do you want bugs? Because that’s how you get bugs.

The big breakthrough for us was when we realized that we could use the schema to *generate the UI*. There were a few libraries out there that could do this, but we settled on the appropriately-named [react-jsonschema-form](https://github.com/mozilla-services/react-jsonschema-form) because it was very customizable and actively maintained.

All we needed to do was feed our widget schemas into react-jsonschema-form and…

![React form](/public/img/cms2.png)


This was HUGE. Now our entire CMS could be powered by our schemas. If we wanted to add a new socialType or add a field to the Social widget or even simply change the label for the input, all we needed to do was update the schema and everything just worked.

---

In addition to defining schemas for our widgets, we needed to define schemas for our different article types so that an entire article’s data could be validated and not just the individual widgets. An article consists of several singular inputs, like the article’s title, and a list of widgets. Here’s an abbreviated example of a schema for our Longform articles:

```json
{
  "type": "object",
  "required": [
    "title",
    "deck",
    "widgets"
  ],
  "properties": {
    "title": {
      "type": "string",
      "title": "Title"
    },
    "deck": {
      "type": "string",
      "title": "Deck"
    },
    "widgets": {
      "type": "array",
      "title": "Widgets",
      "items": {
        "oneOf": [
          {"$ref": "#/definitions/content"},
          {"$ref": "#/definitions/photo"},
          {"$ref": "#/definitions/pullQuote"},
          {"$ref": "#/definitions/video"},
          {"$ref": "#/definitions/social"}
        ]
      }
    }
  },
  "definitions": {...}
}
```

Notice that this schema makes use of the `oneOf` keyword, which tells us that each item in the `widgets` array must validate against exactly one of the schemas listed. This allows our article schemas to be very flexible in the size and structure of articles that validate against it, but it presented us with another problem: react-jsonschema-form didn’t know how to render this kind of schema.

To solve this problem, we kept track of a special schema — a “dynamic” schema — inside our application’s state that react-jsonschema-form *did* know how to render. Whenever a user wanted to add a new widget to their article, we added that widget’s schema to the dynamic schema and then fed that back into react-jsonschema-form.

Here’s what the dynamic schema would look like after the article was first created:

```json
{
  "type": "object",
  "required": [...],
  "properties": {
    "title": {...},
    "deck": {...},
    "widgets": {
      "type": "array",
      "title": "Widgets",
      "items": []
    }
  },
  "definitions": {...}
}
```

And here’s what the dynamic schema would look like after the user added a Content widget, and Social widget, and another Content widget:

```json
{
  "type": "object",
  "required": [...],
  "properties": {
    "title": {...},
    "deck": {...},
    "widgets": {
      "type": "array",
      "title": "Widgets",
      "items": [
          {"$ref": "#/definitions/content"},
          {"$ref": "#/definitions/social"},
          {"$ref": "#/definitions/content"}
      ]
    }
  },
  "definitions": {...}
}
```

We made use of react-jsonschema-form’s customizability in order to give the user the ability to add new widgets to an article. We overrode the default library behavior to add an “Add Widget” button beneath each of the widgets:

![Add Widget](/public/img/cms3.png)


That same kind of customizability also allowed us to do things like integrate a rich text editor component (we chose [react-rte](https://github.com/sstur/react-rte)) so that our writers never had to write HTML again and add the ability to reorder and remove widgets to enable users to quickly restructure their content.

The new CMS was originally built to power our Magazine, but it can drive other parts of Jetsetter.com too. All we need to do is define the document schema and any new widget schemas we need. Then we can generate the UI automatically, collect user input, and construct a JSON object that can be stored in our database and used however we want. We’ve already created a Homepage document with a Homepage Hero Item widget that controls the carousel of images on Jetsetter’s homepage.

The combination of React, JSON Schema, and react-jsonschema-form resulted in a relatively small codebase that powers an extremely flexible and robust system that is easy to use, maintain, and expand.

<video width="100%" autoplay loop>
  <source src="/public/video/cms-test1.m4v" type="video/mp4">
</video>