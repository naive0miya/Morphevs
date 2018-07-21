# Kotlin 带参数的单例

> 原文 (Medium) ：[Kotlin singletons with argument](https://medium.com/@BladeCoder/kotlin-singletons-with-argument-194ef06edd9e)
>
> 作者 ：[Christophe Beyls ](https://medium.com/@BladeCoder?source=post_header_lockup)

> 对象有其局限性

在 Kotlin，单例模式被用来代替那种编程语言中不存在的静态成员和字段。单例是通过简单地[声明一个对象](https://kotlinlang.org/docs/reference/object-declarations.html#object-declarations)而创建的。

```kotlin
object SomeSingleton
```

与类相反，对象不能有任何构造函数，但是如果需要一些初始化代码，则允许 init 块。

```kotlin
object SomeSingleton {
    init {
        println("init complete")
    }
}
```

对象将被实例化，其内部块将在第一次访问时以线程安全的方式懒惰地执行。 为了实现这一点，Kotlin 对象实际上依赖于一个 Java 静态初始化块。 上述 Kotlin 对象将编译成以下等效的 Java 代码:

```Java
public final class SomeSingleton {
   public static final SomeSingleton INSTANCE;

   private SomeSingleton() {
      INSTANCE = (SomeSingleton)this;
      System.out.println("init complete");
   }

   static {
      new SomeSingleton();
   }
}
```

这是在 JVM 上实现单例的首选方法，因为它可以实现线程安全的惰性初始模式，而无需依赖复杂的[双重检查锁定](https://en.wikipedia.org/wiki/Double-checked_locking#Usage_in_Java)模式这样的锁定算法。 通过在 Kotlin 使用一个对象声明，保证你将得到一个安全和高效的单例实现。

#### 传递参数

但是如果初始化代码需要一些额外的参数呢？ 因为 Kotlin 对象不能有任何构造函数，所以你不能传递任何参数给它。

有一些有效的例子表明，将参数传递给单例初始化块是推荐的模式。 这种替代方案要求单例类别知道一些外部组件能够检索该参数，违反了关注点分离原则，使代码不那么可重用。 为了缓解这个问题，外部组件可能是一个依赖注入系统。 这是一个有效的解决方案，但是你并不总是想要使用那种类型的库，有些情况下它不能被使用，就像我们将在下面的 Android 示例中看到的那样。

另一种情况是，你必须依赖一种不同的方式来管理 Kotlin 单例实例的情况是，单例的具体实现是由外部工具或库(Retrofit，Room，...)生成单例的具体实现，并使用自定义生成器或工厂方法检索其实例。 在这种情况下，你通常将单例声明为接口类或抽象类，它不能是对象。

#### 一个 Android 用例

在 Android 平台上，你通常需要通过上下文实例来初始化单例组件的块，这样他们就可以检索文件路径、读取设置或访问服务，但是你也要避免对它保持静态引用(即使静态引用上下文在技术上是安全的)。 有两种方法可以达到这个目的:

早期初始化: 在应用程序上下文可用的情况下，通过在 Application.onCreate ( ) 中在启动时调用公共 init 函数来初始化所有组件。 这个简单的解决方案通过阻塞主线程来减缓应用程序启动速度，初始化所有组件，包括那些不会立即使用的组件。 另一个不为人所知的问题是，在调用此方法之前可能会创建内容提供程序(如文档中所述) ，因此，如果内容提供者使用全局应用程序组件，则该组件必须能够在 Application.onCreate ( ) 或应用程序崩溃之前被初始化

惰性初始模式: 这是推荐的方法。 组件是一个单例，函数返回其实例需要一个上下文参数。 在首次调用该实例时，将使用此参数创建和初始化单例实例。 这需要一些同步机制才能保证线程安全。 一个使用这种模式的标准安卓组件的例子是 LocalBroadcastManager:

```kotlin
LocalBroadcastManager.getInstance(context).sendBroadcast(intent)
```

#### 一个可重用的 Kotlin 实现

我们可以封装逻辑，懒惰地创建和初始化一个 SingletonHolder 类中带有参数的单例。 为了使逻辑线程安全，我们需要实现一个同步的算法和最有效的算法，也是最难做到的，是双重检查锁定模式算法。

```kotlin
open class SingletonHolder<out T, in A>(creator: (A) -> T) {
    private var creator: ((A) -> T)? = creator
    @Volatile private var instance: T? = null

    fun getInstance(arg: A): T {
        val i = instance
        if (i != null) {
            return i
        }

        return synchronized(this) {
            val i2 = instance
            if (i2 != null) {
                i2
            } else {
                val created = creator!!(arg)
                instance = created
                creator = null
                created
            }
        }
    }
}
```

请注意，为了使算法正常工作，需要向实例字段添加 @volatile 注解。

这可能不是最紧凑或者最优雅的 Kotlin 代码，但它是为双重检查锁定模式算法生成最有效的字节码的代码。 相信 Kotlin 的作者: 这个代码实际上是直接从 Kotlin 标准库中的[懒惰函数](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/lazy.html)的实现中借用的，这个函数在默认情况下是同步的。 它被修改以允许将一个参数传递给创建者函数。

考虑到它的相对复杂性，它不是那种你想要多次写(或读)的代码类型，所以目标是在每次必须实现带有参数的单独实现时重用 SingletonHolder 类。

声明 getInstance ( ) 函数的逻辑位置是在单例类的伴随对象中，这样就可以简单地使用单例类名称作为限定符来调用 getInstance 函数，类似于 Java 中的静态方法。 Kotlin 伴随对象提供的一个强大特征是它们能够像其他对象一样从基类继承，这使得某些东西可以与纯静态继承相媲美。 在这种情况下，我们希望使用 SingletonHolder 作为单例对象的基类，以便在单例类中重用并自动公开 getInstance ( ) 函数。

至于作为 SingletonHolder 构造函数的参数传递的创建函数，可以将自定义的 lambda 声明为内联，但通常的解决方案是引用 singleton 类的私有构造函数。 一个具有参数的单例的最终代码是这样的:

```kotlin
class Manager private constructor(context: Context) {
    init {
        // Init using context argument
    }

    companion object : SingletonHolder<Manager, Context>(::Manager)
}
```

不像对象符号那么短，但可能是最好的东西。

现在可以使用下面的语法调用 singleton，它的初始化将是懒惰和线程安全的:

```kotlin
Manager.getInstance(context).doStuff()
```

当外部库生成单例实现时，你也可以使用这个习惯用法，而构建器需要一个参数。 以下是一个使用 Android 的 [Room 持久化库](https://developer.android.com/topic/libraries/architecture/room.html)的例子:

```kotlin
@Database(entities = arrayOf(User::class), version = 1)
abstract class UsersDatabase : RoomDatabase() {

    abstract fun userDao(): UserDao

    companion object : SingletonHolder<UsersDatabase, Context>({
        Room.databaseBuilder(it.applicationContext,
                UsersDatabase::class.java, "Sample.db")
                .build()
    })
}
```

**注意:** 当生成器不需要参数时，你可以简单地使用一个懒惰的委托属性来代替:

```kotlin
interface GitHubService {

    companion object {
        val instance: GitHubService by lazy {
            val retrofit = Retrofit.Builder()
                    .baseUrl("https://api.github.com/")
                    .build()
            retrofit.create(GitHubService::class.java)
        }
    }
}
```

我希望你发现这些代码段很有用。 如果你想提出其他方法来解决这个问题或者有问题，不要犹豫在评论部分开始讨论。 谢谢你的阅读！





