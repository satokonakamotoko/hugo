+++
title = "cordapp-example"
date = "2017-09-28T12:01:00+09:00"
tags = ["Corda"]
categories = ["Code Reading"]
banner = "img/banners/banner-1.jpg"
+++

Corda のドキュメント内でも紹介されていますが、Corda のリポジトリには、CordApp の example である、[cordapp-example](https://github.com/corda/cordapp-example) があります。

<!--more-->

`cordapp-example` の内容については、後ほど詳しくみていくとして、ここでは `generateFlowDiagram` という gradle タスクを実行して、[シーケンス図](example_flow.plantuml)が生成されることを確認します。（シーケンス図の生成には、PlantUM が使用されています）このシーケンス図は、CordApp の基本的なシーケンスをあらわしています。

## build.gradle
[build.gradle](https://github.com/corda/cordapp-example/blob/release-M14.0/kotlin-source/build.gradle)
```
task generateFlowDiagram(type: JavaExec) {
    classpath = files("build/jythonDeps/plantuml-8039.jar")
    main = "net.sourceforge.plantuml.Run"
    args = [
            "-tsvg",
            "-output", "../kotlin-source/build/doc",
            "-exclude", "inc_*",
            "../doc/*.plantuml"
    ]
}
```

## PlantUML
[PlantUML](https://github.com/plantuml)
