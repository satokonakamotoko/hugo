+++
title = "Node Database"
date = "2017-10-07T06:00:00+09:00"
tags = ["Corda"]
categories = ["Code Reading"]
banner = "img/banners/banner-5.jpg"
+++

さて、Corda Node は、データをどこに保存するのでしょうか。

<!--more-->

[ドキュメント](https://docs.corda.net/releases/release-M14.0/node-database.html)にはこのように書いてあるように、現在のところ、ローカルの H2 データベースに保存されています。
`Currently, nodes store their data in an H2 database. In the future, we plan to support a wide range of databases.`
ドキュメントには、H2 データベースに接続する方法が書かれているので、簡単にデータを確認することができます。

## Node.kt
[Node.kt](https://github.com/corda/corda/blob/release-M14.0/node/src/main/kotlin/net/corda/node/internal/Node.kt)
```
    override fun initialiseDatabasePersistence(insideTransaction: () -> Unit) {
        val databaseUrl = configuration.dataSourceProperties.getProperty("dataSource.url")
        val h2Prefix = "jdbc:h2:file:"
        if (databaseUrl != null && databaseUrl.startsWith(h2Prefix)) {
            val h2Port = databaseUrl.substringAfter(";AUTO_SERVER_PORT=", "").substringBefore(';')
            if (h2Port.isNotBlank()) {
                val databaseName = databaseUrl.removePrefix(h2Prefix).substringBefore(';')
                val server = org.h2.tools.Server.createTcpServer(
                        "-tcpPort", h2Port,
                        "-tcpAllowOthers",
                        "-tcpDaemon",
                        "-key", "node", databaseName)
                runOnStop += server::stop
                val url = server.start().url
                printBasicNodeInfo("Database connection url is", "jdbc:h2:$url/node")
            }
        }
        super.initialiseDatabasePersistence(insideTransaction)
    }
```

## CordaPersistence.kt
[CordaPersistence.kt](https://github.com/corda/corda/blob/release-M14.0/node/src/main/kotlin/net/corda/node/utilities/CordaPersistence.kt)
```
fun configureDatabase(props: Properties): CordaPersistence {
    val config = HikariConfig(props)
    val dataSource = HikariDataSource(config)
    val persistence = CordaPersistence.connect(dataSource)

    //org.jetbrains.exposed.sql.Database will be removed once Exposed library is removed
    val database = Database.connect(dataSource) { _ -> ExposedTransactionManager() }
    persistence.database = database

    // Check not in read-only mode.
    persistence.transaction {
        persistence.dataSource.connection.use {
            check(!it.metaData.isReadOnly) { "Database should not be readonly." }
        }
    }
    return persistence
}
```

## AttachmentsSchema.kt
[AttachmentsSchema.kt](https://github.com/corda/corda/blob/release-M14.0/node-schemas/src/main/kotlin/net/corda/node/services/persistence/schemas/requery/AttachmentsSchema.kt)
```
@Table(name = "attachments")
```

## JDBCHashMap.kt
[JDBCHashMap.kt](https://github.com/corda/corda/blob/release-M14.0/node/src/main/kotlin/net/corda/node/utilities/JDBCHashMap.kt)
```
open class JDBCHashedTable(tableName: String) : Table(tableName) {
    val keyHash = integer("key_hash").index()
    val seqNo = integer("seq_no").autoIncrement().index().primaryKey()
}
```

## DatabaseSupport.kt
[DatabaseSupport.kt](https://github.com/corda/corda/blob/release-M14.0/node/src/main/kotlin/net/corda/node/utilities/DatabaseSupport.kt)
```
/**
 * Table prefix for all tables owned by the node module.
 */
const val NODE_DATABASE_PREFIX = "node_"
```
