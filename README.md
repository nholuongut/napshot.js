# Snapshot JS

![](https://i.imgur.com/waxVImv.png)
### [View all Roadmaps](https://github.com/nholuongut/all-roadmaps) &nbsp;&middot;&nbsp; [Best Practices](https://github.com/nholuongut/all-roadmaps/blob/main/public/best-practices/) &nbsp;&middot;&nbsp; [Questions](https://www.linkedin.com/in/nholuong/)
<br/>

Node.js app for slicing and dicing paginated chunks of data with easy sorting and filtering.

Installing via npm
-----------

Snapshot is added to the npm registry, and can therefore be downloaded with `npm install node-snapshot`.

Dependencies
-----------

All dependencies can be installed with `bower install` and `npm install`, however the list is as follows:

 * <a href="http://socket.io/">Socket.IO</a> (client and server);
 * <a href="http://underscorejs.org/">Underscore.js</a> (server);
 * <a href="http://nodejs.org/">Node.js</a> (server);
 * <a href="http://square.github.io/crossfilter/">Crossfilter</a> (server);
 * <a href="http://mochajs.org/">Mocha</a> &ndash; with Should.js (grunt);

Quick Start
-----------

Once a WebSocket connection has been successfully established, you're able to bootstrap Snapshot, passing in the WebSocket (Socket.IO) reference as a dependency.


```javascript
var $snapshot = new Snapshot().bootstrap(socket).useDelta(false);
```

You then need to tell Snapshot which collection it's going to be creating snapshots of. You can pass in an optional string for the `primaryKey` &ndash; if you omit the `primaryKey` then the first key of the first model is used.

```javascript
$snapshot.setCollection(collection, primaryKey);
```

<small>(`primaryKey` is an optional parameter.)</small>

Snapshot listens for three events natively, and these are handled automatically.

 * `snapshot/:namespace/perPage` &ndash; Set amount per page;
 * `snapshot/:namespace/pageNumber` &ndash; Set current page number;
 * `snapshot/:namespace/sortBy` &ndash; Set order column and ascending/descending;

As Snapshot supports multiple instances, a namespace is used to distinguish the events. If you don't explicitly specify a namespace in the instantiation then it will be `default`. Therefore all of your events will be: `snapshot/default/perPage`, `snapshot/default/pageNumber` and `snapshot/default/sortBy`.

<small>(`:namespace` is the name you provided upon instantiation of `Snapshot` &ndash; if you didn't, then it's `default`.)</small>

When the collection has been updated, Snapshot emits the `snapshot/:namespace/contentUpdated` event, passing through the snapshot as the first argument, and statistics relating to the request as the second argument.

For sorting by any given column, you can emit the `snapshot/:namespace/sortBy` event passing in the `key` and `direction` (ascending/descending). If you omit the `direction` property (or set its value to `false`) then Snapshot will cleverly invert the current sorting direction for you.

```javascript
socket.emit('snapshot/:namespace/sortBy', property, 'descending');
```

Filtering
-----------

In addition to sorting and limiting, Snapshot also allows for the filtering of the collection. For this you can use the `applyFilter` method. Unfortunately you will need to read <a href="https://github.com/square/crossfilter/wiki/API-Reference" target="_blank">Crossfilter's API Reference</a> before you begin filtering &ndash; or for simple filtering you can use Snapshot's <a href="#bundled-filters">primitive bundled filters</a>.

```javascript
socket.emit('filterByWord', text);
```

You can apply a filter however you like. It doesn't necessarily need to be applied via WebSockets, you could just as well use vanilla Node.js or Express.js. In our example though, we emit the `filterByWord` event to the Node.js server, and then we need to listen for that event.

```javascript
socket.on('filterByWord', function(text) {

    $snapshot.applyFilter('word', function(dimension) {

        dimension.filterFunction(function(d) {
            var regExp = new RegExp(text, 'i');
            return d.match(regExp);
        });

    });

});
```

You essentially invoke the `applyFilter` on the `snapshot` object. Snapshot will pass in the `dimension` argument to your lambda function &ndash; `this` context is preserved. It's then entirely up to you to apply that dimension to the collection.

If you would like to clear a specific dimension, then you can use the `clearFilter` method &ndash; which takes the property name as its one and only argument.

```javascript
$snapshot.clearFilter('word');
```

You can also clear every single filter by using the `clearFilters` method.

```javascript
$snapshot.clearFilters();
```

Sometimes you may wish to clear certain filters before applying other filters, in which case you should utilise the <a href="#composite-filters">composite filters</a> approach by passing through the dimensions you wish to clear as well. You can then invoke the `filterAll` method on the dimension(s) you wish to clear.

```javascript
$snapshot.applyFilter(['latitude', 'longitude'], function(latitudeDimension, longitudeDimension) {

    // Clear the latitude dimension.
    latitudeDimension.filterAll();

    // But then apply the longitude dimension.
    longitudeDimension.filterFunction(function(d) {
        return (d > 0);
    });

}, 'reduce');
```

<h3>Composite Filters</h3>

Before the release of 0.2.7 you could only apply filters one-at-a-time, which meant if you grouped filters together then the `contentUpdated` event would be emitted for each filter.

However, 0.2.7 introduced the concept of composite filters, which are nothing more than groups of filters which can be applied together, and have the event emitted once all filters have been applied. In order to create a composite filter, simply pass in an array of properties to filter on to the `applyFilter` method &ndash; the callback in the second argument will be the dimension of each property.

```javascript
$snapshot.applyFilter(['latitude', 'longitude'], function(latitudeDimension, longitudeDimension) {

    // Filter to ensure the latitude is above 50, and longitude above 0.

    latitudeDimension.filterFunction(function(d) {
        return (d > 50);
    });

    longitudeDimension.filterFunction(function(d) {
        return (d > 0);
    });

    // Event will now be fired to reflect the current collection state.

}, 'reduce');
```

<h3>Bundled Filters</h3>

In light of Crossfilter's learning curve, Snapshot ships with a handful of bundled filters for common filtering techniques. These can all be invoked by emitting an event with a corresponding value.

 * `snapshot/:namespace/fuzzyFilter` `{String}`
 * `snapshot/:namespace/exactFilter` `{String}`
 * `snapshot/:namespace/rangeFilter` `{Array}`
 * `snapshot/:namespace/regExpFilter` `{String}` `{String}`
 * `snapshot/:namespace/inArrayFilter` `{String}` `{Array}`
 * `snapshot/:namespace/notInArrayFilter` `{String}` `{Array}`
 * `snapshot/:namespace/clearFilter` `{String}`

```javascript
socket.emit('snapshot/default/fuzzyFilter', 'word', 'abc');
socket.emit('snapshot/default/exactFilter', 'word', 'kittens');
socket.emit('snapshot/default/rangeFilter', 'word', [12, Infinity]);
socket.emit('snapshot/default/regExpFilter', 'word', '[a-z0-9]+', 'ig');
socket.emit('snapshot/default/inArrayFilter', 'word', ['firstWord', 'secondWord']);
socket.emit('snapshot/default/notInArrayFilter', 'word', ['thirdWord']);
socket.emit('snapshot/default/clearFilter', 'word');
```

Each bundled filter expects the event name (`snapshot/default/fuzzyFilter`), the key (`word`), and value (`abc`).

<h3>Filtering Types</h3>

By default when you apply a filter, the previous filter will be cleared which is mostly likely the behaviour you're looking for. However, what if the user clicks **red**, and then clicks **blue**? Wouldn't it be nice if we could filter by both **red** and **blue**? In that case you're looking for the third argument of the `applyFilter` method.

```javascript
$snapshot.applyFilter('word', ..., 'reduce');
```

 * `afresh` &ndash; filtering cleared before each filter;
 * `reduce` &ndash; filtering applied on current collection;

Ranges
-----------

Since the browser does not download every model into the browser, it's impossible to determine what the minimum/maximum for any given key is without loading all models. Snapshot therefore allows you to specify which columns you wish to generate minimum/maximum ranges for.

Please be careful with these as too many may noticeably slow down your Snapshots.

The following would specify that you wish to retrieve the ranges for the `id` property on every content change.

```javascript
$snapshot.setRanges(['id']);
```

When the content changes you can access the range with `stats.ranges.id.min` and `stats.ranges.id.max`.

Multiple Instances
-----------

When instantiating Snapshot you should pass in the namespace for the current collection &ndash; that way you could create a new instance of `Snapshot` with a unique collection of models.

```javascript
var $dogs       = new Snapshot('dogs').bootstrap(socket).useDelta(false);
var $cats       = new Snapshot('cats').bootstrap(socket).useDelta(false);
var $rabbits    = new Snapshot('rabbits').bootstrap(socket).useDelta(false);
```

In the above example Snapshot will have 9 events to listen to (3 events * 3 snapshots):

 * `snapshot/dogs/perPage`, `snapshot/dogs/pageNumber`, `snapshot/dogs/sortBy`
 * `snapshot/cats/perPage`, `snapshot/cats/pageNumber`, `snapshot/cats/sortBy`
 * `snapshot/rabbits/perPage`, `snapshot/rabbits/pageNumber`, `snapshot/rabbits/sortBy`

And it will emit 3 events:

 * `snapshot/dogs/contentUpdated`
 * `snapshot/cats/contentUpdated`
 * `snapshot/rabbits/contentUpdated`

If you don't create a namespace then the namespace will be set to `default`.

Delta Updates
-----------

There may be instances where sending delta updates is preferable to re-sending whole models. Snapshot supports the providing of delta updates &ndash; essentially, any models that have already been transmitted across the wire will not be sent again in their entirety; instead only their primary ID is sent.

```javascript
var $snapshot = new Snapshot().bootstrap(socket).useDelta(true);
```

Once you've enabled delta updates using `useDelta(true)` as part of the bootstrap process, Snapshot will keep a history of transmitted models. It's crucial that you set the appropriate primary ID when invoking `setCollection`, otherwise a default primary key will be assumed.

```javascript
$snapshot.setCollection([{ id: 1 }, { id: 2 }, { id: 3 }], 'id');
```

**Note:** You can suppress the initial event when invoking `setCollection` by passing `true` as the third argument.

Since unique models will <strong>only</strong> ever be transmitted once, it's imperative that you keep a history of all models from the `snapshot/:namespace/contentUpdated` event, and then to utilise those from your local cache when you come across a delta model.

Delta models are nothing more than the primary key of the model, which will help you lookup the model from your own collection cache. Therefore to detect a delta model, simply use something like `Number.isFinite` (or Underscore's `_.isNumber`) on the returned collection.

```javascript
socket.on('snapshot/:namespace/contentUpdated', function(models, stats) {

    _.forEach(models, function(model) {

        if (_.isNumber(model)) {
            // Delta model!
            return;
        }

        // ...

    });

});
```

Example
-----------

Snapshot comes bundled with an example to get you started.

 * Navigate to `example/server` and run `node default.js`;
 * Open `example/client/index.html` in your browser;

Architecture
-----------

Below is a simple diagram of how Snapshot works. It demonstrates how the `snapshot/:namespace/pageNumber` event operates &ndash; which is also the same way other native Snapshot events function. It also demonstrates the flow of custom filters.

<img src="6o0Nw5Y.png" alt="Snapshot Architecture" />

 * Browser establishes a WebSocket connection to Node.js &ndash; models are added;
 * Browser emits `snapshot/:namespace/pageNumber` event with data (example);
 * Snapshot along with Crossfilter updates the collection <i>snapshot</i>;
 * Snapshot emits `snapshot/:namespace/contentUpdated` event with the updated collection;
 * Browser emits a custom event (`customFilterApplied`) with the data;
 * Node.js listens for the `customFilterApplied` event and then interacts with Snapshot;
 * Snapshot emits the `snapshot/:namespace/contentUpdated` event with the updated filter applied;

Testing
-----------

<h3>Unit Testing</h3>

Grunt is a prerequisite to run the Mocha tests, which is installed when you run `npm install`. Afterwards all of Snapshot's unit tests can be run with the `grunt test` command from the terminal.

<h3>Cucumber</h3>

Snapshot also comes bundled with a handful of Cucumber tests.

 * `cd tests/cucumber`
 * `bundle install`
 * `bundle exec cucumber`

Philosophy
-----------

Loading a large collection of models into the browser is slow and unnecessary, instead Snapshot uses WebSockets to serve snapshots of those models to the browser when requested. It retains the state of the models, and so if a filter is changed, or the page number incremented, it will modify the snapshot <strong>only</strong> for that client.

Snapshot is also tremendously fast because of its use of Socket.io and Crossfilter. Snapshot listens for events to change the state of the collection of models, and fires another event to let the client know the snapshot was updated. Crossfilter allows Snapshot to quickly slice and dice models &ndash; in the example, slicing and dicing takes 0-1 milliseconds for 1,000 models.

Since Snapshot uses Node.js, the browser support is that of Socket.io, which essentially means Snapshot supports Internet Explorer 5.5+.

 <h3>Example</h3>

 * Browser connects to Snapshot on Node.js server;
 * Snapshot emits `snapshot/default/contentUpdated` with first page's 50 models;
 * Browser increments the page number;
 * Snapshot emits `snapshot/default/contentUpdated` with second page's 50 models;
 * Browser applies filter to select only red items;
 * Snapshot emits `snapshot/default/contentUpdated` to supply second page's red models;
 * Browser sorts the models by their colour;
 * Snapshot emits `snapshot/default/contentUpdated` to supply second page's red models ordered globally by colour;

Caching
-----------

You may wish to cache the collection loaded into Snapshot &ndash; for this we recommend something like <a href="https://github.com/nholuongut/rediscache" target="_blank">RedisCache</a>.

# 🚀 I'm are always open to your feedback.  Please contact as bellow information:
### [Contact ]
* [Name: nho Luong]
* [Skype](luongutnho_skype)
* [Github](https://github.com/nholuongut/)
* [Linkedin](https://www.linkedin.com/in/nholuong/)
* [Email Address](luongutnho@hotmail.com)

![](https://i.imgur.com/waxVImv.png)
![](Donate.png)
[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/nholuong)

# License
* Nho Luong (c). All Rights Reserved.🌟
