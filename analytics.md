# Analytics Introduction
_Information here accurate as of 21/04/2017_

This is the proposed definition of Photobox's implementation of a Javascript library for tracking analytics data within its applications. How this generic Javascript library should be used specifically within a React application is defined in the separate [analytics-react.md](analytics-react.md) document.

This library is agnostic to any 3rd party analytics services and therefore has no external dependencies. It is flexible enough to allow us to send data to multiple sources.

Its API has been defined around what our requirements are right now, but also keeping in mind what we will likely need in future. Therefore, this is a proposal for what the library will potentially look like in its entirety and is not a proposal for what we should build right now.

The last section in this document covers how we might break this down into multiple iterations of work, with the first version being the minimal viable product that covers our current requirements.

This has been written to be as clear as possible to any team member, not just the technical team. It is to be used to agree upon the approach and then to help determine the level of work involved for production planning.

_NOTE: This covers the technical requirements and additional coding work for the web application and not the additional requirements for the data that the app receives e.g. extra product data needed for analytics. The extra work the analytics team require around data is a discussion with the data team._

##### A distinction between an 'event' and a 'trigger'

Two words you will see a lot throughout this documentationn. An 'event' is an action that occurs in our application, whether that be from user input such a click or typing into a text field, or non-user interaction driven by our code, such as the event of retrieving data. 'product data loaded' would be an event.

A 'trigger' is something that happens in response to an event; in this context, the sending of analytics data. I.e. "The sending of product specific analytics data was triggered off the back of a product CTA click event". Multiple types of events can trigger the same thing.

### What We Have Already Done vs. The Work We Need To Do
For clarity, the initial analytics work we did was around how we capture any event that can happen in our app to then trigger the sending of analytics data. Whether that be on page load, button click or even on a non user interaction such as a forced page redirect or a pop-up that we automatically display after a certain period of time.

Due to the nature of our app, we cannot rely on 3rd party analytics services to do any of the triggering work for us as with a traditional website e.g. Google Analytics automatically detecting and sending data on page load. This is something we need to take complete control of within our code.

We required a flexible, robust, centralised and simple way for Developers to add and maintain triggering the sending of analytics data off the back of any event, as well as a mechanism to pull out any of the app's data at any given time. It also needed to be done in a way that fits in with technology we are using. Even though this work was required, it has given us a great deal of power over what data we send and when, with little development time.

Now we have the "How do we trigger analytics?" we need to answer the "Now what data do we send?" and in a way that is consistent, robust and easy to maintain. This is what this work will cover.

_NOTE: As a starting point and proof of concept, we are currently sending a small amount of basic analytics data, but directly to GTM, using GTM's own data layer, in a GTM's specific format and in the simplest possible way. We now need to expand upon that._

# Photbox Analytics Data Layer
The library we will build will return a plain Javascript `Array` that has been extended with the functionality required to process and send analytics data to various sources. We will extend it with the following:

### Data Schemas

##### A `dataLayer.addSchema()` method
This will provide a means for us to define any number of data schemas. A schema represents the structure and expected values of the list of analytics data to send to our external source after a given event e.g. a 'Main Menu' schema may have values specific only to menus, so we define a 'menuItemId' and 'menuItemName' which we send on the event of a menu item click. We may also have a 'Product' schema which has product specific data such as 'productId' and 'productCategory' which we send on the click of a product CTA.

##### Schema Validation
In a schema, we define the names of the values of what we need to send, such as a 'productCategory'. We can also specify the expected format of that value, such as 'text', 'number' or perhaps one out of a fixed list of values such as 'book', 'mug' or 'canvas'. We can also specify whether the value is required or optional or if it has a default value in the absence of another. If validation fails, we will be alerted to fix the problem. This will help ensure that the data we send is always there, in the format that we expect and if it isn't we will know about it.

##### Schema Nesting
The ability to 'nest' schemas. Using the 'Main Menu' and 'Product' schema examples above, both will always include page level data such as 'pageHost' and 'pageTitle'. Rather than repeating those same values across both schemas, we can create a parent schema called 'Page' that specifies those values and then add the 'Main Menu' and 'Product' schemas as children of 'Page'. The child schemas then inherit those value definitions when used so we don't have to add and maintain the same list across multiple schemas (see the example below).

##### Schema Example
```js
import dataLayer from 'photobox-data-layer';
import format from 'joi'; // this does most our work for us. See: https://www.npmjs.com/package/joi

dataLayer.addSchema({
  name: 'Page',
  properties: {
    pageHost: format.string().hostname().required(), // required, must be text, in a correct hostname format
    pageTitle: format.string().required(),
    appVersion: format.string().regex(/^[0-9]+\.[0-9]$/).default('1.0').required() // ensures the format 'N.N' and this value is hard coded here using default(x)  by a developer
  }
  childSchemas: [{
    name: 'Main Menu',
    properties: {
      menuItemId: format.integer().required(),
      menuItemName: format.string().required()
    }
  }, {
    name: 'Product',
    properties: {
      productId: format.integer().required(),
      productCategory: format.string().allow('book', 'mug', 'canvas').required()
    }
  }]
});

// NOTE on formats:
// using `format.optional()` is the difference between the field being included
// but having an empty value and the field not being including in the data at all.

// We can even reference values from other fields to validate against. e.g.
// 'loggedInUser' can contain values only if the value of 'loggedIn' is 'yes'.
// See: https://github.com/hapijs/joi/blob/v10.4.1/API.md#refkey-options.
```

#### An (overridden) `dataLayer.push()` method
With our schemas defined, we can then start pushing analytics data to the Data Layer. We will override the default behaviour of the plain Javascript `Array.push()` to push the the analytics data along with the event and the schema name to validate against: `dataLayer.push(eventName, schemaName, data)`

##### `dataLayer.push()` Example
```js
import dataLayer from 'photobox-data-layer';

dataLayer.push('ProductCTA', 'Product', {
  pageHost: '/range/canvas',
  pageTitle: 'Photobox - Canvas',
  productId: 123,
  productCategory: 'book'
});
```

#### Plugins for External Services
This will allow us to push analytics data to any number of external analytics services. The Data Layer will provide an adapter interface which accepts adapters specific to a service e.g. a Google Tag Manager adapter. The job of the adapter is to take the data we push to our own Data Layer and subsequently push it to the 3rd party service, taking the data we have passed, translating it into a format that the external service understands and then sending it to where it needs to go.

##### GTM Adapter Example
```js
import dataLayer from 'photobox-data-layer';

const gtmAdapter = (window) => (dataLayer) => ({
  // this adapter's push is triggered when we do dataLayer.push(eventName, schemaName, data)
  // but only if the data has passed the schema validation
  push: (eventName, data) => {
    // this is GTM's own data layer
    window.dataLayer.push({
      event: eventName,
      ...data
    });
  }
});

// this adapter needs to be added BEFORE we start pushing data
dataLayer.addAdapter(gtmAdapter(window));
```

#### Debugging & Testing
Because our Data Layer is a standard Javascript `Array` (albeit, extended), we can easily test what is being pushed into it by our app either through the standard tools Developers use such as Chrome Dev Tools or we could eventually take that further and build a more in-depth and user friendly debugging tool to be used by non-Developers for QA (much like GTM's own). This can also be tested by our automated testing pipeline.

# Iterations of Work
This is to be discussed, but as a starting point:

#### Version 1
All features above except:
- ~~Multiple schema support. Just one big schema for now with a lot of fields marked as 'not required' so we can freely pick and choose values to fill in where it makes sense.~~ Adding multiple schema support would actually take very little extra work.
- No nested schema functionality.
- No custom debug tools for non-technicals.

#### Version 2
This inludes everything excluded in version 1 except:
- No custom debug tools for non-technicals.

#### Version 3
Everything including a user friendly debug tool for QA by non-technicals.
