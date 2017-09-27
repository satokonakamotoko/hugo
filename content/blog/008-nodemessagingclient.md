+++
title = "NodeMessagingClient"
date = "2017-09-20T13:01:00+09:00"
tags = ["Corda"]
categories = ["Code Reading"]
banner = "img/banners/banner-3.jpg"
+++

さきに、メッセージングサーバーの起動をみましたが、あわせてメッセージングクライアントも起動します。

<!--more-->

メッセージングクライアントにも、ActiveMQ Artemis が使われます。`processMessage` をループで実行し、受信した Artemis のメッセージを `artemisToCordaMessage` で Corda のメッセージに変換します。

## NodeMessagingClient.kt
[NodeMessagingClient.kt](https://github.com/corda/corda/blob/release-M14.0/node/src/main/kotlin/net/corda/node/services/messaging/NodeMessagingClient.kt)
```
    /**
     * Starts the p2p event loop: this method only returns once [stop] has been called.
     *
     * This actually runs as two sequential loops. The first subscribes for and receives only network map messages until
     * we get our network map fetch response.  At that point the filtering consumer is closed and we proceed to the second loop and
     * consume all messages via a new consumer without a filter applied.
     */
    fun run(serverControl: ActiveMQServerControl) {
        try {
            // Build the network map.
            runPreNetworkMap(serverControl)
            // Process everything else once we have the network map.
            runPostNetworkMap()
        } finally {
            shutdownLatch.countDown()
        }
    }
```

```
    private fun runPreNetworkMap(serverControl: ActiveMQServerControl) {
        val consumer = state.locked {
            check(started) { "start must be called first" }
            check(!running) { "run can't be called twice" }
            running = true
            rpcServer!!.start(serverControl)
            (verifierService as? OutOfProcessTransactionVerifierService)?.start(verificationResponseConsumer!!)
            p2pConsumer!!
        }

        while (!networkMapRegistrationFuture.isDone && processMessage(consumer)) {
        }
        with(networkMapRegistrationFuture) {
            if (isDone) getOrThrow() else andForget(log) // Trigger node shutdown here to avoid deadlock in shutdown hooks.
        }
    }
```

```
    private fun processMessage(consumer: ClientConsumer): Boolean {
        // Two possibilities here:
        //
        // 1. We block waiting for a message and the consumer is closed in another thread. In this case
        //    receive returns null and we break out of the loop.
        // 2. We receive a message and process it, and stop() is called during delivery. In this case,
        //    calling receive will throw and we break out of the loop.
        //
        // It's safe to call into receive simultaneous with other threads calling send on a producer.
        val artemisMessage: ClientMessage = try {
            consumer.receive()
        } catch(e: ActiveMQObjectClosedException) {
            null
        } ?: return false

        val message: ReceivedMessage? = artemisToCordaMessage(artemisMessage)
        if (message != null)
            deliver(message)

        // Ack the message so it won't be redelivered. We should only really do this when there were no
        // transient failures. If we caught an exception in the handler, we could back off and retry delivery
        // a few times before giving up and redirecting the message to a dead-letter address for admin or
        // developer inspection. Artemis has the features to do this for us, we just need to enable them.
        //
        // TODO: Setup Artemis delayed redelivery and dead letter addresses.
        //
        // ACKing a message calls back into the session which isn't thread safe, so we have to ensure it
        // doesn't collide with a send here. Note that stop() could have been called whilst we were
        // processing a message but if so, it'll be parked waiting for us to count down the latch, so
        // the session itself is still around and we can still ack messages as a result.
        state.locked {
            artemisMessage.acknowledge()
        }
        return true
    }
```

```
    private fun artemisToCordaMessage(message: ClientMessage): ReceivedMessage? {
        try {
            val topic = message.required(topicProperty) { getStringProperty(it) }
            val sessionID = message.required(sessionIdProperty) { getLongProperty(it) }
            val user = requireNotNull(message.getStringProperty(HDR_VALIDATED_USER)) { "Message is not authenticated" }
            val platformVersion = message.required(platformVersionProperty) { getIntProperty(it) }
            // Use the magic deduplication property built into Artemis as our message identity too
            val uuid = message.required(HDR_DUPLICATE_DETECTION_ID) { UUID.fromString(message.getStringProperty(it)) }
            log.trace { "Received message from: ${message.address} user: $user topic: $topic sessionID: $sessionID uuid: $uuid" }

            return ArtemisReceivedMessage(TopicSession(topic, sessionID), X500Name(user), platformVersion, uuid, message)
        } catch (e: Exception) {
            log.error("Unable to process message, ignoring it: $message", e)
            return null
        }
    }
```
