---
title: go语言操作mongodb
categories: [技术, Golang]
tags: [mongodb]
---


## 安装

```shell
# 初始化go模块，取名为learn-mongo
mkdir learn-mongo && cd learn-mongo
go mod init learn-mongo

# 获取go mongo模块依赖
go get go.mongodb.org/mongo-driver/mongo
```

## 客户端连接

客户端连接主要有两种方式：

1.  先New一个客户端实例，然后进行Connect；
2.  直接Connect的同时获得一个客户端实例。 通常使用第二种方法，代码更为简洁。

对`mongo`的任何操作，包括`Connect`、`CURD`、`Disconnect`等都离不开一个操作的上下文`Context`环境，需要一个`Context`实例作为操作的第一个参数。

## 使用NewClient实例连接

```GO
package main

import (
  "context"
  "fmt"
  "time"

  "go.mongodb.org/mongo-driver/mongo"
  "go.mongodb.org/mongo-driver/mongo/options"
)

func main() {
  client, err := mongo.NewClient(options.Client().ApplyURI("mongodb://localhost:27017"))
  if err != nil {
    fmt.Errorf("client establish failed. err: %v", err)
  }
  // ctx
  ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
  defer cancel()

  // connect
  if err = client.Connect(ctx); err == nil {
    fmt.Println("connect to db success.")
  }

  // 实例化client后，延迟调用断开连接函数
  defer func() {
    if err = client.Disconnect(ctx); err != nil {
      panic(err)
    }
  }()
}
```

## 使用mongo.Connect()函数连接

### 直接连接

```go
package main

import (
  "context"
  "go.mongodb.org/mongo-driver/mongo"
  "go.mongodb.org/mongo-driver/mongo/options"
  "log"
)

func main() {
  clientOpts := options.Client().ApplyURI("mongodb://localhost:27017/?connect=direct")
  client, err := mongo.Connect(context.TODO(), clientOpts)
  if err != nil {
      log.Fatal(err)
  }
}
```

### 用户名密码认证连接

```GO
package main

import (
  "context"
  "go.mongodb.org/mongo-driver/mongo"
  "go.mongodb.org/mongo-driver/mongo/options"
  "log"
)

func main() {
  credential := options.Credential{

      Username: "username",
      Password: "password",
  }
  clientOpts := options.Client().ApplyURI("mongodb://localhost:27017").SetAuth(credential)
  // 上述可以直接使用带用户名和密码的uri连接
  // clientOpts := options.Client().ApplyURI("mongodb://username:password@localhost:27017")
  client, err := mongo.Connect(context.TODO(), clientOpts)
  if err != nil {
      log.Fatal(err)
  }
}
```

### replicaSet副本集连接

```GO
package main

import (
  "context"
  "go.mongodb.org/mongo-driver/mongo"
  "go.mongodb.org/mongo-driver/mongo/options"
  "log"
)

func main() {
  clientOpts := options.Client().ApplyURI("mongodb://localhost:27017,localhost:27018/?replicaSet=replset")
  client, err := mongo.Connect(context.TODO(), clientOpts)
  if err != nil {
      log.Fatal(err)
  }
}
```

### shard分片连接

```GO
package main

import (
  "context"
  "go.mongodb.org/mongo-driver/mongo"
  "go.mongodb.org/mongo-driver/mongo/options"
  "log"
)

func main() {
  clientOpts := options.Client().ApplyURI("mongodb://localhost:27017,localhost:27018")
  client, err := mongo.Connect(context.TODO(), clientOpts)
  if err != nil {
      log.Fatal(err)
  }
}
```

## 模块配置式连接

在实际项目中，通常都会有配置模块，因此最常用的连接方式是将连接配置写在专门的配置模块文件中：

```go
package config

import (
  "time"

  "go.mongodb.org/mongo-driver/mongo/options"
  "go.mongodb.org/mongo-driver/mongo/readpref"
)

// MONGO SETTINGS
var (
  credentials = options.Credential{
    AuthMechanism: "SCRAM-SHA-1",
    AuthSource:    "anquan",
    Username:      "ysj",
    Password:      "123456",
  }
  // direct                = true
  connectTimeout        = 10 * time.Second
  hosts                 = []string{"localhost:27017", "localhost:27018"}
  maxPoolSize    uint64 = 20
  minPoolSize    uint64 = 5
  readPreference        = readpref.Primary()
  replicaSet            = "replicaSetDb"

  // ClientOpts mongoClient 连接客户端参数
  ClientOpts = &options.ClientOptions{
    Auth:           &credentials,
    ConnectTimeout: &connectTimeout,
    //Direct:         &direct,
    Hosts:          hosts,
    MaxPoolSize:    &maxPoolSize,
    MinPoolSize:    &minPoolSize,
    ReadPreference: readPreference,
    ReplicaSet:     &replicaSet,
  }
)
```

上述写法虽然没有错，但显得过于臃肿，可以使用更优雅简洁的链式调用：

```go
package config

import (
	"time"

	"go.mongodb.org/mongo-driver/mongo/options"
	"go.mongodb.org/mongo-driver/mongo/readpref"
)
// ClientOpts mongoClient 连接客户端参数
var ClientOpts = options.Client().
  SetAuth(options.Credential{
      AuthMechanism: "SCRAM-SHA-1",
      AuthSource:    "anquan",
      Username:      "ysj",
      Password:      "123456",
  }).
  SetConnectTimeout(10 * time.Second).
  SetHosts([]string{"localhost:27017"}).
  SetMaxPoolSize(20).
  SetMinPoolSize(5).
  SetReadPreference(readpref.Primary()).
  SetReplicaSet("replicaSetDb")
```

然后主模块中引入配置

```go
package main

import (
  "context"
  "log"
  "mongo-notes/config" // 引入配置模块

  "go.mongodb.org/mongo-driver/mongo"
)

func main(){
  client, err := mongo.Connect(context.TODO(), config.ClientOpts)
  if err != nil {
    log.Fatal(err)
  }
}
```

## CURD操作

## 先了解bson

使用`mongo-driver`操作`mongodb`需要用到该模块提供的`bson`。主要用来写查询的筛选条件filter、构造文档记录以及接收查询解码的值，也就是在go与mongo之间做`序列化`。其中，基本只会用到如下三种数据结构：

+   `bson.D{}`: 对文档(Document)的**有序**描述，key-value以逗号分隔；
+   `bson.M{}`: Map结构，key-value以冒号分隔，**无序**，使用最方便；
+   `bson.A{}`: 数组结构，元素要求是**有序**的文档描述，也就是元素是`bson.D{}`类型。

## 获取db

```GO
// 获取db
db := client.Database("test")
collectionNames, err := db.ListCollectionNames(ctx, bson.M{})
fmt.Println("collectionNames:", collectionNames)
```

## 获取collection

```GO
// 获取collecton
collection := db.Collection("person")
fmt.Printf("collection: %s \n", collection.Name())
```

### 创建索引

```GO
// 创建索引
indexView := collection.Indexes()
indexName := "index_by_name"
indexBackground := true
indexUnique := true

indexModel := mongo.IndexModel{
  Keys: bson.D{{"name", 1}},
  Options: &options.IndexOptions{
    Name:       &indexName,
    Background: &indexBackground,
    Unique:     &indexUnique,
  },
}
index, err := indexView.CreateOne(ctx, indexModel)
if err != nil {
  log.Fatalf("index created failed. err: %v \n", err)
  return
}
fmt.Println("new index name:", index)
```

### 查看索引

```GO
// 查看索引
indexCursor, err := collection.Indexes().List(ctx)
var indexes []bson.M
if err = indexCursor.All(ctx, &indexes); err != nil {
  log.Fatal(err)
}
fmt.Println(indexes)
```

## Insert

### InsertOne

```GO
collection := client.Database("test").Collection("person")
// InsertOne
insertOneResult, err := collection.InsertOne(ctx, bson.M{"name": "虎子", "gender": "男", "level": 1})
if err != nil {
  log.Fatal(err)
}
fmt.Println("id:", insertOneResult.InsertedID)
```

### InsertMany

```go
// InsertMany
docs := []interface{}{
  bson.M{"name": "5t5", "gender": "男", "level": 0},
  bson.M{"name": "奈奈米", "gender": "男", "level": 1},
}
// Ordered 设置为false表示其中一条插入失败不会影响其他文档的插入，默认为true，一条失败其他都不会被写入
insertManyOpts := options.InsertMany().SetOrdered(false)
insertManyResult, err := collection.InsertMany(ctx, docs, insertManyOpts)
if err != nil {
  log.Fatal(err)
}
fmt.Println("ids:", insertManyResult.InsertedIDs)
```

## Find

### Find

```go
import (
  "go.mongodb.org/mongo-driver/bson"
)
// Find
findOpts := options.Find().SetSort(bson.D{{"name", 1}})
findCursor, err := collection.Find(ctx, bson.M{"level": 0}, findOpts)
var results []bson.M
if err = findCursor.All(ctx, &results); err != nil {
  log.Fatal(err)
}
for _, result := range results {
  fmt.Println(result)
}
```

### FindOne

```go
import (
  "go.mongodb.org/mongo-driver/bson"
)
// FindOne

// 可以使用struct来接收解码结果

// var result struct {
// 	Name   string
// 	Gender string
// 	Level  int
// }

// 或者更好的是直接使用bson.M
var result bson.M

// 按照name排序并跳过第一个, 且只需返回name、level字段
findOneOpts := options.FindOne().SetSkip(1).SetSort(bson.D{{"name", 1}}).SetProjection(bson.D{{"name", 1}, {"level", 1}})

singleResult := collection.FindOne(ctx, bson.M{"name": "5t5"}, findOneOpts)
if err = singleResult.Decode(&result); err == nil {
  fmt.Printf("result: %+v\n", result)
}
```

### FindOneAndDelete

```go
import (
  "go.mongodb.org/mongo-driver/bson"
)
// FindOneAndDelete
findOneAndDeleteOpts := options.FindOneAndDelete().SetProjection(bson.D{{"name", 1}, {"level", 1}})
var deletedDoc bson.M
singleResult := collection.FindOneAndDelete(ctx, bson.D{{"name", "虎子"}}, findOneAndDeleteOpts)
if err = singleResult.Decode(&deletedDoc); err != nil {
  if err == mongo.ErrNoDocuments {
      return
  }
  log.Fatal(err)
}
fmt.Printf("deleted document: %+v \n", deletedDoc)
```

### FindOneAndReplace

```go
// FindOneAndReplace
// 注意： 返回的是被替换前的document，满足条件的doc将会被完全替换为replaceMent
_id, err := primitive.ObjectIDFromHex("5fde05b9612cb3d19c4b25e8")
findOneAndReplaceOpts := options.FindOneAndReplace().SetUpsert(true)
replaceMent := bson.M{"name": "5t5", "skill": "六眼"}
var replaceMentDoc bson.M
err = collection.FindOneAndReplace(ctx, bson.M{"_id": _id}, replaceMent , findOneAndReplaceOpts).Decode(&replaceMentDoc)
if err != nil {
  if err==mongo.ErrNoDocuments {
    return
  }
  log.Fatal(err)
}
fmt.Printf("document before replacement: %v \n", replaceMentDoc)
```

### FindOneAndUpdate

```go
// FindOneAndUpdate
// 注意：返回的结果仍然是更新前的document
_id, err := primitive.ObjectIDFromHex("5fde05b9612cb3d19c4b25e8")
findOneAndUpdateOpts := options.FindOneAndUpdate().SetUpsert(true)
update := bson.M{"$set": bson.M{"level": 0, "gender": "男"}}
var toUpdateDoc bson.M
err = collection.FindOneAndUpdate(ctx, bson.M{"_id": _id}, update, findOneAndUpdateOpts).Decode(&toUpdateDoc)
if err != nil {
  if err == mongo.ErrNoDocuments {
    return
  }
  log.Fatal(err)
}
fmt.Printf("document before updating: %v \n", toUpdateDoc)

```

## Update

### UpdateOne

```go
//UpdateOne
updateOneOpts := options.Update().SetUpsert(true)
updateOneFilter := bson.M{"name": "娜娜明"}
updateOneSet := bson.M{"$set": bson.M{"skill": "三七分"}}
updateResult, err := collection.UpdateOne(ctx, updateOneFilter, updateOneSet, updateOneOpts)
fmt.Printf(
  "matched: %d  modified: %d  upserted: %d  upsertedID: %v\n",
  updateResult.MatchedCount,
  updateResult.ModifiedCount,
  updateResult.UpsertedCount,
  updateResult.UpsertedID,
)
```

### UpdateMany

```go
// UpdateMany
updateManyFilter := bson.M{"name": "虎子"}
updateManySet := bson.M{"$set": bson.M{"level": 1}}
updateManyResult, err := collection.UpdateMany(ctx,updateManyFilter, updateManySet)
fmt.Printf(
  "matched: %d  modified: %d  upserted: %d  upsertedID: %v\n",
  updateManyResult.MatchedCount,
  updateManyResult.ModifiedCount,
  updateManyResult.UpsertedCount,
  updateManyResult.UpsertedID,
)
```

### ReplaceOne

```go
// ReplaceOne
// 就影响数据层面和FindOneAndReplace没有差别
replaceOpts := options.Replace().SetUpsert(true)
replaceMent := bson.M{"name": "小黑", "level": 2, "gender": "男"}
updateResult, err := collection.ReplaceOne(ctx, bson.M{"name": "虎子"}, replaceMent, replaceOpts)
fmt.Printf(
  "matched: %d  modified: %d  upserted: %d  upsertedID: %v\n",
  updateResult.MatchedCount,
  updateResult.ModifiedCount,
  updateResult.UpsertedCount,
  updateResult.UpsertedID,
)
```

## Delete

### DeleteOne

```go
// DeleteOne
deleteOneOpts := options.Delete().SetCollation(&options.Collation{
  // 忽略大小写
  CaseLevel: false,
})
deleteResult, err := collection.DeleteOne(ctx, bson.D{{"name", "虎子"}}, deleteOneOpts)
fmt.Println("deletet count:", deleteResult.DeletedCount)
```

### DeleteMany

```go
// DeleteMany
deleteManyOpts := options.Delete().SetCollation(&options.Collation{
  // 忽略大小写
  CaseLevel: false,
})
deleteManyResult, err := collection.DeleteMany(ctx, bson.D{{"name", "虎子"}}, deleteOneOpts)
fmt.Println("deletet count:", deleteManyResult.DeletedCount)
```

## 批量操作BulkWrite

```go
// BulkWrite
names := []string{"5t5", "娜娜明", "小黑", "蔷薇", "虎子"}
models := []mongo.WriteModel{}
updateOperation := bson.M{"$set": bson.M{"Animation": "咒术回战"}}
for _, name := range names {
  updateOneModel := mongo.NewUpdateOneModel().SetFilter(bson.M{"name": name}).SetUpdate(updateOperation).SetUpsert(true)
  models = append(models, updateOneModel)
}
bulkWriteOpts := options.BulkWrite().SetOrdered(false)
bulkWriteResults, err := collection.BulkWrite(ctx, models, bulkWriteOpts)
if err != nil {
  log.Fatal(err)
}
fmt.Printf(
  "matched: %d  modified: %d  upserted: %d  upsertedIDs: %v\n",
  bulkWriteResults.MatchedCount,
  bulkWriteResults.ModifiedCount,
  bulkWriteResults.UpsertedCount,
  bulkWriteResults.UpsertedIDs,
)
```

## 事务transaction

事务能保证我们的操作要么都成功，要么都失败。使用事务最大的好处就是避免某一个操作失败对现有数据库环境造成影响，曾听一个朋友吐槽因为没有使用事务，操作失败后只能去生产数据库里一条一条的删数据。所以，使用事务是一个良好的习惯，尽管代码看起来会臃肿一点。

`mongo-driver`的事务需要使用`session`开启，任何事务操作需要处于`session context`中。以下是利用不同的api进行事物操作的比较：

| 使用不同api | 主动session管理 | 主动sessionContext管理 | 主动事务abort/commit | 便利性 |
| --- | --- | --- | --- | --- |
| mongo.NewSessionContext | 是 | 是 | 是 | \* |
| mongo.WithSession | 是 | 否 | 是 | \*\* |
| client.UseSessionWithOptions | 否 | 否 | 是 | \*\*\* |
| session.WithTransaction | 是 | 否 | 否 | \*\*\*\* |

## mongo.NewSessionContext

```go
package main

import (
  "context"
  "fmt"
  "log"
  "mongo-notes/config"
  "time"

  "go.mongodb.org/mongo-driver/bson"
  "go.mongodb.org/mongo-driver/mongo"
)

func main() {
  // ctx
  ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
  defer cancel()

  // client
  client, err := mongo.Connect(ctx, config.ClientOpts)
  if err != nil {
    log.Fatal(err)
  }

  // collection
  collection := client.Database("test").Collection("person")

  // session
  session, err := client.StartSession()
  if err != nil {
    panic(err)
  }
  defer session.EndSession(context.TODO())

  // session context
  sessionCtx := mongo.NewSessionContext(ctx, session)

  // session开启transaction
  if err = session.StartTransaction(); err != nil {
      panic(err)
  }
  // transaction: insertOne in sessionCtx
  insertOneResult, err := collection.InsertOne(sessionCtx, bson.D{{"name", "大boss"}})
  if err != nil {
    // 使用context.Background()可以保证abort能够成功，即使mongo的ctx已经超时
    _ = session.AbortTransaction(context.Background())
    panic(err)
  }

  // transaction: findOne in sessionCtx
  var result bson.M
  if err = collection.FindOne(sessionCtx, bson.D{{"_id", insertOneResult.InsertedID}}).Decode(&result); err != nil {
    //使用context.Background()可以保证abort能够成功，即使mongo的ctx已经超时
    _ = session.AbortTransaction(context.Background())
    panic(err)
  }
  fmt.Printf("result: %v\n", result)

  // 使用context.Background()可以保证commit能够成功，即使mongo的ctx已经超时
  if err = session.CommitTransaction(context.Background()); err != nil {
    panic(err)
  }
}

```

## mongo.WithSession

```go
package main

import (
  "context"
  "fmt"
  "log"
  "mongo-notes/config"
  "time"

  "go.mongodb.org/mongo-driver/bson"
  "go.mongodb.org/mongo-driver/mongo"
  "go.mongodb.org/mongo-driver/mongo/options"
  "go.mongodb.org/mongo-driver/mongo/readconcern"
)

func main() {
  // ctx
  ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
  defer cancel()

  // client
  client, err := mongo.Connect(ctx, config.ClientOpts)
  if err != nil {
      log.Fatal(err)
  }

  // collection
  collection := client.Database("test").Collection("person")

  // session
  sessionOpts := options.Session().SetDefaultReadConcern(readconcern.Majority())
  session, err := client.StartSession(sessionOpts)
  if err != nil {
      log.Fatal(err)
  }
  defer session.EndSession(context.TODO())

  // transaction
  err = mongo.WithSession(ctx, session, func(sessionCtx mongo.SessionContext) error {

    if err := session.StartTransaction(); err != nil {
        return err
    }

    insertOneResult, err := collection.InsertOne(sessionCtx, bson.D{{"name", "小boss"}})
    if err != nil {
        // 使用context.Background()可以保证abort能够成功，即使mongo的ctx已经超时
        _ = session.AbortTransaction(context.Background())
        return err
    }

    var result bson.M
    if err = collection.FindOne(sessionCtx, bson.D{{"_id", insertOneResult.InsertedID}}).Decode(&result); err != nil {
        // 使用context.Background()可以保证abort能够成功，即使mongo的ctx已经超时
        _ = session.AbortTransaction(context.Background())
        return err
    }
    fmt.Println(result)

    // 使用context.Background()可以保证commit能够成功，即使mongo的ctx已经超时
    return session.CommitTransaction(context.Background())
  })
}
```

## client.UseSessionWithOptions

```go
package main

import (
  "context"
  "fmt"
  "log"
  "mongo-notes/config"
  "time"

  "go.mongodb.org/mongo-driver/bson"
  "go.mongodb.org/mongo-driver/mongo"
  "go.mongodb.org/mongo-driver/mongo/options"
  "go.mongodb.org/mongo-driver/mongo/readconcern"
)

func main() {
  // ctx
  ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
  defer cancel()

  // client
  client, err := mongo.Connect(ctx, config.ClientOpts)
  if err != nil {
    log.Fatal(err)
  }

  // collection
  collection := client.Database("test").Collection("person")

  // session 
  sessionOpts := options.Session().SetDefaultReadConcern(readconcern.Majority())
  // transaction
  err = client.UseSessionWithOptions(ctx, sessionOpts, func(sessionCtx mongo.SessionContext) error {

    if err := sessionCtx.StartTransaction(); err != nil {
        return err
    }

    insertOneResult, err := collection.InsertOne(sessionCtx, bson.D{{"name", "战五渣"}})
    if err != nil {
        // 使用context.Background()可以保证abort能够成功，即使mongo的ctx已经超时
        _ = sessionCtx.AbortTransaction(context.Background())
        return err
    }

    var result bson.M
    if err = collection.FindOne(sessionCtx, bson.D{{"_id", insertOneResult.InsertedID}}).Decode(&result); err != nil {
       _ = sessionCtx.AbortTransaction(context.Background())
       return err
    }
    fmt.Println(result)

    // 使用context.Background()可以保证commit能够成功，即使mongo的ctx已经超时
    return sessionCtx.CommitTransaction(context.Background())
  })
  if err != nil {
      log.Fatal(err)
  }

}

```

## session.WithTransaction

```go
package main

import (
  "context"
  "fmt"
  "log"
  "mongo-notes/config"
  "time"

  "go.mongodb.org/mongo-driver/bson"
  "go.mongodb.org/mongo-driver/mongo"
  "go.mongodb.org/mongo-driver/mongo/options"
  "go.mongodb.org/mongo-driver/mongo/readconcern"
  "go.mongodb.org/mongo-driver/mongo/readpref"
)

func main() {
  // ctx
  ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
  defer cancel()

  client, err := mongo.Connect(ctx, config.ClientOpts)
  if err != nil {
    log.Fatal(err)
  }
  // collection
  collection := client.Database("test").Collection("person")

  // session 读取策略
  sessOpts := options.Session().SetDefaultReadConcern(readconcern.Majority())
  session, err := client.StartSession(sessOpts)
  if err != nil {
    log.Fatal(err)
  }
  defer session.EndSession(context.TODO())

  // transaction 读取优先级
  transacOpts := options.Transaction().SetReadPreference(readpref.Primary())
  // 插入一条记录、查找一条记录在同一个事务中
  result, err := session.WithTransaction(ctx, func(sessionCtx mongo.SessionContext) (interface{}, error) {
    // insert one
    insertOneResult, err := collection.InsertOne(sessionCtx, bson.M{"name": "无名小角色", "level": 5})
    if err != nil {
      log.Fatal(err)
    }
    fmt.Println("inserted id:", insertOneResult.InsertedID)

    // find one
    var result struct {
      Name  string `bson:"name,omitempty"`
      Level int    `bson:"level,omitempty"`
    }
    singleResult := collection.FindOne(sessionCtx, bson.M{"name": "无名小角色"})
    if err = singleResult.Decode(&result); err != nil {
      return nil, err
    }

    return result, err

  }, transacOpts)

  if err != nil {
      log.Fatal(err)
  }
  fmt.Printf("find one result: %+v \n", result)
}
```

> 注意： 如果数据库中不存在有`name`为`"无名小角色"`的记录，上述操作都会失败，尽管在事务中先插入了一条`name`相同的记录，但事务是一个整体，查找的是事务开始之前的记录，因此查找会失败从而导致插入操作同样会失败。

## Distinct和Count

## Distinct

```GO
// Distinct
distinctOpts := options.Distinct().SetMaxTime(2 * time.Second)
// 返回所有不同的人名
distinctValues, err := collection.Distinct(ctx, "name", bson.M{}, distinctOpts)
if err != nil {
  log.Fatal(err)
}
for _, value := range distinctValues {
  fmt.Println(value)
}
```

## Count

```go
// EstimatedDocumentCount
totalCount, err := collection.EstimatedDocumentCount(ctx)
if err != nil {
  log.Fatal(err)
}
fmt.Println("totalCount:", totalCount)

// CountDocuments
count, err := collection.CountDocuments(ctx, bson.M{"name": "5t5"})
if err != nil {
  log.Fatal(err)
}
fmt.Println("count:", count)
```

## 聚合Aggregate

```go
// Aggregate
// 按性别分组求和，统计出现次数
groupStage := bson.D{
  {"$group", bson.D{
    {"_id", "$gender"},
    {"numTimes", bson.D{
      {"$sum", 1},
    }},
  }},
}
opts := options.Aggregate().SetMaxTime(2 * time.Second)
aggCursor, err := collection.Aggregate(ctx, mongo.Pipeline{groupStage}, opts)
if err != nil {
  log.Fatal(err)
}

var results []bson.M
if err = aggCursor.All(ctx, &results); err != nil {
  log.Fatal(err)
}
for _, result := range results {
  fmt.Printf("gender %v appears %v times\n", result["_id"], result["numTimes"])
}
```

## 事件监控Watch

## client监控

```go
// 监控所有db中的所有collection的插入操作
matchStage := bson.D{{"$match", bson.D{{"operationType", "insert"}}}}
opts := options.ChangeStream().SetMaxAwaitTime(2 * time.Second)
changeStream, err := client.Watch(ctx, mongo.Pipeline{matchStage}, opts)
if err != nil {
  log.Fatal(err)
}

for changeStream.Next(ctx) {
  fmt.Println(changeStream.Current)
}

// 向test.person插入一篇document
{"_id": {"_data": "825FDE28AE000000022B022C0100296E5A10046470407E872C47AFB61ECB12299F90D646645F69640
0645FDE28AE4D0D028CFA4B6B140004"},"operationType": "insert","clusterTime":
 {"$timestamp":{"t":"1608394926","i":"2"}},"fullDocument": {"_id": 
{"$oid":"5fde28ae4d0d028cfa4b6b14"},"name": "蔷薇","level": 
{"$numberInt":"2"},"gender": "女"},"ns": {"db": "test","coll": 
"person"},"documentKey": {"_id": {"$oid":"5fde28ae4d0d028cfa4b6b14"}}}
```

## database监控

```go
// db监控所有collection的插入操作
matchStage := bson.D{{"$match", bson.D{{"operationType", "insert"}}}}
opts := options.ChangeStream().SetMaxAwaitTime(2 * time.Second)
changeStream, err := db.Watch(context.TODO(), mongo.Pipeline{matchStage}, opts)
if err != nil {
  log.Fatal(err)
}

for changeStream.Next(ctx) {
  fmt.Println(changeStream.Current)
}
```

## collection监控

```go
// 只监控当前collection的插入操作
matchStage := bson.D{{"$match", bson.D{{"operationType", "insert"}}}}
opts := options.ChangeStream().SetMaxAwaitTime(2 * time.Second)
changeStream, err := collection.Watch(context.TODO(), mongo.Pipeline{matchStage}, opts)
if err != nil {
  log.Fatal(err)
}

for changeStream.Next(ctx) {
  fmt.Println(changeStream.Current)
}
```


## 参考资料

1.  [github.com/mongodb/mon…](https://github.com/mongodb/mongo-go-driver "https://github.com/mongodb/mongo-go-driver")
2.  [pkg.go.dev/go.mongodb.…](https://pkg.go.dev/go.mongodb.org/mongo-driver/mongo#section-documentation "https://pkg.go.dev/go.mongodb.org/mongo-driver/mongo#section-documentation")