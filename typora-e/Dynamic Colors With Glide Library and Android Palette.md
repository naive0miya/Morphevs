# 使用 Glide 库和 Android 调色板动态色彩

> 原文 (Medium) ：[Dynamic Colors With Glide Library and Android Palette](https://android.jlelse.eu/dynamic-colors-with-glide-library-and-android-palette-5be407049d97)
>
> 作者 ：[Vortana Say](https://android.jlelse.eu/@sayvortana.itc?source=post_header_lockup)

最近，我需要根据这些特定屏幕中图像选择主色，为每个屏幕以不同的颜色设置 Android 视图。 这可能听起来有点混乱，所以我做了一个示例项目来向你展示我的意思，以及我是如何做到的。

在示例应用程序中，我们在第一个屏幕上列出了不同的电影海报。 从那里，我们可以选择任何电影海报，第二个屏幕将显示该电影海报以及电影的一些信息。 同时，详细信息屏幕（第二个屏幕）上的 Android 视图将呈现该电影海报的主色调。 这意味着不同海报的 Android 视图具有不同的颜色。

我们开始吧！ 简单来说，我们只关注三个 android 视图: 状态栏、操作栏和视图。 通过将 Glide 库和 Android 调色板结合起来，我们可以让我们的应用看起来更有趣。 下面是样本应用程序的截图。

![](https://ws3.sinaimg.cn/large/006tNc79gy1ftbvcok0rbj30zk0r01kx.jpg)

**Glide 是什么？**

> Glide 支持获取，解码和显示视频静态图像，图像和动画 GIF。 Glide 包含一个灵活的 API，允许开发人员插入几乎任何网络堆栈。 默认情况下，Glide 使用基于自定义 HttpUrlConnection 的堆栈，但也包括实用程序库插入到 Google 的 Volley 项目或 Square 的 OkHttp 库中。
>
> Glide 的主要焦点是尽可能平滑和快速地滚动任何类型的图像列表，但对于几乎任何需要获取，调整大小和显示远程图像的情况，Glide 都是有效的。

**调色板是什么？**

[Selecting Colors with the Palette API | developer.android.com](https://developer.android.com/training/material/palette-colors)

> 调色板对象可让你访问图像中的主要颜色以及覆盖文本的相应颜色。 使用调色板设计应用程序的样式，并根据给定的源图像动态更改应用程序的配色方案。

因此，我们使用 Glide 库异步地请求图片，但它也有助于保持这些图片缓存，这将帮助应用程序运行更加顺畅。 在这之后，我们使用 Android 调色板来获取所选图像的主色调。

#### Glide 实现

为了获得 Android 调色板图像，我们需要从 Glide 获取图片的位图。 Glide v3和 Glide v4在使用上有很大的不同，获取位图资源的方式也不同。

- **Glide v3**

如果你使用 Glide 版本3，你可以遵循下面的说明(Glide 文档) :

[bumptech/glide | github.com](https://github.com/bumptech/glide/wiki/Custom-targets#palette-example)

Glide v3 需要额外的类来获取位图资源。 上面文档中提到的所需的其他类是：

```Java
public class PaletteBitmap {
    public final Palette palette;
    public final Bitmap bitmap;

    public PaletteBitmap(@NonNull Bitmap bitmap, @NonNull Palette palette) {
        this.bitmap = bitmap;
        this.palette = palette;
    }
}
```



```Java
public class PaletteBitmapResource implements Resource<PaletteBitmap> {
    private final PaletteBitmap paletteBitmap;
    private final BitmapPool bitmapPool;

    public PaletteBitmapResource(@NonNull PaletteBitmap paletteBitmap, @NonNull BitmapPool bitmapPool) {
        this.paletteBitmap = paletteBitmap;
        this.bitmapPool = bitmapPool;
    }

    @Override 
    public PaletteBitmap get() {
        return paletteBitmap;
    }

    @Override 
    public int getSize() {
        return Util.getBitmapByteSize(paletteBitmap.bitmap);
    }

    @Override 
    public void recycle() {
        if (!bitmapPool.put(paletteBitmap.bitmap)) {
            paletteBitmap.bitmap.recycle();
        }
    }
}
```



```Java
public class PaletteBitmapTranscoder implements ResourceTranscoder<Bitmap, PaletteBitmap> {
    private final BitmapPool bitmapPool;

    public PaletteBitmapTranscoder(@NonNull Context context) {
        this.bitmapPool = Glide.get(context).getBitmapPool();
    }

    @Override 
    public Resource<PaletteBitmap> transcode(Resource<Bitmap> toTranscode) {
        Bitmap bitmap = toTranscode.get();
        Palette palette = new Palette.Builder(bitmap).generate();
        PaletteBitmap result = new PaletteBitmap(bitmap, palette);
        return new PaletteBitmapResource(result, bitmapPool);
    }

    @Override 
    public String getId() {
        return PaletteBitmapTranscoder.class.getName();
    }
}
```

实现情况:

```Java
Glide.with(mContext)
        .load(imageURL)
        .asBitmap()
        .transcode(new PaletteBitmapTranscoder(mContext), PaletteBitmap.class)
        .fitCenter()
        .placeholder(R.drawable.placeholder)
        .error(R.drawable.placeholder_error)
        .diskCacheStrategy(DiskCacheStrategy.ALL)
        .into(new ImageViewTarget<PaletteBitmap>(holder.imageView) {
            @Override
            protected void setResource(PaletteBitmap resource) {
                super.view.setImageBitmap(resource.bitmap);
                Palette p = resource.palette;
                holder.mPaletteColor = p.getMutedColor(ContextCompat.getColor(mContext, R.color.default));           
            }
        });
```

重要的是要注意到还有其他方法调用:

- asBitmap ( )
- transcode (new PaletteBitmapTranscoder (mContext) , PaletteBitmap.class)
- into (imageViewTarget class)

可以看到，在 setResource ( ) 重写方法中，我们得到了 PaletteBitmap，它是将用于 Android 调色板的位图。

-  **Glide v4**

在这个版本的 Glide 中，我们不需要创建额外的类: PaletteBitmap，PaletteBitmapResource 和 PaletteBitmapTranscoder。

— 在 Gradle 中导入库：

```groovy
implementation 'com.github.bumptech.glide:glide:4.7.1'
annotationProcessor 'com.github.bumptech.glide:compiler:4.7.1'
```

最新版本: <https://github.com/bumptech/glide>

— 创建 AppGlideModule 类

```Java
import com.bumptech.glide.annotation.GlideModule;
import com.bumptech.glide.module.AppGlideModule;

@GlideModule
public final class MyAppGlideModule extends AppGlideModule {}
```

— 实现

```Java
GlideApp.with(context)
        .asBitmap()
        .load(imageURL)
        .diskCacheStrategy(DiskCacheStrategy.ALL)
        .placeholder(R.mipmap.ic_launcher)
        .listener(new RequestListener<Bitmap>() {
              @Override
              public boolean onLoadFailed(@Nullable GlideException e, Object model, Target<Bitmap> target, boolean isFirstResource) {
                    mParentActivity.startPostponedEnterTransition();
                    return false;
              }

              @Override
              public boolean onResourceReady(Bitmap resource, Object model, Target<Bitmap> target, DataSource dataSource, boolean isFirstResource) {
                    mParentActivity.startPostponedEnterTransition();
                    if (resource != null) {
                        Palette p = Palette.from(resource).generate();
                        // Use generated instance
                        holder.mColorPalette = p.getMutedColor(ContextCompat.getColor(mParentActivity, R.color.movieDetailTitleBg));
                    }
                    return false;
              }
        })
        .into(holder.mImageView);
```

在我们的细节活动屏幕中，我们可以得到主要的安卓调色板，并根据我们的需求使用它。

#### 风格化视图

- 风格化状态栏和操作栏

```Java

// Show the Up button in the action bar.
ActionBar actionBar = getSupportActionBar();
if (actionBar != null) {
    actionBar.setDisplayHomeAsUpEnabled(true);
    //set color action bar
    actionBar.setBackgroundDrawable(new ColorDrawable(ColorUtils.manipulateColor(mPaletteColor, 0.62f)));

    //set color status bar
    getWindow().setStatusBarColor(ColorUtils.manipulateColor(mPaletteColor, 0.32f));
}
```

在这里，我们可以使用 manipulateColor 作为一种方法，根据给定的颜色(我们的调色板颜色)获得颜色更深和更浅的颜色。

```Java
/**
 * https://stackoverflow.com/questions/33072365/how-to-darken-a-given-color-int
 * @param color color provided
 * @param factor factor to make color darker
 * @return int as darker color
 */
public static int manipulateColor(int color, float factor) {
    int a = Color.alpha(color);
    int r = Math.round(Color.red(color) * factor);
    int g = Math.round(Color.green(color) * factor);
    int b = Math.round(Color.blue(color) * factor);
    return Color.argb(a,
            Math.min(r, 255),
            Math.min(g, 255),
            Math.min(b, 255));
}
```

- 风格化视图背景

```Java
if ((mColorPalette != 0)) {
    mViewBackground.setBackgroundColor(mColorPalette);
} else {
    if (getActivity() != null) {
        mViewBackground.setBackgroundColor(ContextCompat.getColor(getActivity(), R.color.movieDetailTitleBg));
    }
}
```

#### 结果

![](https://cdn-images-1.medium.com/max/1600/1*pQ9TiJyo7Zexd3LlBFDJ4g.gif)

通过以下方式获取示例项目:

[vsay01/PopularMovie | github.com](vsay01/PopularMovie | github.com)





