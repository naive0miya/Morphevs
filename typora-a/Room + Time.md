# 在 Room 中使用时间

> 原文 (Medium)：[Room + Time](https://medium.com/@chrisbanes/room-time-2b4cf9672b98)
>
> 作者： [Chris Banes](https://medium.com/@chrisbanes)

[TOC]



如果你已经开始使用 Room（如果你没有使用 [Room](https://developer.android.com/topic/libraries/architecture/room.html)），很有可能你需要存储+取回某种日期/时间。Room 并不提供任何支持，而是提供了可扩展的 [@TypeConverter](https://developer.android.com/reference/android/arch/persistence/room/TypeConverter.html) 注解，它允许你从任意对象到 Room 可以理解的类型提供映射，反之亦然。

该 API 的[文档](https://developer.android.com/topic/libraries/architecture/room.html#type-converters)中的规范示例实际上是日期/时间：
```kotlin
public class Converters {
    @TypeConverter
    public static Date fromTimestamp(Long value) {
        return value == null ? null : new Date(value);
    }

    @TypeConverter
    public static Long dateToTimestamp(Date date) {
        return date == null ? null : date.getTime();
    }
}
```
我在我的应用程序中使用了这个确切的代码，虽然它在技术上有效，但它有两个大问题。 首先是它使用了 Date 类，几乎在所有情况下都应该避免。Date 的主要问题是它不支持时区。第二个问题是它作为一个简单的 Long 保留了这个值，它不能存储任何时区信息。 

所以假设我们使用上面的转换器将 Date 实例保存到数据库中，然后再检索它。 你怎么知道原始值是从哪个时区来的？ 答案很简单，你不能知道。 你能做的最好的事情就是尝试确保所有 Date 实例都使用一个通用的时区，如 UTC。 虽然这允许你比较不同的检索值(即排序) ，但是你永远无法找到原始的时区。 

我决定花一个小时试图解决我的应用程序中的时区问题。

## SQLite + 日期/时间
我调查的第一件事是 SQLite 对日期和时间值的支持，事实上它确实支持它们。 当你使用 Room 时，它控制哪些 SQL 数据键入你的类值映射到。 例如，String 将映射到 TEXT，Int 到 INTEGER 等。 但是我们如何告诉 Room 映射我们的对象日期 + 时间值呢？ 答案很简单，我们不需要这样做。 

SQLite 是松散类型的数据库系统，它将所有值存储为 NULL，INTEGER，TEXT，REAL 或 BLOB。没有特殊的日期或时间类型可以在其他数据库系统中找到。 相反，他们提供了下列关于如何存储日期 / 时间值的[文档](https://sqlite.org/datatype3.html#datetime): 

>SQLite 没有专门用于存储日期和/或时间的存储类。相反，SQLite 的内置日期和时间函数能够将日期和时间存储为 TEXT，REAL 或 INTEGER 值
>正是这些日期和时间功能，使我们能够以最小/无精度损失存储高保真日期 - 时间值，特别是使用支持 [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) 字符串的 TEXT 类型。

因此，我们只需要将我们的值保存为特别格式化的文本，它包含了我们需要的所有信息。 然后，我们可以使用上面提到的 SQLite 函数将我们的文本转换为 SQL 中的日期 / 时间。 我们唯一需要做的就是确保我们的代码使用正确的格式。 

## 回到应用程序

所以我们知道 SQLite 支持我们的需求，但是我们需要决定如何在我们的应用程序中表示这一点。 

我在我的应用程序中使用了 [ThreeTen-BP](http://www.threeten.org/threetenbp/)，这是 JDK 8 日期和时间库（JSR-310）的后端，但是在 JDK 6 + 上工作。  这个库支持时区，因此我们将使用其中的一个类来表示 app 中的日期 + 时间: OffsetDateTime。 这个类是一个时间和日期的不可变表示，在 utc / gmt 的特定偏移量内。 

因此，当我们查看我的一个实体时，我们现在使用 OffsetDateTime 而不是 Date :

```kotlin
@Entity(tableName = "users")
data class User(
        @PrimaryKey val id: Long? = null,
        val username: String,
        val joined_date: OffsetDateTime? = null
)
```
这是实体更新，但现在我们必须更新我们的 TypeConverters，以便 Room 了解如何持久/恢复 OffsetDateTime 值：
```kotlin
object TiviTypeConverters {
    private val formatter = DateTimeFormatter.ISO_OFFSET_DATE_TIME

    @TypeConverter
    @JvmStatic
    fun toOffsetDateTime(value: String?): OffsetDateTime? {
        return value?.let {
            return formatter.parse(value, OffsetDateTime::from)
        }
    }

    @TypeConverter
    @JvmStatic
    fun fromOffsetDateTime(date: OffsetDateTime?): String? {
        return date?.format(formatter)
    }
}
```
我们现在映射 OffsetDateTime / String ，而不是我们以前的日期映射 Data / Long。

方法很简单：一个将 OffsetDateTime 格式化为一个 String，另一个将 String 解析为 OffsetDateTime。 这里关键的难题是确保我们使用正确的字符串格式。感谢 ThreeTen-BP 提供了一个兼容的 DateTimeFormatter.ISO 偏移日期时间。 

```kotlin
DateTimeFormatter.ISO_OFFSET_DATE_TIME.
```
不过你可能不会使用此库，所以让我们看看一个示例格式化字符串:  2013-10-07T17：23：19.540-04：00。 希望你能看到这代表什么日期：2013年10月7日，17：23：19.540 UTC-4。 只要你格式化/解析成这样的字符串，SQLite 将能够理解它。

所以在这一点上，我们差不多完成了。 如果你运行应用程序，使用适当的数据库版本增加 + 迁移，你会发现所有的东西都应该工作得很好。 

有关与 Room 迁移的更多信息，请参阅 [Florina Muntenescu](https://medium.com/@florina.muntenescu) 的[帖子](https://medium.com/google-developers/understanding-migrations-with-room-f01e04b07929)：

## Room 查询
有一件事我们还没有解决，那就是在 SQL 中查询日期列。 之前的 Date / Long 映射有一个隐含的好处，即数字对排序和查询极为有效。 移动到一个字符串在某种程度上打破了这一点，所以让我们来解决它。 

假设我们之前有一个查询，该查询返回所有用户按照他们的加入日期。 你可能会遇到这样的情况: 

```kotlin
@Dao
interface UserDao {
    @Query("SELECT * FROM users ORDER BY joined_date")
    fun getOldUsers(): List<User>
}
```
由于 joined_date 是一个数字（long，记住），SQLite 会做一个简单的数字比较，并返回结果。如果你用新的文本实现运行相同的查询，你可能会注意到结果看起来是一样的，但是它们是吗？ 

那么答案是肯定的，大部分时间。随着文本的实现，SQLite 正在做一个文本排序而不是数字排序，这在大多数情况下是正确的。让我们看看一些示例数据：
```
id  |  joined_date
------------------------------------
1   |  2017-10-17T07:23:19.120+00:00
2   |  2017-10-17T09:36:27.526+00:00
3   |  2017-10-17T11:01:12.972+00:00
4   |  2017-10-17T17:57:01.784+00:00
```
一个简单的从左到右的字符串排序在这里工作，因为所有的字符串组件都处于递减顺序(年、月、日等)。 问题与字符串的最后一个组件一起出现，时区被抵消了。 让我们稍微调整一下数据，看看会发生什么: 

```
id  |  joined_date
------------------------------------
1   |  2017-10-17T07:23:19.120+00:00
2   |  2017-10-17T09:36:27.526+00:00
3   |  2017-10-17T11:01:12.972-02:00
4   |  2017-10-17T17:57:01.784+00:00
```
你可以看到第三行的时区已经从 UTC 变为 UTC-2。 这导致它的联接时间实际上是09:01:12在 UTC，因此它应该被分类为第二行。 不过，归还的名单上的顺序和以前一样。 这是因为我们仍然使用字符串排序，这不考虑时区。 

## SQLite 日期/时间函数

那么我们如何解决这个问题呢？ 还记得那些 SQLite 日期 / 时间函数吗？ 我们只需要确保在与 SQL 中的任何日期 / 时间列进行交互时使用它们。 提供了5个[函数](https://sqlite.org/lang_datefunc.html): 

1. date（...）只返回日期。
2. time（...）只返回时间。
3. datetime（...）返回日期和时间。
4. julianday（...）返回 [Julian Day](https://en.wikipedia.org/wiki/Julian_day)。
5. strftime（...）返回与给定格式字符串格式化的值。前四个可以被认为是具有预定义格式的 strftime 的变体。

由于我们想要对日期和时间进行排序，我们可以使用 datetime (...) 函数。 如果我们回到我们的 DAO，查询现在变成: 

```kotlin
@Dao
interface UserDao {
    @Query("SELECT * FROM users ORDER BY datetime(joined_date)")
    fun getOldUsers(): List<User>
}
```
够简单吧？在我们做出这个改变之后，我们现在得到了正确的语义顺序：
```
id  |  joined_date
------------------------------------
1   |  2017-10-17T07:23:19.120+00:00
3   |  2017-10-17T11:01:12.972-02:00
2   |  2017-10-17T09:36:27.526+00:00
4   |  2017-10-17T17:57:01.784+00:00
```
那就是我的工作完成时间！我们现在在 Room 里支持时间划分的日期/时间。

[all code in gist](https://github.com/chrisbanes/tivi/commit/bd517e2b2fb54046a5e3c8bdcd31133ad1db1b19)






