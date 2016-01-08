![Hoverfly](static/images/hf-logo-std-r-transparent-medium.png)
## Dependencies without the sting

[Hoverfly](http://hoverfly.io) is a lightweight, open source [service virtualization](https://en.wikipedia.org/wiki/Service_virtualization) tool.
Using Hoverfly, you can virtualize your application dependencies to create a self-contained development or test environment.

Hoverfly is a proxy written in [Go](https://github.com/golang/go). It can capture HTTP(s) traffic between an application under test
and external services, and then replace the external services. Another powerful feature: middleware modules, where users
can introduce their own custom logic. **Middleware modules can be written in any language**.

More information about Hoverfly and how to use it:
* https://www.specto.io/speeding-up-your-slow-dependencies/
* https://www.specto.io/service-virtualization-modifying-traffiic/
* https://www.specto.io/api-mocking-for-dev-and-e2e-testing-from-zero-to-hero/

## Installation

### Build it yourself  
This project uses the [Glide](https://github.com/Masterminds/glide) project to manage dependencies in combination with Git [submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules), and therefore you must have Go > 1.5 installed, and the ['Go 1.5 Vendor Experiment'](https://docs.google.com/document/d/1Bz5-UB7g2uPBdOx-rw5t9MxJwkfpx90cqG9AFL0JAYo/edit) flag enabled:

    export GO15VENDOREXPERIMENT=1

Use [Glide](https://github.com/Masterminds/glide) to fetch the dependencies (or you can also use _git submodule init_) with:

    glide up

As this project uses Glide to manage dependencies please note that you must clone the Hoverfly repository within your $GOPATH (you can read more about on how [Glide works](https://github.com/Masterminds/glide#user-content-how-it-works)), or else you may see strange errors like "_cannot find package "github.com/Sirupsen/logrus" in any of: XXX_"

Then build Hoverfly:

    go build

And run it:

    ./hoverfly

### Pre-built binary

Pre-built Hoverfly binaries are available [here](https://github.com/SpectoLabs/hoverfly/releases/).
You may find it easier to download a binary - however since the Hoverfly admin UI requires static files you will need
to clone the Hoverfly repo first, and then copy the binary to the Hoverfly directory before executing it.

## Admin UI

The Hoverfly admin UI is available at [http://localhost:8888/](http://localhost:8888/). It uses the API
(as described below) to change state. It also allows you to wipe the captured requests/responses and shows the number
of captured records. For other functions, such as export/import, you can use the API directly.

## Hoverfly is a proxy

Configuring your application to use Hoverfly is simple. All you have to do is set the HTTP_PROXY environment variable:

     export HTTP_PROXY=http://localhost:8500/

## Destination configuration

You can specify which site to capture or virtualize with a regular expression (by default, Hoverfly processes everything):

    ./hoverfly --destination="."

## Modes (Virtualize / Capture / Synthesize / Modify)

Hoverfly has different operating modes. Each mode changes the behavior of the proxy. Based on the selected mode, Hoverfly can
either capture the requests and responses, look for them in the cache, or send them directly to the middleware and
respond with a payload that is generated by the middleware (more on middleware below).

### Virtualize

By default, the proxy starts in virtualize mode. You can apply middleware to each response.

### Capture

When capture mode is active, Hoverfly acts as a "man-in-the-middle". It makes requests on behalf of a client and records
the responses. The response is then sent back to the original client.

To switch to capture mode, you can add the "--capture" flag during startup:

    ./hoverfly --capture

Or you can select "Capture" using the admin UI at [http://localhost:8888/](http://localhost:8888/)

Or you can use API call to change proxy state while running (see API below).

Do a curl request with proxy details:

    curl http://mirage.readthedocs.org --proxy http://localhost:8500/

###  Synthesize

Hoverfly can create responses to requests on the fly. Synthesize mode intercepts requests (it also respects the --destination flag)
and applies the supplied middleware (the user is required to supply --middleware flag, you can read more about it below). The middleware
is expected to populate response payload, so Hoverfly can then reconstruct it and return it to the client. An example of a synthetic
service can be found in _this_repo/examples/middleware/synthetic_service/synthetic.py_. You can test it by running:

    ./hoverfly --synthesize --middleware "./examples/middleware/synthetic_service/synthetic.py"

### Modify

Modify mode applies the selected middleware (the user is required to supply the --middleware flag - you can read more about this below)
to outgoing requests and incoming responses (before they are returned to the client). It
allows users to use Hoverfly without storing any data about requests. It can be used when it's difficult to - or there is little point
in - describing a service, and you only need to change certain parts of the requests or responses (eg: adding authentication headers through with middleware).
The example below changes the destination host to "mirage.readthedocs.org" and sets the method to "GET":

    ./hoverfly --modify --middleware "./examples/middleware/modify_request/modify_request.py

## HTTPS capture

Add ca.pem to your trusted certificates or turn off verification. With curl you can make insecure requests with -k:

    curl https://www.bbc.co.uk --proxy http://localhost:8500 -k


## API

You can access the administrator API under default port 8888:

* Recorded requests: GET http://proxy_hostname:8888/records ( __curl http://proxy_hostname:8888/records__ )
* Wipe cache: DELETE http://proxy_hostname:8888/records ( __curl -X DELETE http://proxy_hostname:8888/records__ )
* Get current proxy state: GET http://proxy_hostname:8888/state ( __curl http://proxy_hostname:8888/state__ )
* Set proxy state: POST http://proxy_hostname:8888/state, where
   + body to start virtualizing: {"mode":"virtualize"}
   + body to start capturing: {"mode":"capture"}
* Exporting recorded requests to a file: __curl http://proxy_hostname:8888/records > requests.json__
* Importing requests from file: __curl --data "@/path/to/requests.json" http://localhost:8888/records__


## Middleware

Hoverfly supports external middleware modules. You can write them in __any language you want__ !.
These middleware modules are expected to take standard input (stdin) and should output a structured JSON string to stdout.
Payload example:

```javascript
{
	"response": {
		"status": 200,
		"body": "body here",
		"headers": {
			"Content-Type": ["text/html"],
			"Date": ["Tue, 01 Dec 2015 16:49:08 GMT"],
		}
	},
	"request": {
		"path": "/",
		"method": "GET",
		"destination": "1stalphaomega.readthedocs.org",
		"query": ""
	},
	"id": "5d4f6b1d9f7c2407f78e4d4e211ec769"
}
```
Middleware is executed only when request is matched so for fully dynamic responses where you are
generating response on the fly you can use ""--synthesize" flag.

In order to use your middleware, just add path to executable:

    ./hoverfly --middleware "./examples/middleware/modify_response/modify_response.py"

#### Python middleware example

Basic example of a Python module to change response body and add 2 second delay:

```python
#!/usr/bin/env python
import sys
import logging
import json
from time import sleep


logging.basicConfig(filename='middleware.log', level=logging.DEBUG)
logging.debug('Middleware is called')


def main():
    data = sys.stdin.readlines()
    # this is a json string in one line so we are interested in that one line
    payload = data[0]
    logging.debug(payload)

    payload_dict = json.loads(payload)

    payload_dict['response']['status'] = 201
    payload_dict['response']['body'] = "body was replaced by middleware"

    # now let' sleep for 2 seconds
    sleep(2)

    # returning new payload
    print(json.dumps(payload_dict))

if __name__ == "__main__":
    main()

```

Save this file with python extension, _chmod +x_ it and run hoverfly:

    ./hoverfly --middleware "./this_file.py"

#### JavaScript middleware example

You can also execute JavaScript middleware using Node. Make sure that you can execute Node, you can brew install it on OSX.

Below is an example how to take data in, parse JSON, modify it and then encode it back to string and return:

```javascript
#!/usr/bin/env node

process.stdin.resume();  
process.stdin.setEncoding('utf8');  
process.stdin.on('data', function(data) {
  var parsed_json = JSON.parse(data);
  // changing response
  parsed_json.response.status = 201;
  parsed_json.response.body = "body was replaced by JavaScript middleware\n";

  // stringifying JSON response
  var newJsonString = JSON.stringify(parsed_json);

  process.stdout.write(newJsonString);
});

```

You see, it's really easy to use it to create a synthetic service to simulate backend when you are working on the frontend side :)


### How middleware interacts with different modes

Each mode is affected by middleware in a different way. Since the JSON payload has request and response structures, some middleware
 will not change anything. Some information about middleware behaviour:
  * __Capture Mode__: middleware affects only outgoing requests.
  * __Virtualize Mode__: middleware affects only responses (cache contents remain untouched).
  * __Synthesize Mode__: middleware creates responses.
  * __Modify Mode__: middleware affects requests and responses.



## Debugging

You can supply "-v" flag to enable verbose logging.


## License

Apache License version 2.0 [See LICENSE for details](https://github.com/SpectoLabs/hoverfly/blob/master/LICENSE).

(c) [SpectoLabs](https://specto.io) 2015.
