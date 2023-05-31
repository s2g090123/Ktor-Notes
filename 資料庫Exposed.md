# 資料庫Exposed

## 什麼是Exposed
![](https://hackmd.io/_uploads/HJz6f7QL3.png)

Exposed是Kotlin的ORM framework，使用Exposed來簡化對資料庫的操作與存取。
就像是Android Room的存在，透過一系列簡潔的API，來達到資料庫的操作。

## Exposed的特點
1. 簡潔的資料庫操作：Exposed提供了一系列簡潔方便的API，不管是在創建表格、執行CRUD等操作都能夠簡單實作
2. 不需要太懂SQL語法：所有功能都透過extension function包裝起來，比較複雜的操作都可以透過程式邏輯實現
3. 支援多種資料庫引擎：常見的有:
   - H2
   - MySQL
   - SQLite
   - ...

當然還是有一些缺點：
1. 在SQL簡易的語法裡，這裡可能需要進行較麻煩的操作
2. Table與我們的DataClass沒有直接關係，因此在修改資料結構上，程序會相當繁瑣

## 安裝Exposed
在你的`gradle`加上(範例使用`H2`資料庫):
```groovy
dependencies {
    implementation("org.jetbrains.exposed:exposed-core:$exposed_version")
    implementation("org.jetbrains.exposed:exposed-dao:$exposed_version")
    implementation("org.jetbrains.exposed:exposed-jdbc:$exposed_version")
    implementation("com.h2database:h2:$h2_version")
}
```

## 新增Table
這是我們原先的`Customer`:
```kotlin
@Serializable
data class Customer(
    val id: String,
    val firstName: String,
    val lastName: String,
    val email: String
)

val customerStorage = mutableListOf<Customer>()
```
當初我們用Memory來紀錄`Customer`的資料，但這樣只要Server關閉後，資料也會跟著不見，因此我們需要先為`Customer`建立資料庫表格的格式：
```kotlin
object Customers : Table() {
    val id = long("id").autoIncrement()
    val firstName = varchar("firstName", 128)
    val lastName = varchar("lastName", 128)
    val email = varchar("email", 255)

    override val primaryKey: PrimaryKey = PrimaryKey(id)
}
```
- 繼承於`Table`
- `long`, `varchar`用來建立`Column<T>`類別的物件，即對應類型的欄位，而後面都會需要一個參數，代表欄位的名稱，例如`id`。還有`integer`, `float`, `boolean`等新增不同類型欄位的方式
- `varchar`後面的數字，代表最長能存多長的字串，例如`firstName`最多只能存128個字元
- `autoIncrement`指的是當新增新的資料時，自動為這個欄位遞增值(只支援`integer`與`long`欄位)
- `PrimaryKey`用於宣告哪一個欄位為PrimaryKey，例如`id`會被宣告為PrimaryKey
- 此外，也可以覆寫`join操作`、`新建的statement`等

## 資料庫連結
建立`DatabaseFactory`:
```kotlin
import org.jetbrains.exposed.sql.Database

object DatabaseFactory {
    fun init() {
        val driverClassName = "org.h2.Driver"
        val jdbcURL = "jdbc:h2:file:./build/db"
        val database = Database.connect(jdbcURL, driverClassName)
    }
}
```
- 建立了`DatabaseFactory#init`來進行資料庫的連線
- 在`Database#connect`中，我們會需要`driverClassName`和`jdbcURL`
- `driverClassName`為使用的資料庫類型，因為範例中，我們使用`H2`資料庫，因此使用`H2`的Driver，若使用其他的資料庫，則需要使用對應的Driver，例如使用`MySQL`的話，則使用`com.mysql.jdbc.Driver`
- `jdbcURL`透過URL來使用Driver建立資料庫與其連線，當中的格式為，`jdbc`協議、`h2`子協議(資料庫類型)、`file`定義資料庫位置
- 這兩項內容為成對存在，因此改了其中一項內容，要注意是否另外一項也要更新
- **`Database.connect`並未實際建立資料庫的連線**，只是用於宣告將來會有與資料庫連線的操作，直到之後呼叫了`transaction`才算是真正的連線。

為`Customer#id`增加預設值:
```kotlin
@Serializable
data class Customer(
    val id: Long = 0,
    val firstName: String,
    val lastName: String,
    val email: String
)
```

### 建立表格
```kotlin
import org.jetbrains.exposed.sql.SchemaUtils
import org.jetbrains.exposed.sql.transactions.transaction

fun init() {
    // ...
    val database = Database.connect(jdbcURL, driverClassName)
    transaction(database) {
        SchemaUtils.create(Customers)
    }
}
```
- 所有與資料庫有關的操作，都需要在`transaction`的範圍內執行
- 此時才真正地建立起資料庫的連線
- 如果你有多個Database的話，可以對`transaction`設定`database`參數，否則預設為最後一個執行`Database#connect`的Database
- 透過`SchemaUtils`來建立`Customer`表格

### 資料庫Query
為了之後方便執行Query，建立`dbQuery`:
```kotlin
import org.jetbrains.exposed.sql.transactions.experimental.newSuspendedTransaction

object DatabaseFactory {
    fun init() {
        // ...
    }

    suspend fun <T> dbQuery(block: suspend () -> T): T =
        newSuspendedTransaction(Dispatchers.IO) { block() }
}
```

### 執行資料庫
在configuration中呼叫`DatabaseFactory#init`:
```kotlin
fun Application.module() {
    DatabaseFactory.init()
    configureRouting()
    configureSerialization()
}
```

## 建立Dao
在Android Room中有Dao，只要填入對應的SQL指令，就可以幫我們執行特定的CRUD操作。
然而，在`Exported`中，並沒有所謂的`DAO`接口，但所有的操作都被包在我們的`Table`中，它透過extension function更加簡潔撰寫這些操作，因此不需要去撰寫額外的SQL指令，只要使用對應的API，也可以達到CRUD的操作。

為`Customer`建立DAO(DataSource):
```kotlin
interface CustomerDAO {
    suspend fun getAllCustomers(): List<Customer>
    suspend fun getCustomer(id: Long): Customer?
    suspend fun addCustomer(firstName: String, lastName: String): Customer?
    suspend fun updateCustomer(id: Long, firstName: String, lastName: String): Boolean
    suspend fun deleteCustomer(id: Long): Boolean
}
```
```kotlin
import com.example.database.DatabaseFactory.dbQuery

class CustomerDAOImpl : CustomerDAO {
    override suspend fun getAllCustomers(): List<Customer> {
        TODO("Not yet implemented")
    }

    override suspend fun getCustomer(id: Long): Customer? {
        TODO("Not yet implemented")
    }

    override suspend fun addCustomer(firstName: String, lastName: String): Customer? {
        TODO("Not yet implemented")
    }

    override suspend fun updateCustomer(id: Long, firstName: String, lastName: String): Boolean {
        TODO("Not yet implemented")
    }

    override suspend fun deleteCustomer(id: Long): Boolean {
        TODO("Not yet implemented")
    }
}
```
- 我們建立了`CustomerDAO`，來定義CRUD的API接口
- 並且建立了`CustomerDAOImpl`來實現這些操作
- **要記得所有和DB有關的操作，都要在`Transaction`內執行**，這裡我們使用剛剛建立`dbQuery`來執行

### 取得所有資料
我們可以透過`Table#selectAll`來取得此Table的所有資料
```kotlin
override suspend fun getAllCustomers(): List<Customer> = dbQuery {
    Customers.selectAll().map { resultRow ->
        Customer(
            id = resultRow[Customers.id],
            firstName = resultRow[Customers.firstName],
            lastName = resultRow[Customers.lastName],
            email = resultRow[Customers.email]
        )
    }
}
```
- `Table#selectAll`可以取得`Query(Iterable<ResultRow>)`物件
- `ResultRow`即為對應的每一列資料
- 透過`ResultRow#get`來取得對應欄位的值，`resultRow[Customers.id]`即為取得這一列`id`欄位的值

為了方便後續轉換Customer:
```kotlin
private fun ResultRow.toCustomer(): Customer {
    return Customer(
        id = get(Customers.id),
        firstName = get(Customers.firstName),
        lastName = get(Customers.lastName),
        email = get(Customers.email)
    )
}
```

### 取得特定資料
透過`Table#select`代入條件式(`where`)，取得對應條件的資料
```kotlin
override suspend fun getCustomer(id: Long): Customer? = dbQuery {
    Customers
        .select { Customers.id eq id }
        .map { resultRow -> resultRow.toCustomer() }
        .singleOrNull()
}
```
- `Table#select`可以取得`Query(Iterable<ResultRow>)`物件
- 可以使用的operations有：比較(`eq`, `less`, `greater`)、算數(`plus`, `times`)、是否存在(`inList`, `notInList`)

### 新增資料
透過`Table#insert`新增資料
```kotlin
override suspend fun addCustomer(firstName: String, lastName: String, email: String): Customer? = dbQuery {
    val insertStatement = Customers.insert { insertStatement ->
        insertStatement[Customers.firstName] = firstName
        insertStatement[Customers.lastName] = lastName
        insertStatement[Customers.email] = email
    }
    insertStatement.resultedValues?.singleOrNull()?.toCustomer()
}
```
- `Table#insert`當中有一個`body`以`InsertStatement`作為參數的函數參數
- 透過`InsertStatement#get`取得對應欄位的`Column<T>`物件
- 將需要新增的資料，設定於對應的`Column`中，例如`insertStatement[Customers.firstName] = firstName`即設定`firstName`這個欄位的值
- 最後，透過返回的`InsertStatement#resultedValues`，來取得新增的結果

### 編輯資料
透過`Table#update`對特定資料進行更新
```kotlin
override suspend fun updateCustomer(id: Long, firstName: String, lastName: String, email: String): Boolean = dbQuery {
    Customers.update({
        Customers.id eq id
    }) { updateStatement ->
        updateStatement[Customers.firstName] = firstName
        updateStatement[Customers.lastName] = lastName
        updateStatement[Customers.email] = email
    } > 0
}
```
- `Table#update`當中有一個`where`參數，可以填入條件值
- `Table#update`當中有一個`body`以`UpdateStatement`作為此參數的函數參數
- 透過`UpdateStatement#get`取得對應欄位的`Column<T>`物件
- 將需要新增的資料，設定於對應的`Column`中，例如`updateStatement[Customers.firstName] = firstName`即更新`firstName`這個欄位的值
- 若成功更新內容，則回傳1，否則回傳0

### 刪除資料
透過`Table#deleteWhere`對特定條件的值進行刪除
```kotlin
override suspend fun deleteCustomer(id: Long): Boolean = dbQuery {
    Customers.deleteWhere { Customers.id eq id } > 0
}
```
- `Table#update`當中有一個`where`參數，可以填入條件值，對符合條件的資料進行刪除
- 若有刪除到內容，則回傳1，否則回傳0

## 更新Route
首先，在`DatabaseFactory`宣告DAO:
```kotlin
object DatabaseFactory {
     val dao: CustomerDAO = CustomerDAOImpl()   
}
```
### 更新GetAllCustomers
```kotlin
get {
    val customers = dao.getAllCustomers()
    if (customers.isNotEmpty()) {
        call.respond(customers)
    } else {
        call.respondText("No customers found", status = HttpStatusCode.OK)
    }
}
```
### 更新GetCustomers
```kotlin
get("{id?}") {
    val id = call.parameters["id"]?.toLongOrNull() ?: return@get call.respondText(
        "Missing id",
        status = HttpStatusCode.BadRequest
    )
    val customer = dao.getCustomer(id) ?: return@get call.respondText(
            "No customer with id $id",
            status = HttpStatusCode.NotFound
        )
    call.respond(customer)
}
```
### 新增Customer
```kotlin
post {
    val customer = call.receive<Customer>()
    dao.addCustomer(customer.firstName, customer.lastName, customer.email)
    call.respondText("Customer stored correctly", status = HttpStatusCode.Accepted)
}
```
### 刪除Customer
```kotlin
delete("{id?}") {
    val id = call.parameters["id"]?.toLongOrNull() ?: return@delete call.respond(HttpStatusCode.BadRequest)
    val isRemoved = dao.deleteCustomer(id)
    if (isRemoved) {
        call.respondText("Customer removed correctly", status = HttpStatusCode.Accepted)
    } else {
        call.respondText("Not Found", status = HttpStatusCode.NotFound)
    }
}
```
### 移除customerStorage
```kotlin
- val customerStorage = mutableListOf<Customer>()
```