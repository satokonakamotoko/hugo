+++
title = "ExampleClientRPC"
date = "2017-09-30T12:00:00+09:00"
tags = ["Corda"]
categories = ["Code Reading"]
banner = "img/banners/banner-3.jpg"
+++

つづいて、CordaRPCClient を使って Node にアクセスする方法についてです。

<!--more-->

`cordapp-example` に、Java, Kotlin それぞれの言語で、`CordaRPCClient` を使用した example client が用意されています。 実際の Node へのアクセスは、`CordaRPCOps` を介して行います。

## ExampleClientRPC.kt
[ExampleClientRPC.kt](https://github.com/corda/cordapp-example/blob/release-M14.0/kotlin-source/src/main/kotlin/com/example/client/ExampleClientRPC.kt)
```
    fun main(args: Array<String>) {
        require(args.size == 1) { "Usage: ExampleClientRPC <node address>" }
        val nodeAddress = args[0].parseNetworkHostAndPort()
        val client = CordaRPCClient(nodeAddress)

        // Can be amended in the com.example.MainKt file.
        val proxy = client.start("user1", "test").proxy

        // Grab all signed transactions and all future signed transactions.
        val (transactions: List<SignedTransaction>, futureTransactions: Observable<SignedTransaction>) =
                proxy.verifiedTransactionsFeed()

        // Log the 'placed' IOU states and listen for new ones.
        futureTransactions.startWith(transactions).toBlocking().subscribe { transaction ->
            transaction.tx.outputs.forEach { output ->
                val state = output.data as IOUState
                logger.info(state.iou.toString())
            }
        }
    }
```

## ExampleClientRPC.java
[ExampleClientRPC.java](https://github.com/corda/cordapp-example/blob/release-M14.0/java-source/src/main/java/com/example/client/ExampleClientRPC.java)
```
    public static void main(String[] args) throws ActiveMQException, InterruptedException, ExecutionException {
        if (args.length != 1) {
            throw new IllegalArgumentException("Usage: ExampleClientRPC <node address>");
        }

        final Logger logger = LoggerFactory.getLogger(ExampleClientRPC.class);
        final NetworkHostAndPort nodeAddress = NetworkHostAndPortKt.parseNetworkHostAndPort(args[0]);
        final CordaRPCClient client = new CordaRPCClient(nodeAddress, null, CordaRPCClientConfiguration.getDefault(), true);

        // Can be amended in the com.example.Main file.
        final CordaRPCOps proxy = client.start("user1", "test").getProxy();

        // Grab all signed transactions and all future signed transactions.
        final DataFeed<List<SignedTransaction>, SignedTransaction> txsAndFutureTxs =
                proxy.verifiedTransactionsFeed();
        final List<SignedTransaction> txs = txsAndFutureTxs.getSnapshot();
        final Observable<SignedTransaction> futureTxs = txsAndFutureTxs.getUpdates();

        // Log the 'placed' IOUs and listen for new ones.
        futureTxs.startWith(txs).toBlocking().subscribe(
                transaction ->
                        transaction.getTx().getOutputs().forEach(
                                output -> {
                                    final IOUState iouState = (IOUState) output.getData();
                                    logger.info(iouState.getIOU().toString());
                                })
        );
    }
```

## CordaRPCOps.kt
[CordaRPCOps.kt](https://github.com/corda/corda/blob/release-M14.0/core/src/main/kotlin/net/corda/core/messaging/CordaRPCOps.kt)
```
/**
 * RPC operations that the node exposes to clients using the Java client library. These can be called from
 * client apps and are implemented by the node in the [net.corda.node.internal.CordaRPCOpsImpl] class.
 */
interface CordaRPCOps : RPCOps {
```
