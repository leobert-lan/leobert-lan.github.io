# 三思系列：Glide 4.X全方位认知 -- 模块职责概览 | Glide（一）

最近在解决一些问题时，进行了一下检索，发现 `绝大多数文章` 是 `基于3.X`。Glide从进入4.X也有两三年了，在3.X的基础上，发生了很多变化。 所幸我对 `4.X`
的源码还比较熟，且Glide的设计也很精彩，索性写一写 `对4.X的剖析`。

当然，对于多数读者而言，因为有一定的基础知识，这些剖析文章并不是满地金砖了，可以泛读查漏。

> 三思系列是我最新的学习、总结形式，着重于:**问题分析**、**技术积累**、**视野拓展**，[了解更多](https://github.com/leobert-lan/Blog/blob/main/info/%E5%85%B3%E4%BA%8E%E4%B8%89%E6%80%9D%E7%B3%BB%E5%88%97.md)

## 本文主旨

**Glide是一个 `庞大` 的项目，这一篇旨在对Glide项目4.X版本进行 `全方位的认知`**

具体为：**了解Glide处理哪些问题，哪些模块与之相对应**

对早年间 Square公司开源的 `Picasso` 比较熟悉的读者而言，不难理解：`Picasso` 是一个 `小而雅` 的框架，聚焦于图片加载的以下4点主要问题：

* `异步过程管理与调度`
* `资源与显示容器多对多关系`
* `多级缓存`
* `图片转换支持ScaleType`

而后起之秀 `Glide` 已经是一个 `博大且细致` 的图片异步加载框架。

在Glide 1.X 时期，两者的关注点基本一致，随着官方支持Glide的发展，它的关注点也越来越多，包括但不限于：

* `资源的封装`
* `资源获取方式与过程`
* `缓存`
* `decode`、`encode` 的过程
* `Target的抽象`，*注：媒体资源获取的最终目标，例如存为文件、加载到ImageView中等*
* `图片转换`
* `资源与Target多对多关系`
* `生命周期感知与控制`

仔细思考这些点之后，我们不难发现，Glide已经着眼于解决这样一个问题：

> 构建一个系统，对指定的标识媒体资源的实体，*其封装形式可以自行扩展*，执行资源的获取，并应用内存、磁盘进行缓存，封装图片的解码过程，按指定要求进行
> 裁切、缩放等图片转换，最终将结果运用到指定的目标。并对整个过程按照Android平台特性进行生命周期管理。

而这个过程中的绝大多数 `节点` 或者称之为 `切面`，均可进行 `外部扩展`

这一点我们可以从项目介绍中得到印证：*看重点单词即可*

> Glide is a fast and efficient open source `media management` and `image loading framework` for Android that wraps `media
> decoding`, `memory and disk caching`, and `resource pooling` into a simple and easy to use interface.
>
> Glide supports `fetching`, `decoding`, and `displaying` video stills, images, and animated GIFs. Glide includes a flexible API
> that allows developers to plug in to almost any network stack. By default Glide uses a custom `HttpUrlConnection` based
> stack, but also includes utility libraries plug in to Google's Volley project or Square's OkHttp library instead.
>
> Glide's `primary focus` is on `making scrolling any kind of a list of images` as `smooth and fast` as possible, but Glide is
> also effective for `almost any case` where you need to `fetch`, `resize`, and `display` a remote image.

## 模块职责划分，从Glide的构建过程窥一斑

不难理解：一个庞大的系统需要在符合 `SRP` 的基础上进行职责划分，这样一来，类就会变得很多，构建过程会变得复杂，利用Builder模式可以很好的解决这一问题。

```java
class GlideBuilder {
    public Glide build(@NonNull Context context) {
        if (sourceExecutor == null) {
            sourceExecutor = GlideExecutor.newSourceExecutor();
        }

        if (diskCacheExecutor == null) {
            diskCacheExecutor = GlideExecutor.newDiskCacheExecutor();
        }

        if (animationExecutor == null) {
            animationExecutor = GlideExecutor.newAnimationExecutor();
        }

        if (memorySizeCalculator == null) {
            memorySizeCalculator = new MemorySizeCalculator.Builder(context).build();
        }

        if (connectivityMonitorFactory == null) {
            connectivityMonitorFactory = new DefaultConnectivityMonitorFactory();
        }

        if (bitmapPool == null) {
            int size = memorySizeCalculator.getBitmapPoolSize();
            if (size > 0) {
                bitmapPool = new LruBitmapPool(size);
            } else {
                bitmapPool = new BitmapPoolAdapter();
            }
        }

        if (arrayPool == null) {
            arrayPool = new LruArrayPool(memorySizeCalculator.getArrayPoolSizeInBytes());
        }

        if (memoryCache == null) {
            memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
        }

        if (diskCacheFactory == null) {
            diskCacheFactory = new InternalCacheDiskCacheFactory(context);
        }

        if (engine == null) {
            engine = new Engine(
                    memoryCache,
                    diskCacheFactory,
                    diskCacheExecutor,
                    sourceExecutor,
                    GlideExecutor.newUnlimitedSourceExecutor(),
                    GlideExecutor.newAnimationExecutor(),
                    isActiveResourceRetentionAllowed);
        }

        RequestManagerRetriever requestManagerRetriever =
                new RequestManagerRetriever(requestManagerFactory);

        return new Glide(
                context,
                engine,
                memoryCache,
                bitmapPool,
                arrayPool,
                requestManagerRetriever,
                connectivityMonitorFactory,
                logLevel,
                defaultRequestOptions.lock(),
                defaultTransitionOptions);
    }
}
```

从这里我们可以收集到一下有效信息

* 缓存与池
    * MemorySizeCalculator
    * LruBitmapPool
    * BitmapPoolAdapter
    * LruArrayPool
    * LruResourceCache
    * InternalCacheDiskCacheFactory
* 资源获取
    * RequestManagerRetriever
    * DefaultConnectivityMonitorFactory
* 核心流程管理
    * Engine
* 线程池
    * GlideExecutor

再看到构造函数：*代码挺长，略去了部分内容*

```java
class Glide {
    Glide(
            @NonNull Context context,
            @NonNull Engine engine,
            @NonNull MemoryCache memoryCache,
            @NonNull BitmapPool bitmapPool,
            @NonNull ArrayPool arrayPool,
            @NonNull RequestManagerRetriever requestManagerRetriever,
            @NonNull ConnectivityMonitorFactory connectivityMonitorFactory,
            int logLevel,
            @NonNull RequestOptions defaultRequestOptions,
            @NonNull Map<Class<?>, TransitionOptions<?, ?>> defaultTransitionOptions) {

        this.engine = engine;
        this.bitmapPool = bitmapPool;
        this.arrayPool = arrayPool;
        this.memoryCache = memoryCache;
        this.requestManagerRetriever = requestManagerRetriever;
        this.connectivityMonitorFactory = connectivityMonitorFactory;

        DecodeFormat decodeFormat = defaultRequestOptions.getOptions().get(Downsampler.DECODE_FORMAT);
        bitmapPreFiller = new BitmapPreFiller(memoryCache, bitmapPool, decodeFormat);

        final Resources resources = context.getResources();

        registry = new Registry();
        registry.register(new DefaultImageHeaderParser());

        Downsampler downsampler = new Downsampler(registry.getImageHeaderParsers(),
                resources.getDisplayMetrics(), bitmapPool, arrayPool);
        ByteBufferGifDecoder byteBufferGifDecoder =
                new ByteBufferGifDecoder(context, registry.getImageHeaderParsers(), bitmapPool, arrayPool);
        ResourceDecoder<ParcelFileDescriptor, Bitmap> parcelFileDescriptorVideoDecoder =
                VideoDecoder.parcel(bitmapPool);
        ByteBufferBitmapDecoder byteBufferBitmapDecoder = new ByteBufferBitmapDecoder(downsampler);
        StreamBitmapDecoder streamBitmapDecoder = new StreamBitmapDecoder(downsampler, arrayPool);
        ResourceDrawableDecoder resourceDrawableDecoder =
                new ResourceDrawableDecoder(context);
        ResourceLoader.StreamFactory resourceLoaderStreamFactory =
                new ResourceLoader.StreamFactory(resources);
        ResourceLoader.UriFactory resourceLoaderUriFactory =
                new ResourceLoader.UriFactory(resources);
        ResourceLoader.FileDescriptorFactory resourceLoaderFileDescriptorFactory =
                new ResourceLoader.FileDescriptorFactory(resources);
        ResourceLoader.AssetFileDescriptorFactory resourceLoaderAssetFileDescriptorFactory =
                new ResourceLoader.AssetFileDescriptorFactory(resources);
        BitmapEncoder bitmapEncoder = new BitmapEncoder(arrayPool);

        BitmapBytesTranscoder bitmapBytesTranscoder = new BitmapBytesTranscoder();
        GifDrawableBytesTranscoder gifDrawableBytesTranscoder = new GifDrawableBytesTranscoder();

        ContentResolver contentResolver = context.getContentResolver();

        registry
                .append(ByteBuffer.class, new ByteBufferEncoder())
                /*略去一段内容，太长了*/
                .register(GifDrawable.class, byte[].class, gifDrawableBytesTranscoder);

        ImageViewTargetFactory imageViewTargetFactory = new ImageViewTargetFactory();
        glideContext =
                new GlideContext(
                        context,
                        arrayPool,
                        registry,
                        imageViewTargetFactory,
                        defaultRequestOptions,
                        defaultTransitionOptions,
                        engine,
                        logLevel);
    }
}
```

这一段可以分为即可部分：

* builder 的 field 对应取值赋值
* 构建非开放的 `BitmapPreFiller` 实例
* 构建 `Registry` 实例 `registry`
* 构建 `DefaultImageHeaderParser` 、 `ResourceLoader` 、`Decoder` 、 `ResourceTranscoder` 并将之注册到 `registry`
* 构建 `GlideContext` 实例 `glideContext`

按照经验推断，Registry 是一个策略注册表，服务于 `策略模式`。

*不难理解：Glide的工作环境非常复杂，例如获取资源，其资源存在位置可能在网络中的某个位置、 可能在应用的Assets中，也可能在其他位置，对于某一个目标，根据场景不同，存在不同的实现策略*

继续按照经验推断，注册会分为不同的策略组，对应不同的目标,这一点我们稍后再看。

再按照我们常见的使用习惯，追踪一下初始化过程：*泛读即可*

```java
class Glide {
    public static Glide get(@NonNull Context context) {
        if (glide == null) {
            synchronized (Glide.class) {
                if (glide == null) {
                    checkAndInitializeGlide(context);
                }
            }
        }

        return glide;
    }

    private static void checkAndInitializeGlide(@NonNull Context context) {
        //...
        initializeGlide(context);
        //...
    }

    //重载略去
    public static RequestManager with(@NonNull Activity activity) {
        return getRetriever(activity).get(activity);
    }

    @NonNull
    private static RequestManagerRetriever getRetriever(@Nullable Context context) {
        // Context could be null for other reasons (ie the user passes in null), but in practice it will
        // only occur due to errors with the Fragment lifecycle.
        Preconditions.checkNotNull(
                context,
                "You cannot start a load on a not yet attached View or a Fragment where getActivity() "
                        + "returns null (which usually occurs when getActivity() is called before the Fragment "
                        + "is attached or after the Fragment is destroyed).");
        return Glide.get(context).getRequestManagerRetriever();
    }
}
```

按照使用习惯，一般从 `Glide.with(...)` 开始得到 `RequestManager`，当然，也可以以 `Glide.get(context).getRequestManagerRetriever()`
方式获取。如果Glide没有初始化，则会调用 `initializeGlide` 进行初始化。

跟进一下代码，*泛读即可*

```java
class Glide {
    private static void initializeGlide(@NonNull Context context) {
        initializeGlide(context, new GlideBuilder());
    }

    @SuppressWarnings("deprecation")
    private static void initializeGlide(@NonNull Context context, @NonNull GlideBuilder builder) {
        Context applicationContext = context.getApplicationContext();
        GeneratedAppGlideModule annotationGeneratedModule = getAnnotationGeneratedGlideModules();
        List<com.bumptech.glide.module.GlideModule> manifestModules = Collections.emptyList();
        if (annotationGeneratedModule == null || annotationGeneratedModule.isManifestParsingEnabled()) {
            manifestModules = new ManifestParser(applicationContext).parse();
        }

        if (annotationGeneratedModule != null
                && !annotationGeneratedModule.getExcludedModuleClasses().isEmpty()) {
            Set<Class<?>> excludedModuleClasses =
                    annotationGeneratedModule.getExcludedModuleClasses();
            Iterator<com.bumptech.glide.module.GlideModule> iterator = manifestModules.iterator();
            while (iterator.hasNext()) {
                com.bumptech.glide.module.GlideModule current = iterator.next();
                if (!excludedModuleClasses.contains(current.getClass())) {
                    continue;
                }
                if (Log.isLoggable(TAG, Log.DEBUG)) {
                    Log.d(TAG, "AppGlideModule excludes manifest GlideModule: " + current);
                }
                iterator.remove();
            }
        }

        if (Log.isLoggable(TAG, Log.DEBUG)) {
            for (com.bumptech.glide.module.GlideModule glideModule : manifestModules) {
                Log.d(TAG, "Discovered GlideModule from manifest: " + glideModule.getClass());
            }
        }

        RequestManagerRetriever.RequestManagerFactory factory =
                annotationGeneratedModule != null
                        ? annotationGeneratedModule.getRequestManagerFactory() : null;
        builder.setRequestManagerFactory(factory);
        for (com.bumptech.glide.module.GlideModule module : manifestModules) {
            module.applyOptions(applicationContext, builder);
        }
        if (annotationGeneratedModule != null) {
            annotationGeneratedModule.applyOptions(applicationContext, builder);
        }
        Glide glide = builder.build(applicationContext);
        for (com.bumptech.glide.module.GlideModule module : manifestModules) {
            module.registerComponents(applicationContext, glide, glide.registry);
        }
        if (annotationGeneratedModule != null) {
            annotationGeneratedModule.registerComponents(applicationContext, glide, glide.registry);
        }
        applicationContext.registerComponentCallbacks(glide);
        Glide.glide = glide;
    }
}
```

泛读之后，我们发现主要有 `三步`：

* 构建GlideBuilder
* 通过反射，加载 `可能存在` 的自定义模块，设置到Builder中
* 构建Glide

> 思危：如果直接使用GlideBuilder进行构建，则无法直接运用自定义模块的内容，需要自行注入。

## 模块职责划分，于Register挖细节

上一节中，我们略去了Glide实例化过程的一些细节，其实现内容为：向Register注册了各种模块，可以泛读一下源码：

```java
 registry
        .append(ByteBuffer.class,new ByteBufferEncoder())
        .append(InputStream.class,new StreamEncoder(arrayPool))
        /* Bitmaps */
        .append(Registry.BUCKET_BITMAP,ByteBuffer.class,Bitmap.class,byteBufferBitmapDecoder)
        .append(Registry.BUCKET_BITMAP,InputStream.class,Bitmap.class,streamBitmapDecoder)
        .append(
        Registry.BUCKET_BITMAP,
        ParcelFileDescriptor.class,
        Bitmap.class,
        parcelFileDescriptorVideoDecoder)
        .append(
        Registry.BUCKET_BITMAP,
        AssetFileDescriptor.class,
        Bitmap.class,
        VideoDecoder.asset(bitmapPool))
        .append(Bitmap.class,Bitmap.class,UnitModelLoader.Factory.<Bitmap>getInstance())
        .append(
        Registry.BUCKET_BITMAP,Bitmap.class,Bitmap.class,new UnitBitmapDecoder())
        .append(Bitmap.class,bitmapEncoder)
        /* BitmapDrawables */
        .append(
        Registry.BUCKET_BITMAP_DRAWABLE,
        ByteBuffer.class,
        BitmapDrawable.class,
        new BitmapDrawableDecoder<>(resources,byteBufferBitmapDecoder))
        .append(
        Registry.BUCKET_BITMAP_DRAWABLE,
        InputStream.class,
        BitmapDrawable.class,
        new BitmapDrawableDecoder<>(resources,streamBitmapDecoder))
        .append(
        Registry.BUCKET_BITMAP_DRAWABLE,
        ParcelFileDescriptor.class,
        BitmapDrawable.class,
        new BitmapDrawableDecoder<>(resources,parcelFileDescriptorVideoDecoder))
        .append(BitmapDrawable.class,new BitmapDrawableEncoder(bitmapPool,bitmapEncoder))
        /* GIFs */
        .append(
        Registry.BUCKET_GIF,
        InputStream.class,
        GifDrawable.class,
        new StreamGifDecoder(registry.getImageHeaderParsers(),byteBufferGifDecoder,arrayPool))
        .append(Registry.BUCKET_GIF,ByteBuffer.class,GifDrawable.class,byteBufferGifDecoder)
        .append(GifDrawable.class,new GifDrawableEncoder())
        /* GIF Frames */
        // Compilation with Gradle requires the type to be specified for UnitModelLoader here.
        .append(
        GifDecoder.class,GifDecoder.class,UnitModelLoader.Factory.<GifDecoder>getInstance())
        .append(
        Registry.BUCKET_BITMAP,
        GifDecoder.class,
        Bitmap.class,
        new GifFrameResourceDecoder(bitmapPool))
        /* Drawables */
        .append(Uri.class,Drawable.class,resourceDrawableDecoder)
        .append(
        Uri.class,Bitmap.class,new ResourceBitmapDecoder(resourceDrawableDecoder,bitmapPool))
        /* Files */
        .register(new ByteBufferRewinder.Factory())
        .append(File.class,ByteBuffer.class,new ByteBufferFileLoader.Factory())
        .append(File.class,InputStream.class,new FileLoader.StreamFactory())
        .append(File.class,File.class,new FileDecoder())
        .append(File.class,ParcelFileDescriptor.class,new FileLoader.FileDescriptorFactory())
        // Compilation with Gradle requires the type to be specified for UnitModelLoader here.
        .append(File.class,File.class,UnitModelLoader.Factory.<File>getInstance())
        /* Models */
        .register(new InputStreamRewinder.Factory(arrayPool))
        .append(int.class,InputStream.class,resourceLoaderStreamFactory)
        .append(
        int.class,
        ParcelFileDescriptor.class,
        resourceLoaderFileDescriptorFactory)
        .append(Integer.class,InputStream.class,resourceLoaderStreamFactory)
        .append(
        Integer.class,
        ParcelFileDescriptor.class,
        resourceLoaderFileDescriptorFactory)
        .append(Integer.class,Uri.class,resourceLoaderUriFactory)
        .append(
        int.class,
        AssetFileDescriptor.class,
        resourceLoaderAssetFileDescriptorFactory)
        .append(
        Integer.class,
        AssetFileDescriptor.class,
        resourceLoaderAssetFileDescriptorFactory)
        .append(int.class,Uri.class,resourceLoaderUriFactory)
        .append(String.class,InputStream.class,new DataUrlLoader.StreamFactory())
        .append(String.class,InputStream.class,new StringLoader.StreamFactory())
        .append(String.class,ParcelFileDescriptor.class,new StringLoader.FileDescriptorFactory())
        .append(
        String.class,AssetFileDescriptor.class,new StringLoader.AssetFileDescriptorFactory())
        .append(Uri.class,InputStream.class,new HttpUriLoader.Factory())
        .append(Uri.class,InputStream.class,new AssetUriLoader.StreamFactory(context.getAssets()))
        .append(
        Uri.class,
        ParcelFileDescriptor.class,
        new AssetUriLoader.FileDescriptorFactory(context.getAssets()))
        .append(Uri.class,InputStream.class,new MediaStoreImageThumbLoader.Factory(context))
        .append(Uri.class,InputStream.class,new MediaStoreVideoThumbLoader.Factory(context))
        .append(
        Uri.class,
        InputStream.class,
        new UriLoader.StreamFactory(contentResolver))
        .append(
        Uri.class,
        ParcelFileDescriptor.class,
        new UriLoader.FileDescriptorFactory(contentResolver))
        .append(
        Uri.class,
        AssetFileDescriptor.class,
        new UriLoader.AssetFileDescriptorFactory(contentResolver))
        .append(Uri.class,InputStream.class,new UrlUriLoader.StreamFactory())
        .append(URL.class,InputStream.class,new UrlLoader.StreamFactory())
        .append(Uri.class,File.class,new MediaStoreFileLoader.Factory(context))
        .append(GlideUrl.class,InputStream.class,new HttpGlideUrlLoader.Factory())
        .append(byte[].class,ByteBuffer.class,new ByteArrayLoader.ByteBufferFactory())
        .append(byte[].class,InputStream.class,new ByteArrayLoader.StreamFactory())
        .append(Uri.class,Uri.class,UnitModelLoader.Factory.<Uri>getInstance())
        .append(Drawable.class,Drawable.class,UnitModelLoader.Factory.<Drawable>getInstance())
        .append(Drawable.class,Drawable.class,new UnitDrawableDecoder())
        /* Transcoders */
        .register(
        Bitmap.class,
        BitmapDrawable.class,
        new BitmapDrawableTranscoder(resources))
        .register(Bitmap.class,byte[].class,bitmapBytesTranscoder)
        .register(
        Drawable.class,
        byte[].class,
        new DrawableBytesTranscoder(
        bitmapPool,bitmapBytesTranscoder,gifDrawableBytesTranscoder))
        .register(GifDrawable.class,byte[].class,gifDrawableBytesTranscoder);
```

`直接讲解` 这段代码是 `无意义` 的，我们从 `Register` 类入手，分析注册的内容。

```java
public class Registry {
    public static final String BUCKET_GIF = "Gif";
    public static final String BUCKET_BITMAP = "Bitmap";
    public static final String BUCKET_BITMAP_DRAWABLE = "BitmapDrawable";
    private static final String BUCKET_PREPEND_ALL = "legacy_prepend_all";
    private static final String BUCKET_APPEND_ALL = "legacy_append";

    private final ModelLoaderRegistry modelLoaderRegistry;
    private final EncoderRegistry encoderRegistry;
    private final ResourceDecoderRegistry decoderRegistry;
    private final ResourceEncoderRegistry resourceEncoderRegistry;
    private final DataRewinderRegistry dataRewinderRegistry;
    private final TranscoderRegistry transcoderRegistry;
    private final ImageHeaderParserRegistry imageHeaderParserRegistry;

    private final ModelToResourceClassCache modelToResourceClassCache =
            new ModelToResourceClassCache();
    private final LoadPathCache loadPathCache = new LoadPathCache();
    private final Pool<List<Throwable>> throwableListPool = FactoryPools.threadSafeList();

    public Registry() {
        this.modelLoaderRegistry = new ModelLoaderRegistry(throwableListPool);
        this.encoderRegistry = new EncoderRegistry();
        this.decoderRegistry = new ResourceDecoderRegistry();
        this.resourceEncoderRegistry = new ResourceEncoderRegistry();
        this.dataRewinderRegistry = new DataRewinderRegistry();
        this.transcoderRegistry = new TranscoderRegistry();
        this.imageHeaderParserRegistry = new ImageHeaderParserRegistry();
        setResourceDecoderBucketPriorityList(
                Arrays.asList(BUCKET_GIF, BUCKET_BITMAP, BUCKET_BITMAP_DRAWABLE));
    }
    //API略去
}
```

### 资源Model加载器

`modelLoaderRegistry` 用于注册 `资源Model加载方式` 信息。例如，资源的定义Model为 `Uri` 类，注册一个 `ModelLoaderFactory` ，生产对应的
`ModelLoader`，将 `Uri` 媒体实体，加载为 `InputStream` 或者其他形式的 `内容`

相关类：

* `ModelLoaderFactory`
* `ModelLoader` 等

### 编码器

`encoderRegistry` 用于注册 `内容encode方式` 信息。例如，媒体实体已被加载为 `InputStream`，注册一个对应处理的 `Encoder<InputStream>`，
用于处理内容的encode，如写入磁盘过程

相关类：`Encoder` 接口及其实现类

相应的，`resourceEncoderRegistry` 用于注册 `Resource 的encode方式`，这些encoder面向 `Resource`，相比于 `InputStream` 等低级形式的编码内容封装，
`Resource` 实现类进行了更高级别的编码内容封装。

相关类：

* `ResourceEncoder` 接口及其实现类
* `Resource` 接口及其实现类
* `EncodeStrategy`

不难理解：媒体资源的数据封装存在不同 `层级`：

* 读取数据的InputSteam
* Bitmap，BitmapDrawable

所以 encode过程 对不同层级的封装类，有不同的处理，当然，encoder可以通过 `wrapper`，`adapter`,`transfer` 等形式，利用其它encoder扩展功能

### 解码器

`decoderRegistry` 用于注册 `媒体数据解码为Resource方式` 的信息。

前面我们提到 `modelLoaderRegistry` 注册的信息中，可以得到 `ModelLoader`， 它们负责将 `资源Model` 加载为 `资源内容`，这种 `资源内容`以 `InputStream` 等低级形式存在即可。

不难理解，这可以有效的解决 `类数量爆炸` 问题。

> 脑暴：假定 Model 的类数量为m个，解码后输出的形式有n个
> * 不采用这种方式处理，一共需要 m*n 个 decoder 类
> * 采用这种方式处理，一共需要 m+n 个 decoder 类
>
> 当 `层级` 增加时，差距会更加明显
>
> 按照经验推断，Glide在处理decode过程时，采用了 `Wrapper/delegate` 模式 或者 `bridge` 模式。

> 思变：项目中是否可以使用类似方式，处理不同层级之间 `数据Bean` 的 `转化` 或者 `包装`？

相关类：

* `ResourceDecoder` 及其实现类
* `Resource` 及其实现类

### 转换器

*虽然直译为转码器，但是容易概念混淆，按照其功能称为转换器*

`transcoderRegistry` 用于注册 `转化器，及其负责的原始类型和转换类型` 信息，转换器将资源从一种形式转换为另一种形式。例如: `Bitmap -> byte[]`

相关类：

* `ResourceTranscoder` 及其实现类
* `Resource` 及其实现类

### 头信息解析器

`imageHeaderParserRegistry` 用于注册 `头信息解析器` 信息

*注，部分操作系统为了更简便的决定文件的处理方式，利用了文件扩展名。但文件实质内容的编码方式信息，存储在文件头信息中，又称： `File Sigs`、`文件魔数` 。*

> 例如 FF D8 代表 Generic JPEG image file，即 JPE, JPEG, JPG

[了解更多关于File Sigs内容](https://www.garykessler.net/library/file_sigs.html)

相关类：

* `ImageHeaderParser` 及其实现类
* `ImageType`

## 模块职责总结

通过前文的分析，并查阅Glide源码，我们可以对模块职责进行一次初步总结。

在之后的文章中，我们将对具体模块进行 `更细致` 的挖掘，此处总结的类信息，就是 `突破口` 。

*考虑到目前收集的信息 `并不全面` ，我不打算绘制一个图，下面的信息也 `并非` 十分重要，所以将其作为一个 `临时的备忘录` 即可。
在后续内容中，我会对各个模块单独总结成图*

下一篇，我们将对 `资源Model加载、获取流程` 进行 `深度挖掘`。

---

### 缓存与池

* MemorySizeCalculator
* LruBitmapPool
* BitmapPoolAdapter
* LruArrayPool
* LruResourceCache
* InternalCacheDiskCacheFactory

### 资源获取

* RequestManagerRetriever
* RequestManager
* RequestBuilder
* Request  
* DefaultConnectivityMonitorFactory

### 核心流程

* Engine

其他信息尚不能确定。主要流程的相关信息如下：

#### 资源Model加载器

* `ModelLoaderFactory`
* `ModelLoader` 等

#### 编码器

* `Encoder` 接口及其实现类
* `ResourceEncoder` 接口及其实现类
* `Resource` 接口及其实现类
* `EncodeStrategy`

#### 解码器

* `ResourceDecoder` 及其实现类
* `Resource` 及其实现类

#### 转换器

* `ResourceTranscoder` 及其实现类
* `Resource` 及其实现类

#### 头信息解析器

* `ImageHeaderParser` 及其实现类
* `ImageType`

### 线程池

* GlideExecutor

### Glide的创建与配置

`GlideBuilder` 用于创建，`com.bumptech.glide.module` 包下的内容用于自定义模块，诸如：

* `AppGlideModule`
* `LibraryGlideModule`

以及一些已废弃的内容，不再赘述

## 思危、思退、思变

*可能有读者习惯性地拉到尾部，前面提到，结尾处的大篇幅内容都是 `备忘录` 性质,而且之后的系列文章会在这些内容的基础上展开，可读可不读*

但既然拉到了这里，就思考一下吧，阅读优秀的框架，不仅仅可以深入了解框架内容，也可以锻炼设计能力：

* 思危：使用GlideBuilder构建Glide实例是否有没有注意到的危险？
* 思危：项目中再次对Glide实现单例是否存在危险？
* 思变：项目中是否存在可以借鉴Glide的设计思路的地方？