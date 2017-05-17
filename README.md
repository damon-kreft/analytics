# Analytics Introduction
This is the proposed definition of the organisation's implementation of a flexible and reusable Javascript library for tracking analytics data within its applications.

_NOTE: How this generic Javascript library can be used specifically within a React application is defined in the separate [analytics-react.md](analytics-react.md) document._

This library is agnostic to any 3rd party analytics services (Adobe Analytics, Google Analytics, Google Tag Manager etc) and therefore has no external dependencies. It is flexible enough to allow us to send data to multiple sources.

This has been written to be as clear as possible to any team member, not just the technical team.

##### A distinction between an 'event' and a 'trigger'

Two words you will see a lot throughout this documentation. An 'event' is an action that occurs in our application, whether that be from user input such a click or typing into a text field, or non-user interaction driven by our code, such as the event of retrieving data. 'product data loaded' would be an event.

A 'trigger' is something that happens in response to an event; in this context, the sending of analytics data. I.e. "The sending of product analytics data was triggered off the back of a product item click event". Multiple types of events can trigger the sending of the same analytics data.

# Analytics Data Layer
The library is inspired by the approach of Google Tag Manager (GTM), as GTM's purpose is to solve the problem of using multiple analytics services within the same application:
- The performance impact of including multiple snippets of analytics code within the same application.
- The development, testing and management overhead of having to capture generally the same analytics data but send it in the multiple different formats that each service expects, using each service's own specific API's and code libraries.

In the future, ideally GTM is the only service we will need, but as it stands we need to run GTM in parallel with Adobe Analytics, hence the requirement for our own variation of GTM's Data Layer.

In the same way that GTM's Data Layer works, this library will return a plain Javascript `Array` that has been extended with the functionality required to process and send analytics data to various sources. It is extended in the following way:

### Data Schemas

##### A `dataLayer.addSchema()` method
This will provide a means for us to define any number of data schemas. A schema represents the structure and expected values of the list of analytics data to send to our external sources after a given event e.g. a 'Main Menu' schema may have values specific only to menus, so we define a 'menuItemId' and 'menuItemName' which we send on the event of a menu item click. We may also have a 'Product' schema which has product specific data such as 'productId' and 'productCategory' which we send on the click of a product item.

##### Schema Validation
In a schema, we define the names of the values of what we need to send, such as a 'productCategory'. We can also specify the expected format of that value, such as 'text', 'number' or perhaps one out of a fixed list of values such as 'book', 'mug' or 'canvas'. We can also specify whether the value is required or optional or if it has a default value in the absence of one provided. If validation fails, we will be alerted to fix the problem. This will help ensure that the data we wish to capture and send is always there, in the format that we expect. If it is not, we will know about it.

##### Schema Nesting
The ability to 'nest' (or 'compose') schemas.

Using the 'Main Menu' and 'Product' schema examples above, both will always include page level data such as 'pageHost' and 'pageTitle'. Rather than repeating those same values across both schemas, we can create a parent schema called 'Page' that specifies those values and then add the 'Main Menu' and 'Product' schemas as children of 'Page'. The child schemas then inherit those value definitions when used so we don't have to add and maintain the same list across multiple schemas (see the example below).

##### Schema Example
```js
import dataLayer from 'data-layer';
import format from 'joi'; // a great lib for schema validation. See: https://www.npmjs.com/package/joi

dataLayer.addSchema({
  name: 'Page Schema',
  properties: {
    pageHost: format.string().hostname().required(), // required, must be text, in a correct hostname format
    pageTitle: format.string().required(),
    appVersion: format.string().regex(/^[0-9]+\.[0-9]$/).default('1.0').required() // ensures the format 'N.N' and this value is hard coded here using default(x)  by a developer
  }
  childSchemas: [{
    name: 'Main Menu Schema',
    properties: {
      menuItemId: format.integer().required(),
      menuItemName: format.string().required()
    }
  }, {
    name: 'Product Schema',
    properties: {
      productId: format.integer().required(),
      productCategory: format.string().allow('book', 'mug', 'canvas').required()
    }
  }]
});

// NOTE on formats:
// using `format.string().required().allow(null)` is the difference between a field being included
// but is allowed to have an empty value and the field not being including at all.

// We can also reference values from other fields to validate against. e.g.
// 'loggedInUser' can contain values only if the value of 'loggedIn' is 'yes'.
// See: https://github.com/hapijs/joi/blob/v10.4.1/API.md#refkey-options.
```

#### An (overridden) `dataLayer.push()` method
With our schemas defined, we can then start pushing analytics data to the Data Layer. We will override the default behaviour of the plain Javascript `Array.push()` to push the the analytics data along with the event and the schema name to validate against: `dataLayer.push(eventName, schemaName, data)`

##### `dataLayer.push()` Example
```js
import dataLayer from 'data-layer';

dataLayer.push('ProductClick', 'Product Schema', {
  pageHost: '/range/books',
  pageTitle: 'Our Store - Books',
  productId: 123,
  productCategory: 'book'
});
```

The data above is validated against the `Product Schema` we have defined and it if fails, an error is thrown for us to handle.

### Plugins for External Services
This will allow us to push analytics data to any number of external analytics services. The Data Layer will provide an adapter interface which accepts adapters specific to a service e.g. a Google Tag Manager adapter. The job of the adapter is to take the data we push to our own Data Layer and subsequently push it to the 3rd party service, taking the data we have passed, translating it into a format that the external service understands and then sending it to where it needs to go.

##### Google Tag Manager Adapter Example
```js
import dataLayer from 'data-layer';

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
Because our Data Layer is based on a plain Javascript `Array` (albeit, extended), we can easily test what is being pushed into it by our app either through the standard tools Developers use such as Chrome Dev Tools or we could eventually take that further and build a more in-depth and user friendly debugging tool to be used by non-Developers for QA (much like GTM's own). Analytics can also be tested automatically using an automated testing pipeline.
