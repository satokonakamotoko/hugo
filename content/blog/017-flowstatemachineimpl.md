+++
title = "FlowStateMachineImpl"
date = "2017-10-04T06:00:00+09:00"
tags = ["Corda"]
categories = ["Code Reading"]
banner = "img/banners/banner-2.jpg"
+++

ExampleApi の createIOU メソッドから、startTrackedFlow を呼び出していました。今回はその続きです。

<!--more-->

`StateMachineManager` を経由して、`co.paralleluniverse.fibers.Fiber` のサブクラスである、`FlowStateMachineImpl` を呼び出します。fiber のライブラリとして `Quasar` が使われています。


## StateMachineManager.kt
[StateMachineManager.kt](https://github.com/corda/corda/blob/release-M14.0/node/src/main/kotlin/net/corda/node/services/statemachine/StateMachineManager.kt)
```
    fun <T> add(logic: FlowLogic<T>, flowInitiator: FlowInitiator): FlowStateMachineImpl<T> {
        // TODO: Check that logic has @Suspendable on its call method.
        executor.checkOnThread()
        // We swap out the parent transaction context as using this frequently leads to a deadlock as we wait
        // on the flow completion future inside that context. The problem is that any progress checkpoints are
        // unable to acquire the table lock and move forward till the calling transaction finishes.
        // Committing in line here on a fresh context ensure we can progress.
        val fiber = database.isolatedTransaction {
            val fiber = createFiber(logic, flowInitiator)
            updateCheckpoint(fiber)
            fiber
        }
        // If we are not started then our checkpoint will be picked up during start
        mutex.locked {
            if (started) {
                resumeFiber(fiber)
            }
        }
        return fiber
    }
```

## FlowStateMachineImpl.kt
[FlowStateMachineImpl.kt](https://github.com/corda/corda/blob/release-M14.0/node/src/main/kotlin/net/corda/node/services/statemachine/FlowStateMachineImpl.kt)
```
    @Suspendable
    override fun run() {
        createTransaction()
        logger.debug { "Calling flow: $logic" }
        val startTime = System.nanoTime()
        val result = try {
            logic.call()
        } catch (e: FlowException) {
            recordDuration(startTime, success = false)
            // Check if the FlowException was propagated by looking at where the stack trace originates (see suspendAndExpectReceive).
            val propagated = e.stackTrace[0].className == javaClass.name
            processException(e, propagated)
            logger.warn(if (propagated) "Flow ended due to receiving exception" else "Flow finished with exception", e)
            return
        } catch (t: Throwable) {
            recordDuration(startTime, success = false)
            logger.warn("Terminated by unexpected exception", t)
            processException(t, false)
            return
        }

        recordDuration(startTime)
        // Only sessions which have done a single send and nothing else will block here
        openSessions.values
                .filter { it.state is FlowSessionState.Initiating }
                .forEach { it.waitForConfirmation() }
        // This is to prevent actionOnEnd being called twice if it throws an exception
        actionOnEnd(Try.Success(result), false)
        _resultFuture?.set(result)
        logic.progressTracker?.currentStep = ProgressTracker.DONE
        logger.debug { "Flow finished with result ${result.toString().abbreviate(300)}" }
    }
```

## Quasar
[Fibers, Channels and Actors for the JVM](http://docs.paralleluniverse.co/quasar/)
