+++
title = "ExampleFlow"
date = "2017-10-05T06:00:00+09:00"
tags = ["Corda"]
categories = ["Code Reading"]
banner = "img/banners/banner-3.jpg"
+++

前回、FlowStateMachineImpl で、logic.call() というコードがありました。

<!--more-->

この `logic` とは、`FlowLogic` のことであり、この example では、`ExampleFlow` のことです。そして送り主（Initiator）と受け取り主（Acceptor）それぞれの `FlowLogic` が用意されており、その `call` メソッドが呼び出されます。


## ExampleFlow.kt
[ExampleFlow.kt](https://github.com/corda/cordapp-example/blob/release-M14.0/kotlin-source/src/main/kotlin/com/example/flow/ExampleFlow.kt)
```
        @Suspendable
        override fun call(): SignedTransaction {
            // Obtain a reference to the notary we want to use.
            val notary = serviceHub.networkMapCache.notaryNodes.single().notaryIdentity

            // Stage 1.
            progressTracker.currentStep = GENERATING_TRANSACTION
            // Generate an unsigned transaction.
            val iouState = IOUState(IOU(iouValue), serviceHub.myInfo.legalIdentity, otherParty)
            val txCommand = Command(IOUContract.Commands.Create(), iouState.participants.map { it.owningKey })
            val txBuilder = TransactionBuilder(TransactionType.General, notary).withItems(iouState, txCommand)

            // Stage 2.
            progressTracker.currentStep = VERIFYING_TRANSACTION
            // Verify that the transaction is valid.
            txBuilder.toWireTransaction().toLedgerTransaction(serviceHub).verify()

            // Stage 3.
            progressTracker.currentStep = SIGNING_TRANSACTION
            // Sign the transaction.
            val partSignedTx = serviceHub.signInitialTransaction(txBuilder)

            // Stage 4.
            progressTracker.currentStep = GATHERING_SIGS
            // Send the state to the counterparty, and receive it back with their signature.
            val fullySignedTx = subFlow(CollectSignaturesFlow(partSignedTx, GATHERING_SIGS.childProgressTracker()))

            // Stage 5.
            progressTracker.currentStep = FINALISING_TRANSACTION
            // Notarise and record the transaction in both parties' vaults.
            return subFlow(FinalityFlow(fullySignedTx, FINALISING_TRANSACTION.childProgressTracker())).single()
        }
```

## ExampleFlow.java
[ExampleFlow.java](https://github.com/corda/cordapp-example/blob/release-M14.0/java-source/src/main/java/com/example/flow/ExampleFlow.java)
```
        @Suspendable
        @Override
        public SignedTransaction call() throws FlowException {
            // Obtain a reference to the notary we want to use.
            final Party notary = getServiceHub().getNetworkMapCache().getNotaryNodes().get(0).getNotaryIdentity();

            // Stage 1.
            progressTracker.setCurrentStep(GENERATING_TRANSACTION);
            // Generate an unsigned transaction.
            IOUState iouState = new IOUState(new IOU(iouValue), getServiceHub().getMyInfo().getLegalIdentity(), otherParty);
            final Command txCommand = new Command(new IOUContract.Commands.Create(),
                    iouState.getParticipants().stream().map(AbstractParty::getOwningKey).collect(Collectors.toList()));
            final TransactionBuilder txBuilder = new TransactionType.General.Builder(notary).withItems(iouState, txCommand);

            // Stage 2.
            progressTracker.setCurrentStep(VERIFYING_TRANSACTION);
            // Verify that the transaction is valid.
            txBuilder.toWireTransaction().toLedgerTransaction(getServiceHub()).verify();

            // Stage 3.
            progressTracker.setCurrentStep(SIGNING_TRANSACTION);
            // Sign the transaction.
            final SignedTransaction partSignedTx = getServiceHub().signInitialTransaction(txBuilder);

            // Stage 4.
            progressTracker.setCurrentStep(GATHERING_SIGS);
            // Send the state to the counterparty, and receive it back with their signature.
            final SignedTransaction fullySignedTx = subFlow(
                    new CollectSignaturesFlow(partSignedTx, CollectSignaturesFlow.Companion.tracker()));

            // Stage 5.
            progressTracker.setCurrentStep(FINALISING_TRANSACTION);
            // Notarise and record the transaction in both parties' vaults.
            return subFlow(new FinalityFlow(fullySignedTx)).get(0);
        }
```
