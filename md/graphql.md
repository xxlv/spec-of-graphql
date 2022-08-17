#  GraphQL in Shopline Spec 
## 类型->Type 
### 类型定义->Type Definition

创建类型是一切的起点，GraphQL 支持如下的类型

- Scalar
    - ID 
    - String
    - Float
    - Int
    - Boolean
- Object
- InputObject
- Enum
- Union

#### Scalar
Scalar 是基本类型。 因此可以用作`输入`或者 `输出`

Example:
    比如 `user(id:ID):User`

Note:
    id 为参数(argument),输出为`User`

##### 约束->Statute
1. ID必须 是有 唯一 业务含义
2. ID必须 属于一个 `Object` 的 字段
4. ID 必须属于某个`领域实体`，而不是`值对象` 
3. Scalar 的类型必须要保证 实现 `valueToLiteral` 

Note:比如`CustomerUser` 拥有ID,但是 `Money`不能拥有`ID` 

Note: 当Scalar 没有实现`valueToLiteral`， 作为输入参数，在Schema自省获取文档的时候，会出现异常。

#### Object 

Object 是`GraphQL` 最重要的Type. `Object` 代表了领域对象。

Note:
关于领域对象，参见[领域对象](#领域)

#### Union & Interface

`Union` 与 `Interface` ，两者都是抽象的。

##### Union

`Union` 拥有一个可能的对象列表。

###### 定义

Example:
    `type MyUnion{ A | B | C }`

`Union` 的命名规则应该于对象保持一致，代表某种特殊的`领域`。 比如一个 `多媒体`, 它可能是

1. 普通的文本
2. 图片
3. 视频流
4. 其他文件
因此 `union` 可以用做表达一些不可共存的资源的共同抽象。 它可能是枚举对象的任何一个。

##### Interface 

跟 {Union} 一样， {Interface} 本身也是一种抽象类型。 Interface 拥有两种能力

1. 更好的组织对象
2. 保持接口的统一

Note:
    面向对象设计，更好的组织对象，比如 `Product` 实现了 `Node` 接口。就具有 `id` 字段。

#### Enum

`Enum` 是一些字面量的枚举。 当对有效资源状态进行跟踪的时候，记住使用`Enum`

Example:
    `enum SEX{M,F}`

Note:
    如果不使用枚举，graph的接口将越来越难懂。因此，所有的状态都 {强制} 要定义枚举。


 

#### Input Object

Input Object 仅仅用来作为输入的参数，代表一组数据的聚合。它可以使用用任何`Scalar`和`InputObject`。但并不能用作`output`。这是它跟对象的最大区别。

##### 约束
以下约束建议`强制`遵守

1. input的命名必须是 {$func}{Input}
2. input用作`Mutation` 的时候，必须包含动词 如{$Resource}{$Method}{Input}

3. input一般用在`Mutation`,减少在query使用,保证`query`接口的干净
Note:
比如 `UserCreateInput`
  

#### Payload

Payload 代表 `Mutation` 的返回

- Payload 
所有的变更，都需要一个 `Payload`对象作为输出。
`Payload` 对象命名必须是 {$Domain}{$operate}Payload 对象, 如 `UserCreatePayload`。
其中 

    1. Domain 代表名词 如 `User`
    2. operate 代表动词 如 `Create`
    3. `Payload` 同时也根据 `mutation`的 `field name` 来决定，在本例中，我们可以知道，`mutation` 是名字 `userCreate`

Note:
 `Payload` **禁止** 共享

#### Connection
参见下文 [分页](#sec--Connections)

Note:
`Connection` **禁止** 共享

## 图->Graph

### 图的上下文
在GraphQL 中，一切都是 Graph。 许多领域都会有关联，比如一个 `Customer`领域 拥有 `Metafield` 的知识，尽管他们之前的关联非常弱。 现在在同一个`graph`中，他们可以互相传递自己的知识。

这意味着，我们在设计 GraphQL 的时候，考虑的 每一个领域都不再是一个单独的 *领域实体* ，应当将领域 理解成为
 *聚合根*。每一个聚合根都可能关联其他聚合根。 聚合根的边界 用来划分不同的团队。每个域的团队都应该慎重的考虑业务模型，基于聚合根的边界来决定对于其他领域的依赖。 比如 `Customer` 不会依赖 `Online Store` 的对象。
 因为，`Customer` 的聚合根应当聚合 从`customer`视角出发观察到的边。  相反的，`Online Store` 却可以包含 `Customer`。

在构建图的时候，需要团队协作。

Note:
团队必须短暂的忘记现有的领域模型，从顶层重新设计基于图的领域模型。这也就是基于GraphQL 实例本身的。

举个例子，比如 `User` 和 `Order`往往是两个团队，他们处于两个不同的 领域聚合根下，但是，在一个 `用户订单图`中，他们是需要合作的，因此顶层设计者需要先设计出 基于当前图的新的领域对象， 如 `UserOrderGraph`, 在当前图下，会产生新的领域聚合根，如 `UserOrderObject`,  `User` 团队 和 `Order` 团队共享 `UserOrderObject`。
他们各自提供一部分能力，但是在设计上不能表达出这种差异。 使用者感知到的是  `UserOrderObject` 对象，他们在开发的时候思维会自动进入到 `UserOrderObject`上下文。 在这个上下文中 `UserOrder`的每个字段都是平等的，就像是一个领域对象一样。

因此，如果以现有的 领域模型来构建图 是不合理的。 应当自上而下的重新设计。
任何两个领域的对象都可能会融合在一起，只要给定一个合理的上下文。也就是GraphQL 上下文。

### 场景

为了确定图的上下文，需要引入一个管理员角色，叫`图之管理员`, 这个角色用来确保 一张图里面要包含 什么。

一张图里包含了该图的 `领域知识`,这些领域知识 并非由于 图的数据支撑者来决定的。而是基于 图本身来确定的。

因此，可以借{DDD}来思考，在图的上下文里，领域又是什么？

思考这个问题不仅仅需要 图之管理者，还需要 `场景支撑者`，场景支撑者 是 领域 的子集。 比如 有10 领域团队，
场景支撑着可能是其中的7个。

Note: 领域团队，一般就是 负责某个领域的实体或个人。

可以按照如下的步骤进行筛选：

1. 头脑风暴，沿着场景，模拟行为
2. 行为产生事件，将事件汇总成表
3. 筛选领域团队 明确事件表，进行场景补充

Note: 可以选择一个角色来担任`图之管理员`.


## 领域->Domain

GraphQL需要将领域对象转化成 图的领域对象。 一般情况下，图的领域对象是需要重建的。比如 `User`对象，可以是 `普通用户`,也可以是一个 `VIP会员`。

当需要构建一张所有会员在的关系图来获取每个会员在其他平台的用户身份时候，就不能直接引入平台A的 `VIP` 领域 或者 平台B的 `User`领域。

因为这两个领域知识是完全隔离的。因此，就需要重建领域，重新构建当前图的 `VIP` 领域和 `User`领域。

比如 平台A的 `VIP` 领域，拥有 `VIP`:`好感值` 这一属性，那么当`VIP` 领域被转移到当前图的时候，则可以安全移除这一属性。我们可以给当前图的`VIP`增加额外的属性， `from`来表明当前VIP来源于哪一个平台。这样外部的领域知识如何变化，影响不到我们图输出的领域知识。因为我们的领域知识是基于当前图的上下文。



### 聚合根

GraphQL 仍旧可以按照 {DDD} 来设计，一个聚合根可以是一个 `Query`对象，也可以是任何 `Type`。

聚合根是数据的提供者，即任何的领域实体/值对象 都不能绑定字段解析函数。所有的函数必须收敛到聚合根里。


在GraphQL中，所有`Query`/`Mutation`/`Subscription` 都是根， 他代表的用户的访问入口。

他们的字段（Field) 每一个字段的类型代表了一个聚合根。

比如定义

```
type Query{
  card(id:String!):Card!

}
```

任何一个`Type` 都可以是一个聚合根。聚合根的每一个字段都可以设置一个字段Resolver。

当一个 `Type` 是聚合根的时候，它可以extend 任意领域的能力。 比如 `User` 是一个聚合根，则可以扩展 `BTC`领域的 `userBtcAddress`.


```
extend User{
  userBtcAddress:BTCAddressCtxAddress!
}

```


### 领域实体
领域实体 代表一种 `Pure Busioness Model`(纯业务模型)，领域实体本身不包含图的行为(不绑定函数)，除此之外，跟{DDD}中的领域实体一样。
领域实体明确了当前需要返回的领域模型， 它的领域聚合根负责通过 Resolver函数将数据映射到 模型之上。

领域实体是基于图的，在很多时候，图的场景的支撑域的领域知识可以迁移到图的上下文。但是并不是完全一致，可以是一个子集。

任何领域实体都应该可以被安全的被其他聚合跟依赖。


### 值对象
值对象 是一些公共的对象和枚举。 如 `PageInfo`和 `Money`对象，值对象本身绝对不能绑定Resolver. 值对象必须是无状态的。

Note: 值对象绑定 Resolver将导致无法被安全的依赖。比如`PageInfo`绑定了解析函数，将导致任何依赖分页的返回错误。

## 分页->Connections
分页采用基于`cursor`的分页规则，每一个分页的对象都必须是一个`Connection`。包含 `pageInfo`,`edges`
    
### Connection
Connection用作分页类型，当一个`query` 需要分页的时候，必须返回一个Connection。
Connection有如下规约:

1. Connection 必须是一个 `Object`
2. Connection 命名必须是 `{$Domain}Connection` 如 `UserConnection`
3. Connection 对象必须包含 两个字段 `edges`,`pageInfo`


- Node
Node 仅仅包含一个字段 `id:ID!`
- Edge 
所有的Edge 都需要实现Node

#### 类型结构
```
{
  "data": {
    "__type": {
      "fields": [
        // May contain other items
        {
          "name": "pageInfo",
          "type": {
            "name": null,
            "kind": "NON_NULL",
            "ofType": {
              "name": "PageInfo",
              "kind": "OBJECT"
            }
          }
        },
        {
          "name": "edges",
          "type": {
            "name": null,
            "kind": "LIST",
            "ofType": {
              "name": "ExampleEdge",
              "kind": "OBJECT"
            }
          }
        }
      ]
    }
  }
}
```
分页 必须接受如下的参数. `first`,`after`,`last`,`before`.

Note: 具体参见 [Relay](https://relay.dev/graphql/connections.htm)

### 分页增强

#### query
真实的场景中，往往需要对分页按照某个条件进行过滤。我们可以将想要的查询条件都追加在分页的定义上

Example:
    page(first,last,before,after,byName,byType,byTag...):PageConnection


这样做的一个点在于分页的参数将越来越多，后面难以维护。因此，我们需要提供一个{字符串} 语法来代表搜索分页的过滤条件: {query}

##### 语法分析

采用 [EBNF](https://zh.m.wikipedia.org/zh/%E6%89%A9%E5%B1%95%E5%B7%B4%E7%A7%91%E6%96%AF%E8%8C%83%E5%BC%8F)来描述语法

如可以生成如下的语法

Example:
    fieldA=1 and fieldB=2


###### 实践
通过 [ANTLT4](https://www.antlr.org/) 可以快速的构建你的词法/语法解析器。 

`query`语法必须是简单的，非图灵完备的。因此，在设计语法的时候，不要定义产生循环的语法。
保持简单，仅仅作为一个过滤器。

Example:
    a:1 and b:2 or c:1


## 权限->Permission

开放给外部的字段有可能需要权限控制。设计的权限点应当能清晰的描述权限的能力。
因此，这里推荐一种基于字段的权限编码约束

1. 所有的读权限都必须包含 `read_$field`
2. 所有的写权限都必须包含 `write_$field`

{读}和{写}可以覆盖到几乎全部的场景，$field 应当是名词，代表操作的资源。

### 权限的校验

权限的检验应当发生在验证阶段,在执行字段解析函数之前，就应当检查字段的权限。

Note:
在Java的 graphql 版本中,可以通过 `FieldValidation` 拦截器进行处理



## 版本->Version

每一个GraphQL 都是一个运行的实例，也代表一个确定的版本，因此，当graphQL 作为一张图来提供服务的时候，就自然要引入版本的概念。

对外的服务需要包含:

- 不稳定版本
- 候选版本
- 稳定版本
- 历史版本（可选）

不稳定版本可以提供一些实现性的能力，这些能力可能会被删除或者移动到稳定版本。 稳定版本必须保持完整向下的兼容逻辑。
比如一张图 `v20220101` 版本必须包含 `v20210728`版本的数据。但高版本新增的服务，则一定不能被低版本感知到。

Example:
  
  在版本 `v20210728` 中存在 `Query` `findByName`，则在版本`v20220101` 中必须要包含findByName，即便是已经被废弃。
  在版本 `v20220101` 中新增的 `Query` `findByNameV2`，则在版本 `v20210728` 中一定不能存在。 因为低版本无法感知到高版本。



### 版本的名称
版本的命名 可以按照时间 的递增序列，如 `20220101` ,`20220501`.递增的版本序列传递了当前版本的迭代周期。有利于开发者在引入版本的时候做决策。

版本的命名 还可以增加自定义的一些属性，用来标识当前版本的某个状态，比如 `2022-01-01-BETA`。这些描述是可选的。但是，在一个产品里面的版本序列的命名规范是统一的。

Note:
graphQL中的版本应该始终有一个单调递增的时间序列来表达，因为graphQL的接口代表图的知识，是随着时间进行迭代的。

### 版本递增约束
不稳定版本 随着时间的推移和沉淀，会变得稳定，因为他首选成为一个{候选版本}。 候选版本会变成稳定版本。 稳定版本应该维护一个时间序列的链。 
当一个版本变成稳定版本的时候，就要考虑移除掉稳定版本链末端的版本。 控制稳定版本的数量，否则它会越来越庞大。

Note:
    如果是对外部用户的graph，推荐用 `3-4` 个稳定版本序列。 如果是对内，推荐 `1` 个稳定版本。
    
### 兼容性
在graphql中，任何一个字段都可以被很轻易的添加在 `graph`中，当非常难以移除。当一个字段开放出去，要移除这个字段就需要漫长的等待。

TODO 



## 错误->Error

### Http

Note:
下面是常见的Http错误枚举，应当在非业务异常是暴露给开发者

|Code|Status|Description|
|---|---|---|
|UNKNOWN|520|Unknown Error|
|INTERNAL|500|Internal Server Error|
|NOT_FOUND|404|Not Found|
|UNAUTHENTICATED|401|Unauthorized|
|PERMISSION_DENIED|403|Forbidden|
|UNAVAILABLE|503|Bad Request|

### 业务错误

#### Query错误
任何一个`query` 都可能产生错误，错误有多个原因，因此错误应该在 `error`数组中。必须包含三个元素

1. message: 代表错误的原因 
2. locations: 代表错误的位置/行
3. path: 代表错误的path 

可选的，我们可以将一些辅助定位业务错误的因素，如{错误码}来添加在 `extensions`中。

完整的例子如下：

```
{
  "errors": [
    {
      "message": "Name for character with ID 1002 could not be fetched.",
      "locations": [ { "line": 6, "column": 7 } ],
      "path": [ "hero", "heroFriends", 1, "name" ],
      "extensions": {
        "code": "CAN_NOT_FETCH_BY_ID",
        "timestamp": "Fri Feb 9 14:33:09 UTC 2018"
      }
    }
  ]
}
```
#### Mutation错误

Mutation的错误最好包含具体的错误原因，一次Mutation中 有很多字段，当发生错误的时候，需要定位到具体哪一个字段导致了错误的发生（当然有可能是多个字段导致）。
因此，Mutation的错误应该定义在Type中。也就是Payload里。

它必须包含三个字段：

1. field:[String!]
2. message String!
3. code:Code    
所以导致错误的字段都应该出现在field 数组里。

## 限速->RateLimiter
GraphQL是一张图，用户端以`DSL`的形式来描述对 资源的访问与变更，若不加限制，恶意的客户端很容易构造一个极负责的查询来消耗服务端CPU/内存资源。
GraphQL 针对访问高并发性能优先的程序和资源优先，可以采用如下几种限速算法

- Time-based 算法
- Scope 算法


### Time-based 算法
TODO


### Scope 算法

每一个对象都有不同的`scope`,比如 查询一个`object` 需要花费 `1`,查询一个分页，返回10个对象，则应该是 `1+10`
通过设置一个全局最大值，来限制用户的查询和保护系统资源。

Note:
    每一个查询都需要消耗CPU和内存资源，因此必须


下面是一张推荐的表，对于每一种类型的对象成本做了预估。

|Type|Cost|
|---|---|
|Scalar|0|
|Enum|0|
|Object|1|
|Interface|1|
|Union|1|
|Connections|2|
|Mutation Field|10|

Note:
    一切都是为了保护系统资源，在特定的场景下，上面的分值可以自行调整。

## 文档->Document

### 自省

graphQL 可[自省](https://graphql.org/learn/introspection/).
这意味着，可以通过`graphql` 的查询语法来观察自身的结构，包括：字段定义、描述、类型
因此，通过自省指令可以观察`schema`本身的结构。

比如如下的查询，可以观察到 `TypeName` 的 `kind` 信息

```
{
  __type(name: "TypeName") {
    name
    kind
  }
}

```
### graphiQL

[GraphiQL](https://github.com/graphql/graphiql) 是一个in-web 的工具，我们可以通过一个`endpoint`，通过自省就能看到文档。它非常有效，可以发起请求。 
因此强烈推荐在开发过程中使用 `graphiql` 来进行测试。 它也可以部署在外部，当需要将`graphql` api 开放出去的时候，最好提供一个`graphiql`。

### 文档多语言

有些应用需要提供多语言版本的文档.

TODO


## 实践->Do

### ID
ID 可以在一个 {endpoint} 中代表唯一的资源。客户端可以用来做缓存。
有如下两种简单的方式来构建ID


#### encodeString
通过一个`base64`编码的字符串来代表ID是最简单的。
但应该尽可能的考虑查询性能，当有多个数据源的时候，如果ID不无法分析出领域，则额外维护一个字典表来映射。因此适当的做法是带有一个前缀来快速识别子域.

Example:
U_aGVsbG8gd29ybGQ=


Note:
不建议采用随机字符串来做graphql的`ID`.尤其是在拥有大量的子域的时候。 
在上面的例子中, `U_` 代表一种前缀，例如代表 `User`。
在数据集群的时候，可以快速的定位到数据的位置。



#### gid 

gid 是一套`ID` 规约（协议）,通过 `gid` 我们可以定义唯一的资源。

`ID` 本身代表唯一资源，并不要求资源是可读的。 因此在同一个`graph`中采用一套约束，可以带来很多优势。

- `gid` 可读 
- `gid` 在图中唯一

##### 协议
`gid://$company/$domain/$id`

协议分为三部分：

1. `gid://` 代表gid协议
2. $company  固定值 代表公司
3. $domain  代表领域 如 `Product` 注意 首字母大写驼峰
4. $id 代表资源的唯一ID  注意！ $id _必须_ 在领域内唯一，比如 123 在Product 内代表 ID为 123的商品 (不能包含其他含义)。

Note:
    $id 建议 是数字，且长度不能超过64位。


Example:
    gid://shopline/Product/1
    
###### 子域
真实场景中，我们的团队往往分为子域。因此 `$domain` 可以具体到子域。

有两种方式，

通过驼峰

Example:
gid://company/ProductVersion/1

通过`::`

Example:
gid://company/Product::Version/1

Note:
    可以选择其中的其中的任意一种。驼峰是shopify的风格，`::` 是gitlab 的风格。

###### 场景

1. Node 节点

Note:
    假如你要返回一个Node节点，作为graphQL的output. 
    ``` node(id:ID):Node!```
    当为`node`配置字段解析的时候，我们可以根据`id`的`gid`约束来获取到查询的$domain,然后获取到对应domain的资源。
     

### node & nodes 

node 可以作为一种最佳实践，用来表达一系列资源的公共抽象。尤其是当子域非常多的时候，我们不可能为每一个子域名都构建一个查询。
因此，可以考虑使用node来构建 graphQL 的核心资源查询能力


#### Node

```

interface Node{
    id:ID!
}

```
##### Node支撑域

Node 可以作为许多子域的抽象返回，用来补充graphql的能力。

比如 有 `Checkout` `Product` `Cart` `Order` 四个子域。 采用Node，我们可以提供一个这样的查询


Example:
    type node(id:ID):Node!
    
我们可以采用如下的查询来获取`Product`


``` 
{
  node(id:"$productId"){
 	... on Product {
    id
  }   
  }
}

```

当需要查询 `Cart`时候， 我们可以采用如下的形式调用

```
{
  node(id:"$cartId"){
 	... on Cart {
    id
  }   
  }
}

```

仅仅一个`node` 字段，就可以查询很多子域的数据。

### 自定义Scalar

Scalar是基本类型，自定义Scalar的时候必须非常小心，切勿包含业务含义。

判断一个类型是否可以被定义成Scalar的要素如下

1. 如果这个Scalar脱离当前{graph}仍旧有含义，则可以定义，如 `JSON` `HTML`
2. 如果这个Scalar在当前{graphq}中可以被共享且没有ID，则可以定义， 如 `Money`
3. 这个值本身可以被非JSON的 简单字面量表达

Note:
    比如一个 `UUID` 的字面量表达是 `c8e75cef-75d3-9f35-ca14-9a6de31bc745`  
    而一个`User` 的字面量可能是 `{"id":1,"name":"Leslie"}` 因此`User` 不能被定义为Scalar


除此之外，不应该将其定义成为Scalar