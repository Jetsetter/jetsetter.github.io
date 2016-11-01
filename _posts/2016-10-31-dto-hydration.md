---
title: "Config-based document hydration"
author: Chris Graves
layout: post
disqus_page_identifier: dto-hydration
date: 2016-10-31
---
# Config-based document hydration

Here at Jetsetter we produce a lot of content.  MikeP [described](http://tech.jetsetter.com/2016/08/19/json-schema-cms/) how we recently revamped our CMS admin tool to enable our editors to produce great looking content without having to type a single HTML tag.

If we have an article talking about why this hotel is just the absolute to-die-for destination this season, and we have this hotel listed on Jetsetter, then wouldn't it be nice to show off the Jetsetter price when linking to the hotel page?

Options:

1. Editors wake up extra early every morning and look up the Jetsetter prices of these hotels and update the articles with those prices
2. When rendering the article, client calls Jetsetter pricing APIs for each hotel to get latest prices
3. The CMS backend calls the APIs and [includes](https://en.wikipedia.org/wiki/Data_transfer_object) pricing details in the article response

Option 1 was not ideal because we like our editors.  Option 2 is workable but adds complexity in each client and can delay the finished rendering of the article.

Option 3 looks good but what backend engineer would agree to the ongoing support of whatever rich content the editors and front end guys decide that they want?  Today they want Jetsetter pricing, tomorrow it's image metadata, the next day it's instagram embeds ....

To flip the last bit on this Win-Win-Lose scenario we need to generalize the approach so that new data sources can be configured without any code change.

The backend doesn't know what resources are referenced in these documents, and doesn't want to.  Nor does the backend know how to fetch said resources.  Let the json-schema producer dev and the desktop client dev work it out between themselves.

Take this simple document as an example:

    {
      "propertyId": 65
    }

We will want to populate this object with the full details of the referenced property if possible.  Even though this is a Jetsetter property, and this is all Jetsetter code, we (the backend) want nothing to do with how to fetch this data.  To accomplish this, the client facing side CMS backend is configuable (via config files) on how to match and fetch any resources referenced in the document.  For example:

    "property": {
        "tokens": {
            "property-id": ["propertyId", "propertyIds"]
        },
        "method": "GET",
        "header": {
            ...
        },
        "ssl": true,
        "host": "api.jetsetter.com",
        "port": 443,
        "path": "/v4/PropertyService/properties/${property-id}",
        "data": {},
        "max-connections": 10,
        "max-retries": 0
    },

Hydration happens in a few steps-

* ### Extract matching metadata
	config describes "tokens" which match on field names in the document and eventually are used when fetching the given resource.  In this case, "property-id": ["propertyId", "propertyIds"] means that any fields called "propertyId" or "propertyIds" will lead to this resource being fetched.
* ### Fetch resources
	replace token in curl with extracted references
	"path": "/v4/PropertyService/properties/${property-id}", -> /v4/PropertyService/properties/65
* ### Infuse API response into document (hydration)

Our original document:

    {
      "propertyId": 65
    }

After matching and fetching and infusing becomes:

    {
      "propertyId": 65,
      "property": [
        {
          "status": 0,
          "data": {
            "name": "Solage Calistoga",
            "url": "/hotels/calistoga/california/65/solage-calistoga",
            ...
    }

What about a more complex example, where one widget might need to be hydrated against different APIs depending on the content?  What if the 'token' is just one piece of the loaded value?  Note the different extractor fields below which will match and capture the relevant piece of the data:

    "twitter-oembed": {
        "tokens": {
            "social-id": ["socialId"]
        },
        "extractors": {
            "social-id": "^(.*twitter.com.*)$"
        },
        "method": "GET",
        "header": {},
        "ssl": true,
        "host": "publish.twitter.com",
        "port": 443,
        "path": "/oembed?url=${social-id}",
        "data": {},
        "max-connections": 10,
        "max-retries": 0
    },
    "instagram-oembed": {
        "tokens": {
            "social-id": ["socialId"]
        },
        "extractors": {
            "social-id": "^(.*instagram.com.*)$"
        },
        "method": "GET",
        "header": {},
        "ssl": true,
        "host": "api.instagram.com",
        "port": 443,
        "path": "/oembed/?url=${social-id}",
        "data": {},
        "max-connections": 10,
        "max-retries": 0
    }

In real life we have many additional configured endpoints and some of the documents reference a great number of external resources.  We would not want the performance of the website to be dependent on all of these external data sources so one final additional step is to load the hydrated documents into an elastic search index.  This gives us horizontal scalability and decouples the serving of the documents with the production/hydration, as well as allows for elastic search to do some of the heavy lifting of querying documents based on different custom fields.
