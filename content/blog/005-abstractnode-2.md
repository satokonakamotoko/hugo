+++
title = "AbstractNode 2"
date = "2017-09-20T06:00:00+09:00"
tags = ["Corda"]
categories = ["Code Reading"]
banner = "img/banners/banner-5.jpg"
+++

引き続き `AbstractNode` に実装されている、ノードの起動処理について見ていきます。

<!--more-->

ノードの起動処理のなかで、多数のサービスを順にセットアップしていくわけですが、それらサービスのなかでも重要なサービスである、メッセージングサービス（メッセージングサーバー）を起動します。Corda では、メッセージングサービスの実装として、Apache ActiveMQ Artemis を使用しています。

## AbstractNode.kt
[AbstractNode.kt](https://github.com/corda/corda/blob/release-M14.0/node/src/main/kotlin/net/corda/node/internal/AbstractNode.kt)
```
    private fun makeServices(): MutableList<Any> {
        checkpointStorage = DBCheckpointStorage()
        _services = ServiceHubInternalImpl()
        attachments = createAttachmentStorage()
        network = makeMessagingService()
        info = makeInfo()

        val tokenizableServices = mutableListOf(attachments, network, services.vaultService, services.vaultQueryService,
                services.keyManagementService, services.identityService, platformClock, services.schedulerService)
        makeAdvertisedServices(tokenizableServices)
        return tokenizableServices
    }
```

```
    private fun makeVaultObservers() {
        VaultSoftLockManager(services.vaultService, smm)
        CashBalanceAsMetricsObserver(services, database)
        ScheduledActivityObserver(services)
        HibernateObserver(services.vaultService.rawUpdates, HibernateConfiguration(services.schemaService))
    }
```

## Apache ActiveMQ Artemis
[a multi-protocol, embeddable, very high performance, clustered, asynchronous messaging system](https://activemq.apache.org/artemis/)

## Node.kt
[Node.kt](https://github.com/corda/corda/blob/release-M14.0/node/src/main/kotlin/net/corda/node/internal/Node.kt)
```
    override fun startMessagingService(rpcOps: RPCOps) {
        // Start up the embedded MQ server
        messageBroker?.apply {
            runOnStop += this::stop
            start()
        }

        // Start up the MQ client.
        (network as NodeMessagingClient).start(rpcOps, userService)
    }
```

```
    override fun makeMessagingService(): MessagingService {
        userService = RPCUserServiceImpl(configuration.rpcUsers)

        val (serverAddress, advertisedAddress) = with(configuration) {
            if (messagingServerAddress != null) {
                // External broker
                messagingServerAddress to messagingServerAddress
            } else {
                makeLocalMessageBroker() to getAdvertisedAddress()
            }
        }

        printBasicNodeInfo("Incoming connection address", advertisedAddress.toString())

        val myIdentityOrNullIfNetworkMapService = if (networkMapAddress != null) obtainLegalIdentity().owningKey else null
        return NodeMessagingClient(
                configuration,
                versionInfo,
                serverAddress,
                myIdentityOrNullIfNetworkMapService,
                serverThread,
                database,
                networkMapRegistrationFuture,
                services.monitoringService,
                advertisedAddress)
    }

    private fun makeLocalMessageBroker(): NetworkHostAndPort {
        with(configuration) {
            messageBroker = ArtemisMessagingServer(this, p2pAddress.port, rpcAddress?.port, services.networkMapCache, userService)
            return NetworkHostAndPort("localhost", p2pAddress.port)
        }
    }
```

## ArtemisMessagingServer.kt
[ArtemisMessagingServer.kt](https://github.com/corda/corda/blob/release-M14.0/node/src/main/kotlin/net/corda/node/services/messaging/ArtemisMessagingServer.kt)
```
    @Throws(IOException::class, KeyStoreException::class)
    private fun configureAndStartServer() {
        val artemisConfig = createArtemisConfig()
        val securityManager = createArtemisSecurityManager()
        activeMQServer = ActiveMQServerImpl(artemisConfig, securityManager).apply {
            // Throw any exceptions which are detected during startup
            registerActivationFailureListener { exception -> throw exception }
            // Some types of queue might need special preparation on our side, like dialling back or preparing
            // a lazily initialised subsystem.
            registerPostQueueCreationCallback { deployBridgesFromNewQueue(it.toString()) }
            if (nodeRunsNetworkMapService) registerPostQueueCreationCallback { handleIpDetectionRequest(it.toString()) }
            registerPostQueueDeletionCallback { address, qName -> log.debug { "Queue deleted: $qName for $address" } }
        }
        activeMQServer.start()
        Node.printBasicNodeInfo("Listening on port", p2pPort.toString())
        if (rpcPort != null) {
            Node.printBasicNodeInfo("RPC service listening on port", rpcPort.toString())
        }
    }
```
