[![build status](https://api.travis-ci.org/disjunction/node-request-plus.png)](https://travis-ci.org/disjunction/node-request-plus)

# request-plus

A wrapper around [request-promise-native](https://www.npmjs.com/package/request-promise-native) module, adding the following features (all are optional):

* `EventEmitter` integration - you can set a generic listener for any fired/successful/failed request
* automatic retry for particular failures
* caching results using [cache-manager](https://www.npmjs.com/package/cache-manager)
* exporting Prometheus monitoring metrics using [prom-client](https://www.npmjs.com/package/prom-client)

... and you can add your own wrappers too!

The only depency is `request-promise-native`, which in its turn has a `request` as a peer dependecy. So you can flexibly use whatever request version you like.

## Usage
```
npm install request request-plus
```

```javascript
// here all wrappers are optional
const request = require('request-promise')({
  event: { /*...*/ },
  retry: { /*...*/ },
  cache: { /*...*/ },
  prom: { /*...*/ },
});
// ... or ..
const request = require('request-promise')
  .plus.add('event', { /*...*/ })
  .plus.add(/* wnother wrapper name * /, { /*...*/ })
  .plus.add(/* wnother wrapper name * /, { /*...*/ })
  // ...

request('http://example.com/some/api')
  .then(data => { /*...*/ })
  .catch(error => { /*...*/ });
```

## Basic Example

```
npm install request request-plus cache-manager --save
```

Let's say we want to get JSON data from some resource, which fails sometimes and is in general quite slow. So we want to cache its results and do retries if it fails with a timeout or typical server errors.

```javascript
// setup a cache object
const cacheManager = require('cache-manager');
const cache = cacheManager.caching({
  store: 'memory',
  max: 500 // keep maximum 500 different URL responses
});

const rp = require('request-plus');

// create a concrete wrapper
// you have can multiple in one project with different settings
const request = rp({
  // use retry wrapper
  retry: {
    attempts: 3
  },

  // use cache wrapper
  cache: {
    cache: cache,
    cacheOptions: {
      ttl: 3600 * 4 // 4 hours
    }
  }
});

// you can use all the options available in basic request module
request({
  uri: "http://some.service/providing/data",
  json: true
})
  .then(data => {
    // we get here if we got a 200 response within 3 retries
  })
  .catch(error => {
    // well get here if the URL failed with 4xx errors,
    // or 3 retry attempts failed
  });
```

## Wrappers

The wrappers can be specified in options when creating a new requestPlus wrapper (simple way), or you can add them one by one (advanced)

When specified in options, the wrappers will be added in a particular (common sense) order, namely: `event`, `retry`, `cache`, `prom`. Another limitation here: you can have only one wrapper of each type.

Sample:
```javascript
const rp = require('request-plus');
const request = rp({
  event: true,
  retry: true,
  prom: {
    metric: myMetric
  }
});
```
When adding one by one you have full control of the order,
and you may add wrappers of the same type.

```javascript
const rp = require('request-plus');
const request = rp()
  .plus.wrap('prom', {
    metric: rawRequestHistogram
  })
  .plus.wrap('retry')
  .plus.wrap('prom', {
    metric: retriedRequestHistogram
  });
```

### Event Wrapper

This wrapper adds `emitter` to the `.plus` container
and fires basic events for each request going through:

* request - on start of the request
* error - on error
* response - on successful response

```javascript
const request = require('request-plus')({event: true});

// always output failed http requests to std error
// together with used request param
// independent of promise chains/catch clauses
request.plus.emitter.on('error', (error, uri) => {
  console.error('http request failed %j', uri);
})
request('http://..soooo...bad...')
.catch(() => {
  console.log("something happen, i don't know what");
}) 
```

(see sourcecode to see additional params provided for each event)

### Retry Wrapper

Params (all optional):
* **attempts** = 3 - number of attempt before giving up
* **delay** = 1000(ms) - delay between retries. You can provide a closure and calculate it to make a progressive delay, e.g. `attempt => 500 * attempt * attempt` 
* **filterError** - closure defining whether a retry should be done. By default it returns `true` for a timeout and `statusCode` in `[500, 502, 503, 504]`

```javascript
const rp = require('request-plus');
const request = rp({
  retry: {
    attempts: 5,
    delay: 1500,

    // retry all errors
    filterErrors: error =>
      error.message === 'Error: ETIMEDOUT'
      || error.statusCode >= 400
  }
});
```

If there is an `event` wrapper initialised, then it will additionally fire events: `retryRequest`, `retryError`, `retrySuccess` providing the current attempt counter.

### Cache Wrapper

Should be used together with a third-party module: [cache-manager](https://www.npmjs.com/package/cache-manager)

Params:
* **cache** (required) - an instance of cache for `cache-manager` module
* **cacheOptions** - options used for `set()`
* **getKey** - closure generating string cache key for given request options. By default for string param - the full URI is used as key, for an object a hash generated for it (see below)
* **hash** - hash function for default **getKey** behavior. By default it generates a key using a very cheap algorithm, but with a significant collision probability

```javascript
const rp = require('request-plus');
const cacheManager = require('cache-manager');
const memeoryCache = cacheManager.caching({store: 'memory'});
const crypto = require('crypto');
const request = rp({
  cache: {
    cache: memeoryCache,
    hash: str => crypto.createHash('md5').update(str).digest("hex")
  }
});
```
If there is an `event` wrapper initialised, then it will additionally fire events: `cacheRequest` and `cacheMiss`. You can use those to gather stats and calculate cache hits as `count(hits) = count(cacheRequests) - count(cacheMisses)`.

### Prometheus Wrapper

Should be used together with a third-party module: [prom-client](https://www.npmjs.com/package/prom-client)

The wrapper takes a prometheus metric and uses it to monitor both successful responses and error. It supports all basic metric types assuming that `Counter` just counts responses and `Gauge`, `Histogram` and `Summary` measure latency.

If this wrapper doesn't need your needs, you might want do your own measurements using `event` wrapper (see above).

Params:
 * **metric** - and instance of prom-client metric
 * **labels** - an object with labels or a closure generating labels on the fly

```javascript
const promClient = require('prom-client');
const testHistogram = new promClient.Histogram(
  'test_histogram',
  'help of test_histogram',
  {
    buckets: [0.01, 1],
    labels: ["status_code"]
  }
);
const request = require('request-plus')({
  prom: {
    metric: testHistogram,
    labels: error => {
      status_code: error ? error.statusCode : 200
    }
  }
});
```

### Adding own wrapper

```javascript
function myWrapper(requester) {
  return function(uri, requestOptions, callback) {
    if (requester.plus.emitter) {
      requester.plus.emitter.emit('myEvent', 'hello from me');
    }
    console.log('the uri is %j', uri);
    return requester(uri, requestOptions, callback);
  };
}

const request = require('request-plus')()
  .plus.wrap('event')
  .plus.wrap(myWrapper);
```
