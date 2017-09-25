+++
title = "NodeWebServer and NodeStartup"
date = "2017-09-19T15:00:00+09:00"
tags = ["Corda"]
categories = ["Code Reading"]
banner = "img/banners/banner-3.jpg"
+++

Capsule によってパッケージングされた2つの jar ファイル `corda-webserver.jar` と `corda.jar` 。

<!--more-->

それぞれのエントリーポイントになっているクラス `NodeWebServer` と `NodeStartup` のコードを見てみましょう。`NodeWebServer` では、Jetty を起動しています。`NodeStartup` のほうはかなり複雑です。次回以降詳細に見ていきます。 


## NodeWebServer.kt
[NodeWebServer.kt](https://github.com/corda/corda/blob/release-M14.0/webserver/src/main/kotlin/net/corda/webserver/internal/NodeWebServer.kt)
```
    private fun initWebServer(localRpc: CordaRPCOps): Server {
        // Note that the web server handlers will all run concurrently, and not on the node thread.
        val handlerCollection = HandlerCollection()

        // Export JMX monitoring statistics and data over REST/JSON.
        if (config.exportJMXto.split(',').contains("http")) {
            val classpath = System.getProperty("java.class.path").split(System.getProperty("path.separator"))
            val warpath = classpath.firstOrNull { it.contains("jolokia-agent-war-2") && it.endsWith(".war") }
            if (warpath != null) {
                handlerCollection.addHandler(WebAppContext().apply {
                    // Find the jolokia WAR file on the classpath.
                    contextPath = "/monitoring/json"
                    setInitParameter("mimeType", MediaType.APPLICATION_JSON)
                    war = warpath
                })
            } else {
                log.warn("Unable to locate Jolokia WAR on classpath")
            }
        }

        // API, data upload and download to services (attachments, rates oracles etc)
        handlerCollection.addHandler(buildServletContextHandler(localRpc))

        val server = Server()

        val connector = if (config.useHTTPS) {
            val httpsConfiguration = HttpConfiguration()
            httpsConfiguration.outputBufferSize = 32768
            httpsConfiguration.addCustomizer(SecureRequestCustomizer())
            val sslContextFactory = SslContextFactory()
            sslContextFactory.keyStorePath = config.sslKeystore.toString()
            sslContextFactory.setKeyStorePassword(config.keyStorePassword)
            sslContextFactory.setKeyManagerPassword(config.keyStorePassword)
            sslContextFactory.setTrustStorePath(config.trustStoreFile.toString())
            sslContextFactory.setTrustStorePassword(config.trustStorePassword)
            sslContextFactory.setExcludeProtocols("SSL.*", "TLSv1", "TLSv1.1")
            sslContextFactory.setIncludeProtocols("TLSv1.2")
            sslContextFactory.setExcludeCipherSuites(".*NULL.*", ".*RC4.*", ".*MD5.*", ".*DES.*", ".*DSS.*")
            sslContextFactory.setIncludeCipherSuites(".*AES.*GCM.*")
            val sslConnector = ServerConnector(server, SslConnectionFactory(sslContextFactory, "http/1.1"), HttpConnectionFactory(httpsConfiguration))
            sslConnector.port = address.port
            sslConnector
        } else {
            val httpConfiguration = HttpConfiguration()
            httpConfiguration.outputBufferSize = 32768
            val httpConnector = ServerConnector(server, HttpConnectionFactory(httpConfiguration))
            httpConnector.port = address.port
            httpConnector
        }
        server.connectors = arrayOf<Connector>(connector)

        server.handler = handlerCollection
        server.start()
        log.info("Starting webserver on address $address")
        return server
    }
```

## NodeStartup.kt
[NodeStartup.kt](https://github.com/corda/corda/blob/release-M14.0/node/src/main/kotlin/net/corda/node/internal/NodeStartup.kt)
```
    open protected fun startNode(conf: FullNodeConfiguration, versionInfo: VersionInfo, startTime: Long, cmdlineOptions: CmdLineOptions) {
        val advertisedServices = conf.calculateServices()
        val node = createNode(conf, versionInfo, advertisedServices)
        node.start()
        printPluginsAndServices(node)

        node.networkMapRegistrationFuture.thenMatch({
            val elapsed = (System.currentTimeMillis() - startTime) / 10 / 100.0
            // TODO: Replace this with a standard function to get an unambiguous rendering of the X.500 name.
            val name = node.info.legalIdentity.name.orgName ?: node.info.legalIdentity.name.commonName
            Node.printBasicNodeInfo("Node for \"$name\" started up and registered in $elapsed sec")

            // Don't start the shell if there's no console attached.
            val runShell = !cmdlineOptions.noLocalShell && System.console() != null
            node.startupComplete.then {
                try {
                    InteractiveShell.startShell(cmdlineOptions.baseDirectory, runShell, cmdlineOptions.sshdServer, node)
                } catch(e: Throwable) {
                    logger.error("Shell failed to start", e)
                }
            }
        }, {})
        node.run()
    }
```
