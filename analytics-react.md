# Using the Analytics Data Layer Within React  
This document covers how to apply analytics tracking using our [Data Layer](README.md), specifically within a React application.

This is a little more technical than the [Data Layer](README.md) document and requires some understanding of how a React application works.

There are three main topics:
- Triggering the push of analytics data to our Data Layer.
- Capturing application level data to be used as part of the analytics data.
- Capturing component level data to be used as part of the analytics data.

### Pushing to the Data Layer
The considerations for where and how we trigger a `dataLayer.push()` are as follows:
- Centralise the configuration for the events that we need to listen to throughout the application (page view, clicks on specific buttons etc) which subsequently trigger pushes to the Data Layer. Dotting analytics specific triggers around our application would become more and more unmanageable over time rather than having a single location where this is configured.
- Tie how we listen to application events in with the technology we are using. We normally deal with application events through Redux actions (a Redux 'action' is broadly speaking synonymous with an 'event') so analytics should be in-keeping with this.
- A way to manage the case where multiple events may tigger the same analytics data. What our application calls its events may not be a 1:1 mapping of what analytics call their events. E.g A user first loading the application, then clicking to change the page and then our code automatically redirecting the user back to the previous page (after the save of a form let's say) may be three distinct events (or Redux actions) in our application; 'app load', 'page change' and 'page redirect'. Analytics would call all three of these the same thing; a page view. We need a mechanism outside of Redux actions where can write code only once to watch for the URL change and trigger page change analytics.

With this in mind, Redux middleware has been created for a Developer to be able to listen to any Redux action and trigger code in response in one single location (in this case, pushing to the Data Layer).

##### Action Listener Example
```js
import dataLayer from 'data-layer';
import { addPostDispatchListeners } from '../redux/listenerMiddleware';

// When any product click happens within the application, push product data to the analytics Data
// Layer.
addPostDispatchListeners('PRODUCT_CLICK', (action, store) => {
  dataLayer.push({
    'pagePath': store.getState().routing.pathname
    'productId': action.id,
    'productName': action.name,
    'productCategory': action.category
  }, 'Product Schema', 'productCTA');
});

// NOTE: there are 'pre' and 'post' dispatch listener methods (`addPreDispatchListeners` and
// `addPostDispatchListeners`) which give you the sate before or after the action updates state
// respectively. This is to cover the cases where we need to capture data from app state before or
// after the action takes its effect.
```
We can also listen to multiple events to trigger the same analytics code. Using the "'app load', 'page change' and 'page redirect'" example above, we could actually do:

```js
import dataLayer from 'data-layer';
import { addPostDispatchListeners } from '../redux/listenerMiddleware';

addPostDispatchListeners(['APP_LOAD', 'PAGE_CHANGE', 'PAGE_REDIRECT'], (action, store) => {
  dataLayer.push({
    'pagePath': store.getState().routing.pathname
  }, 'Page Schema', 'pageView')
});
```
The problem with this approach for that specific example is, if in future we create other events that result in a page change, we have to always remember to add the new event to this list.

##### State Listener Example
To solve the problem above, we are also able to listen for any changes in the application's state after an action has been triggered and taken its effect:
```js
import { addStateListeners } from '../redux/stateObserver';

addStateListeners((oldState, newState) => {
  const oldPath = oldState && oldState.routing.pathname;
  const newPath = newState && newState.routing.pathname;

  if (oldPath !== newPath) {
    dataLayer.push({
      'pagePath': newPath
    }, 'Page Schema', 'pageView')
  }
});

// NOTE: any number of state listeners can be added to `addStateListeners`
// e.g. `addStateListeners(listener1, listener2);``
```
Now we do not need to manage that list of events. We can simply watch for every URL path change that happens in the application's state.

_NOTE: state listeners behave similarly to Redux reducers, in that all listeners are triggered after every single triggered action, which is why we need to do an old/new state comparison._

Action listeners and state listeners should cover the vast majority of our triggering requirements. The only bugbear conceived is there will inevitably be occasions where we need to trigger an action solely for purpose of sending analytics data, without needing to update the application's state. An example would be tracking the click of external links. But these cases should be far and few between and don't really cause issue. More an undesirable hidden inefficiency that has no impact on users, just a Developer's mentality.

### Application Level Data
Because we use Redux to store the entirety of the application's state in a single place, a keen eye will notice that the listeners in the above examples have access to this state e.g. `store.getState()`. Therefore, when we need to push data to the Data Layer, we have access to almost any data from any part of the application at that given moment. Whether that be the current URL path, a list of products, any form data a user has filled in, if the user is logged in or not etc.

### Component Level Data
The only information we do not have access to using the above methods is the view. The physical page layout that has been generated as a result of the current state. There is one requirement we currently know of that the above methods do not cover and that is knowing exactly at what part of the application the user interacted with to trigger an event and subsequent push to the Data Layer. A product click can happen in several parts of the application e.g. On a product range page or in a 'featured products' listing box on a number of other pages. When we send the product analytics data, we need to know where the user currently sits within the application when they make that action. E.g. 'Books Range > Product Listing > Product Card > Product CTA' or 'Home Page > Todays Deals > Product Card > Product Image'. This is especially important when we have the same page displaying multiple sets of products under multiple different contexts.

We need to extend our visual components with analytics functionality so we can capture this data. One method is to 'decorate' a component as being 'analytics aware' using a Higher Order Component within React. With this we need to achieve two things:
- Allow us to give a component a more human readable, analytics friendly name, but still keep Developers free to name their components as they need in a way that makes sense within the context of the application. We need to be able to set this name on a global level and also have the option to override it in the place it is used (examples below as to why).
- A method for the component to discover its position with in the view, but only relative to other 'analytics aware' components.

The process is this:
1. As is the Redux recommended way, actions are passed to a component via its props and triggered usually by some kind of user interaction within the component, such as a button click. When this action is passed into the component, the analytics HOC will hijack it and attach the component's current location within the view e.g. 'Books Range > Product Listing > Product Card > Product CTA'. When the action is triggered, this information will then be passed along with it.
2. The Redux middleware that is responsible for triggering our analytics action listeners will pick up this location information that is now attached to the action and pass it in as the 3rd argument to our action listeners when called e.g. `productClickListener(action, store, location)`.
3. The listener is then free to use the location as needed.

This will happen unobtrusively, under the covers, so actions are still triggered in exactly the same way without any additional work required from the Developer. The only thing the Developer needs to do is essentially mark the component as 'analytics aware' using the HOC and give it an analytics friendly name.

##### Example HOC Usage:

```javascript
// Generic shared Button component used throughout our application. We have given it the analytics
// name of 'Button' which probably won't be very useful to analytics so we'll need to override that
// in the place the Button is actually used.
import React from 'react';
import { AnalyticsHoc } from 'react-analytics';

const Button = ({ children }) => {
  return (
    <button className="pb-button">{children}</button>
  );
};

export default AnalyticsHoc('Button')(Button);
```
```javascript
// Example Button usage
import React from 'react';
import Button from './components/Button.jsx';

const ProductCard = ({
  image,
  title,
  description
}) => {
  return (
    <div>
      <img src={img} />
      <h4>{title}</h4>
      <p>{description}</p>
      // The HOC exposes an analyticsName prop we can use to override the default 'Button' name
      <Button analyticsName="Product CTA">
        See Product
      </Button>
    </div>
  );
};

// We probably don't need to override this name wherever a ProductCard is used
export default AnalyticsHoc('Product Card')(ProductCard);
```
Alternatively we can wrap the Button in the HOC on-the-fly in the place it is used which is perhaps a better option in this case:
```javascript
import React from 'react';

const Button = ({ children }) => {
  return (
    <button className="pb-button">{children}</button>
  );
};

export default Button;
```
```javascript
// Example Button usage
import React from 'react';
import Button from './components/Button.jsx';
import { AnalyticsHoc } from 'react-analytics';

const ProductCTAButton = AnalyticsHoc('Product CTA')(Button);

const ProductCard = ({
  image,
  title,
  description
}) => {
  return (
    <div>
      <img src={img} />
      <h4>{title}</h4>
      <p>{description}</p>
      <ProductCTAButton>
        See Product
      </ProductCTAButton>
    </div>
  );
};

export default AnalyticsHoc('Product Card')(ProductCard);
```
