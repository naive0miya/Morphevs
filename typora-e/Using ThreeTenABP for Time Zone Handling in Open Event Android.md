# 在 Android 中使用 ThreeTenABP 进行时区处理

> 原文 ：[Using ThreeTenABP for Time Zone Handling in Open Event Android](https://blog.fossasia.org/using-threetenabp-for-time-zone-handling-in-open-event-android/)
>
> 作者 ：[fossasia](https://blog.fossasia.org/)

![](https://ws4.sinaimg.cn/large/006tNc79gy1ftbw5287xoj30mx0bqdg9.jpg)

[Open Event Android](https://github.com/fossasia/open-event-android) 项目帮助活动组织者为他们的活动 / 会议组织事件和生成应用程序(apk 格式) ，提供使用 [Open Event server](https://github.com/fossasia/open-event-orga-server) 生成的 API 端点或压缩。 对于任何事件应用程序来说，正确地处理时区是非常重要的。 在 Open Event Android 应用程序中，有一个改变时区设置的选项。 用户可以在应用中的事件时区和本地时区中查看事件和会话的日期和时间。**ThreeTenABP** 提供 [Java SE 8](https://docs.oracle.com/javase/8/docs/api/java/time/package-summary.html) 的日期时间类的后端端口到 Java SE 6和7。 在这篇博客中，我解释了如何使用 [ThreeTenABP](https://github.com/JakeWharton/ThreeTenABP) 来处理安卓系统中的时区。

### 增加依赖项

要在应用程序中使用 ThreeTenABP，你必须在应用程序模块的 build.gradle 文件中添加依赖项。

```kotlin
dependencies {
      compile    'com.jakewharton.threetenabp:threetenabp:1.0.5'
      testCompile   'org.threeten:threetenbp:1.3.6'
}
```

### 初始化 ThreeTenABP

现在在应用程序类的 onCreate 方法中初始化了 ThreeTenABP。

```java
AndroidThreeTen.init(this);
```

### 创建 getZoneId () 方法

首先创建 getZoneId ( ) 方法，该方法将根据用户的偏好返回 ZoneId。 此方法将用于格式化和解析日期这里 showLocal 是用户首选项。 如果 showLocal 为 true，则此函数将返回默认本地 ZoneId，否则返回事件的ZoneId。

这里 getEventTimeZone ( ) 方法返回事件的时区字符串。

Threetenabp 主要有两个类，表示日期和时间。

- ZonedDateTime : ‘2011-12-03T10:15:30+01:00[Europe/Paris]’
- LocalDateTime : ‘2011-12-03T10:15:30’

ZonedDateTime 在结尾包含时区信息。 LocalDateTime 不包含时区。

### 创建解析和格式的方法

现在创建 getDate ( ) 方法，这个方法将采用 isoDateString，并将返回 ZonedDateTime 对象。

```Java
public static ZonedDateTime getDate(@NonNull String isoDateString) {
        return ZonedDateTime.parse(isoDateString).withZoneSameInstant(getZoneId());;
}
```

首先采用两个参数的格式化日期方法是格式字符串，第二个是 isoDateString。 此方法将返回格式化字符串。

```Java
public static String formatDate(@NonNull String format, @NonNull String isoDateString) {
        return DateTimeFormatter.ofPattern(format).format(getDate(isoDateString));
}
```

### 使用方法

现在我们已经准备好格式化和解析 isoDateString 了。 让我们举个例子。 让 "2017-11-09 t23:08:56 + 08:00" 是 isoDateString。 我们可以使用 getDate ( ) 方法解析这个 isoDateString，该方法将返回 ZonedDateTime 对象。

解析:

```Java
String isoDateString = "2017-11-09T23:08:56+08:00";

DateConverter.setEventTimeZone("Asia/Singapore");
DateConverter.setShowLocalTime(false);
ZonedDateTime dateInEventTimeZone = DateConverter.getDate(isoDateString);

dateInEventTimeZone.toString();  //2017-11-09T23:08:56+08:00[Asia/Singapore]


TimeZone.setDefault(TimeZone.getDefault());
DateConverter.setShowLocalTime(true);
ZonedDateTime dateInLocalTimeZone = DateConverter.getDate(dateInLocalTimeZone);

dateInLocalTimeZone.toString();  //2017-11-09T20:38:56+05:30[Asia/Kolkata]
```

格式:

```java
String date = "2017-03-17T14:00:00+08:00";
String formattedString = formatDate("dd MM YYYY hh:mm:ss a", date));

formattedString // "17 03 2017 02:00:00 PM"
```

#### 结论

正如你所看到的，ThreeTenABP 使得时区处理非常容易。 它还支持默认格式化和方法。 要了解更多关于 ThreeTenABP 的信息，请参考下面的链接。

- Github 仓库:  [ThreeTenABP | github.com](https://github.com/JakeWharton/ThreeTenABP) 
- Jakob Jenkov 的日期/时间格式化教程:  [DateTimeFormatter | tutorials.jenkov.com](http://tutorials.jenkov.com/java-date-time/datetimeformatter.html) 
- Open Event Android PR: [Open Event Android | github.com](https://github.com/fossasia/open-event-android/pull/1879) 

