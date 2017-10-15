+++
title = "reference.conf"
date = "2017-10-08T06:00:00+09:00"
tags = ["Corda"]
categories = ["Code Reading"]
banner = "img/banners/banner-1.jpg"
+++

Corda Node が、データをローカルの H2 データベースに保存することはわかりました。

<!--more-->

ではこのデータベースが、具体的にどこのディレクトリにどのようなファイルとして存在するのか確認してみましょう。default の設定ファイルとして、`reference.conf` というファイルが用意されていることがわかります。

## reference.conf
[reference.conf](https://github.com/corda/corda/blob/release-M14.0/node/src/main/resources/reference.conf)
```
"dataSource.url" = "jdbc:h2:file:"${basedir}"/persistence;DB_CLOSE_ON_EXIT=FALSE;LOCK_TIMEOUT=10000;WRITE_DELAY=100;AUTO_SERVER_PORT="${h2port}
```

## ConfigUtilities.kt
[ConfigUtilities.kt](https://github.com/corda/corda/blob/release-M14.0/node/src/main/kotlin/net/corda/node/services/config/ConfigUtilities.kt)
```
    fun loadConfig(baseDirectory: Path,
                   configFile: Path = baseDirectory / "node.conf",
                   allowMissingConfig: Boolean = false,
                   configOverrides: Config = ConfigFactory.empty()): Config {
        val parseOptions = ConfigParseOptions.defaults()
        val defaultConfig = ConfigFactory.parseResources("reference.conf", parseOptions.setAllowMissing(false))
        val appConfig = ConfigFactory.parseFile(configFile.toFile(), parseOptions.setAllowMissing(allowMissingConfig))
        val finalConfig = configOf(
                // Add substitution values here
                "basedir" to baseDirectory.toString())
                .withFallback(configOverrides)
                .withFallback(appConfig)
                .withFallback(defaultConfig)
                .resolve()
        log.info("Config:\n${finalConfig.root().render(ConfigRenderOptions.defaults())}")
        return finalConfig
    }
```
