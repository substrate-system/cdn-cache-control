# cdn cache control
![tests](https://github.com/substrate-system/cdn-cache-control/actions/workflows/nodejs.yml/badge.svg)
[![types](https://img.shields.io/npm/types/@substrate-system/icons?style=flat-square)](README.md)
[![module](https://img.shields.io/badge/module-ESM%2FCJS-blue?style=flat-square)](README.md)
[![semantic versioning](https://img.shields.io/badge/semver-2.0.0-blue?logo=semver&style=flat-square)](https://semver.org/)
[![Common Changelog](https://nichoth.github.io/badge/common-changelog.svg)](./CHANGELOG.md)
[![install size](https://flat.badgen.net/packagephobia/install/@substrate-system/cdn-cache-control)](https://packagephobia.com/result?p=@substrate-system/cdn-cache-control)
[![dependencies](https://img.shields.io/badge/dependencies-zero-brightgreen.svg?style=flat-square)](package.json)
[![license](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square)](LICENSE)

<p align="center">
  <img alt="cdn-cache-control" src="https://github.com/user-attachments/assets/62464d67-cbd5-48b5-a36d-800678ae22bb" />
</p>

Easy, opinionated CDN cache header handling.

Modern CDNs allow very fine-grained control over the cache. This is particularly
useful for server-side rendering of web content, as it allows you to manually
handle the invalidation of content, ensuring it stays fast and fresh. This
package provides a subclass of the `Headers` class that makes it easier to set
cache control headers for content served through a modern CDN. It provides a
simple, chainable API with sensible defaults for common use cases. It works by
setting the `Cache-Control` and `CDN-Cache-Control` headers to the appropriate
values. If run on a supported platform it will use the more specific header for
that CDN. e.g. on Netlify it will use the `Netlify-CDN-Cache-Control` header.

e.g.

```js
// Expires in 1 minute, but use stale-while-revalidate to
// serve stale content after that
const headers = new CacheHeaders().ttl(ONE_MINUTE).swr();
```

<details><summary><h2>Contents</h2></summary>

<!-- toc -->

- [fork](#fork)
- [Installation](#installation)
- [Usage](#usage)
  * [Use cases](#use-cases)
- [API](#api)
- [:wrench: Constants](#wrench-constants)
  * [:gear: ONE_MINUTE](#gear-one_minute)
  * [:gear: ONE_HOUR](#gear-one_hour)
  * [:gear: ONE_DAY](#gear-one_day)
  * [:gear: ONE_WEEK](#gear-one_week)
  * [:gear: ONE_YEAR](#gear-one_year)
- [:factory: CacheHeaders](#factory-cacheheaders)
  * [Methods](#methods)

<!-- tocstop -->

</details>

## fork
This is a fork of [ascorbic/cdn-cache-control](https://github.com/ascorbic/cdn-cache-control).


## Installation

```bash
npm i -S cdn-cache-control
```

## Usage

The module exports a single class, `CacheHeaders`, which is a subclass of the
fetch [`Headers`](https://developer.mozilla.org/en-US/docs/Web/API/Headers)
class. It provides a chainable API for setting cache headers. By default
it sets the `Cache-Control` and `CDN-Cache-Control` headers to sensible defaults
for content that should be cached by the CDN and revalidated by the browser.

It can be instantiated with a `HeadersInit` value, which lets you base it on an
existing `Headers` object, or an object or array with existing header values. In
that case it will default to using existing `s-maxage` directives if present.

You can pass a `cdn` value as the second argument to set the CDN cache control
header. In some cases this will enable targeted cache header names. e.g. for
Netlify it will use the `Netlify-CDN-Cache-Control` header. Currently supported
values are `netlify`, `vercel`, `cloudflare` and `akamai`. If you don't pass a
value, or pass an unsupported one it will use the generic `CDN-Cache-Control`
header. It will also attempt to detect the platform automatically on
Vercel and Netlify.

### Use cases

If you have content that you want to have the CDN cache until it is manually
revalidated or purged with a new deploy, you can use the default values:

```js
import { CacheHeaders } from "cdn-cache-control";

const headers = new CacheHeaders();
```

This sets the `CDN-Cache-Control` header to
`public,s-maxage=31536000,must-revalidate`, which tells the CDN to cache the
content for a year.

It sets `Cache-Control` to `public,max-age=0,must-revalidate`,
which tells the browser to always check with
the CDN for a fresh version.

You should combine this with an `ETag` or `Last-Modified` header to allow
the CDN to serve a `304 Not Modified` response when the content hasn't changed.

#### stale-while-revalidate

You can enable `stale-while-revalidate` with the `swr` method, optionally
passing a value for the time to serve stale content (defaults to one week):

```js
import { CacheHeaders } from "cdn-cache-control";

const headers = new CacheHeaders().swr();
```

This tells the CDN to serve stale content while revalidating the content in the
background. Combine with the `ttl` method to set the time for which the content
will be considered fresh (default is zero, meaning the CDN will
always revalidate):

```js
import { CacheHeaders, ONE_HOUR } from "cdn-cache-control";

const headers = new CacheHeaders().swr().ttl(ONE_HOUR);
```

#### Immutable content

If you are serving content that is guaranteed to never change then you can set
it as immutable. You should only do this for responses with unique URLs, because
there will be no way to invalidate it from the browser cache if it ever changes.

```js
import { CacheHeaders } from "cdn-cache-control";
const headers = new CacheHeaders().immutable();
```

This will set the CDN and browser caches to expire in 1 year, and add the
immutable directive.

#### Cache tags

Some CDNs support the use of cache tags, which allow you to purge content from
the cache in bulk. The `tag()` function makes it simple to add tags. You can
call it with a string or array of strings.

```js
import { CacheHeaders } from "cdn-cache-control";
const headers = new CacheHeaders().tag(["blog", "blog:1"]);
```

You can then purge the tagged items from the cache using the CDN API. e.g. for
Netlify the API is:

```ts
import { purgeCache } from "@netlify/functions";

export default async function handler(req: Request) => {
  await purgeCache({
    tags: ["blog", "blog:1", "blog:2"],
  });
  return new Response("Purged!", { status: 202 })
};

```

#### Using the generated headers

The headers object can be used anywhere that accepts a `fetch` `Headers` object.
This includes most serverless hosts. It can also be used directly in many
framework SSR functions. Some APIs need a plain object rather than a `Headers`
object. For these you can use the `toObject()` method, which returns a plain
object with the header names and values.

```ts
import { CacheHeaders } from "cdn-cache-control"

export default async function handler(request:Request):Promise<Response> {
  const headers = new CacheHeaders().swr()
  // The `Response` constructor accepts the object directly
  return new Response('Hello', { headers })
}
```

Some frameworks use a readonly `Response` object, so you need to use an
existing `headers` object. In this case you can use the `copyTo` method to
copy the headers to the response:

```astro
---
import { CacheHeaders, ONE_HOUR } from "cdn-cache-control";

new CacheHeaders().swr(ONE_HOUR).copyTo(Astro.response.headers);
---

```

## API

<!-- TSDOC_START -->

## :wrench: Constants

- [ONE_MINUTE](#gear-one_minute)
- [ONE_HOUR](#gear-one_hour)
- [ONE_DAY](#gear-one_day)
- [ONE_WEEK](#gear-one_week)
- [ONE_YEAR](#gear-one_year)

### :gear: ONE_MINUTE

Number of seconds in one minute

| Constant     | Type |
| ------------ | ---- |
| `ONE_MINUTE` | `60` |

### :gear: ONE_HOUR

Number of seconds in one hour

| Constant   | Type   |
| ---------- | ------ |
| `ONE_HOUR` | `3600` |

### :gear: ONE_DAY

Number of seconds in one day

| Constant  | Type    |
| --------- | ------- |
| `ONE_DAY` | `86400` |

### :gear: ONE_WEEK

Number of seconds in one week

| Constant   | Type     |
| ---------- | -------- |
| `ONE_WEEK` | `604800` |

### :gear: ONE_YEAR

Number of seconds in one year

| Constant   | Type       |
| ---------- | ---------- |
| `ONE_YEAR` | `31536000` |

## :factory: CacheHeaders

### Methods

- [Installation](#installation)
- [Usage](#usage)
	- [Use cases](#use-cases)
		- [stale-while-revalidate](#stale-while-revalidate)
		- [Immutable content](#immutable-content)
		- [Cache tags](#cache-tags)
		- [Using the generated headers](#using-the-generated-headers)
- [API](#api)
- [:wrench: Constants](#wrench-constants)
	- [:gear: ONE\_MINUTE](#gear-one_minute)
	- [:gear: ONE\_HOUR](#gear-one_hour)
	- [:gear: ONE\_DAY](#gear-one_day)
	- [:gear: ONE\_WEEK](#gear-one_week)
	- [:gear: ONE\_YEAR](#gear-one_year)
- [:factory: CacheHeaders](#factory-cacheheaders)
	- [Methods](#methods)
		- [:gear: tag](#gear-tag)
		- [:gear: swr](#gear-swr)
		- [:gear: immutable](#gear-immutable)
		- [:gear: ttl](#gear-ttl)
		- [:gear: toObject](#gear-toobject)
		- [:gear: copyTo](#gear-copyto)
		- [:gear: getCdnCacheControl](#gear-getcdncachecontrol)
		- [:gear: setCdnCacheControl](#gear-setcdncachecontrol)
		- [:gear: getCacheControl](#gear-getcachecontrol)
		- [:gear: setCacheControl](#gear-setcachecontrol)
		- [:gear: getCacheTags](#gear-getcachetags)
		- [:gear: setCacheTags](#gear-setcachetags)

#### :gear: tag

Adds a cache tag to the cache tags header. Cache tags are used to invalidate the cache for a URL.

| Method | Type                                                   |
| ------ | ------------------------------------------------------ |
| `tag`  | `(tag: string or string[], ...tags: string[]) => this` |

Parameters:

- `tag`: The cache tag to add. Can be a string or an array of strings.

#### :gear: swr

Sets stale-while-revalidate directive for the CDN cache. By default the browser is sent a must-revalidate
directive to ensure that the browser always revalidates the cache with the server.

| Method | Type                       |
| ------ | -------------------------- |
| `swr`  | `(value?: number) => this` |

Parameters:

- `value`: The number of seconds to set the stale-while-revalidate directive to. Defaults to 1 week.

#### :gear: immutable

Sets cache headers for content that should be cached for a long time and never revalidated.
The CDN cache will cache the content for the specified time, and the browser will cache the content
indefinitely without revalidating. Do not use this unless the URL is fingerprinted or otherwise unique.
Otherwise, the browser will cache the content indefinitely and never check for updates, including for new deploys.

| Method      | Type                       |
| ----------- | -------------------------- |
| `immutable` | `(value?: number) => this` |

Parameters:

- `value`: The number of seconds to set the CDN cache-control s-maxage directive to. Defaults to 1 year.

#### :gear: ttl

Sets the s-maxage for items in the CDN cache. This is the maximum amount of time that the CDN will cache the content.
If used with swr, the content will revalidate in the background after the max age has passed. Otherwise, the content will be
removed from the cache after the max age has passed.

| Method | Type                      |
| ------ | ------------------------- |
| `ttl`  | `(value: number) => this` |

#### :gear: toObject

Returns the headers as a plain object.

| Method     | Type                           |
| ---------- | ------------------------------ |
| `toObject` | `() => Record<string, string>` |

#### :gear: copyTo

Copy the headers from this instance to another Headers instance.

| Method   | Type                                   |
| -------- | -------------------------------------- |
| `copyTo` | `<T extends Headers>(headers: T) => T` |

#### :gear: getCdnCacheControl

The parsed cache-control header for the CDN cache.

| Method               | Type                           |
| -------------------- | ------------------------------ |
| `getCdnCacheControl` | `() => Record<string, string>` |

#### :gear: setCdnCacheControl

| Method               | Type                                           |
| -------------------- | ---------------------------------------------- |
| `setCdnCacheControl` | `(directives: Record<string, string>) => void` |

#### :gear: getCacheControl

The parsed cache-control header for the browser cache.

| Method            | Type                           |
| ----------------- | ------------------------------ |
| `getCacheControl` | `() => Record<string, string>` |

#### :gear: setCacheControl

| Method            | Type                                           |
| ----------------- | ---------------------------------------------- |
| `setCacheControl` | `(directives: Record<string, string>) => void` |

#### :gear: getCacheTags

The parsed content of the cache tags header.

| Method         | Type             |
| -------------- | ---------------- |
| `getCacheTags` | `() => string[]` |

#### :gear: setCacheTags

| Method         | Type                       |
| -------------- | -------------------------- |
| `setCacheTags` | `(tags: string[]) => void` |

<!-- TSDOC_END -->
