# mrequests

An HTTP client module for MicroPython with an API *similar* to [requests].

This is an evolution of the [urequests] module from [micropython-lib] with a few
extensions and many fixes and convenience features.


## Features & Limitations


### Compatibility

Supports many MicroPython ports as well as CPython.

| Port    | Tested on                                     | Notes                                                                                                                        |
| ------- | --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| esp32   | GOOUUU-ESP32 (ESP-WROOM-32)                   |                                                                                                                              |
| esp32s3 | Waveshare ESP32-S3-Zero                       |                                                                                                                              |
| esp32c3 | Seeed Xiao-ESP32-C3                           |                                                                                                                              |
| rp2     | Raspberry Pi Pico W                           |
| esp8266 | LoLin NodeMcu v3 (ESP8266MOD)                 | Limited SSL/TLS support due to problems with ports `ssl` module                                                              |
| stm32   | STM32F407VET6 / WIZNET W5500 Ethernet adapter | (Custom) firmware with network/SSL support enabled required. Limited SSL/TLS support due to problems with ports `ssl` module |
| unix    | Arch Linux                                    | Limited SSL/TLS support due to problems with ports `ssl` module                                                              |


### Features

* Supports SSL/TLS with server certificate validation (as far as supported by
  the `ssl` module of a given MicroPython port).
* Supports redirection with absolute and relative URLs (see below for details).
* Supports HTTP basic authentication (requires `binascii` module).
* Supports socket timeouts.
* Response headers can optionally be saved in the response object.
* Respects `Content-length` header in response.
* Supports responses with chunked transfer encoding.
* `Response` objects have a `readinto` method to store the response body to a
  given buffer (or `memoryview`) in chunks of maximum `len(buffer)` size.
* `Response` objects have `save` and `saveto` methods to save the response
  body to a file (given by filename resp. file object), reading the response
  data and writing the file in small chunks.
* The `Response` class for response objects can be substituted by a custom
  response class (usually defined by subclassing `Response`).


### Limitations

* `mrequests.request` is a synchroneous, blocking function.
* The code is *not* interrupt save and a fair amount of memory allocation is
  happening in the process of handling a request.
* URL parsing does not cover all corner cases (see [test_urlparse] for details).
* URLs with authentication credentials in the host part (e.g.
  `http://user:secret@myhost/`) are *not supported*. Pass authentication
  credentials separately via the `auth` argument instead.
* SSL/TLS support on the MicroPython `unix`, `stm32` and `esp8266` ports is
  limited. In particular, their `ssl` module does not support all encryption
  schemes commonly in use by popular servers, meaning that trying to connect
  to them via HTTPS will fail with various cryptic error messages.

    On the `esp8266` port no TLS server certificate validation is performed.
* Request and JSON data may be passed in as `bytes` or strings and the request
  data will be encoded to `bytes`, if necessary, using the encoding given with
  the `encoding` parameter. But be aware that encodings other than `utf-8` are
  *not supported* by most (any?) MicroPython implementations.
* Custom headers may be passed as a dictionary with `bytes` keys and values
  and keys and values must contain only ASCII chars. If you need header values
  to use non-ASCII chars, you need to encode them according to RFC 8187.

    The header dictionary *may* contain also string keys and values, but:

    * Using MicroPython, it causes `Warning: Comparison between bytes and str`
      to be printed to the standard error output when calling `request`.
    * It causes additional memory allocations.
    * The "Host" header (if present) must always use `b"Host"` as the header
      dictionary key, i.e. it must be of type `bytes` and use this exact
      capitalization (unless the headers dictionary uses an implementation with
      case-insensitive key lookup).
* The URL and specifically any query string parameters it contains will not be
  URL-encoded, and it may contain only ASCII chars. Make sure you encode the
  query string part of the URL with `urlencode.quote` before passing it, if
  necessary.
* When encoding `str` instances via `urlencode.urlencode` or `urlencode.quote`,
  the `encoding` and `errors` arguments are currently ignored by MicroPython and
  it behaves as if their values were `"utf-8"` resp. `"ignore"`.
* In responses using "chunked" transfer-encoding, chunk extensions and trailers
  are ignored.


### Redirection Support

* Can follow redirects for response status codes 301, 302, 303, 307 and 308.
* The HTTP method is changed to `GET` for redirects, unless the original
  method was `HEAD` or the status code is 307 or 308.
* For status code 303, if the method of the request resulting in a redirection
  (which may have been the result of a previous redirection) is `GET`, the
  redirection is not followed, since the `Location` header is supposed to
  indicate a non-HTTP resource then.
* Redirects are allowed to change the protocol from `http` to `https`,
  but redirects changing from `https` to `http` will not be followed.
* The `request` function has an additional keyword argument `max_redirects`,
  defaulting to 1, which controls how many levels of redirections are followed.
  If this is exceeded, the function raises a `ValueError`.
* The code does not check for infinite redirection cycles. It is advised to
  keep `max_redirects` to a low number instead.


## Installation

While there are multiple ways to install the library from your PC's command
line, three different installation methods are provided via different scripts:

* `install_mpremote.sh`(Bash script using [mpremote] binary)
* `install.sh` (Bash script using [rshell] binary, legacy)
* `install.py` (Python script using `mpremote` as a library)


### `install_mpremote.sh`

The following should be installed and in your shell's `PATH`:

* `mpy-cross`
* `mpremote`

Run the command:

```con
./install_mpremote.sh
```

This will compile the Python modules with `mpy-cross` and copy the resulting
`.mpy` files to the board's flash using the `mpremote` command.


### `install.sh`

The following should be installed and in your shell's `PATH`:

* `mpy-cross`
* `rshell`

For boards with the `stm32` port run:

```con
DESTDIR=/flash ./install.sh
```

For boards with the `esp8266` or `esp32` port run:

```con
DESTDIR=/pyboard PORT=/dev/ttyUSB0 BAUD=115200 ./install.sh
```

This will compile the Python modules with `mpy-cross` and copy the resulting
`.mpy` files to the board's flash using the `rshell` comamnd.


### `install.py`

The following should be installed and in your shell's `PATH`:

* `mpy-cross`

Also, `mpremote` should be available on your `PYTHONPATH`, e.g. installed via
`pip`:

```con
python -m pip install mpremote
```

Then simply run `install.py`:

```con
./install.py
```

This will compile the Python modules with `mpy-cross` and copy the resulting
`.mpy` files to the board's flash using the `mpremote` library. Run
`install.py -h` to review installation options.


### Manual installation

For the `unix` port, just copy the `mrequest` directory with all files in it
to a directory, which is in `sys.path`, e.g. `~/.micropython/lib`, or set the
`MICROPYPATH` environment variable to a colon-separated list of directories
including the one to which you copied the package directory.

Note: the `mrequests/mrequests.py` module has no dependencies besides modules
usually already built in to the MicroPython firmware on all ports (as of
version >= 1.15) and can be installed and used on its own if flash storage
space or available RAM is scarce. The other modules in the `mrequests` package
provide support for specific tasks like, for example, sending form-encoded
request parameters or data or parsing url-encoded strings (see the scripts in
the `examples` directory for examples of their use).


## Examples

Below are some basic usage examples of the `mrequests` module. More examples
for special tasks can be found in the scripts in the [examples](./examples)
directory. Some of these examples require extra modules from [micropython-lib].
To install these requirements to a MicroPython board you can use [mpremote]:

```con
mpremote mip install collections-defaultdict
```


### Simple GET request with JSON response

```py
>>> import mrequests as requests
>>> r = requests.get("http://httpbin.org/get",
                     headers={"Accept": "application/json"})
>>> print(r)
<Response object at 7f6f91631be0>
>>> print(r.content)
b'{\n  "args": {}, \n  "headers": {\n    "Accept": "application/json", \n
"Host": "httpbin.org", \n    "X-Amzn-Trace-Id": "Root=1-[redacted]"\n  }, \n
"origin": "[redacted]", \n  "url": "http://httpbin.org/get"\n}\n'
>>> print(r.text)
{
  "args": {},
  "headers": {
    "Accept": "application/json",
    "Host": "httpbin.org",
    "X-Amzn-Trace-Id": "Root=1-[redacted]"
  },
  "origin": "[redacted]",
  "url": "http://httpbin.org/get"
}

>>> print(r.json())
{'url': 'http://httpbin.org/get', 'headers': {'X-Amzn-Trace-Id':
'Root=1-[redacted]', 'Host': 'httpbin.org', 'Accept': 'application/json'},
'args': {}, 'origin': '[redacted]'}
>>> r.close()
```

It is mandatory to close response objects as soon as you finished working with
them. On MicroPython platforms without full-fledged OS, not doing so may lead
to resource leaks and malfunction.


### HTTP Basic Auth

```py
>>> import mrequests as requests
>>> user = "joedoe"
>>> password = "simsalabim"
>>> url = "http://httpbin.org/basic-auth/%s/%s" % (user, password)
>>> r = requests.get(url, auth=(user, password))
>>> print(r.text)
{
  "authenticated": true,
  "user": "joedoe"
}
>>> r.close()
```


## Reference

The main function provided by `mrequests` is `request`, which takes an HTTP
method and a URL as positional arguments and several optional keyword arguments
and returns a `Response` object:

```py
request(method, url, data=None, json=None, headers={}, auth=None,
        encoding=None, response_class=Response, save_headers=False,
        max_redirects=1, timeout=None, ssl_context=None)
```

Parameters:

*method (str)* - the HTTP request method as a string in all-caps.

*url (str)* - the URL of the request as a string. This must be an absolute URL,
including the protocol (only `http://` and `https://` are supported) and a host
name or IP address. The server name may be suffixed with a port number,
separated by a colon. The default port is 80 for `http` URLs and 443 for
`https`.

Authentication credentials in the host part are not supported (see the *auth*
parameter below).

The URL may contain only ASCII chars. The caller is responsible for encoding
any non-ASCII chars in the path or the query string (for example with
`urlencode.quote`) or encoding IDN host names if necessary.

*data (bytes, str)* - the request body data as `bytes` or `str` instance. If
given as a string, it will be converted to `bytes` using the encoding given
with the `encoding` parameter (requiring additional memory to allocate the new
bytes object). The caller is responsible for formatting the request data
according to the content type specified in the request headers.

*json (obj)* - an object, which will be encoded as JSON and sent as the request
body data. Also adds a `Content-Type` header with the value `application/json`.
This overwrites data passed with the *data* parameter and allocates memory for
the request data. To avoid allocation, pass a JSON-encoded byte string with the
*data* parameter instead.

*headers (dict)* - a dictionary of additional request headers to sent. Keys and
values should be `bytes` instances and may contain only ASCII chars. They *may*
be `str` instances, but see "Limitations". `str` keys and values will be
converted to `bytes` using ASCII encoding, which causes memory allocation.

A `Content-Length` header will always be added, using the length of the request
data as the value. If no `b"Host"` key is present in the header dictionary, the
`Host` header value will be generated based on the host name from the URL.

*auth (tuple)* - HTTP basic authentication credentials given as a
`(user, password)` tuple of `bytes` objects or a callable, returning such a
tuple. Some MicroPython versions may also accept `str` instances as the tuple
elements but may issue a warning message. This will overwrite `Authorization`
headers passed in with the *headers* parameter.

*encoding (str)* - the encoding of the request body data, if the the data is
passed in as a `str` instance with the *data* or *json* parameters. Defauts to
`utf-8`.

*response_class (obj)* - the class to use for the returned response objects.
Defaults to `mrequests.Response`. Custom response classes should sub-class
`mrequests.Response` and must take the same constructor arguments.

*save_headers (bool)* - a boolean, which is passed to the constructor of the
response class instance, which determines whether it keeps a reference to the
response headers in the instance. This is set to `False` by default to save
memory. If set to `True`, the default response class will make the reponse
header lines available via its `headers` instance attribute as a list of
unparsed `bytes` objects.

*max_redirects (int)* - the maximum number of valid redirections to follow.
Defaults to 1. If too many redirections are encountered, a `ValueError` is
raised.

*timeout (float)* - sets the timeout for the connection socket as a
non-negative float in seconds. Defaults to `None`, which blocks indefinitely.
If a non-zero value is given, connection attempts or socket read/write
operations will raise `OSError` if the timeout period value has elapsed before
the operation has completed. If zero is given, the socket is put in
non-blocking mode.

*ssl_context (ssl.SSLContext)* - pass a custom `ssl.SSLContext` instance to
configure TLS encryption and certificate handling. If performing an HTTPS
request and no SSL context is passed, a default one is created, which allows
an encrypted connection, but does not require the server to present a valid
certificate. If the MicroPython port has proper SSL support, it is strongly
recommended to pass your own SSL context instance, which has its `verify_mode`
attribute set to `ssl.CERT_REQUIRED` and the required certificates loaded.
See the documentation of the [ssl module] for details.

---

Several convenience wrappers for creating request using common HTTP methods are
available:

```py
head(url, **kw)

get(url, **kw)

post(url, **kw)

put(url, **kw)

patch(url, **kw)

delete(url, **kw)
```

The url and all keyword arguments are simply passed to `request`.


## Authors

**mrequests** is based on [urequests], written by *Paul Sokolovsky* and
part of [micropython-lib] and licensed under the [MIT license]. It was further
developed and is maintained by *Christopher Arndt*.


## License

**mrequests** is distributed under the terms of the [MIT license] and is free
and Open Source software.

Please see the file [LICENSE](./LICENSE) for details.

[micropython-lib]: https://github.com/micropython/micropython-lib
[mit license]: http://opensource.org/licenses/MIT
[mpremote]: https://docs.micropython.org/en/latest/reference/mpremote.html
[requests]: https://github.com/psf/requests
[rshell]: https://pypi.org/project/rshell/
[ssl module]: https://docs.micropython.org/en/latest/library/ssl.html#class-sslcontext
[test_urlparse]: ./tests/test_urlparse.py
[urequests]: https://github.com/micropython/micropython-lib/blob/master/urequests/urequests.py
