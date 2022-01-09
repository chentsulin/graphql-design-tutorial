# 教學：設計 GraphQL API

這份教學最初是由 [Shopify](https://www.shopify.ca/) 建立，用於內部用途。但我們認為它能幫到任何正在打造 GraphQL API 的人，因此我們建立了這份公開版本。

這些內容是基於最近 3 年來，Shopify 在生產環境中打造並持續演進 schema 所獲得的經驗和教訓。這份教學經過了一段時間的演進並且會在未來持續改變，所以沒有什麼是一定不會變的。

我們相信這些設計準則在大多數情況都適用。但它們可能不是全部都適合你。即使在 Shopify 內部，我們依然會質疑它們並存在例外，因為大部分的準則都不是 100% 適用。所以請不要盲目地照搬並實施所有的準則。挑選那些對你以及你的使用情境合理的準則。

目錄
=================
* [簡介](#簡介)
* [步驟 0：背景設定](#step-zero-background)
* [步驟 1：全局概覽](#step-one-a-birds-eye-view)
* [步驟 2：開頭](#step-two-a-clean-slate)
  * [呈現 CollectionMembership](#呈現-collectionmembership)
  * [呈現 Collection](#呈現-collection)
  * [結論](#結論)
* [步驟 3：補上細節](#step-three-adding-detail)
  * [起始點](#starting-point)
  * [ID 和 Node 介面](#ids-and-the-node-interface)
  * [Rules and Subobjects](#rules-and-subobjects)
  * [Lists and Pagination](#lists-and-pagination)
  * [字串](#字串)
  * [ID 和關聯](#id-和關聯)
  * [Naming and Scalars](#naming-and-scalars)
  * [Pagination Again](#pagination-again)
  * [列舉](#列舉)
* [步驟 4：商業邏輯](#step-four-business-logic)
* [步驟 5：Mutation](#step-five-mutations)
  * [拆分合理的操作](#拆分合理的操作)
  * [為 Mutation 命名](#naming-the-mutations)
  * [操作關聯](#操作關聯)
  * [Input：結構，第一部分](#input-structure-part-1)
  * [Input：Scalars](#input-scalars)
  * [Input：結構，第二部分](#input-structure-part-2)
  * [Output](#output)
* [TLDR：設計準則](#tldr-the-rules)
* [尾聲](#尾聲)

## 簡介

歡迎！這份文件將帶你走過設計一個新的 GraphQL API（或對既有的 GraphQL API 擴充）的過程。API 設計是一項極具挑戰的任務，需要對你的業務領域有徹底的了解並不斷的迭代跟實驗。

## 步驟 0：背景設定

為了達到這份教學的目的，想像你在一家電子商務公司工作。
你有一個既有的 GraphQL API 可以查詢你的 product 資訊，但沒什麼其他的東西。然而，你的團隊剛完成一個專案在後端實作了「collection」而且也想要可以透過 API 查詢 collection。

Collection 是對 product 進行分組的新方法；例如，你可能擁有一個包含你所有 t-shirt 的 collection。Collection 可以在你瀏覽網站時被用做顯示用途，也可以用在程式化任務（例如：你可能想要對特定 collection 裡的 product 做折扣）。

在後端，新功能已經實作如下：
- 所有的 collection 都有一些簡單的屬性，像是：標題、描述（可能包含 HTML 格式）、以及一張圖。
- 你有兩種 collection：自行列出希望加入 product 的「手動」collection，以及指定一些規則然後讓 collection 自己建立的「自動」collection。
- 因為 product 跟 collection 的關聯是多對多，你會在中間有一個關聯表叫做 `CollectionMembership`。
- Collection，就像之前的 product 一樣，可以被發布（在店面上顯示）也可以不發布。

有了這樣的背景知識，你就可以開始思考你的 API 設計了。

## 步驟 1：全局概覽

粗略版的 schema 可能看起來像這樣（省略所有已存在的類型，例如 `Product`）：
```graphql
interface Collection {
  id: ID!
  memberships: [CollectionMembership!]!
  title: String!
  imageId: ID
  bodyHtml: String
}

type AutomaticCollection implements Collection {
  id: ID!
  rules: [AutomaticCollectionRule!]!
  rulesApplyDisjunctively: Boolean!
  memberships: [CollectionMembership!]!
  title: String!
  imageId: ID
  bodyHtml: String
}

type ManualCollection implements Collection {
  id: ID!
  memberships: [CollectionMembership!]!
  title: String!
  imageId: ID
  bodyHtml: String
}

type AutomaticCollectionRule {
  column: String!
  relation: String!
  condition: String!
}

type CollectionMembership {
  collectionId: ID!
  productId: ID!
}
```

雖然只有 4 個物件和 1 個介面，但乍看之下已經非常複雜。而且，如果我們打算使用這個 API 來打造例如行動應用程式的 collection 功能，它顯然也沒有實作我們所有需要的功能。

讓我們先退一步思考一下。一個非常複雜的 GraphQL API 會由許多的物件組成，藉由許多的路徑關聯在一起，並擁有許多的欄位。試圖一次設計這樣子的東西會導致混亂和錯誤。因此，你應該從更高階的概覽開始，只關注類型跟它們的關聯，而不要擔心具體的欄位或 mutation。
基本上像是在思考一個[實體關聯模型（Entity-Relationship model）](https://en.wikipedia.org/wiki/Entity%E2%80%93relationship_model) 但是混入一些 GraphQL 特定的概念。如果我們這樣去縮減我們的粗略版 schema，我們最後會得到以下結果：

```graphql
interface Collection {
  Image
  [CollectionMembership]
}

type AutomaticCollection implements Collection {
  [AutomaticCollectionRule]
  Image
  [CollectionMembership]
}

type ManualCollection implements Collection {
  Image
  [CollectionMembership]
}

type AutomaticCollectionRule { }

type CollectionMembership {
  Collection
  Product
}
```

為了獲得這種簡化的表示形式，我移除了所有常量（scalar）欄位、所有欄位名稱、以及所有可接收空值的資訊。剩下的東西看起來還是有點像 GraphQL，但它讓你能從更高層級專注在類型跟它們的關聯。

*準則 #1：在處理特定欄位之前，總是從物件以及它們之間關聯的高階概覽開始。*

## 步驟 2：開頭

現在我們有了一些簡單的東西，我們可以透過這個去解決主要的缺陷。

如先前所述，我們的實作定義了手動和自動的 collection， 以及使用一個中間的關聯表。我們粗略版的 API 設計很明顯地圍繞著我們的實作去架構，但這是個錯誤。

這個方法的根本問題是，API 運作的目的跟實作不同，經常處於不同的抽象層級。在這種情況下，我們的實作使我們在許多不同的方面誤入歧途。

### 呈現 `CollectionMembership`

其中一個非常明顯而且你可能已經注意到的是，schema 中包含 `CollectionMembership` 類型。這個 collection membership 表是用來代表 product 跟 collection 之間的多對多關聯。
現在再讀一遍最後一句話：關聯是*介於 product 跟 collection 之間*；從語義、業務領域的角度來看， collection membership 跟所有東西都無關。它們是實作細節。

這代表它不應該存在於我們的 API。反而，我們的 API 應該直接暴露 product 的實際業務領域關聯。如果我們移除 collection membership，這樣產生的高階設計看起來像這樣：

```graphql
interface Collection {
  Image
  [Product]
}

type AutomaticCollection implements Collection {
  [AutomaticCollectionRule]
  Image
  [Product]
}

type ManualCollection implements Collection {
  Image
  [Product]
}

type AutomaticCollectionRule { }
```

這樣好多了。

*準則 #2：永遠不要在你的 API 設計中暴露實作細節。*

### 呈現 Collection

這個 API 設計仍然有一個重大的缺陷，雖然如果沒有對業務領域有透徹的了解就不會顯得那麼明顯。在我們現有的設計中，我們將 AutomaticCollection 和 ManualCollection 建立成兩個不同的類型，實作一個共同的 Collection 介面。直覺上這有一定的道理：它們有許多共同的欄位，但它們的關聯和它們的一些行為仍然明顯不同（AutomaticCollection 有規則）。

但從商業模式的角度來看，這些差異基本上也是實作細節。collection 定義的行為是它對 product 進行分組；挑選這些 product 的方式是次要的。我們可以在某個時間點擴展我們的實作來允許某個第三種挑選 product 的方法（機器學習？）或是允許混合方法（一些規則和一些的手動添加 product） 而*他們仍然會是 collection*。你甚至可以說我們現在不允許混合的狀況是個失敗的實作。也就是說，我們 API 的形狀應該看起來更像這樣：

```graphql
type Collection {
  [CollectionRule]
  Image
  [Product]
}

type CollectionRule { }
```

這樣非常好。此時你最擔心的可能是，我們現在是在假裝 ManualCollections 有規則，不過請記得，這個關聯是個列表。在我們新的 API 設計中，「ManualCollection」只是一個規則列表為空的 Collection。

### 結論

在這個抽象層級選擇最好的 API 設計，必然會需要你對正在建模的問題領域有非常深入的了解。
在教學設定中，很難為特定主題提供如此深入的情境，不過希望這個 collection 設計足夠簡單並且整個推導仍然合理。即使你對 collection 沒有這種深度的理解，你依然絕對需要掌握你實際在建模的領域。在設計你的 API 時，問自己這些棘手的問題至關重要，而不要盲目地遵照實作方式。

與此密切相關的是，，一個好的 API 也不會依照使用者介面建模。
實作方式和使用者介面都可以當作 API 設計的靈感和輸入，但最後驅動你決策的還是必須是業務領域。

更重要的是，一定不要照搬現有的 REST API 的設計。REST 和 GraphQL 背後的設計準則可能會導致非常不同的選擇，所以不要假設那些對你的 REST API 有用的東西也是 GraphQL 的好設計。

盡可能放下包袱，從頭開始。

*準則 #3：圍繞業務領域來設計你的 API，而不要圍繞實作、使用者介面、或是舊的 API。*

## 步驟 3：補上細節

現在我們有一個乾淨的結構來建模我們的類型，我們可以加回我們的欄位，並重新回來處理那個層級的細節。

在我們開始添加細節之前，問問自己這些東西此時是否真的需要。只是因為那個資料庫欄位、模型屬性或 REST 屬性存在，並不代表需要將它自動地夾到 GraphQL schema 中。

要暴露一個 schema 元素（欄位、參數、類型，等等）應該由真實的需求跟使用案例驅動。GraphQL schema 可以簡單地透過添加元素來演進，但是更改或是移除它們是破壞性的變更而難上許多。

*準則 #4：移除欄位遠比添加欄位來的困難。*

### 起始點

Restoring our naive fields adjusted for our new structure，我們會得到：

```graphql
type Collection {
  id: ID!
  rules: [CollectionRule!]!
  rulesApplyDisjunctively: Boolean!
  products: [Product!]!
  title: String!
  imageId: ID
  bodyHtml: String
}

type CollectionRule {
  column: String!
  relation: String!
  condition: String!
}
```

現在我們要解決一系列全新的設計問題。我們將按照順序從上到下看過這些欄位，一路修好它們。

### ID 和 `Node` 介面

在我們的 Collection 類型中，最上面的欄位是一個 ID 欄位，這很正常也很合理；我們需要使用這個 ID 來識別整個 API 中的 collection，特別是在執行修改或刪除它們等操作的時候。
然而，我們設計的這一部分缺少了一個東西：`Node` 介面。這是一個非常常用的介面，已經存在於大多數的 schema 之中，它看起來像這樣：
```graphql
interface Node {
  id: ID!
}
```
它向客戶端提示該物件可以透過給定的 ID 來持久化和取回資料，這允許客戶端精準有效地管理本地快取以及其他技巧。你絕大多數可識別的業務物件（例如，product、collection，等等）應該實作 `Node` 介面。

我們初始的設計現在看起來像這樣：
```graphql
type Collection implements Node {
  id: ID!
}
```

*準則 #5：主要的業務物件類型都應該實作 `Node` 介面。*

### Rules 和子物件

我們接下來將一起考慮 Collection 類型上的兩個欄位：`rules` 以及 `rulesApplyDisjunctively`。第一個非常直白：一個 rule 的列表。請注意，b列表本身和列表中的項目都被標記不可為空值：這樣很好，因為 GraphQL 確實會區分 `null`、`[]` 和 `[null]`對手動 collection 來說，這列表可以是空列表，但它不能為 null 也不能包含 null。

*專業提醒：列表類型的欄位通常總是不可為空值的列表包含著不可為空值的項目。如果你想要一個可以為空值的列表，請確保真的能在語意上區分空列表以及空值。*

第二個欄位有點奇怪：它是一個布林欄位，指出這些規則是否分開套用。它也不允許空值，但是在這裡我們遇到了一個問題：手動 collection 的這個欄位應該取什麼值？不管將它設定成 false 還是 true 感覺會產生誤導，但是使該欄位可以為空值，然後在處理自動 collection 時，讓它變成一個有三種的奇怪狀態這樣也很怪。當我們對此感到困惑時，還有另一件事值得一提：這兩個欄位明顯且錯綜複雜地相關。
這在語義上是正確的，我們選擇了具有共用前綴的名稱的這一事實也暗示了這一點。有沒有辦法以某種方式在 schema 中指出這個關聯？

事實上，we can solve all of these problems in one fell swoop by deviating even further from our underlying implementation and introducing a new GraphQL type with no direct model equivalent：`CollectionRuleSet`。當你有一組密切相關的欄位，而且其值和行為相互關聯時，這樣做通常是很合理的。通過在 API 層級將這兩個欄位分組到它們自己的類型中，我們提供了一個清晰的語義指示，也解決了我們所有關於可允許空值的問題：對於手動 collection 來說，這個 rule-set 本身為空值。而布林欄位可以維持不允許空值。這把我們導向了以下的設計：

```graphql
type Collection implements Node {
  id: ID!
  ruleSet: CollectionRuleSet
  products: [Product!]!
  title: String!
  imageId: ID
  bodyHtml: String
}

type CollectionRuleSet {
  rules: [CollectionRule!]!
  appliesDisjunctively: Boolean!
}

type CollectionRule {
  column: String!
  relation: String!
  condition: String!
}
```

*專業提醒：就像列表一樣，布林欄位通常總是不為空值。如果你想要一個可為空值的布林，請確保真的能在語意上區分這三個狀態（null/false/true）並且這不意味著一個更大的設計缺陷。*

*準則 #6：把密切相關的欄位抽出來放到子物件。*

### Lists and Pagination

Next on the chopping block is our `products` field. This one might seem safe; after all we already "fixed" this relation back when we removed our `CollectionMembership`type, but in fact there's something else wrong here.

The field as currently defined returns an array of products, but collections can easily have many tens of thousands of products, and trying to gather all of those into a single array would be incredibly expensive and inefficient. For situations like this, GraphQL provides lists pagination.

Whenever you implement a field or relation returning multiple objects, always ask yourself if the field should be paginated or not. How many of this object can there be? What quantity is considered pathological?

Paginating a field means you need to implement a pagination solution first.
This tutorial uses [Connections](https://graphql.org/learn/pagination/#complete-connection-model) which is defined by the [Relay Connection spec](https://facebook.github.io/relay/graphql/connections.htm).

In this case, paginating the products field in our design is as simple as changing its definition to `products: ProductConnection!`。Assuming you have connections implemented, your types would look like this：

```graphql
type ProductConnection {
  edges: [ProductEdge!]!
  pageInfo: PageInfo!
}

type ProductEdge {
  cursor: String!
  node: Product!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
}
```


*準則 #7：總是檢查清單欄位是否應該被分頁。*

### 字串

下一個要看的是 `title` 欄位。This one is legitimately fine the way it is。It's a simple string, and it's marked non-null because all collections must have a title。

*專業提醒：跟布林以及列表一樣，值得注意 GraphQL 有區分空字串（`""`）以及空值（`null`），所以如果你需要一個可以為空值的字串，請確保不存在（`null`）和存在但為空（`""`）之間存在合理的語意差異。You can often think of empty strings as meaning 「applicable, but not populated」, and null strings meaning 「not applicable」。*

### ID 和關聯

Now we come to the `imageId` field。This field is a classic example of what happens when you try and apply REST designs to GraphQL。In REST APIs it's pretty common to include the IDs of other objects in your response as a way to link together those objects, but this is a major anti-pattern in GraphQL.
Instead of providing an ID, and forcing the client to do another round-trip to get any information on the object, we should just include the object directly into the graph - that's what GraphQL is for after all。In REST APIs this pattern often isn't practical, since it inflates the size of the response significantly when the included objects are large。However, this works fine in GraphQL because every field must be explicitly queried or the server won't return it.

As a general rule, the only ID fields in your design should be the IDs of the object itself。Any time you have some other ID field, it should probably be an object reference instead。Applying this to our schema so far, we get:

```graphql
type Collection implements Node {
  id: ID!
  ruleSet: CollectionRuleSet
  products: ProductConnection!
  title: String!
  image: Image
  bodyHtml: String
}

type Image {
  id: ID!
}

type CollectionRuleSet {
  rules: [CollectionRule!]!
  appliesDisjunctively: Boolean!
}

type CollectionRule {
  column: String!
  relation: String!
  condition: String!
}
```

*Rule #8：總是使用物件參照而不是 ID 欄位。*

### Naming and Scalars

The last field in our simple `Collection` type is `bodyHtml`。To a user who is unfamiliar with the way that collections were implemented, it's not entirely obvious what this field is for; it's the body description of the specific collection。The first thing we can do to make this API better is just to rename it to `description`, which is a much clearer name。

*準則 #9：根據合理性選擇欄位名稱，而不要根據實作或是舊的 API 怎麼稱呼它。*

Next, we can make it non-nullable。As we talked about with the title field, it doesn't make sense to distinguish between the field being null and simply being an empty string, so we don't expose that in the API。Even if your database schema does allow records to have a null value for this column, we can hide that at the implementation layer。

Finally, we need to consider if `String` is actually the right type for this field。GraphQL provides a decent set of built-in scalar types （`String`, `Int`, `Boolean`, et）but it also lets you define your own, and this is a prime use case for that feature。Most schemas define their own set of additional scalars depending on their use cases。These provide additional context and semantic value for clients。In this case, it probably makes sense to define a custom `HTML` scalar for use here（and potentially elsewhere）when the string in question must be valid HTML。

Whenever you're adding a scalar field, it's worth checking your existing list of custom scalars to see if one of them would be a better fit。If you're adding a field and you think a new custom scalar would be appropriate, it's worth talking it over with your team to make sure you're capturing the right concept。

*準則 #10：當你開放查詢欄位的值有特殊語意，使用自定義的 scalar 類型。*

### Pagination Again

That covers all of the fields in our core `Collection` type。The next object is `CollectionRuleSet`, which is quite simple。The only question here is whether or not the list of rules should be paginated。In this case the existing array actually makes sense; paginating the list of rules would be overkill。Most collections will only have a handful of rules, and there isn't a good use case for a collection to have a large rule set。Even a dozen rules is probably an indicator that you need to rethink that collection, or should just be manually adding products。

### 列舉

這將我們帶到了 schema 中的最後一個類型，`CollectionRule`。每條 rule 是由一個要匹配的 column（例如，product title）、一種關聯類型（例如，equality）和要實際使用的值（例如，「Boots」）組成這有點混淆地被稱作為 `condition`。那最後一個欄位可以重新命名，而且 `column` 也是；column 是一個非常特定於資料庫的用詞，而我們是在做 GraphQL。`field` 可能是個更好的選擇。

As far as types go, both `field` and `relation` are probably implemented internally as enumerations（assuming your language of choice even has enumerations）。Fortunately GraphQL has enums as well, so we can convert those two fields to enums。我們完成的 schema 設計現在看起來像這樣：

```graphql
type Collection implements Node {
  id: ID!
  ruleSet: CollectionRuleSet
  products: ProductConnection!
  title: String!
  image: Image
  description: HTML!
}

type CollectionRuleSet {
  rules: [CollectionRule!]!
  appliesDisjunctively: Boolean!
}

type CollectionRule {
  field: CollectionRuleField!
  relation: CollectionRuleRelation!
  value: String!
}

enum CollectionRuleField {
  TAG
  TITLE
  TYPE
  INVENTORY
  PRICE
  VENDOR
}

enum CollectionRuleRelation {
  CONTAINS
  ENDS_WITH
  EQUALS
  GREATER_THAN
  LESS_THAN
  NOT_CONTAINS
  NOT_EQUALS
  STARTS_WITH
}
```

*準則 #11：對只能接受一組特定值的欄位使用列舉。*

## 步驟 4：商業邏輯

We now have a minimal but well-designed GraphQL API for collections。There is a lot of detail to collections that we haven't dealt with - any real implementation of this feature would need a lot more fields to deal with things like product sort order, publishing, etc。- but as a rule those fields will all follow the same design patterns laid out here。However, there are still a few things which bear looking at in more detail。

For this section, it is most convenient to start with a motivating use case from the hypothetical client of our API。Let us therefore imagine that the client developer we have been working with needs to know something very specific: whether a given product is a member of a collection or not。Of course, this is something that the client can already answer with our existing API: we expose the complete set of products in a collection, so the client simply has to iterate through, looking for the product they care about。

This solution has two problems though。The first, obvious problem is that it's inefficient; collections can contain millions of products, and having the client fetch and iterate through them all would be extremely slow。The second, bigger problem, is that it requires the client to write code。This last point is a critical piece of design philosophy: the server should always be the single source of truth for any business logic。An API almost always exists to serve more than one client, and if each of those clients has to implement the same logic then you've effectively got code duplication, with all the extra work and room for error which that entails。

*準則 #12：API 應該提供商業邏輯，而不只是資料。複雜的計算應該在伺服器上統一完成，而不是在許多的客戶端上做。*

Back to our client use-case, the best answer here is to provide a new field specifically dedicated to solving this problem。Practically, this looks like：
```graphql
type Collection implements Node {
  # ...
  hasProduct(id: ID!): Boolean!
}
```
This field takes the ID of a product and returns a boolean based on the server determining if a product is in the collection or not。The fact that this sort-of duplicates the data from the existing `products` field is irrelevant。GraphQL returns only what clients explicitly ask for, so unlike REST it does not cost us anything to add a bunch of secondary fields。The client doesn't have to write any code beyond querying an additional field, and the total bandwidth used is a single ID plus a single boolean。

One follow-up warning though: just because we're providing business logic in a situation does not mean we don't have to provide the raw data too。Clients should be able to do the business logic themselves, if they have to。You can’t predict all of the logic a client is going to want, and there isn't always an easy channel for clients to ask for additional fields（though you should strive to ensure such a channel exists as much as possible）。

*準則 #13：也要提供原始資料，即使它周圍已經有商業邏輯。*

最後，不要讓商業邏輯的欄位影響整個 API 的形狀。
The business domain data is still the core model。如果你發現 the business logic doesn't really fit, then that's a sign that maybe your underlying model isn't right。

## 步驟 5：Mutation

現在我們的 GraphQL schema 設計還缺少的最後一個部分是實際去更改值的能力：建立、更新、以及刪除 collection 和關聯的部分。
跟 schema 的可讀部分一樣，我們應該從高階概覽開始：在這種情況下，只包含我們想要實作的各種 mutation，而不用擔心它們的個別的 input 跟 output。不經思考的話，我們可能會遵照 CRUD 範式並只有 `create`、`delete`、以及 `update` mutation。
雖然這是一個不錯的起點，但對於合適的 GraphQL API 來說還不夠。

### 拆分合理的操作

The first thing we might notice if we were to stick to just CRUD is that our `update` mutation quickly becomes massive, responsible not just for updating simple scalar values like title but also for performing complex actions like publishing/unpublishing, adding/removing/reordering the products in the collection, changing the rules for automatic collections, etc。This makes it hard to implement on the server and hard to reason about for the client。
Instead, we can take advantage of GraphQL to split it apart into more granular, logical actions。As a very first pass, we can split out publish/unpublish resulting in the following mutation list：
- create
- delete
- update
- publish
- unpublish

*Rule #14：為資源上邏輯獨立的操作撰寫不同的 mutation。*

### 為 Mutation 命名

作為一個起點，不要預設使用 CRUD 動詞名稱。用 CRUD 動詞可以很好的描述資料庫的陳述句，不過它們是應該對 API 使用者隱藏的實作細節。考慮到你的領域、情境以及 mutation 的作用，CRUD 動詞很少能充分描述一個業務操作。如果有更有意義的動詞可以使用，那就傾向使用它。例如，如果主要的作用是取消發布一個 collection，不要使用 `collectionDelete` 當作名字；而將它命名為 `collectionUnpublish`。

### 操作關聯

`update` mutation 仍然負責太多事情，所以繼續拆分它是合理的，但我們將分別處理這些動作，因為它們也值得從另一個維度思考：物件關聯的操作（例如：一對多、多對多）。我們已經思考過 ID 以及嵌入的用法，還有在讀取 API 中分頁以及陣列的用法，在改變這些關係時也有一些類似的問題需要處理。

For the relationship between products and collections, there are a couple of styles we could broadly consider:
- Embedding the entire relationship (e.g. `products: [ProductInput!]!`) into the update mutation is the CRUD-style default, but of course it quickly becomes inefficient when the list is large.
- Embedding "delta" fields（e.g. `productsToAdd: [ID!]!` and `productsToRemove: [ID!]!`）into the update mutation is more efficient since only the changed IDs need to be specified instead of the entire list, but it still keeps the actions tied together.
- Splitting it up entirely into separate mutations（`addProduct`, `removeProduct`, etc.）is the most powerful and flexible but also the most work.

The last option is generally the safest call, especially since mutations like this will usually be distinct logical actions anyway. However, there are a lot of factors to consider:
- Is the relationship large or paginated？If so, embedding the entire list is definitely impractical, however either delta fields or separate mutations could still work. If the relationship is always small though (especially if it's one-to-one), embedding may be the simplest choice.
- Is the relationship ordered？The product-collection relationship is ordered, and permits manual reordering. Order is naturally supported by the embedded list or by separate mutations（you can add a `reorderProducts` mutation）but isn't an option for delta fields.
- Is the relationship mandatory？Products and collections can both exist on their own outside of the relationship, with their own create/delete lifecycle. If the relationship were mandatory（i.e. products must be in a collection）then this would strongly suggest separate mutations because the action would actually be to *create* a product, not just to update the relationship.
- Do both sides have IDs？The collection-rule relationship is mandatory（rules can't exist without collections）but rules don't even have IDs; they are clearly subservient to their collection, and since the relationship is also small, embedding the list is actually not a bad choice here. Anything else would require rules to be individually identifiable and that feels like overkill.

*準則 #15：操作物件關聯非常複雜，不容易總結成一條簡潔的規則。*

如果將所有這些加在一起，針對 collection，我們最終得到以下的 mutation 列表：
- create
- delete
- update
- publish
- unpublish
- addProducts
- removeProducts
- reorderProducts

Products we split into their own mutations, because the relationship is large and ordered。Rules we left inline because the relationship is small, and rules are sufficiently minor to not have IDs。

Finally, you may note our product mutations act on sets of products, for example `addProducts` and not `addProduct`。This is simply a convenience for the client, since the common use case when manipulating this relationship will be to add, remove, or reorder more than one product at a time。

*準則 #16：當為關聯撰寫個別的 mutation 時，考慮一次對多個項目進行操作是否會對這個 mutation 有用。*

### Input：結構，第一部分

Now that we know which mutations we want to write, we get to figure out what their input structures look like。If you've been browsing any of the real production schemas that are publicly available, you may have noticed that many mutations define a single global `Input` type to hold all of their arguments: this pattern was a requirement of some legacy clients but is no longer needed for new code; we can ignore it。

For many simple mutations, an ID or a handful of IDs are all that is needed, making this step quite simple。Among collections, we can quickly knock out the following mutation arguments：
- `delete`, `publish` and `unpublish` all simply need a single collection ID
- `addProducts` and `removeProducts` both need the collection ID as well as a list of product IDs

This leaves us with only three remaining "complicated" inputs to design：
- create
- update
- reorderProducts

Let's start with create。A very naive input might look kind of like our original naive collection model when we started, but we can already do better than that。
Based on our final collection model and the discussion of relationships above, we can start with something like this：

```graphql
type Mutation {
  collectionDelete(collectionId: ID!)
  collectionPublish(collectionId: ID!)
  collectionUnpublish(collectionId: ID!)
  collectionAddProducts(collectionId: ID!, productIds: [ID!]!)
  collectionRemoveProducts(collectionId: ID!, productIds: [ID!]!)
  collectionCreate(title: String!, ruleSet: CollectionRuleSetInput, image: ImageInput, description: HTML!)
}

input CollectionRuleSetInput {
  rules: [CollectionRuleInput!]!
  appliesDisjunctively: Boolean!
}

input CollectionRuleInput {
  field: CollectionRuleField!
  relation: CollectionRuleRelation!
  value: String!
}
```

First a quick note on naming：you'll notice that we named all of our mutations in the form `collection<Action>` rather than the more naturally-English `<action>Collection`。Unfortunately, GraphQL does not provide a method for grouping or otherwise organizing mutations, so we are forced into alphabetization as a workaround。Putting the core type first ensures that all of the related mutations group together in the final list。

*準則 #17：為了按照字母分組，把 mutation 操作的物件名稱前綴在 mutation 名稱上（例如，使用 `orderCancel` 而不是 `cancelOrder`）*

### Input：Scalars

這個草稿已經比最早粗略的版本好上不少，但依然不夠完美。特別是，`description` input 欄位有一些問題。A non-null `HTML` field makes sense for the output of a collection's description, but it doesn't work as well for input for a couple of reasons。First-off, while `!` denotes non-nullability on output, it doesn't mean quite the same thing on input; instead it denotes more the concept of whether a field is "required"。A required field is one the client must provide in order for the request to proceed, and this isn't true for `description`。We don't want to prevent clients from creating collections if they don't provide a description（or equivalently, we don't want to force them to provide a useless `""`）, so we should make `description` non-required。

*準則 #18：只有 mutation 語義上確實需要該 input 欄位才能執行，才把它標為必填。*

The other issue with `description` is its type; this may seem counter-intuitive since it is already strongly-typed（`HTML` instead of `String`）and we've been all about strong typing so far。But again, inputs behave a little differently。
Validation of strong typing on input happens at the GraphQL layer before any "userspace" code gets run, which means that realistically clients have to deal with two layers of errors: GraphQL-layer validation errors, and business-layer validation errors（for example something like: you've reached the limit of collections you can create with your current storage）。In order to simplify this process, we intentionally weakly type input fields when it might be difficult for the client to validate up-front。This lets the business-logic side handle all of the validation, and lets the client only deal with errors from one spot。

*準則 #19：當格式明確而且客戶端驗證很複雜時，使用較弱的類型來限制 input（例如：用 `String` 來取代 `Email`）。這讓伺服器可以一次執行所有複雜的驗證並用單一格式統一回傳錯誤，簡化了客戶端的複雜度。*

It is important to note, though, that this is not an invitation to weakly-type all your inputs. We still use strongly-typed enums for the `field` and `relation` values on our rule input, and we would still use strong typing for certain other inputs like `DateTime`s if we had any in this example. The key differentiating factors are the complexity of client-side validation and the ambiguity of the format. HTML is a well-defined, unambiguous specification, but is quite complex to validate. On the other hand, there are hundreds of ways to represent a date or time as a string, all of them reasonably simple, so it benefits from a strong scalar type to specify which format we expect.

*準則 #20：當格式可能不明確且客戶端驗證很簡單時，使用較強的類型來限制 input（例如：用 `DateTime` 來取代 `String`）。這樣清楚明瞭並鼓勵客戶端使用比較嚴格的輸入控制（例如：用日期選擇器取代自由的文字輸入框）。*

### Input：結構，第二部分

我們繼續處理 update mutation 的部分，它可能看起來像這樣：

```graphql
type Mutation {
  # ...
  collectionCreate(title: String!, ruleSet: CollectionRuleSetInput, image: ImageInput, description: String)
  collectionUpdate(collectionId: ID!, title: String, ruleSet: CollectionRuleSetInput, image: ImageInput, description: String)
}
```

你會注意到，它跟我們的 create mutation 非常相似，但有兩個不同之處：多了一個 `collectionId` 參數來決定要更新哪個 collection，以及不再需要 `title` 因為這個 collection 已經有了。Ignoring the title's required status for a moment, our example mutations have four duplicate arguments, and a complete collections model would include quite a few more。

While there are some arguments for leaving these mutations as-is, we have decided that situations like this call for DRYing up the common portions of the arguments, even at the cost of requiredness。這樣有幾個優點：
- We end up with a single input object representing the concept of a collection and mirroring the single `Collection` type our schema already has。
- 客戶端可以在 create 跟 update 表單之間共享程式碼 （一種常見的模式）因為他們最終操作的是同一種 input 物件。
- Mutation 只有幾個頂層參數，維持輕量以及可讀。

當然，主要的代價是，從 schema 上不再能清楚知道在建立時 title 是必要的。我們的 schema 最後看起來像這樣：

```graphql
type Mutation {
  # ...
  collectionCreate(collection: CollectionInput!)
  collectionUpdate(collectionId: ID!, collection: CollectionInput!)
}

input CollectionInput {
  title: String
  ruleSet: CollectionRuleSetInput
  image: ImageInput
  description: String
}
```

*準則 #21：結構化 mutation 的 input 來減少重複，即便這需要放寬某些欄位的必填限制。*

### Output

我們需要處理的最候一個設計問題是 mutation 的回傳值。 通常情況下，mutation 可能成功也可能失敗，雖然 GraphQL 確實有明確地支援查詢層級的錯誤，但這對業務層級的 mutation 失敗並不理想。因此，我們將這些頂層的錯誤保留給客戶端（例如：請求一個不存在的欄位）而不要用在用戶上。因此，每一個 mutation 都應該定義一個「payload」類型，除了其他可能有用的值以外，還要包含一個使用者錯誤的欄位。對於 create 來說，它可能看起來像這樣：

```graphql
type CollectionCreatePayload {
  userErrors: [UserError!]!
  collection: Collection
}

type UserError {
  message: String!

  # 導致錯誤的 input 欄位的路徑。
  field: [String!]
}
```

在這裡，成功的 mutation 會回傳一個空的列表給 `userErrors`，並會回傳剛建立的 collection 給 `collection` 欄位。不成功成功的 mutation 會回傳一個以上的 `UserError` 物件，並回傳 `null` 給 collection。

*準則 #22：Mutation 應該透過 mutation payload 的 `userErrors` 欄位來提供使用者/業務層級錯誤。底層的查詢錯誤欄位是保留給客戶端和伺服器層級的錯誤。*

在許多實作中，大多是自動提供這種結構，而你需要定義的只是 `collection` 回傳欄位。

update mutation 的部分，我們遵循完全相同的模式：

```graphql
type CollectionUpdatePayload {
  userErrors: [UserError!]!
  collection: Collection
}
```

值得注意的是，甚至在這裡，`collection` 仍然可以為空值
，因為如果提供的 ID 不代表有效的 collection，則沒有要回傳的 collection。

*準則 #23：大部分 mutation 的 payload 欄位應該要可以接受空值，除非在每種可能的錯誤情況下都確實有要回傳的值。*

## TLDR：設計準則

- 準則 #1：在處理特定欄位之前，總是從物件以及它們之間關聯的高階概覽開始。
- 準則 #2：永遠不要在你的 API 設計中暴露實作細節。
- 準則 #3：圍繞業務領域來設計你的 API，而不要圍繞實作、使用者介面、或是舊的 API。
- 準則 #4：移除欄位遠比添加欄位來的困難。
- 準則 #5：主要的業務物件類型都應該實作 `Node` 介面。
- 準則 #6：把密切相關的欄位抽出來放到子物件。
- 準則 #7：總是檢查清單欄位是否應該被分頁。
- 準則 #8：總是使用物件參照而不是 ID 欄位。
- 準則 #9：根據合理性選擇欄位名稱，而不要根據實作或是舊的 API 怎麼稱呼它。
- 準則 #10：當你開放查詢欄位的值有特殊語意，使用自定義的 scalar 類型。
- 準則 #11：對只能接受一組特定值的欄位使用列舉。
- 準則 #12：API 應該提供商業邏輯，而不只是資料。複雜的計算應該在伺服器上統一完成，而不是在許多的客戶端上做。
- 準則 #13：也要提供原始資料，即使它周圍已經有商業邏輯。
- 準則 #14：為資源上邏輯獨立的操作撰寫不同的 mutation。
- 準則 #15：操作物件關聯非常複雜，不容易總結成一條簡潔的規則。
- 準則 #16：當為關聯撰寫個別的 mutation 時，考慮一次對多個項目進行操作是否會對這個 mutation 有用。
- 準則 #17：為了按照字母分組，把 mutation 操作的物件名稱前綴在 mutation 名稱上（例如，使用 `orderCancel` 而不是 `cancelOrder`）。
- 準則 #18：只有 mutation 語義上確實需要該 input 欄位才能執行，才把它標為必填。
- 準則 #19：當格式明確而且客戶端驗證很複雜時，使用較弱的類型來限制 input（例如：用 `String` 來取代 `Email`）。這讓伺服器可以一次執行所有複雜的驗證並用單一格式統一回傳錯誤，簡化了客戶端的複雜度。
- 準則 #20：當格式可能不明確且客戶端驗證很簡單時，使用較強的類型來限制 input（例如：用 `DateTime` 來取代 `String`）。這樣清楚明瞭並鼓勵客戶端使用比較嚴格的輸入控制（例如：用日期選擇器取代自由的文字輸入框）。
- 準則 #21：結構化 mutation 的 input 來減少重複，即便這需要放寬某些欄位的必填限制。
- 準則 #22：Mutation 應該透過 mutation payload 的 `userErrors` 欄位來提供使用者/業務層級錯誤。底層的查詢錯誤欄位是保留給客戶端和伺服器層級的錯誤。
- 準則 #23：大部分 mutation 的 payload 欄位應該要可以接受空值，除非在每種可能的錯誤情況下都確實有要回傳的值。

## 尾聲

感謝你閱讀這份教學！希望此時你對於如何設計一個好的 GraphQL API 已經擁有確切的想法。

一旦你設計了一個滿意的 API，就是時候動手實作了！
