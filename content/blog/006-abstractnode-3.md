+++
title = "AbstractNode 3"
date = "2017-09-20T06:01:00+09:00"
tags = ["Corda"]
categories = ["Code Reading"]
banner = "img/banners/banner-1.jpg"
+++

まだまだ AbstractNode の起動処理は続きます。

<!--more-->

ここで、Corda の重要な概念のひとつ [Flow](https://docs.corda.net/releases/release-M14.0/key-concepts-flows.html) をインストールしています。Flow については後ほど詳しく見ていきましょう。そして `StateMachineManager` `NodeSchedulerService` と重要なサービスを開始していきます。

## AbstractNode.kt
[AbstractNode.kt](https://github.com/corda/corda/blob/release-M14.0/node/src/main/kotlin/net/corda/node/internal/AbstractNode.kt)
```
    private fun installCoreFlows() {
        installCoreFlow(FetchTransactionsFlow::class) { otherParty, _ -> FetchTransactionsHandler(otherParty) }
        installCoreFlow(FetchAttachmentsFlow::class) { otherParty, _ -> FetchAttachmentsHandler(otherParty) }
        installCoreFlow(BroadcastTransactionFlow::class) { otherParty, _ -> NotifyTransactionHandler(otherParty) }
        installCoreFlow(NotaryChangeFlow::class) { otherParty, _ -> NotaryChangeHandler(otherParty) }
        installCoreFlow(ContractUpgradeFlow::class) { otherParty, _ -> ContractUpgradeHandler(otherParty) }
        installCoreFlow(TransactionKeyFlow::class) { otherParty, _ -> TransactionKeyHandler(otherParty) }
    }
```

```
    private fun scanCordapps(): ScanResult? {
        val scanPackage = System.getProperty("net.corda.node.cordapp.scan.package")
        val paths = if (scanPackage != null) {
            // Rather than looking in the plugins directory, figure out the classpath for the given package and scan that
            // instead. This is used in tests where we avoid having to package stuff up in jars and then having to move
            // them to the plugins directory for each node.
            check(configuration.devMode) { "Package scanning can only occur in dev mode" }
            val resource = scanPackage.replace('.', '/')
            javaClass.classLoader.getResources(resource)
                    .asSequence()
                    .map {
                        val uri = if (it.protocol == "jar") {
                            (it.openConnection() as JarURLConnection).jarFileURL.toURI()
                        } else {
                            URI(it.toExternalForm().removeSuffix(resource))
                        }
                        Paths.get(uri)
                    }
                    .toList()
        } else {
            val pluginsDir = configuration.baseDirectory / "plugins"
            if (!pluginsDir.exists()) return null
            pluginsDir.list {
                it.filter { it.isRegularFile() && it.toString().endsWith(".jar") }.collect(toList())
            }
        }

        log.info("Scanning CorDapps in $paths")

        // This will only scan the plugin jars and nothing else
        return if (paths.isNotEmpty()) FastClasspathScanner().overrideClasspath(paths).scan() else null
    }
```

## StateMachineManager.kt
[StateMachineManager.kt](https://github.com/corda/corda/blob/release-M14.0/node/src/main/kotlin/net/corda/node/services/statemachine/StateMachineManager.kt)
```
    fun start() {
        restoreFibersFromCheckpoints()
        listenToLedgerTransactions()
        serviceHub.networkMapCache.mapServiceRegistered.then { executor.execute(this::resumeRestoredFibers) }
    }
```

## NodeSchedulerService.kt
[NodeSchedulerService.kt](https://github.com/corda/corda/blob/release-M14.0/node/src/main/kotlin/net/corda/node/services/events/NodeSchedulerService.kt)
```
    fun start() {
        mutex.locked {
            recomputeEarliest()
            rescheduleWakeUp()
        }
    }
```
