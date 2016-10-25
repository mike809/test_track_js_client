# TestTrack JS Client

This is the JavaScript client library for the [TestTrack](https://github.com/Betterment/test_track) system.

It provides client-side split-testing and feature-toggling through a simple, mostly declarative API.

This library intends to obscure the details of assignment and visitor session management, allowing you to focus entirely on the experience a visitor should have when she has been assigned a variant.

If you're looking to do server-side assignment and you're using Rails, then check out our [Rails client](https://github.com/Betterment/test_track_rails_client).

## Installation

You can add the test track js client to your application via bower or npm.

```
bower install test_track_js_client --save
```

```
npm install test_track_js_client --save
```

You can find the latest version of the test track JS client [here](https://github.com/Betterment/test_track_js_client/releases).

The test track JS client currently has the following dependencies: `blueimp-md5`, `node-uuid`, `jquery` and `jquery.cookie`.

If you're using a fancy build pipeline ([grunt](http://gruntjs.com/), [gulp](http://gulpjs.com/), [broccoli](http://broccolijs.com/), [webpack](https://webpack.github.io/)), then you are all set. If not, you have a few other [options](#alternative-setup) for loading the client into your page.

## Configuration

Before using the client you must call `TestTrack.initialize()`. This method also takes some optional [configuration parameters](#advanced-configuration), if you fancy.

### API

#### `.vary(split_name, options)`

The `vary` method is used to perform a split. It takes 2 arguments.

* `split_name` --  The first argument is the name of the split. This will be a snake_case string, e.g. `"homepage_redesign_q1_2015"`.
* `options` -- The second argument is an object that contains the `context` of the assignment, a variant/callback configuration (`variants`), and a default variant (`defaultVariant`).
  * `context` -- is a string that the developer provides so that the test track server can record where an assignment was first created. If a call to `vary` is made in more than one place for a given split, you'll be able to see which codepath was hit first.

  * `variants` -- The variant/callback configuration is an object whose keys are the variant names and whose values are function handlers for each of those variants.

  * `defaultVariant` -- The default variant is used if the user is assigned to a variant that is not represented in the `variants` object. When this happens, Test Track will execute the handler of the default variant and re-assign the user to the default variant. **You should not rely on this defaulting behavior, it is merely provided to ensure we don't break the customer experience.** You should instead make sure that you represent all variants of the split in your `variants` and if variants are added to the split on the backend, update your code to reflect the new variants. Because this defaulting behavior re-assigns the user to the `defaultVariant`, no data will be recorded for the variant that is not represented. This will impede our ability to collect meaningful data for the split.

Here is an example of a 4-way split where `'variant_4'` is the default variant. Let's say `'variant_5'` was added to this split on the backend but this code did not change to reflect that new variant. Any users that Test Track assigns to `'variant_5'` will be re-assigned to `'variant_4'`.

  ```js
  TestTrack.vary('name_of_split', {
      context: 'homepage',
      variants: {
          variant_1: function() {
              // do variant 1 stuff
          },
          variant_2: function() {
              // do variant 2 stuff
          },
          variant_3: function() {
            // do variant 3 stuff
          },
          variant_4: function() {
            // do variant 4 stuff
          }
      },
      defaultVariant: 'variant_4'  // default to variant_4 (this is required)
  });
  ```

#### `.ab(split_name, options)`

The `ab` method is used exclusively for two-way splits and feature toggles. It takes 2 arguments.

* `split_name` --  The first argument is the name of the split. This will be a snake_case string, e.g. `"homepage_chat_bubble"`.
* `options` -- The second argument is an object that contains the `context`, an optional `trueVariant`, and a `callback` function.
  * `context` -- is a string that the developer provides so that the test track server can record where an assignment was first created. If a call to `vary` is made in more than one place for a given split, you'll be able to see which codepath was hit first.
  * `trueVariant` -- an optional parameter that specifies which variant is the "true" variant and the other variant will be used as the default. Without the true variant, `ab` will assume that the variants for the split are named `'true'` and `'false'`.
  * `callback` -- a single function that will be called for all variants. If the `trueVariant` is assigned to the visitor then `true` will be passed to the `callback`.

  ```js
  TestTrack.ab('name_of_split', {
      context: 'homepage',
      trueVariant: 'variant_name',
      callback: function(hasVariantName) {
          if (hasVariantName) {
              // do something
          } else {
              // do something else
          }
      }
  });
  ```

  ```js
  TestTrack.ab('some_new_feature', {
      context: 'homepage',
      callback: function(hasFeature) {
          if (hasFeature) {
              // do something
          }
      }
  });
  ```

#### `.logIn(identifier, value)`

The `logIn` method is used to ensure a consistent experience across devices. For instance, when a user logs in to your app on a new device, you should also log the user into Test Track in order to grab their existing split assignments instead of treating them like a new visitor. It takes 2 arguments.

* `identifier` --  The first argument is the name of the identifier. This will be a snake_case string, e.g. `"myapp_user_id"`.
* `value` -- The second argument is a primitive value, e.g. `12345`, `"abcd"`

```js
TestTrack.logIn('myapp_user_id', 12345).then(function() {
  // From this point on you have existing split assignments from a previous device.
});
```

## Advanced Configuration

When you call `TestTrack.initialize()` you can optionally pass in an analytics object, an error logger, and a callback that will run after the test track visitor has loaded, but before any analytics events have fired. For example:

```js
TestTrack.initialize({
    analytics: {
        trackAssignment: function(visitorId, assignment, callback) {
            var props =  {
                SplitName: assignment.getSplitName(),
                SplitVariant: assignment.getVariant(),
                SplitContext: assignment.getContext()
            };

            remoteAnalyticsService.track('SplitAssigned', props, callback);
        },
        identify: function(visitorId) {
            remoteAnalyticsService.identify(visitorId);
        },
        alias: function(visitorId) {
            remoteAnalyticsService.alias(visitorId);
        }
    },
    errorLogger: function(message) {
        RemoteLoggingService.log(message); // logs remotely so that you can be alerted to any misconfigured splits
    },
    onVisitorLoaded: function(visitor) {
        // callback that will run after the test track visitor has loaded, but before any analytics events have fired
    }
});
```

## Alternative Setup

**This is only if you're not using a build pipeline**

### Simple HTML setup

You can load the dependencies and the library via `script` tags, like this:
```html
<script type="text/javascript" src="path/to/deps/jquery/dist/jquery.js"></script>
<script type="text/javascript" src="path/to/deps/jquery.cookie/jquery.cookie.js"></script>
<script type="text/javascript" src="path/to/deps/blueimp-md5/js/md5.js"></script>
<script type="text/javascript" src="path/to/deps/node-uuid/uuid.js"></script>
<script type="text/javascript" src="path/to/deps/test_track_js_client/dist/testTrack.min.js"></script>
```

Or, you can load the bundled and minified version of the client that includes all of the dependencies for you (except jQuery), like this:
```html
<script type="text/javascript" src="path/to/deps/jquery/dist/jquery.js"></script>
<script type="text/javascript" src="path/to/deps/test_track_js_client/dist/testTrack.bundled.min.js"></script>
```

### RequireJS setup

You must provide aliases for the test track JS client's dependencies in your RequireJS config like so:

```js
require.config({
    paths: {
        'jquery': 'path/to/deps/jquery/dist/jquery.js',
        'jquery.cookie': 'path/to/deps/jquery.cookie/jquery.cookie.js',
        'node-uuid': 'path/to/deps/node-uuid/uuid',
        'blueimp-md5': 'path/to/deps/blueimp-md5/js/md5'
    }
});
```
Then you can require the test track client anywhere you need it in classic requirejs style:
```js
var TestTrack = require('path/to/deps/test_track_js_client/dist/testTrack');
```
OR
```js
define([
'path/to/deps/test_track_js_client/dist/testTrack'
], function(TestTrack) {

});
```

## Development

### Running tests
1. Clone this repo
1. run `npm install` to download npm dependencies
1. run `bower install` to download bower dependencies
1. run `grunt` to run the tests and build the distributables
