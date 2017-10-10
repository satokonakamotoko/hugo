+++
title = "DriverDSL"
date = "2017-09-27T16:00:00+09:00"
tags = ["Corda"]
categories = ["Code Reading"]
banner = "img/banners/banner-4.jpg"
+++

さてここまで、runnodes スクリプトによる Corda Node の起動シーケンスについてみてきました。

<!--more-->

Corda には `runnodes` とは別の Node の起動方法が用意されており、それは `Driver.kt` で提供されています。`Driver.kt` の冒頭ファイルコメントを紹介します。
```
This file defines a small "Driver" DSL for starting up nodes that is only intended for development, demos and tests.
```

## Main.kt
[Main.kt](https://github.com/corda/cordapp-example/blob/release-M14.0/kotlin-source/src/test/kotlin/com/example/Main.kt)
```
fun main(args: Array<String>) {
    // No permissions required as we are not invoking flows.
    val user = User("user1", "test", permissions = setOf())
    driver(isDebug = true) {
        startNode(X500Name("CN=Controller,O=R3,OU=corda,L=London,C=UK"), setOf(ServiceInfo(ValidatingNotaryService.type)))
        val (nodeA, nodeB, nodeC) = Futures.allAsList(
                startNode(X500Name("CN=NodeA,O=NodeA,L=London,C=UK"), rpcUsers = listOf(user)),
                startNode(X500Name("CN=NodeB,O=NodeB,L=New York,C=US"), rpcUsers = listOf(user)),
                startNode(X500Name("CN=NodeC,O=NodeC,L=Paris,C=FR"), rpcUsers = listOf(user))).getOrThrow()

        startWebserver(nodeA)
        startWebserver(nodeB)
        startWebserver(nodeC)

        waitForAllNodesToFinish()
    }
}
```

## Driver.kt
[Driver.kt](https://github.com/corda/corda/blob/release-M14.0/test-utils/src/main/kotlin/net/corda/testing/driver/Driver.kt)
```
        private fun startInProcessNode(
                executorService: ListeningScheduledExecutorService,
                nodeConf: FullNodeConfiguration,
                config: Config
        ): ListenableFuture<Pair<Node, Thread>> {
            return executorService.submit<Pair<Node, Thread>> {
                log.info("Starting in-process Node ${nodeConf.myLegalName.commonName}")
                // Write node.conf
                writeConfig(nodeConf.baseDirectory, "node.conf", config)
                val clock: Clock = if (nodeConf.useTestClock) TestClock() else NodeClock()
                // TODO pass the version in?
                val node = Node(nodeConf, nodeConf.calculateServices(), MOCK_VERSION_INFO, clock, initialiseSerialization = false)
                node.start()
                val nodeThread = thread(name = nodeConf.myLegalName.commonName) {
                    node.run()
                }
                node to nodeThread
            }.flatMap { nodeAndThread -> addressMustBeBoundFuture(executorService, nodeConf.p2pAddress).map { nodeAndThread } }
        }
```
