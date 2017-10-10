+++
title = "ExampleApi 2"
date = "2017-10-02T12:00:00+09:00"
tags = ["Corda"]
categories = ["Code Reading"]
banner = "img/banners/banner-5.jpg"
+++

前回は、ExampleApi という REST API を提供するクラスを用意して、Node にアクセスする方法について見ていきました。

<!--more-->

ただしこの方法を有効にするには、もうひとつ作業が必要になります。`WebServerPluginRegistry` を実装したプラグインクラスを用意し、Java の ServiceLoader の仕組みを使って REST API サービスを有効にします。


## ExamplePlugin.java
[ExamplePlugin.java](https://github.com/corda/cordapp-example/blob/release-M14.0/java-source/src/main/java/com/example/plugin/ExamplePlugin.java)
```
public class ExamplePlugin implements WebServerPluginRegistry {
    /**
     * A list of classes that expose web APIs.
     */
    private final List<Function<CordaRPCOps, ?>> webApis = ImmutableList.of(ExampleApi::new);

    /**
     * A list of directories in the resources directory that will be served by Jetty under /web.
     */
    private final Map<String, String> staticServeDirs = ImmutableMap.of(
            // This will serve the exampleWeb directory in resources to /web/example
            "example", getClass().getClassLoader().getResource("exampleWeb").toExternalForm()
    );

    @Override public List<Function<CordaRPCOps, ?>> getWebApis() { return webApis; }
    @Override public Map<String, String> getStaticServeDirs() { return staticServeDirs; }
}
```

## ExamplePlugin.kt
[ExamplePlugin.kt](https://github.com/corda/cordapp-example/blob/release-M14.0/kotlin-source/src/main/kotlin/com/example/plugin/ExamplePlugin.kt)
```
class ExamplePlugin : WebServerPluginRegistry {
    /**
     * A list of classes that expose web APIs.
     */
    override val webApis: List<Function<CordaRPCOps, out Any>> = listOf(Function(::ExampleApi))

    /**
     * A list of directories in the resources directory that will be served by Jetty under /web.
     */
    override val staticServeDirs: Map<String, String> = mapOf(
            // This will serve the exampleWeb directory in resources to /web/example
            "example" to javaClass.classLoader.getResource("exampleWeb").toExternalForm()
    )
}
```

## net.corda.webserver.services.WebServerPluginRegistry
[net.corda.webserver.services.WebServerPluginRegistry](https://github.com/corda/cordapp-example/blob/release-M14.0/kotlin-source/src/main/resources/META-INF/services/net.corda.webserver.services.WebServerPluginRegistry)
```
# Register a ServiceLoader service extending from net.corda.webserver.services.WebServerPluginRegistry
com.example.plugin.ExamplePlugin
```
