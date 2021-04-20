+++
title = "CloudFront, S3 and CORS"
+++

Found a great cache-poisoning DOS issue with our CloudFront/S3 setup.

## TL;DR:

S3 doesn’t follow W3 guidelines with the `Vary` header, be careful when doing CORS with S3/CloudFront to avoid non-CORS requests hosing subsequent CORS requests for the same resources.

## Background

The services I work on publish objects to S3 that are used to provide both user configuration and social/analytics data to our customers. The data is a mixture of
user-defined configuration that's published on-demand by user edits & explicit "publish" actions, and data we fetch from the core APIs, which is automatically refreshed on a schedule.

For quite some time we've occasionally seen an issue upon launching a freshly created display page where the browser logs some CORS/network request failures to the console that usually clears up
after a few minutes, and wouldn't be reproduceable on a second computer loading the same resource. I'd initially assumed this was caused by some subtle race condition in which the browser was
requesting the resource from CloudFront before it was fully available on S3, as a 404/403 from S3 to CloudFront can also show up in the browser as a CORS failure in some cases, and it always
got better after we'd looked at it for a minute or two. Once it recovered it stayed working, so some sort of timing issue with publishing felt right.

## The Problem

CloudFront, by default, does not cache based on HTTP headers. For CORS to work, the client must indicate it's interested in CORS headers being added to the response.
When doing a GET request, this is done by including an `Origin` header on the request to show what domain is initiating the response. The server can then decide if
it wants to honor the CORS request of the client for the specified domain, and will respond with something like `Access-Control-Allow-Origin: *`. Our CloudFront settings
were already allowing the necessary headers through from the client to S3, so in the simple case of a client requesting a given resource, the headers were passed through
to S3 and the response included the expected response headers.

S3 does not send `Vary: Origin` even when CORS is enabled on the bucket, which is dumb and weird as it would just solve this by instructing CloudFront (and browsers, ect)
to always treat differences in the `Origin` header as cache-invalidating, i.e. part of the effective cache key.

### request/response flows
{% mermaid() %}
sequenceDiagram
participant P as Publisher
participant S3
participant CF as CloudFront
participant C1 as Client1
P-->>S3: Publish Object
C1->>CF: GET Object (w/Origin header)
loop Check Cache
	rect rgb(220,140,140)
	CF->>CF: Cache Miss!
	end
end
CF->>S3: GET Object (w/Origin header)
S3-->>CF: RESP Object (w/CORS headers)
loop Cache Put
	rect rgb(150,150,230)
	CF-->>CF: Cache Object (w/CORS headers)
	end
end
CF-->>C1: RESP Object (w/CORS headers)
Note right of C1: Happy client, got its<br/>CORS headers!
{% end %}

Here everything has worked as expected, so CORS is working and everyone's happy.

But what happens if we have two clients, one asking for CORS headers and another one NOT asking for them?
Well, the answer depends on who asks first!

First up, a CORS-enabled client asks for the resource with an Origin header and gets CORS in the response.
Then a non-CORS-enabled client asks for the same resource without an Origin header, and this is a cache hit in CloudFront, so the same response
which includes CORS headers is sent to the second client. It doesn't care about these headers and just ignores them, this causes no problems
for either client, and everything is fine.
{% mermaid() %}
sequenceDiagram
participant P as Publisher
participant S3
participant CF as CloudFront
participant C1 as Client1
participant C2 as Client2
P-->>S3: Publish Object
C1->>CF: GET Object (w/Origin header)
loop Check Cache
	rect rgb(220,140,140)
	CF->>CF: Cache Miss!
	end
end
CF->>S3: GET Object (w/Origin header)
S3-->>CF: RESP Object (w/CORS headers)
loop Cache Put
	rect rgb(150,150,230)
	CF-->>CF: Cache Object (w/CORS headers)
	end
end
CF-->>C1: RESP Object (w/CORS headers)
Note right of C1: Happy client, got its<br/>CORS headers!

C2->>CF: GET Object (no CORS)
loop Check Cache
	rect rgb(140,220,140)
	CF-->>CF: Cache Hit! Object (w/CORS headers)
	end
end
CF-->>C2: RESP Object (w/CORS)
Note right of C2: Happy client, ignores<br/>CORS headers.
{% end %}

But if the order of requests is changed and the non-CORS request occurs first, things are very different. In this case, the non-CORS client still
has everything it needs, but the client expecting CORS doesn't get back the headers it needs and the user agent will refuse to load the content.

{% mermaid() %}
sequenceDiagram
participant Publisher
participant S3
participant CloudFront
participant Client1
participant Client2
Publisher-->>S3: Publish Object
Client1->>CloudFront: GET Object (no CORS)
loop Check Cache
	rect rgb(220,140,140)
	CloudFront->>CloudFront: Cache Miss!
	end
end
CloudFront->>S3: GET Object (no CORS)
S3-->>CloudFront: RESP Object (no CORS)
loop Cache Put
	rect rgb(150,150,230)
	CloudFront-->>CloudFront: Cache Object (no CORS headers)
	end
end
CloudFront-->>Client1: RESP Object (no CORS)
Note right of Client1: Happy client, doesn't<br/>need CORS headers 

Client2->>CloudFront: GET Object (w/Origin header)
loop Check Cache
	rect rgb(140,220,140)
	CloudFront-->>CloudFront: Cache Hit! Object (no CORS headers)
	end
end
CloudFront-->>Client2: RESP Object (no CORS)
Note right of Client2: SAD client, didn't<br/>get CORS headers!
{% end %}

Further complicating matters, if a good request followed a subsequent publish to the same object in S3, the newly-cached version in CloudFront would now have the CORS headers on it.

The heart of the problem here is that CloudFront isn't considering the client's inclusion of the `Origin` header to be relevant to the cached object, even though it _includes_ this header in the request to S3!

## Subtleties

This is all particularly insidious for a couple reasons.
1. CloudFront is passing all the right headers on to S3, so any single request for a given resource works correctly. This means you can test out your setup and everything looks just fine, as it usually works as expected!
2. If S3 is configured to respond to `Origin` headers with `Access-Control-Allow-Origin: *`, the same cached object will work for all domains specified in an incoming `Origin` header.
3. As soon as the publisher writes an updated object to S3, it invalidates the cache. So it resets the conditions, and the first client to make a new request for the object gets its request honored.
  Our publisher automatically writes objects on a schedule, so someone encountering this issue loading up the requested resource in the browser is likely to get "lucky" and make the winning request
  after a publish, so no issue is seen.

## The solution

We have to make CloudFront treat headers as relevant parts of the request for the purpose of establishing the cache key. This way multiple requests for the same resource will each get response headers that correspond to their request headers.

The relevant configuration is in the CloudFront Behaviors section, under "Cache Based on Selected Request Headers", select the "Whitelist Headers" option and choose, at a minimum, `Origin`. If `OPTIONS` is enabled under "Allowed HTTP Methods", `Access-Control-Request-Headers` and `Access-Control-Request-Method` should also be enabled so that `OPTIONS` responses get cached appropriately - these are used for CORS preflighting, which is more relevant to font and image `GET` requests, and `POST` requests, but might as well for completeness.


## Example request/response using curl

```sh
OBJECT_URL='https://example.com/6NWUzMjlmYjFkNzZkZmI2ODM1YzEwZWU5/data.json?Expires=1582239185&Signature=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX&Key-Pair-Id=APKAIXXXXXXXXXXXXXXX'

# First, make request without Origin header.
# If this is the FIRST request to hit CloudFront after most-recent write to the
# corresponding S3 object, there will be no CORS headers returned
curl -o /dev/null -D - \
	-H 'accept: application/json, text/plain, */*' \
	"$OBJECT_URL" --compressed
					
content-type: application/json
content-length: 2705
date: Fri, 14 Feb 2020 19:49:52 GMT
last-modified: Fri, 14 Feb 2020 19:49:23 GMT
cache-control: max-age=0
content-encoding: gzip
accept-ranges: bytes
server: AmazonS3
x-cache: RefreshHit from cloudfront

# Subsequent request with Origin header will have the same CORS response headers.
# If the previous request without Origin header was first to this resource, this
# means we fail to provide CORS headers.
# Either way, the headers will be the same on this request.
curl -o /dev/null -D - \
	-H 'accept: application/json, text/plain, */*' \
	-H 'origin: https://example.com' \
	"$OBJECT_URL" --compressed

content-type: application/json
content-length: 2705
date: Fri, 14 Feb 2020 19:49:53 GMT
last-modified: Fri, 14 Feb 2020 19:49:23 GMT
cache-control: max-age=0
content-encoding: gzip
accept-ranges: bytes
server: AmazonS3
x-cache: RefreshHit from cloudfront

# With the CloudFront settings updated to use the Origin header as part of the
# cache key, this same request will include CORS headers in the response:
curl -o /dev/null -D - \
	-H 'accept: application/json, text/plain, */*' \
	-H 'origin: https://example.com' \
	"$OBJECT_URL" --compressed

content-type: application/json
content-length: 2705
date: Fri, 14 Feb 2020 19:49:53 GMT
access-control-allow-origin: *
access-control-allow-methods: GET, PUT
access-control-max-age: 3000
last-modified: Fri, 14 Feb 2020 19:49:23 GMT
cache-control: max-age=0
content-encoding: gzip
accept-ranges: bytes
server: AmazonS3
vary: Origin,Access-Control-Request-Headers,Access-Control-Request-Method
x-cache: RefreshHit from cloudfront

```

## References

<https://forums.aws.amazon.com/thread.jspa?threadID=156134>
<https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/header-caching.html#header-caching-web-cors>
<https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS>
<https://fetch.spec.whatwg.org/#http-access-control-request-headers>
