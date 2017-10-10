+++
title = "Shell"
date = "2017-09-29T12:00:00+09:00"
tags = ["Corda"]
categories = ["Code Reading"]
banner = "img/banners/banner-2.jpg"
+++

ここまで Node の起動プロセス、起動方法、起動設定についてみてきました。

<!--more-->

ここからは、起動した Node にアクセスする方法について見ていきましょう。 まずは Node の起動と同時にたちあがる Shell を使ってみます。`help` コマンドの出力のうち、`start` `run` `flow` が corda 独自で用意されているコマンドです。


## Shell
[Shell](https://docs.corda.net/releases/release-M14.0/shell.html)
```
>>> help   
Try one of these commands with the -h or --help switch:                                                                                                                                
                                                                                                                                                                                       
NAME      DESCRIPTION                                                                                                                                                                  
dashboard a monitoring dashboard                                                                                                                                                       
egrep     search file(s) for lines that match a pattern                                                                                                                                
env       display the term env                                                                                                                                                         
filter    a filter for a stream of map                                                                                                                                                 
java      various java language commands                                                                                                                                               
jdbc      JDBC connection                                                                                                                                                              
jndi      Java Naming and Directory Interface                                                                                                                                          
jpa       Java persistance API                                                                                                                                                         
jul       java.util.logging commands                                                                                                                                                   
jvm       JVM informations                                                                                                                                                             
less      opposite of more                                                                                                                                                             
man       format and display the on-line manual pages                                                                                                                                  
shell     shell related command                                                                                                                                                        
sleep     sleep for some time                                                                                                                                                          
sort      sort a map                                                                                                                                                                   
system    vm system properties commands                                                                                                                                                
thread    JVM thread commands                                                                                                                                                          
help      provides basic help                                                                                                                                                          
repl      list the repl or change the current repl                                                                                                                                     
start     An alias for 'flow start'                                                                                                                                                    
run       Runs a method from the CordaRPCOps interface on the node.                                                                                                                    
flow      Commands to work with flows. Flows are how you can change the ledger.  
```

## InteractiveShell.kt
[InteractiveShell.kt](https://github.com/corda/corda/blob/release-M14.0/node/src/main/kotlin/net/corda/node/shell/InteractiveShell.kt)
```
    @JvmStatic
    fun runRPCFromString(input: List<String>, out: RenderPrintWriter, context: InvocationContext<out Any>): Any? {
        val parser = StringToMethodCallParser(CordaRPCOps::class.java, context.attributes["mapper"] as ObjectMapper)

        val cmd = input.joinToString(" ").trim { it <= ' ' }
        if (cmd.toLowerCase().startsWith("startflow")) {
            // The flow command provides better support and startFlow requires special handling anyway due to
            // the generic startFlow RPC interface which offers no type information with which to parse the
            // string form of the command.
            out.println("Please use the 'flow' command to interact with flows rather than the 'run' command.", Color.yellow)
            return null
        }

        var result: Any? = null
        try {
            InputStreamSerializer.invokeContext = context
            val call = parser.parse(context.attributes["ops"] as CordaRPCOps, cmd)
            result = call.call()
            if (result != null && result !is kotlin.Unit && result !is Void) {
                result = printAndFollowRPCResponse(result, out)
            }
            if (result is Future<*>) {
                if (!result.isDone) {
                    out.println("Waiting for completion or Ctrl-C ... ")
                    out.flush()
                }
                try {
                    result = result.get()
                } catch (e: InterruptedException) {
                    Thread.currentThread().interrupt()
                } catch (e: ExecutionException) {
                    throw e.rootCause
                } catch (e: InvocationTargetException) {
                    throw e.rootCause
                }
            }
        } catch (e: StringToMethodCallParser.UnparseableCallException) {
            out.println(e.message, Color.red)
            out.println("Please try 'man run' to learn what syntax is acceptable")
        } catch (e: Exception) {
            out.println("RPC failed: ${e.rootCause}", Color.red)
        } finally {
            InputStreamSerializer.invokeContext = null
            InputStreamDeserializer.closeAll()
        }
        return result
    }
```
