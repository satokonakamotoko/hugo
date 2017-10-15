+++
title = "FinalityFlow"
date = "2017-10-06T06:00:00+09:00"
tags = ["Corda"]
categories = ["Code Reading"]
banner = "img/banners/banner-4.jpg"
+++

Corda ではいくつもの [Flow](https://docs.corda.net/releases/release-M14.0/key-concepts-flows.html) があらかじめ[用意](https://github.com/corda/corda/tree/release-M14.0/core/src/main/kotlin/net/corda/core/flows)されています。

<!--more-->

例えば、そのうちのひとつに `FinalityFlow` があります。この Flow は Transaction の完了を記録します。


## FinalityFlow.kt
[FinalityFlow.kt](https://github.com/corda/corda/blob/release-M14.0/core/src/main/kotlin/net/corda/core/flows/FinalityFlow.kt)
```
    @Suspendable
    private fun notariseAndRecord(stx: SignedTransaction): SignedTransaction {
        val notarised = if (needsNotarySignature(stx)) {
            val notarySignatures = subFlow(NotaryFlow.Client(stx))
            stx + notarySignatures
        } else {
            stx
        }
        serviceHub.recordTransactions(notarised)
        return notarised
    }
```
