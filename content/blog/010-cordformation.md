+++
title = "Cordformation"
date = "2017-09-28T12:00:00+09:00"
tags = ["Corda"]
categories = ["Code Reading"]
banner = "img/banners/banner-5.jpg"
+++

続いて Node の設定について見てみましょう。Node の設定ファイルとして node.conf が使用されていることがわかります。

<!--more-->

`node.conf` の設定項目は、[こちら](https://docs.corda.net/releases/release-M14.0/corda-configuration-file.html)にまとめられています。そして、この `node.conf` の生成や、その他 node の起動に関する様々なタスクをサポートする `Cordformation` という gradle プラグインが用意されています。

## build.gradle
[build.gradle](https://github.com/corda/cordapp-example/blob/release-M14.0/kotlin-source/build.gradle)
```
task deployNodes(type: net.corda.plugins.Cordform, dependsOn: ['jar']) {
    directory "./build/nodes"
    networkMap "CN=Controller,O=R3,OU=corda,L=London,C=UK"
    node {
        name "CN=Controller,O=R3,OU=corda,L=London,C=UK"
        advertisedServices = ["corda.notary.validating"]
        p2pPort 10002
        rpcPort 10003
        webPort 10004
        cordapps = []
    }
    node {
        name "CN=NodeA,O=NodeA,L=London,C=UK"
        advertisedServices = []
        p2pPort 10005
        rpcPort 10006
        webPort 10007
        cordapps = []
        rpcUsers = [[ user: "user1", "password": "test", "permissions": []]]
    }
    node {
        name "CN=NodeB,O=NodeB,L=New York,C=US"
        advertisedServices = []
        p2pPort 10008
        rpcPort 10009
        webPort 10010
        cordapps = []
        rpcUsers = [[ user: "user1", "password": "test", "permissions": []]]
    }
    node {
        name "CN=NodeC,O=NodeC,L=Paris,C=FR"
        advertisedServices = []
        p2pPort 10011
        rpcPort 10012
        webPort 10013
        cordapps = []
        rpcUsers = [[ user: "user1", "password": "test", "permissions": []]]
    }
}
```

## Cordformation.groovy
[Cordformation.groovy](https://github.com/corda/corda/blob/release-M14.0/gradle-plugins/cordformation/src/main/groovy/net/corda/plugins/Cordformation.groovy)
```
/**
 * The Cordformation plugin deploys nodes to a directory in a state ready to be used by a developer for experimentation,
 * testing, and debugging. It will prepopulate several fields in the configuration and create a simple node runner.
 */
class Cordformation implements Plugin<Project> {
```
