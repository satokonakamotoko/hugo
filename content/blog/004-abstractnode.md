+++
title = "AbstractNode"
date = "2017-09-19T15:01:00+09:00"
tags = ["Corda"]
categories = ["Code Reading"]
banner = "img/banners/banner-4.jpg"
+++

NodeStartup のなかで、node.start() が呼ばれていました。

<!--more-->

start メソッドは、ノードのベースクラスである `AbstractNode` に実装されており、その名前のとおりノードの起動処理が書かれています。この start メソッドを起点にして、多数の [services](https://github.com/corda/corda/tree/release-M14.0/node/src/main/kotlin/net/corda/node/services) が利用可能な状態にセットアップされていきます。

## AbstractNode.kt
[AbstractNode.kt](https://github.com/corda/corda/blob/release-M14.0/node/src/main/kotlin/net/corda/node/internal/AbstractNode.kt)
```
    open fun start() {
        require(!started) { "Node has already been started" }

        if (configuration.devMode) {
            log.warn("Corda node is running in dev mode.")
            configuration.configureWithDevSSLCertificate()
        }
        validateKeystore()

        log.info("Node starting up ...")

        // Do all of this in a database transaction so anything that might need a connection has one.
        initialiseDatabasePersistence {
            val tokenizableServices = makeServices()

            smm = StateMachineManager(services,
                    checkpointStorage,
                    serverThread,
                    database,
                    busyNodeLatch)

            smm.tokenizableServices.addAll(tokenizableServices)

            if (serverThread is ExecutorService) {
                runOnStop += {
                    // We wait here, even though any in-flight messages should have been drained away because the
                    // server thread can potentially have other non-messaging tasks scheduled onto it. The timeout value is
                    // arbitrary and might be inappropriate.
                    MoreExecutors.shutdownAndAwaitTermination(serverThread as ExecutorService, 50, SECONDS)
                }
            }

            makeVaultObservers()

            checkpointStorage.forEach {
                isPreviousCheckpointsPresent = true
                false
            }
            startMessagingService(rpcOps)
            installCoreFlows()

            val scanResult = scanCordapps()
            if (scanResult != null) {
                installCordaServices(scanResult)
                registerInitiatedFlows(scanResult)
                findRPCFlows(scanResult)
            }

            // TODO Remove this once the cash stuff is in its own CorDapp
            registerInitiatedFlow(IssuerFlow.Issuer::class.java)

            initUploaders()

            runOnStop += network::stop
            _networkMapRegistrationFuture.setFuture(registerWithNetworkMapIfConfigured())
            smm.start()
            // Shut down the SMM so no Fibers are scheduled.
            runOnStop += { smm.stop(acceptableLiveFiberCountOnStop()) }
            _services.schedulerService.start()
        }
        started = true
    }
```

## NodeConfiguration.kt
[NodeConfiguration.kt](https://github.com/corda/corda/blob/release-M14.0/node/src/main/kotlin/net/corda/node/services/config/NodeConfiguration.kt)

## CordaPersistence.kt
[CordaPersistence.kt](https://github.com/corda/corda/blob/release-M14.0/node/src/main/kotlin/net/corda/node/utilities/CordaPersistence.kt)
```
fun configureDatabase(props: Properties): CordaPersistence {
    val config = HikariConfig(props)
    val dataSource = HikariDataSource(config)
    val persistence = CordaPersistence.connect(dataSource)

    //org.jetbrains.exposed.sql.Database will be removed once Exposed library is removed
    val database = Database.connect(dataSource) { _ -> ExposedTransactionManager() }
    persistence.database = database

    // Check not in read-only mode.
    persistence.transaction {
        persistence.dataSource.connection.use {
            check(!it.metaData.isReadOnly) { "Database should not be readonly." }
        }
    }
    return persistence
}
```
## HikariCP
[A solid high-performance JDBC connection pool](http://brettwooldridge.github.io/HikariCP/)
