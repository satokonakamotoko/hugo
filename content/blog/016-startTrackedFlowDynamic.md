+++
title = "startTrackedFlowDynamic"
date = "2017-10-03T06:00:00+09:00"
tags = ["Corda"]
categories = ["Code Reading"]
banner = "img/banners/banner-1.jpg"
+++

前回は、ExampleApi という REST API を提供するクラスを用意して、Node にアクセスする方法について見ていきました。

<!--more-->

ただしこの方法を有効にするには、もうひとつ作業が必要になります。`WebServerPluginRegistry` を実装したプラグインクラスを用意し、Java の ServiceLoader の仕組みを使って REST API サービスを有効にします。


## ExampleApi.kt
[ExampleApi.kt](https://github.com/corda/cordapp-example/blob/release-M14.0/kotlin-source/src/main/kotlin/com/example/api/ExampleApi.kt)
```
    @PUT
    @Path("create-iou")
    fun createIOU(@QueryParam("iouValue") iouValue: Int, @QueryParam("partyName") partyName: X500Name): Response {
        val otherParty = services.partyFromX500Name(partyName) ?:
                return Response.status(Response.Status.BAD_REQUEST).build()

        var status: Response.Status
        var msg: String
        try {
            val flowHandle = services.startTrackedFlow(::Initiator, iouValue, otherParty)
            flowHandle.progress.subscribe { println(">> $it") }

            // The line below blocks and waits for the future to resolve.
            val result = flowHandle
                    .returnValue
                    .getOrThrow()

            status = Response.Status.CREATED
            msg = "Transaction id ${result.id} committed to ledger."

        } catch (ex: Throwable) {
            status = Response.Status.BAD_REQUEST
            msg = ex.message!!
            logger.error(msg, ex)
        }

        return Response.status(status).entity(msg).build()
    }
```

## CordaRPCOpsImpl.kt
[CordaRPCOpsImpl.kt]()
```
    override fun <T : Any> startTrackedFlowDynamic(logicType: Class<out FlowLogic<T>>, vararg args: Any?): FlowProgressHandle<T> {
        val stateMachine = startFlow(logicType, args)
        return FlowProgressHandleImpl(
                id = stateMachine.id,
                returnValue = stateMachine.resultFuture,
                progress = stateMachine.logic.track()?.second ?: Observable.empty()
        )
    }
```

## AbstractNode.kt
[AbstractNode.kt]()
```
        override fun <T> startFlow(logic: FlowLogic<T>, flowInitiator: FlowInitiator): FlowStateMachineImpl<T> {
            return serverThread.fetchFrom { smm.add(logic, flowInitiator) }
        }
```
