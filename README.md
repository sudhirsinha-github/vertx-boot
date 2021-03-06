# vertx-boot
Annotation based library to fast-develop vertx web applications. This is in-line to the Spring boot with minimal functionalities for now.

## Getting Started

Git clone the project on your local machine and import it to your favorite ide.

### Prerequisites

For runnning this, you will need
- Java 1.8
- Gradle support - In Eclipse editor, goto help -> eclipse marketplace -> search for buildship (buildship gradle integration) and install it.

## Brief
This library helps you to fast-build vertx web applications. This is a sort of boiler plate which will take care of injecting routes and deploying verticles so that you can work on your concrete APIs straightaway. To Brief
- **AppLauncher**        -> The starting point of the application. It is used to set the app configuration.
- **MainVerticle**       -> Main verticle deploys all the other verticles used in the program.
- **Handlers**           -> Handlers are basically the controllers which receives the input, process the input and returns the Json response back to the user. Helper Handlers are added and one can extend these as per the need.

## Build Vertx web applications using vertx-boot
For consuming vertx-boot madness :- (Will upload a sample project later to showcase it's usage)
- Add **vertx-boot** as dependency to your project.
- Create any **Handler** class which extends **com.greyseal.vertx.boot.handler.BaseHandler**. Sample PingHandler <br /><br />
```
@RequestMapping(path = "/ping")
public class PingHandler extends BaseHandler {

    public PingHandler(Vertx vertx) {
        super(vertx);
    }

    @Override
    @Protected
    @RequestMapping(method = HttpMethod.GET, consumes = MediaType.APPLICATION_JSON_VALUE, produces = MediaType.APPLICATION_JSON_VALUE)
    public void handle(RoutingContext ctx) {
        try {
            JsonObject result;
            result = new JsonObject().put("status", "OK");
            ctx.setBody(Buffer.buffer(result.toString()));
            ctx.response().putHeader("Custom", "header");
            ctx.next();
        } catch (Exception ex) {
            ctx.fail(ex);
        }
    }
}
```
- Add a **Verticle** to your project. Sample HttpServerVerticle <br /> <br/>
```
@Verticle(type = VerticleType.STANDARD, configuration = "httpServerVerticle")
public class HttpServerVerticle extends BaseVerticle {
    public static String CONTEXT_PATH = VertxBootConfig.INSTANCE.getConfig().getString(Configuration.CONTEXT_PATH);
    protected static Logger logger = LoggerFactory.getLogger(HttpServerVerticle.class);
    private Single<HttpServer> server;
    private Router mainRouter;

    @Override
    public void start(Future<Void> startFuture) {
        try {
            super.start();
            this.server = createHttpServer(createOptions(ConfigHelper.isHTTP2Enabled()), buildRouter());
            server.subscribe((result -> {
                startFuture.complete();
                logger.info("HTTP server running on port {}", result.actualPort());
            }), ex -> {
                startFuture.fail(ex);
            });
        } catch (Exception ex) {
            logger.error("Failed to start HTTP Server ", ex);
            startFuture.fail(ex);
        }
    }

    private Single<HttpServer> createHttpServer(final HttpServerOptions httpOptions, final Router router) {
        return vertx.createHttpServer(httpOptions).requestHandler(router::accept).rxListen(ConfigHelper.getPort(),
                ConfigHelper.getHost());
    }

    private HttpServerOptions createOptions(boolean http2) {
        HttpServerOptions serverOptions = new HttpServerOptions(ConfigHelper.getHTTPServerOptions());
        if (http2) {
            serverOptions.setSsl(true)
                    .setKeyCertOptions(
                            new PemKeyCertOptions().setCertPath("server-cert.pem").setKeyPath("server-key.pem"))
                    .setUseAlpn(true);
        }
        return serverOptions;
    }

    private Router buildRouter() {
        this.mainRouter = Router.router(vertx).exceptionHandler((error -> {
            logger.error("Routers not injected ", error);
        }));
        mainRouter.route(CONTEXT_PATH + "/*").handler(BodyHandler.create());
        mainRouter.route().handler(ResponseContentTypeHandler.create());
        mainRouter.route(CONTEXT_PATH + "/*").handler(
                CorsHandler.create("*").allowedHeaders(getAllowedHeaders()).exposedHeaders(getAllowedHeaders()));
        AnnotationProcessor.init(this.mainRouter, vertx);
        mainRouter.route(CONTEXT_PATH + "/*").last().failureHandler(ErrorHandler.create(vertx));
        return this.mainRouter;
    }

    public HttpClient buildHttpClient() {
        HttpClientOptions options = new HttpClientOptions();
        if (ConfigHelper.isSSLEnabled()) {
            options.setSsl(true);
            options.setTrustAll(true);
            options.setVerifyHost(false);
        }
        options.setTryUseCompression(true);
        options.setKeepAlive(true);
        options.setMaxPoolSize(50);
        return vertx.createHttpClient(options);
    }

    private Set<String> getAllowedHeaders() {
        Set<String> allowHeaders = new HashSet<>();
        allowHeaders.add("X-Requested-With");
        allowHeaders.add("Access-Control-Allow-Origin");
        allowHeaders.add("Origin");
        allowHeaders.add("Content-Type");
        allowHeaders.add("Accept");
        allowHeaders.add(HttpHeaders.AUTHORIZATION.toString());
        return allowHeaders;
    }
}
```
That's it. Just debug or run your project.

## Built With

* [Vertx](http://vertx.io/) - The web framework used
* [Gradle](https://gradle.org/) - Dependency Management
