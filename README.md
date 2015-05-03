# BrowserMob Proxy

BrowserMob Proxy is a simple utility that makes it easy to capture performance data from browsers, typically written using automation toolkits such as Selenium and Watir.

Version 2.1 is currently in development, and contains a number of improvements over 2.0. We highly recommend that you use a 2.1 beta version instead of the previous 2.0.0 release.

To build the current development version, see the [build instructions](https://github.com/lightbody/browsermob-proxy#creating-the-batch-files-from-source).

The previous version of BrowserMob Proxy is 2.0.0 -- see the [2.0.x README](https://github.com/lightbody/browsermob-proxy/tree/2.0) for usage information.

## Important Changes since 2.0.0

Since the 2.1 release is still in beta, some features and functionality (including the BrowserMobProxy interface) may change, although the new interface is largely stable. The most important changes are:

- [Separate REST API and Embedded Mode modules](#embedded-mode). Include only the functionality you need.
- [New BrowserMobProxy interface](https://github.com/lightbody/browsermob-proxy/blob/master/browsermob-core/src/main/java/net/lightbody/bmp/BrowserMobProxy.java). The new interface will completely replace the legacy 2.0 ProxyServer contract in version 3.0 and higher.
- [LittleProxy support](#littleproxy-support). More powerful than the legacy Jetty back-end. For 2.1 releases, LittleProxy support will be provided through the browsermob-core-littleproxy module.

See the [New Interface Compatibility Guide](new-interface-compatibility.md) for information on using the new BrowserMobProxy interface with the legacy ProxyServer implementation.

### New BrowserMobProxy API

BrowserMob Proxy 2.1 includes a [new BrowserMobProxy interface](https://github.com/lightbody/browsermob-proxy/blob/master/browsermob-core/src/main/java/net/lightbody/bmp/BrowserMobProxy.java) to interact with BrowserMob Proxy programmatically. The new interface defines the functionality that BrowserMob Proxy will support in future releases (including 3.0+).

### Deprecated LegacyProxyServer Interface

The legacy interface, implicitly defined by the ProxyServer class, has been extracted into `net.lightbody.bmp.proxy.LegacyProxyServer` and is now officially deprecated. The existing ProxyServer implementation will continue to implement the LegacyProxyServer interface, and the new LittleProxy-based implementation will also implement LegacyProxyServer for all 2.1.x releases. LegacyProxyServer will not be supported after 3.0 is released.

**All users are highly encouraged to use the `net.lightbody.bmp.BrowserMobProxy` interface for all new code.** The new interface provides additional functionality and is compatible with both the legacy Jetty-based ProxyServer implementation [(with some exceptions)](new-interface-compatibility.md) and the new LittleProxy implementation. Using the new interface will greatly increase the likelihood of a smooth transition to 3.0, when the legacy ProxyServer implementation will not be supported.

### LittleProxy Support

BrowserMob Proxy now supports using LittleProxy instead of Jetty 5 + Apache HTTP Client. To enable LittleProxy support, include the `browsermob-core-littleproxy` artifact:
```xml
    <dependency>
        <groupId>net.lightbody.bmp</groupId>
        <artifactId>browsermob-core-littleproxy</artifactId>
        <version>2.1.0-beta-1-SNAPSHOT</version>
        <scope>test</scope>
    </dependency>
```

Instead of creating a `ProxyServer` instance, create a `BrowserMobProxyServer` instance:
```java
    BrowserMobProxy proxy = new BrowserMobProxyServer();
    proxy.start();
```

## Features and Usage

The proxy is programmatically controlled via a REST interface or by being embedded directly inside Java-based programs and unit tests. It captures performance data in the [HAR format](http://groups.google.com/group/http-archive-specification). In addition it can actually control HTTP traffic, such as:

 - blacklisting and whitelisting certain URL patterns
 - simulating various bandwidth and latency
 - remapping DNS lookups
 - flushing DNS caching
 - controlling DNS and request timeouts
 - automatic BASIC authorization

### REST API

**New in 2.1:** The REST API now supports LittleProxy. When running browsermob-proxy, specify `--use-littleproxy true` to enable LittleProxy support.

To get started, first start the proxy by running `browsermob-proxy` or `browsermob-proxy.bat` in the bin directory:

    $ sh browsermob-proxy -port 9090 --use-littleproxy true
    INFO 05/31 03:12:48 o.b.p.Main           - Starting up...
    2011-05-30 20:12:49.517:INFO::jetty-7.3.0.v20110203
    2011-05-30 20:12:49.689:INFO::started o.e.j.s.ServletContextHandler{/,null}
    2011-05-30 20:12:49.820:INFO::Started SelectChannelConnector@0.0.0.0:9090

Once started, there won't be an actual proxy running until you create a new proxy. You can do this by POSTing to /proxy:

    [~]$ curl -X POST http://localhost:9090/proxy
    {"port":9091}

or optionally specify your own port:

    [~]$ curl -X POST -d 'port=9099' http://localhost:9090/proxy
    {"port":9099}

or if running BrowserMob Proxy in a multi-homed environment, specify a desired bind address (default is `0.0.0.0`):

    [~]$ curl -X POST -d 'bindAddress=192.168.1.222' http://localhost:9090/proxy
    {"port":9096}

Once that is done, a new proxy will be available on the port returned. All you have to do is point a browser to that proxy on that port and you should be able to browse the internet. The following additional APIs will then be available:

 - GET /proxy - get a list of ports attached to `ProxyServer` instances managed by `ProxyManager`
 - PUT /proxy/[port]/har - creates a new HAR attached to the proxy and returns the HAR content if there was a previous HAR. Supports the following parameters:
  - initialPageRef - the string name of the first page ref that should be used in the HAR. Defaults to "Page 1".
  - initialPageTitle - the title of first HAR page. Defaults to initialPageRef.
  - captureHeaders - Boolean, capture headers
  - captureContent - Boolean, capture content bodies
  - captureBinaryContent - Boolean, capture binary content
 - PUT /proxy/[port]/har/pageRef - starts a new page on the existing HAR. Supports the following parameters:
  - pageRef - the string name of the first page ref that should be used in the HAR. Defaults to "Page N" where N is the next page number.
  - pageTitle - the title of new har page. Defaults to pageRef.
 - DELETE /proxy/[port] - shuts down the proxy and closes the port
 - GET /proxy/[port]/har - returns the JSON/HAR content representing all the HTTP traffic passed through the proxy
 - GET /proxy/[port]/whitelist - Displays whitelisted items
 - PUT /proxy/[port]/whitelist - Sets a list of URL patterns to whitelist. Takes the following parameters:
  - regex - a comma separated list of regular expressions
  - status - the HTTP status code to return for URLs that do not match the whitelist
 - DELETE /proxy/[port]/whitelist - Clears all URL patterns from the whitelist 
 - GET /proxy/[port]/blacklist - Displays blacklisted items
 - PUT /proxy/[port]/blacklist - Set a URL to blacklist. Takes the following parameters:
  - regex - the blacklist regular expression
  - status - the HTTP status code to return for URLs that are blacklisted
  - method - regular expression for matching method., e.g., POST. Emtpy for matching all method.
 - DELETE /proxy/[port]/blacklist - Clears all URL patterns from the blacklist
 - PUT /proxy/[port]/limit - Limit the bandwidth through the proxy. Takes the following parameters:
  - downstreamKbps - Sets the downstream bandwidth limit in kbps
  - upstreamKbps - Sets the upstream bandwidth limit kbps
  - downstreamMaxKB - Specifies how many kilobytes in total the client is allowed to download through the proxy.
  - upstreamMaxKB - Specifies how many kilobytes in total the client is allowed to upload through the proxy.
  - latency - Add the given latency to each HTTP request
  - enable - (true/false) a boolean that enable bandwidth limiter. By default the limit is disabled, although setting any of the properties above will implicitly enable throttling
  - payloadPercentage - a number ]0, 100] specifying what percentage of data sent is payload. e.g. use this to take into account overhead due to tcp/ip.
  - maxBitsPerSecond - The max bits per seconds you want this instance of StreamManager to respect.
 - GET /proxy/[port]/limit - Displays the amount of data remaining to be uploaded/downloaded until the limit is reached.
 - POST /proxy/[port]/headers - Set and override HTTP Request headers. For example setting a custom User-Agent.
  - Payload data should be json encoded set of headers (not url-encoded)
 - POST /proxy/[port]/hosts - Overrides normal DNS lookups and remaps the given hosts with the associated IP address
  - Payload data should be json encoded set of name/value pairs (ex: {"example.com": "1.2.3.4"})
 - POST /proxy/[port]/auth/basic/[domain] - Sets automatic basic authentication for the specified domain
  - Payload data should be json encoded username and password name/value pairs (ex: {"username": "myUsername", "password": "myPassword"}
 - PUT /proxy/[port]/wait - wait till all request are being made
  - quietPeriodInMs - Sets quiet period in milliseconds
  - timeoutInMs - Sets timeout in milliseconds 
 - PUT /proxy/[port]/timeout - Handles different proxy timeouts. Takes the following parameters:
  - requestTimeout - request timeout in milliseconds. A timeout value of -1 is interpreted as infinite timeout. It equals -1 by default.
  - readTimeout - read timeout in milliseconds. Which is the timeout for waiting for data or, put differently, a maximum period inactivity between two consecutive data packets). A timeout value of zero is interpreted as an infinite timeout. It equals 60000 by default
  - connectionTimeout - Determines the timeout in milliseconds until a connection is established. A timeout value of zero is interpreted as an infinite timeout. It eqauls 60000 by default
  - dnsCacheTimeout - Sets the maximum length of time that records will be stored in this Cache. A nonpositive value disables this feature (that is, sets no limit). It equals 0 y default
 - PUT /proxy/[port]/rewrite - Redirecting URL's
  - matchRegex - a matching URL regular expression
  - replace - replacement URL
 - DELETE /proxy/[port]/rewrite - Removes all URL redirection rules currently in effect
 - PUT /proxy/[port]/retry - Setting the retry count
  - retrycount - the number of times a method will be retried
 - DELETE /proxy/[port]/dns/cache - Empties the Cache.

For example, once you've started the proxy you can create a new HAR to start recording data like so:

    [~]$ curl -X PUT -d 'initialPageRef=Foo' http://localhost:8080/proxy/9091/har

Now when traffic goes through port 9091 it will be attached to a page reference named "Foo". Consult the HAR specification for more info on what a "pageRef" is. You can also start a new pageRef like so:

    [~]$ curl -X PUT -d 'pageRef=Bar' http://localhost:8080/proxy/9091/har/pageRef

That will ensure no more HTTP requests get attached to the old pageRef (Foo) and start getting attached to the new pageRef (Bar). You can also get the HAR content at any time like so:

    [~]$ curl http://localhost:8080/proxy/9091/har

Sometimes you will want to route requests through an upstream proxy server. In this case specify your proxy server by adding the httpProxy parameter to your create proxy request:

    [~]$ curl -X POST http://localhost:9090/proxy?httpProxy=yourproxyserver.com:8080
    {"port":9091}

Alternatively, you can specify the upstream proxy config for all proxies created using the standard JVM [system properties for HTTP proxies](http://docs.oracle.com/javase/6/docs/technotes/guides/net/proxies.html).
Note that you can still override the default upstream proxy via the POST payload, but if you omit the payload the JVM
system properties will be used to specify the upstream proxy.

*TODO*: Other REST APIs supporting all the BrowserMob Proxy features will be added soon.

### Command-line Arguments

 - -port \<port\>
  - Port on which the API listens. Default value is 8080.
 - -address <address>
  - Address to which the API is bound. Default value is 0.0.0.0.
 - -proxyPortRange \<from\>-\<to\>
  - Range of ports reserved for proxies. Only applies if *port* parameter is not supplied in the POST request. Default values are \<port\>+1 to \<port\>+500+1.
 - -ttl \<seconds\>
  - Proxy will be automatically deleted after a specified time period. Off by default.

### Embedded Mode

**New in 2.1:** New Embedded Mode module

**New in 2.1:** New [BrowserMobProxy interface](#new-browsermobproxy-api) for Embedded Mode

BrowserMob Proxy 2.1 separates the Embedded Mode and REST API into two modules. If you only need Embedded Mode functionality, add the `browsermob-core` artifact (or `browsermob-core.jar` file) as a dependency. The REST API artifact is `browsermob-rest`.

If you're using Java and Selenium, the easiest way to get started is to embed the project directly in your test. First, you'll need to make sure that all the dependencies are imported in to the project. You can find them in the *lib* directory. Or, if you're using Maven, you can add this to your pom:
```xml
    <dependency>
        <groupId>net.lightbody.bmp</groupId>
        <artifactId>browsermob-core</artifactId>
        <version>2.1.0-beta-1-SNAPSHOT</version>
        <scope>test</scope>
    </dependency>
```

Once done, you can start a proxy using `net.lightbody.bmp.BrowserMobProxy`:
```java
    BrowserMobProxy server = new ProxyServer();
    server.start(0);
    // get the JVM-assigned port and get to work!
    int port = server.getPort();
    //...
    
```

Consult the Javadocs on the `net.lightbody.bmp.BrowserMobProxy` class for the full API.

### Using With Selenium

You can use the REST API with Selenium however you want. But if you're writing your tests in Java and using Selenium 2, this is the easiest way to use it:
```java
    // start the proxy
    BrowserMobProxy server = new ProxyServer();
    server.start(0);

    // get the Selenium proxy object
    Proxy proxy = ClientUtil.createSeleniumProxy(server);

    // configure it as a desired capability
    DesiredCapabilities capabilities = new DesiredCapabilities();
    capabilities.setCapability(CapabilityType.PROXY, proxy);

    // start the browser up
    WebDriver driver = new FirefoxDriver(capabilities);

    // create a new HAR with the label "yahoo.com"
    server.newHar("yahoo.com");

    // open yahoo.com
    driver.get("http://yahoo.com");

    // get the HAR data
    Har har = server.getHar();
```

### HTTP Request Manipulation

**HTTP request manipulation is changing with LittleProxy.** The LittleProxy-based interceptors are easier to use and more reliable. The legacy ProxyServer implementation **will not** support the new interceptor methods.

#### LittleProxy interceptors

There are four new methods to support request and response interception in LittleProxy:

  - `addRequestFilter`
  - `addResponseFilter`
  - `addFirstHttpFilterFactory`
  - `addLastHttpFilterFactory`

For most use cases, including inspecting and modifying requests/responses, `addRequestFilter` and `addResponseFilter` will be sufficient. The request and response filters are easy to use:
```java
	proxy.addRequestFilter(new RequestFilter() {
            @Override
            public HttpResponse filterRequest(HttpRequest request, HttpMessageContents contents, HttpRequest originalRequest) {
                if (request.getUri().equals("/some-endpoint-to-intercept")) {
                    // retrieve the existing message contents as a String or, for binary contents, as a byte[]
                    String messageContents = contents.getTextContents();

                    // do some manipulation of the contents
                    String newContents = messageContents.replaceAll("original-string", "my-modified-string");
                    //[...]

                    // replace the existing content by calling setTextContents() or setBinaryContents()
                    contents.setTextContents(newContents);
                }

                // in the request filter, you can return an HttpResponse object to "short-circuit" the request
                return null;
            }
        });
        
        // responses are equally as simple:
        proxy.addResponseFilter(new ResponseFilter() {
            @Override
            public void filterResponse(HttpResponse response, HttpMessageContents contents, HttpRequest originalRequest) {
                if (/*...some filtering criteria...*/) {
                    contents.setTextContents("This message body will appear in all responses!");
                }
            }
        });
```

With Java 8, the syntax is even more concise:
```java
        proxy.addResponseFilter((response, contents, originalRequest) -> {
            if (/*...some filtering criteria...*/) {
                contents.setTextContents("This message body will appear in all responses!");
            }
        });
```

See the javadoc for the `RequestFilter` and `ResponseFilter` classes for more information.

For fine-grained control over the request and response lifecycle, you can add "filter factories" directly using `addFirstHttpFilterFactory` and `addLastHttpFilterFactory` (see the examples in the InterceptorTest unit tests).

#### REST API interceptors with LittleProxy

When running the REST API with LittleProxy enabled, you cannot use the legacy `/:port/interceptor/` endpoints. Instead, POST the javascript payload to the new `/:port/filter/request` and `/:port/filter/response` endpoints.

##### Request filters

Javascript request filters have access to the variables `request` (type `io.netty.handler.codec.http.HttpRequest`), `contents` (type `net.lightbody.bmp.util.HttpMessageContents`), and `originalRequest` (type `io.netty.handler.codec.http.HttpRequest`). `originalRequest` contains the original request received from the client and does not reflect any changes made by previous filters. If the javascript returns an object of type `io.netty.handler.codec.http.HttpResponse`, the HTTP request will "short-circuit" and return the response immediately.

**Example: Modify User-Agent header**

```sh
curl -i -X POST -H 'Content-Type: text/plain' -d "request.headers().remove('User-Agent'); request.headers().add('User-Agent', 'My-Custom-User-Agent-String 1.0');" http://localhost:8080/proxy/8081/filter/request
```

##### Response filters

Javascript response filters have access to the variables `response` (type `io.netty.handler.codec.http.HttpResponse`), `contents` (type `net.lightbody.bmp.util.HttpMessageContents`), and `originalRequest` (type `io.netty.handler.codec.http.HttpRequest`). As in the request filter, `originalRequest` contains the original request received from the client and does not reflect any changes made by previous filters.

**Example: Modify response body**

```sh
curl -i -X POST -H 'Content-Type: text/plain' -d "contents.setTextContents('<html><body>Response successfully intercepted</body></html>');" http://localhost:8080/proxy/8081/filter/response
```

#### Legacy interceptors

If you are using the legacy ProxyServer implementation, you can manipulate the requests like so:
```java
    BrowserMobProxy server = new ProxyServer();
    ((LegacyProxyServer)server).addRequestInterceptor(new RequestInterceptor() {
        @Override
        public void process(BrowserMobHttpRequest request, Har har) {
            request.getMethod().removeHeaders("User-Agent");
            request.getMethod().addHeader("User-Agent", "Bananabot/1.0");
        }
    });
```
You can also POST a JavaScript payload to `/:port/interceptor/request` and `/:port/interceptor/response` using the REST interface. The functions will have a `request`/`response` variable, respectively, and a `har` variable (which may be null if a HAR isn't set up yet). The JavaScript code will be run by [Rhino](https://github.com/mozilla/rhino) and have access to the same Java API in the example above:

    [~]$ curl -X POST -H 'Content-Type: text/plain' -d 'request.getMethod().removeHeaders("User-Agent");' http://localhost:9090/proxy/9091/interceptor/request
    
Consult the Java API docs for more info.

### SSL Support

While the proxy supports SSL, it requires that a Certificate Authority be installed in to the browser. This allows the browser to trust all the SSL traffic coming from the proxy, which will be proxied using a classic man-in-the-middle technique. IT IS CRITICAL THAT YOU NOT INSTALL THIS CERTIFICATE AUTHORITY ON A BROWSER THAT IS USED FOR ANYTHING OTHER THAN TESTING.

If you're doing testing with Selenium, you'll want to make sure that the browser profile that gets set up by Selenium not only has the proxy configured, but also has the CA installed (Firefox set up by Selenium has installed CA by default). Unfortuantely, there is no API for doing this in Selenium, so you'll have to solve it uniquely for each browser type. We hope to make this easier in upcoming releases.

### NodeJS Support

NodeJS bindings for browswermob-proxy are available [here](https://github.com/zzo/browsermob-node).  Built-in support for [Selenium](http://seleniumhq.org) or use [CapserJS-on-PhantomJS](http://casperjs.org) or anything else to drive traffic for HAR generation.

### Logging

When running in stand-alone mode, the proxy loads the default logging configuration from the conf/bmp-logging.yaml file. To increase/decrease the logging level, change the logging entry for net.lightbody.bmp.

**New in 2.1:** Neither Embedded Mode nor the REST API include an slf4j static binding, so you no longer need to exclude the slf4j-jdk14 dependency when including `browsermob-core` or `browsermob-rest`.

### DNS Resolution

**New in 2.1:** BrowserMob Proxy enables native DNS resolution by default.

The legacy Jetty-based ProxyServer implementation uses XBill (dnsjava) resolution, but automatically falls back to the default JVM DNS resolution if XBill cannot resolve the address. To disable native DNS fallback, set the `bmp.allowNativeDnsFallback` JVM property to `false`. You can also use the `BrowserMobProxy.setHostNameResolver()` method to disable native DNS fallback and/or dnsjava resolution itself.

When running from the command line:

    $ JAVA_OPTS="-Dbmp.allowNativeDnsFallback=false" sh browsermob-proxy

or in Windows:

    C:\browsermob-proxy\bin> set JAVA_OPTS="-Dbmp.allowNativeDnsFallback=false"
    C:\browsermob-proxy\bin> browsermob-proxy.bat

If you are running in Embedded Mode (for example, within a Selenium test) you can disable native fallback or dnsjava by setting the implementation directly:

```java
    BrowserMobProxy proxyServer = new ProxyServer();
    // use only dnsjava
    proxyServer.setHostNameResolver(ClientUtil.createDnsJavaResolver());
    // or use only native resolution
    proxyServer.setHostNameResolver(ClientUtil.createNativeCacheManipulatingResolver());
    //...
    proxyServer.start(0);
```

## Creating the batch files from source

You'll need maven (`brew install maven` if you're on OS X); use the `release` profile to generate the batch files from this repository. Optionally, proceed at your own risk and append the `-DskipTests` option if the tests are failing.

    [~]$ mvn -P release
    [~]$ mvn -DskipTests -P release
