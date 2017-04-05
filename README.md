# scirra-offline-sw

This is a Service Worker script for offline support used for Construct 2 and Construct 3. It supports up-front and lazy caching of static resources for offline use. It aims to work like a better AppCache with a simpler JSON-based file list format.

Many thanks to Jake Archibald for putting up with endless questions and writing some code that influenced the design of this version. I doubt I could have done it without him!

## Features

- an **external JSON file** with a version number and list of URLs to cache (separating code and data, making updates smaller and quicker, never need to update the Service Worker itself, no special rules around updating)
- optional **paths to lazy-load** (where requests are not pre-cached, but are cached on first request)
- implicit caching of the index page (no need to predict in the file list if it's index.html or main.aspx etc, also correctly handles query strings e.g. if main page is /?foo=bar)
- offline-first (works offline; when online serves cache-first saving bandwidth; uncached responses default to network)
- checks for updates in the background, identified by changing "version" field in the JSON file
- **cache-busting updates** to avoid stale HTTP cache entries, but does not bypass on first cache to avoid duplicated requests
- avoids mixing resource versions in the same pageload (always sources from same cache; runs upgrade & cleanup on "navigate" requests with no other clients open)
- attempts to handle updates in an atomic fashion to rule out partial caches (although the necessary API features to actually guarantee this are missing)
- avoids conflicting with other caches on the same origin, even from this same library
- vanilla JS with no assumptions about your build/deployment methods (i.e. this repo is just a .js file) - but note it does depend on localforage for lazy-load storage
- work with arbitrary server configurations, e.g. no need to specially configure caching on the Service Worker script or any other files (which cannot be specified anyway if you develop frameworks/middleware), no server-side scripts
- update upon pressing the browser reload button (note this is a bit hacky, see below)
- **sends messages over a BroadcastChannel indicating update events** (e.g. downloading update, update ready) so pages can notify users accordingly
- **robust for production use** - deployed with [Construct 2](https://www.scirra.com/) where it has been battle-tested in a variety of environments, and also used for the Construct 3 editor itself.

## Dependencies

This script requires localforage.js in the same path. Get it from [LocalForage on Github](https://github.com/localForage/localForage)

Localforage is used as a simple way to store the lazy-load paths so they can be accessed in fetch events.

## How to use it

Copy `sw.js` (and `localforage.js` if you don't already use it) to the same folder as your index page, and install the service worker from a script like this:

```
navigator.serviceWorker.register("sw.js", { scope: "./" });
```

(You'll want to add error checking, as well as testing if `navigator.serviceWorker` is supported)

**Tip:** Delay registration until the page has finished loading to allow the first load to finish without contending against the SW caching in the background. For example in our Construct 2 games, we wait for the game to finish loading before registering the SW, so loading isn't held up by the SW requesting music files that aren't immediately required. This makes the first load more efficient than with AppCache! Also the Service Worker's first cache does not do cache-busting, which allows all the resources requested so far to be returned from the HTTP cache rather than from the network.
...but **note** that if you do that with an AppCache fallback, the browser will start loading the AppCache, then disable it and throw it away when you register your SW :-\ also note [crbug.com/410665](https://crbug.com/410665)

### offline.js format

This is basically a JSON file with a list of static resources to cache and a version number. It looks like this:

```
{
	"version": 1,
	"fileList": [
		"info.txt",
		"image.png"
	],
	"lazyLoad": [
		"myPath/",
		"myOtherPath/"
	]
}
```

Note both the detected main page URL and the root `/` path are implicitly cached. For example if you visit `https://example.com/foo/index.html`, then both `index.html` and `/` (corresponding to `https://example.com/foo/`) are added to the cache without having to specify them in the file list.

Any requests made under any of the `lazyLoad` paths are not cached up-front, but are cached the first time they are fetched. This is useful for large optional files, such as templates or examples in an editor app.

To issue an update, update the files as normal then change the `"version"` field (which can be any arbitrary number, e.g. a timestamp). Note the version field does not need to actually increase, it only looks for a different version number to one it's seen before, however obviously this should be different for every update and not re-use old values.

Note the file is called `offline.js` by default instead of `offline.json`, since almost all servers have a MIME type set for .js, but not all have one set for .json. So if we named the file .json, in some cases the server will return 404 Not Found. If you want a .json extension just rename the file and change `OFFLINE_DATA_FILE` at the top of `sw.js`.

### BroadcastChannel messages

To find out about update events like "update ready", create a BroadcastChannel named "offline" and listen for messages like this:

```
// note: check BroadcastChannel is supported first
let broadcastChannel = new BroadcastChannel("offline");
broadcastChannel.onmessage = function (e)
{
	const data = e.data;
	const messageType = data.type;
	
	// messageType can be:
	// "downloading-update": has started downloading a new version in the background
	//     (data.version indicates which)
	// "update-ready": an update has finished downloading in the background and is
	//     now ready to use after a reload (data.version indicates which)
	// "update-pending": an update is available but needs a reload to start using it
	// "up-to-date": identified that the latest version is already in use
	// "downloading": started first-time caching of the current version for offline use
	// "offline-ready": finished first-time caching, so app can now work offline
};
```

There is a caveat with this: the SW can generate messages before the page has loaded enough to have created the BroadcastChannel to receive the message. To try to resolve this, there is a hack: the SW artificially delays all messages by 3 seconds to try to make sure the page has loaded enough to be listening. Then to make sure your page receives the message, you have to make sure early on in loading it loads a script which creates the BroadcastChannel and starts listening. Of course there's still no guarantee this will happen within 3 seconds. Ideally the SW would buffer messages itself until clients say they are ready for them. This is pretty tricky though (the SW APIs make this hard to design), so it's left as a TODO.


## Implementation notes / TODOs

- writing the cache is not technically atomic, although in practice it should be OK. Caching waits for all requests to complete successfully before opening the cache and writing everything in one go, so an incomplete cache lasts for as short a time as possible. However if the SW is for some reason terminated during that writing, an incomplete cache will be left behind and offline support will not work correctly. SW does not yet have the necessary features to do this writing atomically, but apparently there is some spec work being done on this. TODO: make cache writing atomic when possible.
- update-on-reload is hacky: Chrome reports 2 clients present when reloading, but Firefox reports 1, so we've had to resort to user agent string inspection to decide when to update.
- support for cache-busting fetches is hacked in with random query string parameters since Chrome does not yet support fetch cache control options ([crbug.com/453190](https://crbug.com/453190)). TODO: remove the query string hack when cache control options supported.
- BroadcastChannel messages are delayed by 3 seconds as a hack to try to make sure the client is listening. TODO: use a per-client message buffer and clients should tell the SW when they are ready to receive messages, guaranteeing clients receive all their messages with no unnecessary delay.
- this has not yet been tested against any SW implementations other than Chrome and Firefox. Notably Microsoft are working on an implementation in Edge which should be tested.

Service Workers still need spec work and browsers need various bug fixes/additions to fulfil the above list.

## License

MIT
