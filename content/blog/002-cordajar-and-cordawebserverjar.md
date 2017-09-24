+++
title = "corda.jar and corda-webserver.jar"
date = "2017-09-19T12:00:00+09:00"
tags = ["Corda"]
categories = ["Code Reading"]
banner = "img/banners/banner-2.jpg"
+++

[CorDapp](https://docs.corda.net/releases/release-M14.0/cordapp-overview.html) には、2つの jar ファイルが組み込まれます。

<!--more-->

1つは、corda-webserver.jar もう1つは、corda.jar です。これら2つの jar ファイルは、Capsule というツールによってパッケージングされます。

## build.gradle(corda-webserver.jar)
[build.gradle](https://github.com/corda/corda/blob/release-M14.0/webserver/webcapsule/build.gradle)
```$xslt
task buildWebserverJar(type: FatCapsule, dependsOn: project(':node').compileJava) {
    applicationClass 'net.corda.webserver.WebServer'
    archiveName "corda-webserver-${corda_release_version}.jar"
    applicationSource = files(
            project(':webserver').configurations.runtime,
            project(':webserver').jar,
            new File(project(':node').buildDir, 'classes/main/CordaCaplet.class'),
            new File(project(':node').buildDir, 'classes/main/CordaCaplet$1.class'),
            "$rootDir/config/dev/log4j2.xml"
    )
    from 'NOTICE' // Copy CDDL notice

    capsuleManifest {
        applicationVersion = corda_release_version
        javaAgents = ["quasar-core-${quasar_version}-jdk8.jar"]
        systemProperties['visualvm.display.name'] = 'Corda Webserver'
        minJavaVersion = '1.8.0'
        minUpdateVersion['1.8'] = java8_minUpdateVersion
        caplets = ['CordaCaplet']

        // JVM configuration:
        // - Constrain to small heap sizes to ease development on low end devices.
        // - Switch to the G1 GC which is going to be the default in Java 9 and gives low pause times/string dedup.
        //
        // If you change these flags, please also update Driver.kt
        jvmArgs = ['-Xmx200m', '-XX:+UseG1GC']
    }
}
```

## Capsule
[Packaging and Deployment for JVM Apps](http://www.capsule.io/)

## build.gradle(corda.jar)
[build.gradle](https://github.com/corda/corda/blob/release-M14.0/node/capsule/build.gradle)
```$xslt
task buildCordaJAR(type: FatCapsule, dependsOn: project(':node').compileJava) {
    applicationClass 'net.corda.node.Corda'
    archiveName "corda-${corda_release_version}.jar"
    applicationSource = files(
            project(':node').configurations.runtime,
            project(':node').jar,
            '../build/classes/main/CordaCaplet.class',
            '../build/classes/main/CordaCaplet$1.class',
            "$rootDir/config/dev/log4j2.xml"
    )
    from 'NOTICE' // Copy CDDL notice


    capsuleManifest {
        applicationVersion = corda_release_version
        appClassPath = ["jolokia-agent-war-${project.rootProject.ext.jolokia_version}.war"]
        // TODO add this once we upgrade quasar to 0.7.8
        // def quasarExcludeExpression = "x(rx**;io**;kotlin**;jdk**;reflectasm**;groovyjarjarasm**;groovy**;joptsimple**;groovyjarjarantlr**;javassist**;com.fasterxml**;com.typesafe**;com.google**;com.zaxxer**;com.jcabi**;com.codahale**;com.esotericsoftware**;de.javakaffee**;org.objectweb**;org.slf4j**;org.w3c**;org.codehaus**;org.h2**;org.crsh**;org.fusesource**;org.hibernate**;org.dom4j**;org.bouncycastle**;org.apache**;org.objenesis**;org.jboss**;org.xml**;org.jcp**;org.jetbrains**;org.yaml**;co.paralleluniverse**;net.i2p**)"
        // javaAgents = ["quasar-core-${quasar_version}-jdk8.jar=${quasarExcludeExpression}"]
        javaAgents = ["quasar-core-${quasar_version}-jdk8.jar"]
        systemProperties['visualvm.display.name'] = 'Corda'
        minJavaVersion = '1.8.0'
        minUpdateVersion['1.8'] = java8_minUpdateVersion
        caplets = ['CordaCaplet']

        // JVM configuration:
        // - Constrain to small heap sizes to ease development on low end devices.
        // - Switch to the G1 GC which is going to be the default in Java 9 and gives low pause times/string dedup.
        //
        // If you change these flags, please also update Driver.kt
        jvmArgs = ['-Xmx200m', '-XX:+UseG1GC']
    }

    // Make the resulting JAR file directly executable on UNIX by prepending a shell script to it.
    // This lets you run the file like so: ./corda.jar
    // Other than being slightly less typing, this has one big advantage: Ctrl-C works properly in the terminal.
    reallyExecutable { trampolining() }
}
```
