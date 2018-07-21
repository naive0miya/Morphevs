# 存储库模式的演变——注意过度抽象

> 原文 (http://hannesdorfmann.com)：[THE EVOLUTION OF THE REPOSITORY PATTERN - BE AWARE OF OVER ABSTRACTION](http://hannesdorfmann.com/android/evolution-of-the-repository-pattern)
>
> 作者：[Hannes Dorfmann](http://hannesdorfmann.com/about/)

[TOC]

我和 Artem Zinnatullin 在我们的播客《[The Context](https://github.com/artem-zinnatullin/TheContext-Podcast)》中，一位听众问我，是否可以给他举一个存储库模式的例子。 所以我四处搜索，偶然发现了一些博客文章，发现存储库模式这个术语用了很多不同的方式来描述。 在这篇博客文章中，我想简要介绍一下存储库模式的历史，我想讨论为什么我认为存储库模式会导致过度抽象和过度工程化。 

在我们开始之前，我想说，存储库模式这个术语在不同的上下文和不同的编程语言中以不同的方式使用和定义。 因此，我把这篇文章分成两部分: 在第一部分中，我想讨论为什么我认为存储库模式的"原始"定义是一个你很少需要的过度设计的抽象概念，在第二部分，我想展示一个普遍和流行的定义存储库模式，可能是一个更好的选择平均使用情况。 

## 原始的存储库模式

据我所知，存储库模式最早是由 Martin Fowler 等人在《[企业应用程序架构模式](http://martinfowler.com/eaaCatalog/repository.html)》一书中首次提出的。 [这篇](https://medium.com/@krzychukosobudzki/repository-design-pattern-bc490b256006#.h7cm2b24q)由 Krzychu Kosobudzki 或 Christian Panadero 的[博客](http://panavtec.me/what-is-the-repository-pattern-and-why-im-not-going-to-use-it-on-android)为我们提供了一个在 android 开发中使用这种模式的例子。在我看来，原始的存储库模式是一个你很少需要的过度设计的抽象层。尤其是在 Android 上。 我说的过度设计是什么意思？ 让我们来看看 Krzychu Kosobudzki 的博客文章中的例子 : 

```java
interface Repository<T> {
    void add(T item);
    void remove(Specification specification);
    List<T> query(Specification specification);
}
```

规范类型是什么？ 基本上，规范只是一个接口，就像这样: 

```java
interface Specification { }

interface SqlSpecification extends Specification {
    String toSqlQuery();
}
```

然后我们可以使用具体的规范来从 SQLite 数据库中查询所有新闻列表，按日期排列，如下所示: 

```java
class NewestNewsesSqlSpecification implements SqlSpecification {

    @Override
    public String toSqlQuery() {
        return String.format(
                "SELECT  FROM %1$s ORDER BY `%2$s` DESC;",
                NewsTable.TABLE_NAME,
                NewsTable.Fields.DATE
        );
    }
}
```

规范的另一个例子如下: 

```java
public class NewsByIdSqlSpecification implements SqlSpecification {
    private int id;

    public NewsByIdSpecification(final int id) {
        this.id = id;
    }

    @Override
    public String toSqlQuery() {
        return String.format(
                "SELECT  FROM %1$s WHERE `%2$s` = %3$d';",
                NewsTable.TABLE_NAME,
                NewsTable.Fields.ID,
                id
        );
    }
}
```

然后使用这样的组件: 

```java
public class NewsSqlRepository implements Repository<News> {

    @Override
    public List<News> query(Specification specification) {
        SqlSpecification sqlSpecification = (SqlSpecification) specification;

        SQLiteDatabase database = openHelper.getReadableDatabase();
        List<News> newses = new ArrayList<>();

        Cursor cursor = database.rawQuery(sqlSpecification.toSqlQuery());

        for (int i = 0, size = cursor.getCount(); i < size; i++) {
            cursor.moveToPosition(i);
            newses.add(toNewsMapper.map(cursor));
        }
        cursor.close();
        return newses;
    }
}
```

同样，上面显示的所有代码片段都是从 Krzychu Kosobudzki [博客文章](https://medium.com/@krzychukosobudzki/repository-design-pattern-bc490b256006#.h7cm2b24q)中借鉴的。

通常你使用 NewsRepository，即在你的 Presenter (MVP) 或 use case / interactctor 中，如果你正在遵循简洁结构 : newsRepository.query（newNewNewsesSqlSpecification ( )）。

到目前为止还不错。 在我们继续之前，我想问你两个重要的问题: 

第一个问题：为什么，什么时候使用存储库模式？ 我想链接到 Martin Fowler 等人的《[企业应用程序架构模式](http://martinfowler.com/eaaCatalog/repository.html)》一书。

> 存储库将在幕后进行适当的操作。 概念上，存储库封装数据存储区中持久存在的对象集以及对它们执行的操作，提供了一个更加面向对象的持久层视图。 存储库还支持在域和数据映射层之间实现简洁分离和单向依赖的目标。 

第二个问题: 抽象的好处是什么？ 

- 单一责任：这个类负责制定一个“标准”。在 NewestNewsesSqlSpecification 的情况下，这个标准被转换成一个 SQL 字符串。
- 隐藏实现细节： SQL 语句隐藏在 NewestNewsesSqlSpecification 中，所以没有其他类需要了解 SQL。
- 我们可以重用规格：设想一个带有 SQL WHERE 子句的规范。我们可以定义一个这样的规范，并且在我们需要一个不同的基于 WHERE 子句的规范时重用它 。
- 开放/封闭的原则：正如 Krzychu Kosobudzki 正确指出的那样，这允许创建新的规范（因此新的功能），而无需触摸代码库中的其他东西。

这种额外的抽象层有什么不利之处？ 现在，无论是谁调用 newsRepository.query (new NewestNewsesSqlSpecification ( )) 必须知道要传递哪个规范。 这本身并不是一个问题，但是开发人员开始过度抽象这种模式。 例如，Krzychu Kosobudzki 说，通过存储库模式，我们可以随着需求的变化而改变具体的实现。 他举了一个使用 Realm 而不是 SQLite 数据库的例子: 

```java
public class NewsRealmRepository implements Repository<News> {

    @Override
    public List<News> query(Specification specification) {
        RealmSpecification realmSpecification = (RealmSpecification) specification;

        Realm realm = Realm.getInstance(realmConfiguration);
        RealmResults<NewsRealm> realmResults = realmSpecification.toRealmResults(realm);

        List<News> newses = new ArrayList<>();

        for (NewsRealm news : realmResults) {
            newses.add(toNewsMapper.map(news));
        }
        realm.close();
        return newses;
    }
}
```

正如你所看到的，还有另外一个子域的规范：

```java
public interface RealmSpecification extends Specification {
    RealmResults<NewsRealm> toRealmResults(Realm realm);
}

public class NewsByIdSpecification implements RealmSpecification {
    private final int id;

    public NewsByIdSpecification(final int id) {
        this.id = id;
    }

    @Override
    public RealmResults<NewsRealm> toRealmResults(Realm realm) {
        return realm.where(NewsRealm.class)
                .equalTo(NewsRealm.Fields.ID, id)
                .findAll();
    }
}
```

问题是，无论是谁调用 newsRepository.query ( ) 都必须知道它是否必须通过  NewestNewsesSqlSpecification 或 NewestNewsesRealmSpecification。那么，把你的实现从 SQLite 改成 Realm 真的很容易吗？ 不，这不是因为你必须在 FooSqlSpecification 到 FooRealmSpecification 的任何地方更改规格。 这里还有一个大的代码味道：

```java
@Override
public List<News> query(Specification specification) {
  SqlSpecification sqlSpecification = (SqlSpecification) specification;
  ...
}
```

```java
@Override
public List<News> query(Specification specification) {
  RealmSpecification realmSpecification = (RealmSpecification) specification;
  ...
}
```

无论何时，当你在用一个接口编程，但是你必须要在一个具体的类中设置一个接口，在 95% 的时间里你都在做错事！ 

Martin Fowler 等人没有对存储库模式和变更的具体实现(即 SQLite 到 Realm)发表声明。 Martin Fowler 等人将存储库模式描述为 

> 添加这个层可以减少重复查询逻辑。 

因此，存储库模式的最初定义是尽量减少重复查询逻辑，而不一定要建立一个抽象层来更改基础数据库。 

我不想责怪任何人。 这不是一个与 Android 相关的问题，我们也经常在 Java EE 世界中看到，特别是在 spring framework 支持的后端。 

## 存储库模式术语的演变

在 android 开发中，存储库模式这个术语经常与 Clean Architecture 结合使用。Fernando Cejas 在他出色的博客文章 [Architecting Android…The clean way?](http://fernandocejas.com/2014/09/03/architecting-android-the-clean-way/) 中也有介绍。 （向下滚动到“数据层”部分）。 有趣的是，他还链接到 Martin Fowler 等人对“存储库模式”的原始定义。然而，Fernando Cejas 表示：

> 应用程序所需的所有数据都来自该层，通过存储库实现(接口位于域层) ，该接口使用存储库模式，其策略是，通过一个工厂，根据某些条件选择不同的数据源。 

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frovbn5r7rj30go0beacs.jpg)

> 所有这一切背后的想法是，数据来源对客户来说是透明的，它不在乎数据是来自内存、磁盘还是云，唯一的真相是数据将会到达并且会被获取。 

正如你在这里看到的，存储库模式这个术语用来描述一些略有不同的东西。如果你在 Github 上查看 Fernando 的 [Clean Architecture](https://github.com/android10/Android-CleanArchitecture/tree/master/data/src/main/java/com/fernandocejas/android10/sample/data/repository) 示例项目，你会发现这一点更加清晰。在他的例子中，他使用了 RxJava。 对于我的博客文章，下面的代码片段中我已经删除了 RxJava，以便更好地理解，如果你没有以前的 RxJava 知识。 我也简化了一些代码，把重点放在重要的事情上。  例如，Fernando 希望通过 id 加载一个用户: 

```java
interface UserRepository {
  User user(int userId);
}
```

该存储库的具体实现：

```java
class UserDataRepository implements UserRepository {

  private UserDataStoreFactory userDataStoreFactory;

  @Override
  public User user(int userId) {
    UserDataStore userDataStore = userDataStoreFactory.create(userId);
    return userDataStore.get(userId);
  }
}
```

一个用户数据存储，正如其名字所暗示的那样，负责从类似存储的磁盘或后端(云)中加载用户。 

```java
interface UserDataStore {
  User getUser(int userId);
}

class DiskUserDataStore implements UserDataStore {
  private DiskCache<Int, User> userCache;

  @Override
  public User getUser(int userId){
    return userCache.get(userId);
  }
}

class CloudUserDataStore implements UserDataStore {
  @Override
  public User getUser(int userId){
    // Make an HTTP request and return User.
  }
}
```

当有人调用 UserRepository.getUser (userId) 时，UserDataStoreFactory 是决定使用 UserDataStore 的组件。 例如，工厂可以检查 DiskUserDataStore 是否有一个用户，如果用户对象已经过期，则工厂返回 CloudUserDataStore，否则 DiskUserDataStore。 该工厂还可以考虑智能手机是否有一个活跃的互联网连接。 我想你明白我的意思。 

## 结束

对存储库模式的解释是其他开发人员的代表，他们现在用类似的方式来解释存储库模式。 然而，它与 Martin Fowler 等人对存储库模式的"原始"定义没有多少共同之处。 存储库模式难道不是用来最小化重复查询逻辑吗？  Martin Fowler  没有提到从不同的用户数据库中加载用户。 规范类在哪里？ 

如果你比较两者，原始的存储库模式和后来的模式，你会发现实际上他们试图解决两种不同类型的问题。 虽然最初的一个尝试尽量减少重复查询逻辑(通过开放 / 封闭原则) ，后者试图隐藏你的应用程序正在对话的具体数据存储。 但是从我的角度来看，你不可能在同一时间以同样的模式解决这两个问题，就像 Krzychu 在他的博客文章中所尝试的那样。 你必须选择你真正想要解决的问题(尽管你可以把这两种模式链接在一起)。 

那么为什么这两种模式都被称为存储模式？ 这在历史上经常发生。 一个名字被用于不同的目的。 例如，Trygve Reenskaug 在20世纪70年代引入了模型视图控制器，但他的定义与我们现在所说的 MVC 没有任何共同之处，也就是说，控制器是 android 上的活动，iOS 上的 UIController，后端的 Controller 等等。 同样的情况也发生在存储模式这个术语上。 

不用担心，进化是好事。 我们应该记住我们从哪里来，为什么它已经进化到它现在的用途，以便更好地理解我们要去的地方。 原始存储库模式是为企业应用程序设计的，以减少查询逻辑的重复。 通常，你的 android 应用程序不是企业应用程序。 通常情况下，在你的 android 应用程序中，你应该遵循 Fernando Cejas 对存储库模式的定义，因为大多数情况下，你希望隐藏底层数据存储实现更加灵活，以适应未来需求的更改和相应的重构。 

但是，如果你仔细观察后来对存储库模式的解释，你会得出结论，基本上它只是说: 对接口进行编程，这样你可以稍后更改实现细节并遵循单一责任原则。 这就是所有简洁架构和存储库模式等等的内容。 

不要过分抽象。 不要过度工程化。 例如，如果你从头开始一个应用程序，是的，你应该定义一个接口，像 UserRepo 和你所有的业务逻辑或者领域层或者任何一个层面应该对这个接口编程。 然而，如果你只有一个后台来加载数据，那么你的 UserRepo 实现应该直接将这些 http 调用变成一个实现 UserRepo 接口的 UserRepo 类中。 不需要使用 UserDataStore，CloudUserDataStore 和 UserDataStoreFactory。 

你可能会问自己: 如果有一天我的需求改变了，我不得不使用 Realm 而不是 SQLite。 然后你提供 UserRepoRealm 实现 UserRepo 而不是 UserRepoSql 实现 UserRepo。 仍然不需要使用 UserDataStore 和 UserDataStoreFactory。 我的信息是: 构建你真正需要的最小的代码片段，不要用将来可能需要或不需要的需求来过度设计它。 说真的，我已经在这个行业工作了很多年了，我从来没有在后台工作过，更不用说一个安卓应用程序，它曾经改变了基础数据库。 不要建立像 SqlSpecification 这样的抽象层，因为这样的未来需求会改变你的想法。 如果有一天你不得不在你的云计算后端添加一些像 DiskCache 这样的东西来进行通信，那么重构你的代码以尊重单一责任原则是有意义的，并添加类如 UserDataStoreFactory，UserDataStore，DiskUserDataStore 和 CloudUserDataStore。 因为你所有的其他层都在用 UserRepo 接口编程，重构这个功能根本不是问题。 

