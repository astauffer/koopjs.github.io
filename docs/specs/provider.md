# Provider

Add support for any remote API or datasource to Koop. Dive into the docs below or check out a working sample [here](https://github.com/koopjs/koop-provider-sample).

## Index.js

Every provider must have a file called `index.js`. Its purpose is to tell Koop how to load and use the provider. The keys and values are enumerated in the example below.
<script src="https://gist.github.com/dmfenton/93379d71f4412716b3e508e1c8612cfa.js"></script>

## Model.js

Every provider must have a Model. This is where almost all of the business logic of the provider will occur. Its primary job is to fetch data from a remote source like an API or database and return GeoJSON to Koop for further processing.

### Function: getData

Models are required to implement a function called `getData`.  It should fetch data from the remote API, translate the data into GeoJSON (if necessary) and call the `callback` function with the GeoJSON as the second parameter. If there is an error in fetching or processing data from the remote API it should call the `callback` function with an error as the first parameter and stop processing.

<script src="https://gist.github.com/dmfenton/066061daa62b53c60f1fcbf94ade9567.js"></script>

## Cached vs Pass-Through

Providers typically fall into two categories: cached and pass-through.

### Pass-Through

Pass-through providers do not store any data, they act as a proxy/translator between the client and the remote API.

[Koop-Provider-Yelp](https://github.com/koopjs/Koop-Provider-Yelp) is a good example. The Yelp API supports filters and geographic queries, but it only returns 20 results at a time and there is no way to download the entire Yelp dataset.

It makes sense to use a pass-through strategy if at least one of the following is true:
- The remote dataset changes frequently
- The remote dataset is very small
- The remote dataset is too large to be fetched as a whole
- The remote API does not allow fetching the entire dataset
- The remote API supports filters and geographic queries
- The remote API is quick to respond

The request below fetches data from yelp and translates it into Geoservices JSON

```
http://localhost:8080/yelp/FeatureServer/0?where=term=pizza
```
GeoJSON can be retrieved as well
```
http://localhost:8080/yelp/FeatureServer/0?where=term=pizza&f=geojson
```

### Cached

Cached providers periodically request entire datasets from the remote API.

[Koop-Provider-Craigslist](https://github.com/dmfenton/koop-provider-craigslist) is a good example. The Craigslist API returns the entire set of postings for a given city and type in one call (e.g. Atlanta apartments). The data also does not change that frequently. Therefore the Craigslist provider uses the Koop cache with a TTL of 1 hour, guaranteeing that data will never be more than an hour out of date.

It makes sense to use a cache strategy if at least one of the following is true:
- The remote dataset does not change often
- The remote dataset is large
- The entire remote dataset can be fetched
- The remote API does not support filters or geographic queries
- The remote API is slow to respond

## Advanced

Providers can do more than simply implement `getData` and hand GeoJSON back to Koop's core. In fact, they can extend the API space in an arbitrary fashion by adding routes that map to controller functions. Those controller functions call functions on the model to fetch or process data from the remote API.

### Enable/Disable parameters

Recall the `getData` function takes in a `req` that has a set of `params`:
- host
- id

By default the host parameter is disabled and the id parameter is enabled. So requests like the following would work:

`/provider/:id/FeatureServer/0`

To enable the `host` parameter, edit the `index.js` file and set `hosts` to `true`. Then requests like the following will work:

`/provider/:host/:id/FeatureServer/0`

Sometimes, your provider may need no parameters at all. To disable the `id` parameter, edit `index.js` and set `disableIdParam` to `true`. Then requests like the following will work:

`/provider/FeatureServer/0`

### Routes.js

This file is simply an array of routes that should be handled in the namespace of the provider e.g. http://adapters.koopernetes.com/agol/arcgis/datasets/e5255b1f69944bcd9cf701025b68f411_0

In the example above, the provider namespace is `agol`, and `arcgis` and `e5255b1f69944bcd9cf701025b68f411_0` are parameters. Anything that matches `/agol/*/datasets/*` will be be handled by the model.

Each route has:
- path: an express.js style route that includes optional parameters
  - see https://www.npmjs.com/package/path-to-regexp for details
- methods: the http methods that should be handled at this route
  - e.g. `get`, `post`, `put`, `delete`, `patch`
- handler: the Controller function that should handle requests at this route

Example:

<script src="https://gist.github.com/dmfenton/a8eacc31afeb460cbc919e7e2095f4ce.js"></script>

### Controller.js

The purpose of the Controller file is to handle additional routes that are specified in `route.js`. It is a class that is instantiated with access to the model. Most processing should happen in the model, while the controller acts as a traffic director.

As of Koop 3.0, you **do not** need to create a controller. If you want to add additional functionality to Koop that is not supported by an output plugin, you can add additional functions to the controller.

Example:

<script src="https://gist.github.com/dmfenton/0a0f5d287b4cc2a7f3fbf9616b004198.js"></script>


### Model.js
In addition to implementing `getData`, the model exists to interact with the remote API and to serve the controller. Any function the controller would need to call to handle a request should be implemented as a public function on the model.

Example:

<script src="https://gist.github.com/dmfenton/ab9ce1deea644c176092a44010b0ede3.js"></script>
