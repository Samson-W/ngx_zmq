# ngx_zmq

An upstream ZeroMQ module for nginx.  It offers the following features:
* Endpoint configuration
* Per location timeout configuration
* Connection pooling

The following features and improvements are planned and in development
* Selection of ZeroMQ socket type
  * Currently defaults to REQ
  * Would like to be able to select PUB or PUSH
* Code cleanup
* Memory profiling/cleanup

## Adding to Nginx
Complation of the module only requires libzmq 3.x+ (as well as nginx).  It has been tested to build against nginx 1.5.12 and 1.7.4 (OpenResty versions). They can be compiled in simply with
```bash
./confugre --add-module=<path_to_ngx_zmq>
``` 
from the nginx source directory, followed by your standard `make` and `make install` commands.


## nginx Configuration
ngx_zmq uses location blocks to set up endpoints, and provides the following directives:
```nginx
location /zmq {
  zmq;
  zmq_endpoint "tcp://localhost:5555"; #endpoint is required
  zmq_timeout 10; #in milliseconds, total time spent in function. optional, defaults to -1 (no timeout)
}
```
Assuming a zmq server (example included in us.cpp) is running, a request over ZMQ can be made via
```bash
$curl -X POST -d 'what_to_send' http://localhost/zmq
```

The sample upstream server can be compiled and run with:
```bash
$g++ us.cpp -lzmq; ./a.out
```
The sample server will always reply with "World"

### With the lua-nginx-module
[As per this issue](https://github.com/openresty/lua-nginx-module/issues/415), we cannot just do a ngx.location.capture() from the lua nginx module.  This requires a bit of a workaround.  Try:
```nginx
location ~ /zmq/proxy/(?<target>[\S]+) {
internal;
proxy_pass http://localhost/zmq/$1;
}

location /zmq/my_endpoint/ {
internal;
zmq_endpoint "tcp://my.endpoint.org:5555";
zmq_timeout 10;
zmq;
}
```
and in Lua:
```lua
local reply, err = readurl.capture(
"/zmq/proxy/my_endpoint/",
{body=thing_to_send},
false,
{failure_log_level=ngx.CRIT, counter_dict=requests_counter}
)
```
Using appropriate options.  You could use other request frameworks as well (one such is mentioned in the above linked issue).

## Error handling
ngx_zmq will give the following codes/request bodies under the specified conditions:

| HTTP Code | Request Body | Condition|
|-----------|--------------|----------|
| Bad Gateway (502) | <empty> | send times out|
| Gateway Timeout (504) | <empty> | recv times out|
| Bad Gateway (502) | ZeroMQ error message | rev fails, but does not timeout (errno != EAGAIN)|
| OK (200) | Response from ZMQ server | successful round-trip (from ZMQ_REQ socket|

## Reference Materials
There does not seem to be a single good set of documentation on writing nginx modules.  The definitive guide linked below is great, but frequently errors happened I wouldn't know how to find.  A list of some of the resources I used:

[Search nginx source](http://lxr.nginx.org/ident)

[simple module](http://www.nginxguts.com/2011/01/how-to-return-a-simple-page/)

[Evan's Definitive guide](http://www.evanmiller.org/nginx-modules-guide.html)

[agentzh explaining nginx](http://openresty.org/download/agentzh-nginx-tutorials-en.html)
