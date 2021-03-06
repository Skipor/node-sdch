# node-sdch

[![Build Status](https://travis-ci.org/baranov1ch/node-sdch.svg?branch=master)](https://travis-ci.org/baranov1ch/node-sdch)

SDCH encoder/decoder for node.js

Refer to [the spec](http://lists.w3.org/Archives/Public/ietf-http-wg/2008JulSep/att-0441/Shared_Dictionary_Compression_over_HTTP.pdf) for more information.

Keep in mind, that it is to accurate in all aspects. For instance:

* Chromium already supports SDCH-over-HTTPS as it is now considered to not
introduce additional risks.

* Chromium does not support comma separated port list. Use multiple headers.

* Chromium downloads only the first dictionary from `Get_Dictionary` header.

This package mimics Chromium behavior rather than follow the spec precisely,
since Chromium is the real consumer of SDCH, not the spec:)

## Quick overview.

Based on [node-vcdiff](http://github.com/baranov1ch/node-vcdiff). In a nutshell,
SDCH adds HTTP layer to VCDIFF compression:

```javascript
var sdch = require('sdch');

var dict = new sdch.SdchDictionary({
  domain: 'kotiki.cc',
  path: '/',
  data: 'Yo dawg I heard you like common substrings in your documents so we ' +
  'put them in your vcdiff dictionary so you can compress while you compress'
});

var testData =
  'Yo dawg I heard you like common substrings somewhere else so we put ' +
  'them in your vcdiff dictionary so you can decompress while you decompress'

var encoded = sdch.sdchEncodeSync(testData, dict);
var decoded = sdch.sdchDecodeSync(encoded, [ dict ]);

sdch.sdchEncode(testData, dict, function(err, enc) {
  sdch.sdchDecode(enc, [ dict ], function(err, dec) {
    assert(testData === dec.toString());
  });
});

var in = createInputStreamSomehow();
var out = createOutputStreamSomehow();
var encoder = sdch.createSdchEncoder(dict);
in.pipe(encoder).pipe(out);

var decoder = sdch.createSdchDecoder([dict]);
out.pipe(decoder).pipe(process.stdout);
```

You may want to use [connect-sdch](http://github.com/baranov1ch/connect-sdch) which provides all basic server-side stuff required to serve
sdch-encoded content. 

## Slow overview

HTTP Server may provide a
dictionary to the client, and the client may use it to decode server responses.
Dictionary in SDCH has to be associated with some domain, optionally path and
ports, and have some properties. These properties are prepended to a VCDIFF
dictionary in HTTP-header format:

```
domain: kotiki.cc
path: /
port: 80
port: 3000
max-age: 86400
```

When the client requests the server, it appends _client hashes_ of available
dictionaries (which the client may have downloaded later). The server chooses
the dictionary to decode with and proceeds. This is why `SdchDecoder` accepts
the list of dictionaries instead of a single one. Decoder do not know which
particular dictionary server whould choose, it will figure it out only when
parsing the response.

SDCH-encoded entity differs from VCDIFF encoded by dictionary _server hash_
appended in a front of vcdiff-encoded body. So SDCH encoder just prepends
this hash + `'\0'` and then streams VCDIFF-encoded data. The decoder parses
this hash and selects the dictionary from provided and decodes the data.

> Well-behaved SDCH client should check a lot of security stuff about the
> dictionaries proposed by the server, particularly scheme, domain, port,
> and path match. This package includes util functions to make all these
> checks (`sdch.clientUtils`). See how [connect-sdch](http://github.com/baranov1ch/connect-sdch) example client
> uses them to validate server provided dictionaries and to choose what
> to advertise. You may also refer to chromium code for more information.

Here is a quick example of how server and client hashes are created:

```javascript
var shasum = crypto.createHash('sha256');
shasum.update(/* concatenated SDCH headers + \n */);
shasum.update(/* vcdiff dictionary data */);
var hash = shasum.digest();

var clientHash = urlsafeEncode(hash.slice(0, 6).toString('base64'));
var serverHash = urlsafeEncode(hash.slice(6, 12).toString('base64'));
```

## API Reference

### Encoding/Decoding

All encoding/decoding functions accepts `options` parameter:
```javascript
sdchEncodeSync(input, dictionary, options)
sdchDecodeSync(input, dictionaries, options)

sdchEncode(input, dictionary, options, callback)
sdchDecode(input, dictionaries, options, callback)

createSdchEncoder(dictionary, options)
createSdchDecoder(dictionaries, options)
```
These options will be passed to underlying `vcdiff` module. For their meaning,
please refer to [node-vcdiff](http://github.com/baranov1ch/node-vcdiff) docs.
*Note*: you should not provide `dictionary` or `hashedDictionary`, it will be
provided by this module.

For decoder, 2 additional options are available:
* `url`, String. URL of the resource being decoded.

* `validationCallback`, Function. Will be used to check if the dictionary is
valid for decoding of the resource. If not provided, default implementation
(uses `clientUtil.canUseDictionary`) will be used

Validation callback is used only if `url` option is provided:

```javascript
sdch.sdchDecode(
  input,
  dictionaries,
  {
    url: 'http://resource.com/path',
    validationCallback: function (dict,           // selected dictionary 
                                  resourceUrl) {  // 'http://resource.com/path'
      if (...)
        return false;

      return true;
    }
  }, function (err, data) {
    if (err) {
      ....
    }
  });

```

### Dictionaries

#### Creation

You may create SDCH dictionary from buffer containing vcdiff dictionary data
(ones generated by femtozip, for instance) and a bunch of SDCH-related options
using `SdchDictionary` constructor:
```javascript
var dict = new sdch.SdchDictionary({
  domain: 'kotiki.cc',                          // String
  url: '/dicts/dict1',                          // String
  data = fs.readFileSync('path-to-your-dict'),  // Buffer or String
  path: '/somepath',                            // String
  formatVersion, '1.0',                         // String
  maxAge: 84600,                                // Int
  ports: [80, 443, 3000],                       // Array of Ints
});
```
Only `domain`, `url` and `data` are required, others are optional. The
constructor will throw if required args are missing or any arg has wrong type.

Sometimes you need to parse the dictionary from some source. For instance,
client will parse the dictionary from the server response. Or if you want to
serve dictionaries as a static resources using nginx, you will have to prepare
the files in advance (prepend SDCH headers before data). Then you may create
`options` from that data using `createDictionaryOptions(dictionaryURL, data)`.
`dictionaryURL` is the url where the dictionary is served. For the client it
should be valid web url, including at least scheme and domain parts, since
security checks hash to be performed against it. For the server any url will do
unless you use [connect-sdch](http://github.com/baranov1ch/node-vcdiff) to serve dictionaries. Then
you need to pass an absolute path from which the dictionary will be served.

Client:
```javascript
// Client just received dict from some URL
var dict;
try {
  var opts = sdch.createDictionaryOptions(
    dictUrl,        // String
    responseBody);  // String
  dict = sdch.clientUtils.createDictionaryFromOptions(opts);
} catch (e) {
  // Whoops... dictionary was invalid for some reason
  console.log(e.message);
}
```

Server:
```javascript
var dict;
try {
  var opts = sdch.createDictionaryOptions(
    dictUrl,                // String
    fs.readFileSync(...));  // String
  // On the server we may be sure that the dictionary is more or less correct.
  // So create it directly.
  dict = new sdch.SdchDictionary(opts);
} catch (e) {
  // Whoops... this time this may only mean syntax error in headers, missing
  // domain header or no url.
  console.log(e.message);
}
```

#### SdchDictionary

##### properties
The class has the following properties (optional params and thie derivatives
may be `undefined`):

* `url` - String. URL from which it was downloaded by the client or on which it
is served.

* `domain` - String. Only this domain and its subdomains are allowed to use
this dictionary

* `path` - String. This dictionary will be used for the specified path only or
for all subpaths also if path ends with `/`. For instance:
`/` matches all paths
`/path` matches only `/path`
`/path/` matches `/path/123`, `path/123/4`, but not `/path`

* `ports` - Array of Integers. Usage of the dictionary should be restricted by
provided ports.

* `data` - Buffer. VCDiff dictionary data.

* `hashedDict` - vcdiff.HashedDictionary instance for encoding.

* `formatVersion` - version. Chromium supports only `'1.0'`.

* `maxAge` - Integer. Dictionary lifetime in seconds.

* `expiration` - Date. Invalidation time for the dictionary. The client should
not use the dictionary if the current time is more than `expiration`.
`expiration` is set to the time of creation of `SdchDictionary` + `maxAge`.

* `clientHash` - String. See above.

* `serverHash` - String. See above.

* `etag` - String. Substring of the SHA256 checksum as well as `clientHash`
and `serverHash`.

##### methods

* `getLength()` returns SDCH dictionary length (SDCH headers + dict data).
May be used to create content-length header.

* `getOutputStream(opts)` returns dictionary content as a stream.
`opts` may include any options valid for `stream.Readable`. Also `opts` may
include 
`range: { start: startPos, end: endPos }`.
It will be used to stream only a part of the dict (useful for serving content
range). Please make sure you pass here a valid range for that dictionary. Use
some range parsers to validate it.
Multi-ranges are not supported as well.

### clientUtils

Set of functions, more or less copy-pasted from the chromium code to aid in
creation of SDCH clients to test SDCH servers.

#### basic client behavior:
* The client willing to accept sdch should advertise it in `Content-Encoding`
HTTP header:

```
Accept-Encoding: sdch,gzip,deflate
```

* When the client receives response, it should check `Get-Dictionary` header.
If it presents, the client may follow url from that header and download the
dictionary.

> *NOTE:* Use `canFetchDictionary` to determine if the client should really
> fetch that dictionary or smth is wrong.


* When the client downloads the dictionary, it parses it and stores somewhere

> *NOTE:* Use `createDictionaryOptions` to parse dictionary params from the
> response and `createDictionaryFromOptions` to check if the dictionary is valid
> and create it if so.

* The client may advertise dictionary to the server in the future requests to
that domain by appending its `clientHash` to `Avail-Dictionary` request header.
`Avail-Dictionary` contain a comma-separated list of client hashes of the
dictionaries, the client is willing to advertise to the server.
Chromium advertises all available dictioinaries for the domain (plus some
checks).

> *NOTE:* Use `canAdvertiseDictionary` to determine if the client should
> advertise the dictionary to the server.

* If the server chooses to SDCH-encode the resource, it will append `sdch` to
`Content-Encoding` header. The server may also compress the response with gzip
or deflate, do the header will be smth like:
```
Content-Encoding: sdch, gzip
```
The client should first decompress the response by gzip/deflate and then pass
it do sdch decoder.

> *NOTE:* There is a lot of mess in that place. Some proxies in the wild tend to
> mess with CE header, so that it may be just `sdch` or `gzip` even if the
> resource id SDCH'ed and gzipped or just SDCH'ed or just gzipped or... you got
> the idea:) so Chromium tries to perform gzip and sdch decoding for every
> resource it has advertised the dictionary, despite the values of the
> `Content-Encoding` header, and falls back if fails. Since we're not writing the
> browser and just want to test our SDCH-server directly, we may skip all that
> magic and trust the header.

If the server decided not to SDCH-encode the response even if the client has
advertised some valid dictionaries, it adds
```
X-SDCH-Encode: 0
```
header to the response. The client should be prepared to handle it.

> *NOTE:* after the client has parsed the dictionary server hash from the
> response it may use `canUseDictionary` to check if dictionary is valid to use.

##### `createDictionaryFromOptions(opts)`

Creates dictionary from provided options, parsed from the dictionary using
`createDictionaryOptions` if they seem valid. Otherwise throws Error.
Chromium performs this checks after it downloads the dictionary. The dictionary
will not be created if:

 * `opts.domain` is `null` or empty string

 * `opts.domain` specifies TLD (via `tldjs` package)

 * `opts.url` hostname is not `opts.domain` or if `opts.domain` starts with `.`
and `opts.url` hostname does not belong to `opts.domain`. Theoretically both
of these mean just not belonging to the domain:)

 * if `opts.ports` provided but does not include `opts.url` port. No port means
 port 80 for http and 443 for https, naturally.

 * if `opts.formatVersion is provided but does not equal to `'1.0'` (this is
what chromium does).

##### `canUseDictionary(dictionary, referringUrl)`

Determines if the client can really use the dictionary for decoding the
resource (from `referringUrl`). This check is performed by the client
(Chromium) when it parses _server hash_ from the first 9 bytes of the response
body and need to verify that dictionary suggested by the server is valid for
that resource.
The function return `false` if:

 * `referringUrl` domain does not match `dictionary.domain`

 * `dictionary.ports` is present but `referringUrl` port does not match them

 * `dictionary.path` is present but `referringUrl` path does not match it (see
description of the `path` field).

 * if `dictionary.url` scheme does not match `referringUrl` one. Simply
speaking, you should not use non-secure dictionaries for secure resources and
vice versa.

##### `canAdvertiseDictionary(dictionary, targetUrl)`

Determines if the client should advertise that `dictionary` is available when
it requests `targetUrl`.
The function return `false` if:

* `targetUrl` domain does not match `dictionary.domain`

* `dictionary.ports` is present but `targetUrl` port does not match them

* if `dictionary.url` scheme does not match `targetUrl` one.

* dictionary is expired (`current time > dictionary.expiration`).

##### `canFetchDictionary(dictionaryUrl, referringUrl)`

Determines if the client should fetch the dictionary located on `dictionaryUrl`
advertised by the server in the `referringUrl` response.
The function return `false` if:

* `referringUrl` and `dictionaryUrl` hash different schemes (http vs https)

* `referringUrl` does not equal to `dictionaryUrl`. Note, that this is exact
match.

* schemes are not `http` or `https`. This is just additional check which exists
in Chromium but not specified in anywhere.
