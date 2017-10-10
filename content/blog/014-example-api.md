+++
title = "ExampleApi"
date = "2017-10-01T12:00:00+09:00"
tags = ["Corda"]
categories = ["Code Reading"]
banner = "img/banners/banner-4.jpg"
+++

引き続き Node にアクセスする方法についてです。これまで Shell を利用する方法、CordaRPCClient を利用する方法について見てきました。 

<!--more-->

これらの方法に加えて、REST API を作成する方法があります。`cordapp-example` に、Java, Kotlin それぞれの言語で、`ExampleApi` という名前で rest api の例が用意されています。`ExampleClientRPC` の例と同様に、実際の Node へのアクセスは、`CordaRPCOps` を介して行います。

## ExampleApi.java
[ExampleApi.java](https://github.com/corda/cordapp-example/blob/release-M14.0/java-source/src/main/java/com/example/api/ExampleApi.java)
```
// This API is accessible from /api/example. All paths specified below are relative to it.
@Path("example")
public class ExampleApi {
    private final CordaRPCOps services;
```

## ExampleApi.kt
[ExampleApi.kt](https://github.com/corda/cordapp-example/blob/release-M14.0/kotlin-source/src/main/kotlin/com/example/api/ExampleApi.kt)
```
// This API is accessible from /api/example. All paths specified below are relative to it.
@Path("example")
class ExampleApi(val services: CordaRPCOps) {
```
