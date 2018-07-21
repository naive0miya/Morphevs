# RecyclerView 进阶技巧 — 第2部分 

> 原文 (Medium) ： [RecyclerView Pro Tips — Part2](https://proandroiddev.com/recyclerview-pro-tips-part-2-341e6f449a62) 
>
> 作者 ：[Arif Khan ](https://proandroiddev.com/@passiondroid?source=post_header_lockup)

在这篇文章中，我将讨论 RecyclerView的 ItemDecoration 和 ItemAnimator，并将尝试通过一个简单的应用程序来解释，这个应用程序也可以在 [Github](https://github.com/passiondroid/Marvels) 上找到。

1. **ItemDecoration** （顾名思义）应该用来装饰 RecyclerView 的视图项目。

ItemDecoration 可用于应用分隔线和其他效果，如填充项或项目视图之间的等距。 为了在项目视图之间添加一个简单的分隔符，有一个类 “DividerItemDecoration”，它支持 lib 版本25.1.0以上版本。 以下代码片段演示了它的实现：

```Java
mDividerItemDecoration = new DividerItemDecoration(recyclerView.getContext(),
             mLayoutManager.getOrientation());
     recyclerView.addItemDecoration(mDividerItemDecoration);
```

要应用自定义项目装饰，最好的方法是扩展 RecyclerView.ItemDecoration 类。 在示例应用程序中，我使用了 GridLayoutManager，并将 CharacterItemDecoration 应用于 recycleroview:

```Java
recyclerView.addItemDecoration(new CharacterItemDecoration(50));
```

在这里，CharacterItemDecoration 在其构造函数中使用50px 作为偏移量，并覆盖 getItemOffsets (...)。 getItemOffsets ( ) 内部 outRect 的每个字段都指定了项目视图应该被嵌入的像素数，类似于填充或边距。 由于我使用 GridLayoutManager，并希望在网格项目之间设置相等的空格，所以我将每个项目的偏移量设置为25px (即偏移量 / 2) ，并在奇数位置为每个项目左偏25px，同时保持顶部偏移相同的所有项目。

![](https://ws2.sinaimg.cn/large/006tNc79gy1ftbvo5vqe3j30ub0f3dla.jpg)

2. **ItemAnimator** 应该用于 RecyclerView 项目内，动画项目或视图。

![](https://ws3.sinaimg.cn/large/006tNc79gy1ftbvoj5tugg305k09w76x.gif)



让我们的应用程序 Instagramicious (不要被这个词吓到)。 事实上，它甚至不是一个词!) 通过扩展 "DefaultItemAnimator" 并重写一些方法。

```Java
public boolean canReuseUpdatedViewHolder(@NonNull RecyclerView.ViewHolder viewHolder) {
        return true;
}
```

当项目数据被更改时，canReuseUpdatedViewHolder (…) 决定是否使用相同的 ViewHolder 来进行动画处理。 如果它返回 false，这两者——旧的和更新的 viewhoders 被传递到 animateChange (…) 方法。

```Java
public ItemHolderInfo recordPreLayoutInformation(@NonNull RecyclerView.State state, @NonNull RecyclerView.ViewHolder viewHolder, int changeFlags, @NonNull List<Object> payloads) {
    if (changeFlags == FLAG_CHANGED) {
        for (Object payload : payloads) {
            if (payload instanceof String) {
                return new CharacterItemHolderInfo((String) payload);
            }
        }
    }
    return super.recordPreLayoutInformation(state, viewHolder, changeFlags, payloads);
}

public static class CharacterItemHolderInfo extends ItemHolderInfo {
        public String updateAction;

        public CharacterItemHolderInfo(String updateAction) {
            this.updateAction = updateAction;
        }
}
```

在布局阶段开始之前，RecyclerView 调用了 recordPreLayoutInformation (…) 方法。 在视图可能反弹、移动或移除之前，ItemAnimator 应该记录关于视图的必要信息。 从该方法返回的数据将传递给相应的 animate** 方法(在我们的例子中是 animateChange (...))。

```Java
@Override
public boolean animateChange(@NonNull RecyclerView.ViewHolder oldHolder,
                                 @NonNull RecyclerView.ViewHolder newHolder,
                                 @NonNull ItemHolderInfo preInfo,
                                 @NonNull ItemHolderInfo postInfo) {

  if (preInfo instanceof CharacterItemHolderInfo) {
    CharacterItemHolderInfo recipesItemHolderInfo = (CharacterItemHolderInfo) preInfo;
    CharacterRVAdapter.CharacterViewHolder holder = (CharacterRVAdapter.CharacterViewHolder) newHolder;
       if (CharacterRVAdapter.ACTION_LIKE_IMAGE_DOUBLE_CLICKED.equals(recipesItemHolderInfo.updateAction)) {
            animatePhotoLike(holder);
          }
        }
       return false;
    }

private void animatePhotoLike(final CharacterRVAdapter.CharacterViewHolder holder) {
     holder.likeIV.setVisibility(View.VISIBLE);
     holder.likeIV.setScaleY(0.0f);
     holder.likeIV.setScaleX(0.0f);
     
     AnimatorSet animatorSet = new AnimatorSet();
     ObjectAnimator scaleLikeIcon = ObjectAnimator.ofPropertyValuesHolder
              (holder.likeIV, PropertyValuesHolder.ofFloat("scaleX", 0.0f, 2.0f), 
              PropertyValuesHolder.ofFloat("scaleY", 0.0f, 2.0f), PropertyValuesHolder.ofFloat("alpha", 0.0f, 1.0f, 0.0f));
     scaleLikeIcon.setInterpolator(DECELERATE_INTERPOLATOR);
     scaleLikeIcon.setDuration(1000);

     ObjectAnimator scaleLikeBackground = ObjectAnimator.ofPropertyValuesHolder
              (holder.characterCV, PropertyValuesHolder.ofFloat("scaleX", 1.0f, 0.95f, 1.0f),
              PropertyValuesHolder.ofFloat("scaleY", 1.0f, 0.95f, 1.0f));
     scaleLikeBackground.setInterpolator(DECELERATE_INTERPOLATOR);
     scaleLikeBackground.setDuration(600);
     animatorSet.playTogether(scaleLikeIcon, scaleLikeBackground);
     animatorSet.start();
}
```

当一个适配器项目出现时，在接收 notifyItemChanged (int) 调用的布局之前和之后，回收视图调用了 animateChange (…) 方法。 当调用 notifyDataSetChanged 时，并且适配器具有稳定的 id，也可以调用此方法 ，这样 RecyclerView 仍然可以将视图重新绑定到相同的 ViewHolders。请注意，这个方法接收 (ViewHolder oldHolder，ViewHolder newHolder，ItemHolderInfo preInfo，ItemHolderInfo postInfo) 作为参数。 由于我们正在重复使用该视图，oldHolder 和 newHolder 都是相同的。

当用户在任何项目上双击时，都会调用下面的方法:

```Java
notifyItemChanged(position, ACTION_LIKE_IMAGE_DOUBLE_CLICKED);
```

这启动了 canReuseUpdatedViewHolder (...)、recordPreLayoutInformation (…) 的整个过程，并最终在 ItemAnimator 中启动了 animateChange (...) ，这反过来又会缩放和淡出心形图标，以及 RecyclerView 项目的 cardview。

