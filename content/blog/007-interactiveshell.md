+++
title = "InteractiveShell"
date = "2017-09-20T13:00:00+09:00"
tags = ["Corda"]
categories = ["Code Reading"]
banner = "img/banners/banner-2.jpg"
+++

Corda ノードの起動と同時に、[Shell](https://docs.corda.net/releases/release-M14.0/shell.html) が立ち上がります。

<!--more-->

Shell には、Java のための shell である CRaSH に、パッチをあてたものが使われます。この Shell 関連のクラスのいくつかは、CRaSH の制限のため Kotlin ではなく、Java で書かれています。

## InteractiveShell.kt
[InteractiveShell.kt](https://github.com/corda/corda/blob/release-M14.0/node/src/main/kotlin/net/corda/node/shell/InteractiveShell.kt)
```
    fun startShell(dir: Path, runLocalShell: Boolean, runSSHServer: Boolean, node: Node) {
        this.node = node
        var runSSH = runSSHServer

        val config = Properties()
        if (runSSH) {
            // TODO: Finish and enable SSH access.
            // This means bringing the CRaSH SSH plugin into the Corda tree and applying Marek's patches
            // found in https://github.com/marekdapps/crash/commit/8a37ce1c7ef4d32ca18f6396a1a9d9841f7ff643
            // to that local copy, as CRaSH is no longer well maintained by the upstream and the SSH plugin
            // that it comes with is based on a very old version of Apache SSHD which can't handle connections
            // from newer SSH clients. It also means hooking things up to the authentication system.
            Node.printBasicNodeInfo("SSH server access is not fully implemented, sorry.")
            runSSH = false
        }

        if (runSSH) {
            // Enable SSH access. Note: these have to be strings, even though raw object assignments also work.
            config["crash.ssh.keypath"] = (dir / "sshkey").toString()
            config["crash.ssh.keygen"] = "true"
            // config["crash.ssh.port"] = node.configuration.sshdAddress.port.toString()
            config["crash.auth"] = "simple"
            config["crash.auth.simple.username"] = "admin"
            config["crash.auth.simple.password"] = "admin"
        }

        ExternalResolver.INSTANCE.addCommand("run", "Runs a method from the CordaRPCOps interface on the node.", RunShellCommand::class.java)
        ExternalResolver.INSTANCE.addCommand("flow", "Commands to work with flows. Flows are how you can change the ledger.", FlowShellCommand::class.java)
        ExternalResolver.INSTANCE.addCommand("start", "An alias for 'flow start'", StartShellCommand::class.java)
        val shell = ShellLifecycle(dir).start(config)

        if (runSSH) {
            // printBasicNodeInfo("SSH server listening on address", node.configuration.sshdAddress.toString())
        }

        // Possibly bring up a local shell in the launching terminal window, unless it's disabled.
        if (!runLocalShell)
            return
        // TODO: Automatically set up the JDBC sub-command with a connection to the database.
        val terminal = TerminalFactory.create()
        val consoleReader = ConsoleReader("Corda", FileInputStream(FileDescriptor.`in`), System.out, terminal)
        val jlineProcessor = JLineProcessor(terminal.isAnsiSupported, shell, consoleReader, System.out)
        InterruptHandler { jlineProcessor.interrupt() }.install()
        thread(name = "Command line shell processor", isDaemon = true) {
            // Give whoever has local shell access administrator access to the node.
            CURRENT_RPC_CONTEXT.set(RpcContext(User(ArtemisMessagingComponent.NODE_USER, "", setOf())))
            Emoji.renderIfSupported {
                jlineProcessor.run()
            }
        }
        thread(name = "Command line shell terminator", isDaemon = true) {
            // Wait for the shell to finish.
            jlineProcessor.closed()
            log.info("Command shell has exited")
            terminal.restore()
            node.stop()
        }
    }
```

## Corda Shell
[This is a patch set on top of the excellent but unmaintained CRaSH project ](https://github.com/corda/crash)

## RunShellCommand.java
[RunShellCommand.java](https://github.com/corda/corda/blob/release-M14.0/node/src/main/java/net/corda/node/shell/RunShellCommand.java)
```
// Note that this class cannot be converted to Kotlin because CRaSH does not understand InvocationContext<Map<?, ?>> which
// is the closest you can get in Kotlin to raw types.
```

