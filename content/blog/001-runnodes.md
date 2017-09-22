+++
title = "runnodes"
date = "2017-09-19T06:00:00+09:00"
tags = ["Corda"]
categories = ["Code Reading"]
banner = "img/banners/banner-1.jpg"
+++

## 概要

まずは、[Quickstart](https://docs.corda.net/releases/release-M14.0/getting-set-up.html#run-from-the-command-prompt)を読んでみる。ドキュメントに従うと、runnodesというスクリプトがあらわれる。その名のとおりCorDapp（Codraで動くアプリケーションをこう呼ぶらしい）のnodeを起動するためのスクリプトのようだ。まずはこのrunnodesを起点にしてCordaのしくみを理解してみることにする。MANIFEST.MFによると、mainクラスは、NodeRunnerKtのようである。Kotlinなので、ソースファイルはNodeRunner.ktである。たしかにmainメソッドがあった。

## runnodes
[runnodes](https://github.com/corda/corda/blob/release-M14.0/gradle-plugins/cordformation/src/main/resources/net/corda/plugins/runnodes)
```$xslt
if which osascript >/dev/null; then
    /usr/libexec/java_home -v 1.8 --exec java -jar runnodes.jar "$@"
else
    "${JAVA_HOME:+$JAVA_HOME/bin/}java" -jar runnodes.jar "$@"
fi
```

```
cat META-INF/MANIFEST.MF 
Manifest-Version: 1.0
Main-Class: net.corda.plugins.NodeRunnerKt
```

## NodeRunner.kt
[NodeRunner.kt](https://github.com/corda/corda/blob/release-M14.0/gradle-plugins/cordformation/src/noderunner/kotlin/net/corda/plugins/NodeRunner.kt)
```$xslt
fun main(args: Array<String>) {
    val startedProcesses = mutableListOf<Process>()
    val headless = GraphicsEnvironment.isHeadless() || (args.isNotEmpty() && args[0] == HEADLESS_FLAG)
    val workingDir = File(System.getProperty("user.dir"))
    val javaArgs = args.filter { it != HEADLESS_FLAG }
    println("Starting nodes in $workingDir")
    workingDir.listFiles { file -> file.isDirectory }.forEach { dir ->
        listOf(NodeJarType, WebJarType).forEach { jarType ->
            jarType.acceptDirAndStartProcess(dir, headless, javaArgs)?.let { startedProcesses += it }
        }
    }
    println("Started ${startedProcesses.size} processes")
    println("Finished starting nodes")
}
```

```$xslt
private object NodeJarType : JarType("corda.jar") {
    override fun acceptNodeConf(nodeConf: File) = true
}

private object WebJarType : JarType("corda-webserver.jar") {
    // TODO: Add a webserver.conf, or use TypeSafe config instead of this hack
    override fun acceptNodeConf(nodeConf: File) = Files.lines(nodeConf.toPath()).anyMatch { "webAddress" in it }
}
```
