# Dagger 2 的依赖注入- 使用 Multibinding 和 AutoFactory 的例子

> 原文 (mirekstanek.online) ：[Inject everything — ViewHolder and Dagger 2 (with Multibinding and AutoFactory example](https://mirekstanek.online/inject-everything%e2%80%8a-%e2%80%8aviewholder-and-dagger-2-with-multibinding-and-autofactory-example/))
>
> 作者 ：[Mirek Stanek](https://twitter.com/froger_mcs)

[TOC]

这篇文章的第一个版本最初写在我的开发博客上：http://frogermcs.github.io/

这篇文章是在 Android 中显示依赖注入与 Dagger 2 框架的系列文章的一部分。今天我们来看看 Multibinding 和 Autofactory，我们将尝试使用 Dagger 2 实现 ViewHolder 模式。



Dagger 2 实现的依赖注入模式的主要目的是将客户端的依赖关系的创建与客户端的行为分开。 实际上，这意味着所有的 new 运算符 newInstance（）和其他的调用都不应该在 Dagger 模块以外的地方被调用。

## 完全 dagger 的代价
这篇文章的目的是展示我们可以做什么，而不是我们应该做什么。 这就是为什么知道我们将创造从行为中分离出来的原因。 如果你想在你的项目中使用 Dagger 2 几乎所有的东西，你会很快看到大量的 64k 方法计数限制被生成的代码用于注入。

## 注入所有的例子
Dagger 2 中最有名的代码示例可能显示了如何将简单对象（例如 Presenter）注入到 Activity 中，并且看起来与此类似：

```java
public class MyActivity extends BaseActivity {

    @Inject MyActivityPresenter presenter;

    @Override
    protected void injectDependencies() {
        getAppComponent().plus(new MyActivityModule(this)).inject(this);
    }
    
    //...
}
```
当我们开始扩展这个代码（假设我们在 Activity 中显示项目列表）时，我们有时会使用快捷方式。这意味着我们并不总是 @Inject 新的对象（如在这种情况下的适配器）：
```java
public class MyActivity extends BaseActivity {

    @Inject MyActivityPresenter presenter;
    MyListAdapter adapter;

    @Override
    protected void injectDependencies() {
        getAppComponent().plus(new MyActivityModule(this)).inject(this);
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        //...
        adapter = new MyListAdapter();
        recyclerView.setAdapter(adapter);
    }
}
```
采用这种方法，我们采取行动反对 DI 和 DI 带来的利润。 调用 Activity 类中的 Adapter 构造函数意味着每当我们需要在其实现中更改某些东西（新的构造函数参数，可能是显示网格而不是列表的全新实现）时，我们必须更新 Activity 代码。

有时采取这样的捷径是有意的（我们知道这个代码将来不会被扩展）。 但是今天我们将尝试使所有与适配器相关的代码成为依赖注入方法的一部分。 作为一个例子，我们将扩展我们的 [GithubClient](https://github.com/frogermcs/GithubClient) 应用程序 - 它的屏幕显示给定用户名的存储库列表。
![|center](https://cdn-images-1.medium.com/max/800/0*B4ev5z70YVEmrAg9.png)
为了使它更复杂，更接近真实的应用程序，我们的适配器将有三种不同的视图类型：普通，扩展和特色。

## 开始
在这里你可以看到我们例子的起始代码库。对我来说，这是实现适配器的第一次迭代在大多数情况下的样子。

RepositoriesListActivity
```java
public class RepositoriesListActivity extends BaseActivity {
    @Bind(R.id.rvRepositories)
    RecyclerView rvRepositories;

    @Inject
    RepositoriesListActivityPresenter presenter;

    private RepositoriesListAdapter repositoriesListAdapter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_repositories_list);
        ButterKnife.bind(this);
        setupRepositoriesListView();
    }

    private void setupRepositoriesListView() {
        repositoriesListAdapter = new RepositoriesListAdapter(this);
        rvRepositories.setAdapter(repositoriesListAdapter);
        rvRepositories.setLayoutManager(new LinearLayoutManager(this));
    }

    @Override
    protected void setupActivityComponent() {
        GithubClientApplication.get(this).getUserComponent()
                .plus(new RepositoriesListActivityModule(this))
                .inject(this);
    }
}
```
RepositoriesListAdapter
```java
public class RepositoriesListAdapter extends RecyclerView.Adapter {

    private RepositoriesListActivity repositoriesListActivity;

    private final List<Repository> repositories = new ArrayList<>();

    public RepositoriesListAdapter(RepositoriesListActivity repositoriesListActivity) {
        this.repositoriesListActivity = repositoriesListActivity;
    }

    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        final RecyclerView.ViewHolder viewHolder;
        if (viewType == Repository.TYPE_NORMAL) {
            View view = LayoutInflater.from(parent.getContext()).inflate(R.layout.list_item_normal, parent, false);
            viewHolder = new RepositoryViewHolderNormal(view);
        } else if (viewType == Repository.TYPE_BIG) {
            View view = LayoutInflater.from(parent.getContext()).inflate(R.layout.list_item_big, parent, false);
            viewHolder = new RepositoryViewHolderBig(view);
        } else if (viewType == Repository.TYPE_FEATURED) {
            View view = LayoutInflater.from(parent.getContext()).inflate(R.layout.list_item_featured, parent, false);
            viewHolder = new RepositoryViewHolderFeatured(view);
        } else {
            return null;
        }

        viewHolder.itemView.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                onRepositoryItemClicked(viewHolder.getAdapterPosition());
            }
        });

        return viewHolder;
    }

    private void onRepositoryItemClicked(int adapterPosition) {
        repositoriesListActivity.onRepositoryClick(repositories.get(adapterPosition));
    }

    @Override
    public void onBindViewHolder(RecyclerView.ViewHolder holder, int position) {
        ((RepositoryViewHolder) holder).bind(repositories.get(position));
    }

    @Override
    public int getItemCount() {
        return repositories.size();
    }

    @Override
    public int getItemViewType(int position) {
        Repository repository = repositories.get(position);
        if (repository.stargazers_count > 500) {
            if (repository.forks_count > 100) {
                return Repository.TYPE_FEATURED;
            }
            return Repository.TYPE_BIG;
        }
        return Repository.TYPE_NORMAL;
    }

    public void updateRepositoriesList(List<Repository> repositories) {
        this.repositories.clear();
        this.repositories.addAll(repositories);
        notifyDataSetChanged();
    }

    public static class RepositoryViewHolderNormal extends RepositoryViewHolder {

        @Bind(R.id.tvName)
        TextView tvName;

        public RepositoryViewHolderNormal(View view) {
            super(view);
            ButterKnife.bind(this, itemView);
        }

        @Override
        public void bind(Repository repository) {
            tvName.setText(repository.name);
        }
    }

    public static class RepositoryViewHolderBig extends RepositoryViewHolder {

        @Bind(R.id.tvName)
        TextView tvName;
        @Bind(R.id.tvStars)
        TextView tvStars;
        @Bind(R.id.tvForks)
        TextView tvForks;

        public RepositoryViewHolderBig(View view) {
            super(view);
            ButterKnife.bind(this, itemView);
        }

        @Override
        public void bind(Repository repository) {
            tvName.setText(repository.name);
            tvStars.setText("Stars: " + repository.stargazers_count);
            tvForks.setText("Forks: " + repository.forks_count);
        }
    }

    public static class RepositoryViewHolderFeatured extends RepositoryViewHolder {

        @Bind(R.id.tvName)
        TextView tvName;
        @Bind(R.id.tvStars)
        TextView tvStars;
        @Bind(R.id.tvForks)
        TextView tvForks;

        public RepositoryViewHolderFeatured(View view) {
            super(view);
            ButterKnife.bind(this, itemView);
        }

        @Override
        public void bind(Repository repository) {
            tvName.setText(repository.name);
            tvStars.setText("Stars: " + repository.stargazers_count);
            tvForks.setText("Forks: " + repository.forks_count);
        }
    }
}
```

## 适配器注入
在开始时，我们将注入我们的适配器对象，而不是在 Activity 类中调用它的构造函数：

RepositoriesListActivity
```java
public class RepositoriesListActivity extends BaseActivity {
    @Bind(R.id.rvRepositories)
    RecyclerView rvRepositories;

    @Inject
    RepositoriesListActivityPresenter presenter;
    @Inject
    RepositoriesListAdapter repositoriesListAdapter;

    //...

    private void setupRepositoriesListView() {
        rvRepositories.setAdapter(repositoriesListAdapter);
        rvRepositories.setLayoutManager(new LinearLayoutManager(this));
    }

    @Override
    protected void setupActivityComponent() {
        GithubClientApplication.get(this).getUserComponent()
                .plus(new RepositoriesListActivityModule(this))
                .inject(this);
    }
}
```
为了使这成为可能，我们必须在 Activity Module 中初始化 RepositoriesListAdapter 对象：
```java
@Module
public class RepositoriesListActivityModule {
    private RepositoriesListActivity repositoriesListActivity;
    
    //...
    
    @Provides
    @ActivityScope
    RepositoriesListAdapter provideRepositoriesListAdapter(RepositoriesListActivity repositoriesListActivity) {
        return new RepositoriesListAdapter(repositoriesListActivity);
    }
}
```
非常简单现在让我们对 RepositoriesListAdapter 类进行一些重构。 ViewHolders 的内部静态类应该移出适配器代码：
```java
public class RepositoriesListAdapter extends RecyclerView.Adapter {

    //...

    public static class RepositoryViewHolderNormal extends RepositoryViewHolder {

        @Bind(R.id.tvName)
        TextView tvName;

        public RepositoryViewHolderNormal(View view) {
            super(view);
            ButterKnife.bind(this, itemView);
        }

        @Override
        public void bind(Repository repository) {
            tvName.setText(repository.name);
        }
    }

    public static class RepositoryViewHolderBig extends RepositoryViewHolder {

        @Bind(R.id.tvName)
        TextView tvName;
        @Bind(R.id.tvStars)
        TextView tvStars;
        @Bind(R.id.tvForks)
        TextView tvForks;

        public RepositoryViewHolderBig(View view) {
            super(view);
            ButterKnife.bind(this, itemView);
        }

        @Override
        public void bind(Repository repository) {
            tvName.setText(repository.name);
            tvStars.setText("Stars: " + repository.stargazers_count);
            tvForks.setText("Forks: " + repository.forks_count);
        }
    }

    public static class RepositoryViewHolderFeatured extends RepositoryViewHolder {

        @Bind(R.id.tvName)
        TextView tvName;
        @Bind(R.id.tvStars)
        TextView tvStars;
        @Bind(R.id.tvForks)
        TextView tvForks;

        public RepositoryViewHolderFeatured(View view) {
            super(view);
            ButterKnife.bind(this, itemView);
        }

        @Override
        public void bind(Repository repository) {
            tvName.setText(repository.name);
            tvStars.setText("Stars: " + repository.stargazers_count);
            tvForks.setText("Forks: " + repository.forks_count);
        }
    }
}
```
要在 Android Studio 中快速执行此操作，只需将光标放在类名上，然后单击 F6（将内部移动到上一级）
![|center](https://cdn-images-1.medium.com/max/800/0*3gHzXvXxRBSoJPxH.png)

## 辅助注入，自动工厂
在我们的重构过程的下一步，我们想从 onCreateViewHolder（）方法移出构建过程：

```java
@Override
public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    final RecyclerView.ViewHolder viewHolder;
    if (viewType == Repository.TYPE_NORMAL) {
        View view = LayoutInflater.from(parent.getContext()).inflate(R.layout.list_item_normal, parent, false);
        viewHolder = new RepositoryViewHolderNormal(view);
    } else if (viewType == Repository.TYPE_BIG) {
        View view = LayoutInflater.from(parent.getContext()).inflate(R.layout.list_item_big, parent, false);
        viewHolder = new RepositoryViewHolderBig(view);
    } else if (viewType == Repository.TYPE_FEATURED) {
        View view = LayoutInflater.from(parent.getContext()).inflate(R.layout.list_item_featured, parent, false);
        viewHolder = new RepositoryViewHolderFeatured(view);
    } else {
        return null;
    }

    //...

    return viewHolder;
}
```
这不像前面的例子那么简单。 正如你可以看到我们的 ViewHolders 有强制性的参数是 View 对象（RecyclerView.ViewHolder 类只有一个构造函数：public ViewHolder（View itemView））。 这意味着每当我们想要创建新的对象时，我们都需要提供视图（这是在运行时创建的，所以我们无法事先在模块类中进行配置）。

这个问题被称为辅助注入，并在 [StackOverflow](http://stackoverflow.com/questions/16040125/using-dagger-for-dependency-injection-on-constructors) 和 [Dagger Discuss 组](https://groups.google.com/forum/#!topic/dagger-discuss/QgnvmZ-dH9c) 中进行了更广泛的描述。

最后的解决方案（直到在 Dagger 2 框架中没有这样做的官方方式）是注入 Factory 对象，它将这些依赖关系作为参数并创建预期的对象。 在提到的 StackOverflow 线程中，您可以看到如何手动执行此操作，但 Google 也提供了一个自动生成这些工厂的解决方案。

[AutoFactory](https://github.com/google/auto/tree/master/factory) 生成的工厂可以单独使用，也可以通过简单的批注与 JSR-330 兼容的依赖注入器一起使用。要在我们的项目中使用它，我们必须在 build.gradle 文件中添加新的依赖关系：
```groovy
compile 'com.google.auto.factory:auto-factory:1.0-beta3'
```
现在我们所要做的就是注解我们想要创建工厂的类：
```java
@AutoFactory(implementing = RepositoriesListViewHolderFactory.class)
public class RepositoryViewHolderNormal extends RepositoryViewHolder {

    @Bind(R.id.tvName)
    TextView tvName;

    public RepositoryViewHolderNormal(ViewGroup parent) {
        super(LayoutInflater.from(parent.getContext()).inflate(R.layout.list_item_normal, parent, false));
        ButterKnife.bind(this, itemView);
    }

    @Override
    public void bind(Repository repository) {
        tvName.setText(repository.name);
    }
}
```
@AutoFactory 注解的参数可以使我们的 Factory 类扩展给定的类或实现给定的接口。在我们的情况下将是：
```java
public interface RepositoriesListViewHolderFactory {
    RecyclerView.ViewHolder createViewHolder(ViewGroup parent);
}
```
此更新之后适配器的 onCreateViewHolder（）方法如下所示：
```java
@Override
public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    final RecyclerView.ViewHolder viewHolder;
    if (viewType == Repository.TYPE_NORMAL) {
        viewHolder = new RepositoryViewHolderNormalFactory().createViewHolder(parent);
    } else if (viewType == Repository.TYPE_BIG) {
        viewHolder = new RepositoryViewHolderBigFactory().createViewHolder(parent);
    } else if (viewType == Repository.TYPE_FEATURED) {
        viewHolder = new RepositoryViewHolderFeaturedFactory().createViewHolder(parent);
    } else {
        return null;
    }

    //...
    return viewHolder;
}
```
现在我们的代码有点干净，但是我们仍然在里面调用构造函数。

## Multibinding
我们重构的最后一步是在 Module 类中初始化我们的 Factory 对象，以免在 Adapter 类中构造函数调用。 我们可以简单地将它们注入为 RepositoriesListAdapter 构造函数参数。 但是这意味着每当我们决定添加/删除新类型的 ViewHolder 时，我们仍然需要手动更新 Adapter 代码。

相反，我们可以使用 [Multibinding](http://google.github.io/dagger/multibindings.html)。由于这个特性，Dagger 允许我们将几个对象绑定到一个集合中（即使对象绑定在不同的模块中）。

我们的 ViewHolder 工厂有一个共同点：interface RepositoriesListViewHolderFactory。这意味着我们可以注入 Integers （Repository 对象的类型被表示为一个 static final int）和 RepositoriesListViewHolderFactory 的 map：

```java
public class RepositoriesListAdapter extends RecyclerView.Adapter {

    private RepositoriesListActivity repositoriesListActivity;
    private Map<Integer, RepositoriesListViewHolderFactory> viewHolderFactories;

    public RepositoriesListAdapter(RepositoriesListActivity repositoriesListActivity,
                                   Map<Integer, RepositoriesListViewHolderFactory> viewHolderFactories) {
        this.repositoriesListActivity = repositoriesListActivity;
        this.viewHolderFactories = viewHolderFactories;
    }

    //...
}
```
在这里你可以看到我们的 map 是如何创建的：

```java
@Module
public class RepositoriesListActivityModule {
    
    //...
    
    @Provides
    @IntoMap
    @IntKey(Repository.TYPE_NORMAL)
    RepositoriesListViewHolderFactory provideViewHolderNormal() {
        return new RepositoryViewHolderNormalFactory();
    }

    @Provides
    @IntoMap
    @IntKey(Repository.TYPE_BIG)
    RepositoriesListViewHolderFactory provideViewHolderBig() {
        return new RepositoryViewHolderBigFactory();
    }

    @Provides
    @IntoMap
    @IntKey(Repository.TYPE_FEATURED)
    RepositoriesListViewHolderFactory provideViewHolderFeatured() {
        return new RepositoryViewHolderFeaturedFactory();
    }
}
```
@IntoMap 注解使这些对象成为 map的一部分，并将它们置于给出的  @IntKey 关键字之下（[这里](https://google.github.io/dagger/api/latest/dagger/multibindings/package-summary.html)您可以找到更多不同类型键的注释）：

最后，在 onCreateViewHolder（）方法的所有代码改进之后，尽可能简单：
```java
@Override
public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    final RecyclerView.ViewHolder viewHolder = viewHolderFactories.get(viewType).createViewHolder(parent);
    viewHolder.itemView.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            onRepositoryItemClicked(viewHolder.getAdapterPosition());
        }
    });
    return viewHolder;
}
```
没有构造函数，没有 if-else 语句。一切都通过适当的 Dagger 图配置完成。

这就是我们的适配器和它的 ViewHolders 现在是 Dagger 2 图的一部分。我们又向前迈进了一步，在我们的代码中注入了一切。使用 AutoFactory 和 Multibinding 将会简单得多。

## 完整的示例代码
在这里你可以找到重构后描述的类的示例代码：
- [RepositoriesListActivity](https://github.com/frogermcs/GithubClient/blob/4ee4a68bf2170295823020c6354976722b1a632a/app/src/main/java/frogermcs/io/githubclient/ui/activity/RepositoriesListActivity.java)
- [RepositoriesListActivityAdapter](https://github.com/frogermcs/GithubClient/blob/4ee4a68bf2170295823020c6354976722b1a632a/app/src/main/java/frogermcs/io/githubclient/ui/adapter/RepositoriesListAdapter.java)
- [RepositoriesListActivityModule](https://github.com/frogermcs/GithubClient/blob/4ee4a68bf2170295823020c6354976722b1a632a/app/src/main/java/frogermcs/io/githubclient/ui/activity/module/RepositoriesListActivityModule.java)

所述项目的完整示例代码 GithubClient 在 Github [存储库](https://github.com/frogermcs/GithubClient)中获取。

