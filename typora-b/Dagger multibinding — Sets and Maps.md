# Dagger 多重绑定 — Sets 和 Maps

> 原文 (Medium)：[Dagger multibinding — Sets and Maps](https://medium.com/@hamidgh/dagger-multibinding-sets-and-maps-713254b7f734)
>
> 作者：[Hamid Gharehdaghi](https://medium.com/@hamidgh?source=post_header_lockup)

[TOC]

已经有一段时间了，Dagger 作为依赖注入库是开发者工具箱中流行和常用的工具。熟悉这个工具的所有特性和用例可以帮助我们编写更清晰的代码。multibinding 我发现与一些设计模式相匹配以解决问题和具有更好的体系结构的特性。 

如果你没有在你的项目中使用 Dagger，我想现在是时候尽快开始使用它了。[在这里你可以找到一个好的起点](https://github.com/codepath/android_guides/wiki/Dependency-Injection-with-Dagger-2)。在这里我假设你已经知道如何使用 Dagger 进行正常的依赖注入。 

## Set multibinding

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frow088seqj30bu0bg3yi.jpg)

作为一个例子，我想实现(部分)一个简单的翻译应用程序，它可以翻译文本。 这个应用支持不同的翻译。 为了专注于这个话题，我会尽可能地跳过细节。 

我开始写一个翻译器接口: 

```java
public interface Translator {
    String getName();
    String translate(String text);
}
```

用于在 spinner 中显示翻译器的适配器: 

```java
public class TranslatorAdapter extends BaseAdapter {
    private List<Translator> translators = new ArrayList<>();

    public TranslatorAdapter(Collection<Translator> translators) {
        this.translators.addAll(translators);
    }

    @Override
    public Object getItem(int position) {
        return translators.get(position);
    }

    @Override
    public View getView(int position, ...) {
        Translator translator = translators.get(position);
        //TODO inflate view and return
    }
    ...
}
```

用于活动，片段，... ... : 

```java
public class TranslatorActivity extends AppCompatActivity {

    @Inject Set<Translator> translators;
    ...
    private void initTranslatorsSpinner() {
        TranslatorAdapter adapter = new TranslatorAdapter(
                                               translators);
        translatorsSpinner.setAdapter(adapter);
    }
```

请注意，我们不只注入一个翻译器，而是一组翻译器。

谷歌翻译：

```java
public class GoogleTranslator implements Translator {
    private static final String NAME = "Google Translator";

    @Override
    public String getName() {
        return NAME;
    }

    @Override
    public String translate(String text) {
        //TODO implement translation by Google services
    }
}
```

最后是 @Module 类：

```java
@Module
public class TranslatorsModule {

    @Provides(type = Provides.Type.SET)
    @Singleton
    public Translator provideGoogleTranslator(){
        return new GoogleTranslator();
    }
}
```

唯一的区别在于（与普通 provider 相比）将提供类型更改为 Provide.Type.SET，这使我们能够注入一组翻译器。

有了这种架构，增加新的翻译是如此简单，并且具有最小的可能的变化 （为了遵守 [OCP](http://www.oodesign.com/open-close-principle.html)）。我们只需为新翻译器实现翻译器接口并提供新翻译器的实例。让我们来实现一个新的翻译器：

```java
public class LongmanTranslator implements Translator {
    private static final String NAME = "Longman Translator";

    @Override
    public String getName() {
        return NAME;
    }

    @Override
    public String translate(String text) {
        //TODO implement translation by Longman dictionary
    }
}
```

并为 @Module 添加一个新的提供者：

```java
@Module
public class TranslatorsModule {
    ...
    @Provides(type = Provides.Type.SET)
    @Singleton
    public Translator provideLongmanTranslator(){
        return new LongmanTranslator();
    }
}
```

就是这样。 你不必改变其他部分(UI 和 UX)。 这就是 Dagger 给我们展示的漂亮的解耦功能。

## Map multibinding

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frow0c21zoj30bs090glw.jpg)

在这里，让我们假设我们有一个销售项目的应用程序，在结账过程中，用户可以选择支付方式来付款。 每个支付系统都有自己的过程和计算。 但是每个卖家都支持不同的支付系统，例如，一个卖家只能支持信用卡，另一个则可以支持 PayPal 和 cash on-delivery。 就像之前的例子一样，我会尝试跳过细节，集中注意力和概念。 

让我们假设当我们通过 API 调用获得一个卖方信息时，我们会得到一个 json 格式的支持支付系统。 通过拥有这个支付列表，我们可以显示支付方法列表在一个 spinner，并且用户将选择其中的一个做支付: 

```java
{
    //Seller information here
    ...
    "payment_methods": [
        "credit-card",
        "paypal",
        ...
    ]
}
```

为了处理支付，让我们为所有处理程序创建一个接口: 

```java
public interface PaymentHandler {
    PaymentResult handlePayment(ShoppingCart cart);
}
```

在结账过程中，我们会有这样的东西: 

```java
public class CheckoutProcess {

    @Inject Map<String, PaymentHandler> paymentHandlers;
    ...
    private void onCheckout() {
        ...
        doPayment(selectedPaymentMethod);
    }
    
    private void doPayment(String selectedPaymentMethod) {
        PaymentHandler paymentHandler =
                        paymentHandlers.get(selectedPaymentMethod);
        paymentHandler.handlePayment(shoppingCart);
    }
}
```

请注意，我们使用 Map 来映射从卖方信息 API 到 PaymentHandler 的支付方法名称。 让我们实现信用卡支付处理程序: 

```java
public CreditCardPaymentHandler implements PaymentHandler {
    @Override
    PaymentResult handlePayment(ShoppingCart cart){
        //TODO here is all the bussiness logic for handling the
        //credit card payment method
    }
}
```

@Module 类中的提供者：

```java
@Module
public class PaymentModule {

    @Provides(type = Provides.Type.MAP)
    @StringKey("credit-card")
    public PaymentHandler provideCreditCardPaymentHandler(){
        return new CreditCardPaymentHandler();
    }
}
```

我们应该使用 Provides.Type.MAP 作为提供者类型，因为 Dagger 想知道在编译时注入 Map 的关键是什么，我们应该使用 @stringkey 注解来指定这一点。 

如果我们要注入的 map 有不同类型的密钥，我们可以使用 @intkey, @longkey, @classkey。 对于更具体的 map 键类型(例如 enum，...) ，我们可以使用 @mapkey，并为此创建一个新的注解。 我现在不打算说更多的细节了。 

有了这个架构，如果将来我们决定添加一个新的支付系统，我们只需要添加一个新的处理程序并提供注入。 让我们添加一个 paypal 支付处理程序: 

```java
public PaypalPaymentHandler implements PaymentHandler {
@Override
    PaymentResult handlePayment(ShoppingCart cart){
        //TODO here is all the bussiness logic for handling the
        //PayPal payment method.
    }
}
```

并在 @module 类中提供: 

```java
@Module
public class PaymentModule {
    ...
    @Provides(type = Provides.Type.MAP)
    @StringKey("paypal")
    public PaymentHandler providePayPalPaymentHandler(){
        return new PaypalPaymentHandler();
    }
}
```

## 结束

我试图通过尽可能简化示例来解释概念并跳过实现细节。如需查找更多详情，请查看 [Dagger 文档](http://google.github.io/dagger/multibindings.html)。

