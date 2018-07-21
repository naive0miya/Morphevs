# Android 应用程序使用 AutoValue

> 原文 (Medium)：[AutoValue in Android Applications](https://medium.com/david-developer/autovalue-in-android-applications-fa424759c320)
>
> 作者：[David Tiago Conceição](https://medium.com/@davidtiagocon?source=post_header_lockup)

[TOC]

少即是多。这是在每个领域更经常重复的短语之一。对于软件开发领域，编写的代码越少意味着更多的稳定性，更简单和更灵活。在本文中，我将介绍一套用于软件开发的优秀工具的总体设想：[AutoValue](https://github.com/google/auto/tree/master/value) 及其扩展。

## AutoValue

在这个例子中，我们将为一个人创建一个包含数据的类。 类个人数据包含一些应用程序的基本数据。 它将数据存储在最终属性中，这些属性可以通过 getter 访问。 它还覆盖了 String ( )、 hashCode ( ) 和 equals (Object) 的方法。 这是 Android 项目(以及一般的 Java 项目)中非常常见的模式，也就是所谓的值类。 我们可以在下面的代码示例中看到，这个类已经有超过70行。 

> 本文包含几个代码示例，以 Gist 的形式呈现。为了减少行数，其中一些行格式非常简单。完整的示例项目可以在 [GitHub](https://github.com/davidtcdeveloper/AutoValueStudy) 上找到。

```java
public class PersonData {

    private final long id;
    private final String name;
    private final int status;
    private final String eMail;
    private final String profileUrl;
    private final String pictureImageUrl;

    public PersonData(long id, String name, int status, String eMail, String profileUrl, String pictureImageUrl) {
        this.id = id;
        this.name = name;
        this.status = status;
        this.eMail = eMail;
        this.profileUrl = profileUrl;
        this.pictureImageUrl = pictureImageUrl;
    }
    public long getId() { return id; }

    public String getName() { return name; }

    public int getStatus() { return status; }

    public String geteMail() { return eMail; }

    public String getProfileUrl() { return profileUrl; }

    public String getPictureImageUrl() { return pictureImageUrl;}

    @Override
    public String toString() {
        return "PersonData{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", status=" + status +
                ", eMail='" + eMail + '\'' +
                ", profileUrl='" + profileUrl + '\'' +
                ", pictureImageUrl='" + pictureImageUrl + '\'' +
                '}';
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;

        PersonData that = (PersonData) o;

        if (id != that.id) return false;
        if (status != that.status) return false;
        if (name != null ? !name.equals(that.name) : that.name != null) return false;
        if (eMail != null ? !eMail.equals(that.eMail) : that.eMail != null) return false;
        if (profileUrl != null ? !profileUrl.equals(that.profileUrl) : that.profileUrl != null)
            return false;
        return pictureImageUrl != null ? pictureImageUrl.equals(that.pictureImageUrl) : that.pictureImageUrl == null;
    }

    @Override
    public int hashCode() {
        int result = (int) (id ^ (id >>> 32));
        result = 31  result + (name != null ? name.hashCode() : 0);
        result = 31  result + status;
        result = 31  result + (eMail != null ? eMail.hashCode() : 0);
        result = 31  result + (profileUrl != null ? profileUrl.hashCode() : 0);
        result = 31  result + (pictureImageUrl != null ? pictureImageUrl.hashCode() : 0);
        return result;
    }
}
```

很多代码需要维护，分析和审查。

现在，为了更进一步的复杂性，让我们假设这个类将会主持一个新的属性。 要做到这一点，我们必须采取几个步骤: 声明属性，在构造函数中包含一个参数，创建 getter 方法，将属性添加到 equals 方法中，将属性添加到 hashCode ( ) 方法中，并将属性添加到 toString 方法中。 如果开发者在其中一个步骤中犯了错误，那么代码将在某个重要时刻失败。 所有这些工作只是为了制作一个简单的类。 如果这个类被用于某种序列化，如 JSON 或者实现一个类似 Parcelable 的接口，那么就需要编写、维护和审查更多的代码。 

AutoValue 及其扩展是为了减少编写简单类的代码的数量。然后开发人员只需声明这个类，而所有繁重的工作都是由代码生成器完成的。为了实现这一点，我们必须设置一些工具来构建脚本，并在我们的类中应用一些注解。 

要启动，请确保你使用的是 Android Build Tools 2.2 或更高版本。否则，你将不得不启用 [android-apt](https://bitbucket.org/hvisser/android-apt) 插件，并从 apt 替换指令 annotationProcessor。

> 此帖子的以前版本声明，android-apt 插件是强制性的。自 Android Build Tools 2.2 发布以来，[情况就不再如此了](http://www.littlerobots.nl/blog/Whats-next-for-android-apt/)。

在下面的步骤中，我们将在模块中包含 AutoValue 依赖项。 annotationProcessor 类型的依赖性表明当代码生成插件运行时将应用 AutoValue。提供的数据表明 AutoValue 不需要包含在项目中，因为它仅在构建时使用。

```groovy
provided 'com.google.auto.value:auto-value:1.2'
annotationProcessor 'com.google.auto.value:auto-value:1.2'
```

现在我们可以将我们的类重构为更简单的形式。添加注解 @AutoValue，表明它必须由工具分析。我们也必须使类抽象，因为它的实现将会被生成而不是手动创建。删除了属性，定义了相应的抽象方法。 每个方法定义都指示相关属性的格式、为其定义的修饰符和注解。 我们的类现在有十多行。 

```java
import com.google.auto.value.AutoValue;

@AutoValue
public abstract class PersonData {
    abstract long id();
    abstract String name();
    abstract int status();
    abstract String eMail();
    abstract String profileUrl();
    abstract String pictureImageUrl();
}
```

由于我们现在有一个抽象类，我们必须定义一种方法来创建它的新实例。按照惯例，AutoValue 类通过发布静态方法 create 来允许新实例。这个方法实例化一个新的对象并以透明的方式返回它。

```java
public static PersonData create(long id, String name, int status, String eMail,String profileUrl, String pictureImageUrl) {
        return new AutoValue_PersonData(id, name, status, eMail, profileUrl, pictureImageUrl);
    }
```

请注意，我们正在调用 AutoValue_PersonData 类的构造函数。这是 AutoValue 生成的类，具体实现了所有我们早期定义的: 属性和方法。 

当我们第一次将这个调用包含在 create 方法中时，代码会显示一个错误，指出 AutoValue_PersonData 还不存在。请记住，AutoValue 类是在构建时生成的，因此每次我们在定义中更改某些内容时都必须重新构建项目。我们在下面的示例中有一个生成的类的片段。

```java
final class AutoValue_PersonData extends PersonData {

  private final long id;
  private final String name;
  private final int status;
  private final String eMail;
  private final String profileUrl;
  private final String pictureImageUrl;

  AutoValue_PersonData(
      long id,
      String name,
      int status,
      String eMail,
      String profileUrl,
      String pictureImageUrl) {
    this.id = id;
    if (name == null) {
      throw new NullPointerException("Null name");
    }
    this.name = name;
    this.status = status;
    if (eMail == null) {
      throw new NullPointerException("Null eMail");
    }
```

我们可以注意到，正如我们预期的那样，具体类继承了我们的抽象概念。 它包括具有所有字段的构造函数和它们的初始化。 具体类也具有与代码的抽象、简化的组织和管理相同的包。 

在某些情况下，使用具有多个参数的构造函数可能并不有趣。 在我们的示例中，我们有一些对于调用者来说非常容易出错的字符串参数。 考虑到这个问题，AutoValue 允许定义 Builders。 这意味着我们可以生成允许使用 [Builder 模式](https://en.wikipedia.org/wiki/Builder_pattern)初始化对象的类。 这样，我们可以通过调用具有属性名称的方法序列来创建新实例。 这样可以防止误导参数的顺序和其他问题。 要创建一个 Builder，我们必须定义一个实习抽象类，如下面的示例。 

```java
 @AutoValue.Builder
    abstract static class Builder {
        abstract Builder id(long value);
        abstract Builder name(String value);
        abstract Builder status(int value);
        abstract Builder eMail(String value);
        abstract Builder profileUrl(String value);
        abstract Builder pictureImageUrl(String value);
        abstract PersonData build();
    }
```

所有的属性都有其等价的方法。创建方法也被新的静态方法取代，如样本。

```java
 static Builder builder() {
        return new AutoValue_PersonData.Builder();
    }
```

现在，我们类的客户端能够以非常流畅的方式轻松地创建新实例。 

```java
PersonData personData = PersonData.builder()
  .id(1)
  .name("David Tiago Conceição")
  .eMail("david@david.com")
  .status(0)
  .profileUrl("twitter.com/davidtiagocon")
  .pictureImageUrl("https://pbs.twimg.com/profile_images/601894402198544384/FJupV0uC.jpg")
  .build();
```

再次查看生成的类，我们可以注意到它中有一些运行时检查。默认情况下，AutoValue 类不允许使用空字段，并引发运行时异常，以警告开发人员可能有错误。

```java
if (pictureImageUrl == null) {
  throw new NullPointerException("Null pictureImageUrl");
}
```

如果你的类必须接受空值，则必须使用 @Nullable 注解该属性。这样，一致性将不会生成，空值将被接受。

```java
 @Nullable
    abstract String pictureImageUrl();
```

```java
abstract Builder pictureImageUrl(@Nullable String value);
```

我们的类现在比第一个版本更简单。我们可以很容易地声明新属性或调整现有属性。我们也有建设者的有用功能。但是，一旦生成的类仍然非常简单，这个类在实际项目中的效用仍然是有限的。

## 扩展

最近版本的 AutoValue 添加了一个非常有用的功能，名为 AutoValue Extensions。使用这些扩展，可以将自定义代码生成器包含到我们的项目中。新的中介类可以在 absctract 和具体的类之间生成，从而提供更多的功能和用例。

在本篇文章中，我们将检查两个 Android 项目的插件：[AutoValue-Parcel](https://github.com/rharter/auto-value-parcel) 和 [AutoValue-GSON](https://github.com/rharter/auto-value-gson)。还有更多的扩展，但我们现在会检查这两个。

首先，让我们设置扩展，以允许生成  Parcelable 类。为此，我们的模块的 build.gradle 添加了一个新的依赖关系。

```groovy
annotationProcessor 'com.ryanharter.auto.value:auto-value-parcel:0.2.2'
```

现在我们只需要让我们的 PersonData 类实现 Parcelable。这个接口的所有方法都是由扩展生成的，开发人员不需要再编写任何代码。

```java
@AutoValue
public abstract class PersonData implements Parcelable {
```

我们的类 AutoValue_PersonData 现在包含 Parcelable 的所有实现。这很简单，不是吗？

```java
 public static final Parcelable.Creator<AutoValue_PersonData> CREATOR = new Parcelable.Creator<AutoValue_PersonData>() {
    @Override
    public AutoValue_PersonData createFromParcel(Parcel in) {
      return new AutoValue_PersonData(
          in.readLong(),
          in.readString(),
          in.readInt(),
          in.readString(),
          in.readString(),
          in.readInt() == 0 ? in.readString() : null
      );
    }
```

GSON 集成扩展需要更多的工作，但值得付出努力。再次，我们开始添加依赖到我们的 build.gradle。

```groovy
annotationProcessor 'com.ryanharter.auto.value:auto-value-gson:0.3.1'

compile 'com.google.code.gson:gson:2.6.2'
```

之后，我们必须定义必须与 GSON 集成的类。为此，类中定义了一个 typeAdapter 方法：

```java
public static TypeAdapter<PersonData> typeAdapter(Gson gson) {
  return new AutoValue_PersonData.GsonTypeAdapter(gson);
}
```

同样，会显示错误，因为我们生成的类已过时。构建项目能使错误消失。这个过程将在我们的项目中生成一个新类：AutoValueGsonTypeAdapterFactory 在我们生成的类和 GSON 之间建立连接。出于这个原因，我们必须在我们的项目中创建的 GSON 实例中注册这个类。

```java
//...
import com.ryanharter.auto.value.gson.AutoValueGsonTypeAdapterFactory;
//...

Gson gson = new GsonBuilder()
  .registerTypeAdapterFactory(new AutoValueGsonTypeAdapterFactory())
  .create();
```

这种集成对于应用程序和后端之间的通信非常有用，也可以与 Retrofit 一起使用。使用扩展可以更容易地定义通信中使用的类，因为开发人员所有的工作都是定义抽象类并添加适配器工厂方法。 

## 还有更多

目前提供的两个扩展对于 Android 项目非常有用。但是对于不同的情况还有更多的扩展，例如 [Moshi 扩展](https://github.com/rharter/auto-value-moshi)， [Cursor 扩展](https://github.com/gabrielittner/auto-value-cursor)，[With 扩展](https://github.com/gabrielittner/auto-value-with)和 [Redact 扩展](https://github.com/square/auto-value-redacted)。

在 Android 项目中使用 AutoValue 和 Extensions 可以使应用程序开发的某些步骤更快，并使开发人员能够关注更重要的步骤。现在简单和重复性的任务已经实现了自动化，创造了一个更高效的环境。



