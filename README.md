# Cloudflare Worker Content Security Policy Nonce Generator
A Cloudflare worker to generate and inject Content Security Policy nonces for HTML pages.

### Motivation
HTTP [Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) is an added layer of security that helps to detect and mitigate certain types of attacks, including [Cross Site Scripting (XSS)](https://developer.mozilla.org/en-US/docs/Glossary/XSS) and data injection attacks. These attacks are used for everything from data theft to site defacement to distribution of malware.

CSP level 1 uses the concept of `whitelist` of domains or entire script paths, however this technique is being flagged as isecure, and there's even a [paper](https://research.google/pubs/pub45542/) around this.

CSP level 2 introduced the concept of `digests` or `nonces` which would allow developers to whitelist using `sha256` checksums or a uniquely randomly generated nonce. The issue is that when we include external `stylesheets` and `scripts` that we don't control (such as Google Analytics, etc) we can't use `sha-256` directives, hence the need of nonces.

CSP level 3 introduces the concept of [`strict-dynamic`](https://content-security-policy.com/strict-dynamic/) which enlargely helps developers to deploy secure websites without the burden of injecting `nonces` everywhere.

### Description

Here, at Move Your Digital, our sites are served from a static hosting and that colides with the uniquely generated nonce technique we need. However, if you use Cloudflare you can use this script to overcome this problem.

This small script is a [Cloudflare Worker](https://workers.cloudflare.com/) and must be deployed as one single worker. Once attached to a route, say `https://example.com/*`, all requests matching `/*` will be intercepted and a completely new and unique `nonce` is generated using the [Web Crypto](https://developers.cloudflare.com/workers/reference/apis/web-crypto/) library for each request.

#### Flow
```
Request -> Cloudflare -> Worker

Worker:
-------
1. Request Origin (or Cloudflare cache)
2. Generate a new nonce
3. Replace new nonce in HTTP Response

Worker -> Cloudflare -> Response
```

### Usage

1. Create a new [Cloudflare Worker](https://workers.cloudflare.com/)
2. Deploy `nonce-worker.js`
3. Within `Workers` tab of domain dashboard [attach routes](https://developers.cloudflare.com/workers/about/routes/)
4. Always respond with the static `DhcnhD3khTMePgXw` string (which is the default string which will be replaced by a new nonce). Change it if needed.

### How does it work by example

#### Example of response from origin
```
HTTP 200 OK
Content-Type: text/html
Content-Security-Policy: default-src 'none'; script-src 'self' 'nonce-DhcnhD3khTMePgXw' 'strict-dynamic';
Cache-Control: no-store

<html>
<head>
  <link rel="stylesheet" href="/styles.css" nonce="DhcnhD3khTMePgXw" />
  <script nonce="DhcnhD3khTMePgXw">
    alert("I'm alive!");
  </script>
</head>
</html>
```

#### Response after Cloudflare worker
```
HTTP 200 OK
Content-Type: text/html
Content-Security-Policy: default-src 'none'; script-src 'self' 'nonce-MWM2ODViMWI3MQ/Yzk5NzJhMzQ2OWQxMTk=' 'strict-dynamic';
Cache-Control: no-cache, max-age=0

<html>
<head>
  <link rel="stylesheet" href="/styles.css" nonce="MWM2ODViMWI3MQ/Yzk5NzJhMzQ2OWQxMTk=" />
  <script nonce="MWM2ODViMWI3MQ/Yzk5NzJhMzQ2OWQxMTk=">
    alert("I'm alive!");
  </script>
</head>
</html>
```

Notice two changes:
- The default `nonce` string is replaced by a randomly generated one
- `Cache-Control` directive was `no-store` and replaced by `no-cache, max-age=0` (can be changed by tweaking `DEFAULT_CACHE_CONTROL` constant)

### Known issues and troubleshooting

#### Never use `304 Not Modified` responses
Unfortunately, one downside of this technique is that you can't respond with 304 code since the nonce must be always regenerated and match both HTTP header and body response.

However, you can cache on browser and change `max-age` directive to something abover 0.

If you absolutely need to respond with `304` you'll need to maintain status within this script. You could use [Cloudflare KV](https://developers.cloudflare.com/workers/reference/apis/kv/) that saves a pair of `__cfduid` cookie value and the generated nonce.

A different approach would be to inject the nonce in the cookie, which still cares proper security validations and we don't recommed it yet.

However, if you find youself solving this issue, please consider to open a pull request here or simply let us know :-)

#### Rocket Loader®

The new [Cloudflare Rocket Loader®](https://blog.cloudflare.com/we-have-lift-off-rocket-loader-ga-is-mobile/) feature injects/replaces itself with a different script. Since we can't whitelist domains using `strict-dynamic` Google Chrome blocks this request and we were unable to find the reason. Sadly, we had to disable Rocket Loader for these websites.

We're working on a solution to overcome this issue.


### Considerations

#### Limits of Workers free plan

As the time of this writing, the limit is 100k/mo. However this can change. If you need more, consider using a paying plan which in fact is very cheap.

### Contributing

You can contribute for this repo. Please open an issue or a pull request.