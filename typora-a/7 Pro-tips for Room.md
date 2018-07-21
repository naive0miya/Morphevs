# 使用Room的7个技巧

> 原文 (Medium)： [7 Pro-tips for Room](https://medium.com/google-developers/7-pro-tips-for-room-fbadea4bfbd1)
> 作者： [Florina Muntenescu](https://medium.com/@florina.muntenescu)

[TOC]

[Room](https://developer.android.com/topic/libraries/architecture/room.html) 是 SQLite 之上的一个抽象层，它使持久化数据变得更容易、更好。 如果你是新手，那么看看这本[入门书](https://medium.com/google-developers/7-steps-to-room-27a5fe5f99b2): 

在这篇文章中，我想分享一些关于充分利用 Room 的技巧：

## 预先填充数据库
你是否需要在数据库创建之后或数据库打开时向数据库添加默认数据？使用 [RoomDatabase#Callback](https://developer.android.com/reference/android/arch/persistence/room/RoomDatabase.Callback.html)！构建 RoomDatabase 时调用 [addCallback ](https://developer.android.com/reference/android/arch/persistence/room/RoomDatabase.Builder.html#addCallback%28android.arch.persistence.room.RoomDatabase.Callback%29) 方法，并覆盖 [onCreate](https://developer.android.com/reference/android/arch/persistence/room/RoomDatabase.Callback.html#onCreate%28android.arch.persistence.db.SupportSQLiteDatabase%29) 或 [onOpen](https://developer.android.com/reference/android/arch/persistence/room/RoomDatabase.Callback.html#onOpen%28android.arch.persistence.db.SupportSQLiteDatabase%29) 。

当数据库第一次创建时，在创建表之后，onCreate 将被调用。 数据库打开时，onOpen 就会被调用。 由于 DAOs 只能在这些方法返回后被访问，所以我们正在创建一个新的线程，在那里我们可以获得对数据库的引用，获取 DAO，然后插入数据。 

```kotlin
Room.databaseBuilder(context.applicationContext,
        DataDatabase::class.java, "Sample.db")
        // 在调用 onCreate 之后预填充数据库
        .addCallback(object : Callback() {
            override fun onCreate(db: SupportSQLiteDatabase) {
                super.onCreate(db)
                // 移动到一个新的线程
                ioThread {
                    getInstance(context).dataDao()
                                        .insert(PREPOPULATE_DATA)
                }
            }
        })
        .build()
```
这里有一个完整的[例子](https://gist.github.com/florina-muntenescu/697e543652b03d3d2a06703f5d6b44b5)。  
**注意：**注意: 当使用 ioThread 方法时，如果你的应用程序在第一次启动时崩溃，在数据库创建和插入之间，数据将永远不会被插入。 

## 使用 DAO 的继承功能
你的数据库中是否有多个表，并发现自己复制了相同的 Insert，Update 和 Delete 方法？ DAO 支持继承，所以创建一个 BaseDao \<T> 类，并在那里定义通用的 @Insert，@Update 和 @Delete 方法。 让每个 DAO 扩展 BaseDao 并添加针对它们的特定方法。 

```kotlin
interface BaseDao<T> {
    @Insert
    fun insert(vararg obj: T)
}
@Dao
abstract class DataDao : BaseDao<Data>() {
    @Query("SELECT * FROM Data")
    abstract fun getData(): List<Data>
}
```
在这里看一个完整的 [例子](https://gist.github.com/florina-muntenescu/1c78858f286d196d545c038a71a3e864)。
DAO 必须是接口或抽象类，因为 Room 会在编译时生成它们的实现，包括来自 BaseDao 的方法。

## 使用最少的样板代码在事务中执行查询
使用 [@Transaction](https://developer.android.com/reference/android/arch/persistence/room/Transaction.html) 注解一个方法可以确保你在该方法中执行的所有数据库操作都将在一个事务中运行。在方法体中抛出异常时，事务将失败。
```kotlin
@Dao
abstract class UserDao {
    
    @Transaction
    open fun updateData(users: List<User>) {
        deleteAllUsers()
        insertAll(users)
    }
    @Insert
    abstract fun insertAll(users: List<User>)
    @Query("DELETE FROM Users")
    abstract fun deleteAllUsers()
}
```
在以下情况下，你可能希望对具有 select 语句的 @Query 方法使用 @Transaction 注解：
- 当查询结果相当大时。 通过在一个事务中查询数据库，你可以确保如果查询结果不适用于单个游标窗口， 

  它不会因[游标窗口交换](https://medium.com/google-developers/large-database-queries-on-android-cb043ae626e8)之间的数据库更改而损坏。

- 当查询的结果是 @Relation 字段的 POJO 。这些字段是分开查询的，因此在单个事务中运行它们将保证查询之间的一致结果。

具有多个参数的 @Delete，@Update 和 @Insert 方法在事务内部自动运行。

## 只读你需要的东西
查询数据库时，是否使用了查询中返回的所有字段？ 请注意你的应用程序使用的内存量，并只加载你将最终使用的字段。 这还可以通过降低 IO 成本来提高查询速度。Room 会为你做 columns  和 object 之间的映射。

考虑这个复杂的用户对象：
```kotlin
@Entity(tableName = "users")
data class User(@PrimaryKey
                val id: String,
                val userName: String,
                val firstName: String, 
                val lastName: String,
                val email: String,
                val dateOfBirth: Date, 
                val registrationDate: Date)
```

在一些屏幕上，我们不需要显示所有这些信息。相反，我们可以创建一个仅保存所需数据的 UserMinimal 对象。

```kotlin
data class UserMinimal(val userId: String,
                       val firstName: String, 
                       val lastName: String)
```

在 DAO 类中，我们定义查询并从 users 表中选择正确的列。
```kotlin
@Dao
interface UserDao {
    @Query(“SELECT userId, firstName, lastName FROM Users)
    fun getUsersMinimal(): List<UserMinimal>
}
```

## 强制使用外键的实体之间的约束
即使 Room [不直接支持关系](https://developer.android.com/topic/libraries/architecture/room.html#no-object-references)，它也允许你定义实体之间的外键约束。

Room 具有 [@ForeignKey](https://developer.android.com/reference/android/arch/persistence/room/ForeignKey.html) 注解，它是 [@Entity](https://developer.android.com/reference/android/arch/persistence/room/Entity.html) 注解的一部分，允许使用 [SQLite 外键](https://sqlite.org/foreignkeys.html) 功能。 它强制跨表的约束，以确保在修改数据库时该关系是有效的。 对于一个实体，定义要引用的父实体、其中的列和当前实体中的列。 

考虑一个用户和宠物类。宠物有一个所有者，这是一个用户 id 被引用为外键。 

```kotlin
@Entity(tableName = "pets",
        foreignKeys = arrayOf(
            ForeignKey(entity = User::class,
                       parentColumns = arrayOf("userId"),
                       childColumns = arrayOf("owner"))))
data class Pet(@PrimaryKey val petId: String,
              val name: String,
              val owner: String)
```

或者，你可以定义在数据库中删除或更新父实体时应采取的操作。你可以选择以下之一：NO_ACTION，RESTRICT，SET_NULL，SET_DEFAULT 或 CASCADE，它们具有与 [SQLite](https://sqlite.org/foreignkeys.html#fk_actions) 中相同的行为。

**注意：**在 Room 中，SET_DEFAULT 作为 SET_NULL 工作，因为 Room 尚不允许为列设置默认值。

## 通过 @Relation 简化一对多的查询
在前面的 User-Pet 例子中，我们可以说我们有一个一对多的关系：一个用户可以有多个宠物。假设我们想要获得一个带有宠物的用户列表：List \<UserAndAllPets>。

```kotlin
data class UserAndAllPets (val user: User,
                           val pets: List<Pet> = ArrayList())
```
为了做到这一点，我们需要实现2个查询：一个获得所有用户的列表，另一个获取基于用户 id 的宠物列表。
```kotlin
@Query(“SELECT * FROM Users”)
public List<User> getUsers();
@Query(“SELECT * FROM Pets where owner = :userId”)
public List<Pet> getPetsForUser(String userId);
```

然后，我们将遍历用户列表并查询 Pets 表。

为了使这个更简单，Room 的 @Relation 注解自动获取相关的实体。@Relation 只能应用于 List 或 Set 对象。UserAndAllPets 类必须更新：
```kotlin
class UserAndAllPets {
   @Embedded
   var user: User? = null
   @Relation(parentColumn = “userId”,
             entityColumn = “owner”)
   var pets: List<Pet> = ArrayList()
}
```
在 DAO 中，我们定义了一个单一的查询，Room 将同时查询 Users 和 Pets 表并处理对象映射。
```kotlin
@Query(“SELECT * FROM Users”)
List<UserAndAllPets> getUsers();
```

## 避免针对可观察查询的误报通知
假设你希望在一个可观察的查询中基于用户 id 获得一个用户: 

```kotlin
@Query(“SELECT * FROM Users WHERE userId = :id)
fun getUserById(id: String): LiveData<User>
// or
@Query(“SELECT * FROM Users WHERE userId = :id)
fun getUserById(id: String): Flowable<User>
```
每当用户更新时，都会得到一个新的 User 对象。但当用户表中出现与你感兴趣的用户无关的其他更改(删除、更新或插入)时，也会得到相同的对象，这些变更与你感兴趣的用户没有任何关系，从而导致错误的正面通知。 更重要的是，如果你的查询涉及多个表，当其中任何一个发生变化时，都会得到一个新的发射。 

以下是背后发生的事情：

- SQLite 支持在表中发生 DELETE，UPDATE 或 INSERT 时触发[触发器](https://sqlite.org/lang_createtrigger.html)。
- Room 创建一个 [InvalidationTracker](https://developer.android.com/reference/android/arch/persistence/room/InvalidationTracker.html)，它使用 Observers 来跟踪被观察表中什么地方发生了变化。
- LiveData 和 Flowable 查询都依赖于[ InvalidationTracker.Observer#onInvalidated](https://developer.android.com/reference/android/arch/persistence/room/InvalidationTracker.Observer.html#onInvalidated%28java.util.Set%3Cjava.lang.String%3E%29) 通知。收到此消息后，会触发重新查询。

Room 只知道表格已被修改，但不知道为什么和发生了什么变化。 因此，在重新查询之后，查询的结果由 LiveData 或 Flowable 发出。 由于 Room 不在内存中保存任何数据，并且不能认为对象具有 equals ( )，所以它不能分辨这是否是相同的数据。

你需要确保你的 DAO 过滤发射的值，并且只对不同的物体做出反应。

如果可观察的查询是使用 Flowables 实现的，请使用 [Flowable#distinctUntilChanged](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html#distinctUntilChanged--) 。

```kotlin
@Dao
abstract class UserDao : BaseDao<User>() {
/**
* Get a user by id.
* @return the user from the table with a specific id.
*/
@Query(“SELECT * FROM Users WHERE userid = :id”)
protected abstract fun getUserById(id: String): Flowable<User>
fun getDistinctUserById(id: String): 
   Flowable<User> = getUserById(id)
                          .distinctUntilChanged()
}
```

如果你的查询返回一个 LiveData，则可以使用 MediatorLiveData，它只允许来自同一个源的不同对象发射。

```kotlin
fun <T> LiveData<T>.getDistinct(): LiveData<T> {
    val distinctLiveData = MediatorLiveData<T>()
    distinctLiveData.addSource(this, object : Observer<T> {
        private var initialized = false
        private var lastObj: T? = null
        override fun onChanged(obj: T?) {
            if (!initialized) {
                initialized = true
                lastObj = obj
                distinctLiveData.postValue(lastObj)
            } else if ((obj == null && lastObj != null) 
                       || obj != lastObj) {
                lastObj = obj
                distinctLiveData.postValue(lastObj)
            }
        }
    })
    return distinctLiveData
}
```
在你的 DAO 中，使方法返回不同的 LiveData public 以及查询 protected 的数据库的方法。
```kotlin
@Dao
abstract class UserDao : BaseDao<User>() {
@Query(“SELECT * FROM Users WHERE userid = :id”)
   protected abstract fun getUserById(id: String): LiveData<User>
fun getDistinctUserById(id: String): 
         LiveData<User> = getUserById(id).getDistinct()
}
```

在这里看到更多的 [代码](https://gist.github.com/florina-muntenescu/fea9431d0151ce0afd2f5a0b8834a6c7) 。

**注意：**如果要返回一个要显示的列表，请考虑使用 [Paging Library](https://developer.android.com/topic/libraries/architecture/paging.html) 并返回一个 [LivePagedListProvider](https://developer.android.com/reference/android/arch/paging/LivePagedListProvider.html)，因为该库将帮助自动计算列表项之间的差异并更新你的 UI 。



