## 重识Runtime _objc_init()实现细节

### 前言

Runtime作为Objective-C的运行时的核心一直是iOS开发者必须要学习了解的，从事iOS开发也有一段时间，之前都是断断续续看Runtime的源码或者是通过别人的博客思路来学习了解Runtime。从今年三月份开始系统的研读了Runtime的源码、发现大致和之前了解的相同(网上各路大神太多了,摊手脸！),刚好最近项目不忙，所以决定写几篇博客记录下这段经历，LET'S GO。

通过翻码发现苹果在库初始化之前由dyld程序注册了三个通知程序，来实现整个RunTime系统的组建、+load的调用、以及资源释放。

_objc_init源码实现如下:

```
void _objc_init(void)
{
    static bool initialized = false;
    if (initialized) return;
    initialized = true;
    
    //Alex注释: 读取Runtime相关的环境变量
    environ_init();
    tls_init();
    static_init();
    lock_init();
    //Alex注释: 初始化libobjc异常处理系统
    exception_init();
    /* Alex注释: 重要的来了，注册通知程序：这里通过名字我们可以猜测出苹果通过dyld注册了
     * 三个程序分别是map_images、load_images、unmap_image通过名字可以大概猜测出他们
     * 的作用、接下来让我们来揭开这层面纱。
     * ***************这里比较遗憾的是没有找到_dyld_objc_notify_register实现的源码
     * ***************如果有大神知道欢迎评论指出
     */
    _dyld_objc_notify_register(&map_images, load_images, unmap_image);
}
```

[dyld源码](https://github.com/opensource-apple/dyld) 感兴趣的可以去了解。

###  一、map_images方法的实现分析

苹果对于map_images的注释如下:
    Process the given images which are being mapped in by dyld.
    Calls ABI-agnostic code after taking ABI-specific locks.
Google翻译之后大致意思是处理由dyld映射的给定images，获取ABI锁并调用与ABI无关的代码。
images：网上很多翻译为镜像，但是我看源码则更像是`资源文件`解析。

map_images的源码实现如下:

```
void map_images(unsigned count, const char * const paths[],
           const struct mach_header * const mhdrs[])
{
    //Alex注释: 开启Runtime锁
    mutex_locker_t lock(runtimeLock);
    //Alex注释: 将具体任务交由map_images_nolock执行
    return map_images_nolock(count, paths, mhdrs);
}
```

map_images将具体的处理交由map_images_nolock方法执行，map_images_nolock实现大概150行代码，其主要做了下面这些事情:
- 1、获取共享缓存的内存区域。
- 2、将dyld中读取的mach_header *mhdrs\[\]转化为header_info *hList\[\]结构体数组。
- 3、将转化出的header_info添加到FirstHeader的结构体链表中。
- 4、初始化静态的全局变量namedSelectors(NXMapTable类型)，并注册我们所熟知的一些方法
   如load、initialize、retain、alloc、dealloc、copy。
- 5、初始化AutoreleasePool，静态全局变量static uint8_t SideTableBuf
   \[sizeof(StripedMap<SideTable>)]。
- 6、最后将转化好的header_info结构体数组交给_read_images进行更细致的解析。

map_images_nolock源码(有精简)实现如下:

```
void map_images_nolock(unsigned mhCount, const char * const mhPaths[],
                  const struct mach_header * const mhdrs[])
{
    static bool firstTime = YES;
    //Alex注释: 初始化header_info结构体数组
    header_info *hList[mhCount];
    //Alex注释: 用于记录hList的大小
    uint32_t hCount;
    //Alex注释: 用于记录方法和消息的个数
    size_t selrefCount = 0;

    if (firstTime) {
        //Alex注释: 获取共享缓存的内存区域
        preopt_init();
    }
    hCount = 0;

    int totalClasses = 0;
    int unoptimizedTotalClasses = 0;
    {
        uint32_t i = mhCount;
        while (i--) {
            const headerType *mhdr = (const headerType *)mhdrs[i];
            /*Alex注释:
             * 将mhdr 转化为 header_info 添加到 FirstHeader 链表中；如果添加成功，addHeader 返回 header_info 结构体 else NULL。
             */
            auto hi = addHeader(mhdr, mhPaths[i], totalClasses, unoptimizedTotalClasses);
            if (!hi) {
                continue;
            }
            
            if (mhdr->filetype == MH_EXECUTE) {
             /*Alex注释:
             * 获取 header_info中的sel数量，这里区分了__OBJC2__和其他版本、是因为OBJC
             * 2之后提出了消息概念、将之前的SEL变为了message_ref_t { IMP imp;  SEL 
             * sel;};的结构体封装，而这正是后来黑魔法实现的基本原理。
             */
#if __OBJC2__
                size_t count;
                _getObjc2SelectorRefs(hi, &count);
                selrefCount += count;
                _getObjc2MessageRefs(hi, &count);
                selrefCount += count;
#else
                _getObjcSelectorRefs(hi, &selrefCount);
#endif
            }
            hList[hCount++] = hi;
        }
    }

    if (firstTime) {
        /*Alex注释:
         * 初始化全局的namedSelectors(NXMapTable类型)变量，并注册必要的的方法如 load、initialize、retain、release、autorelease、retainCount等...
         */
        sel_init(selrefCount);
        /*Alex注释:
         * 初始化自动释放池AutoreleasePool(AutoreleasePoolPage)、全局SideTableBuf变量(StripedMap<SideTable>)
         * SideTableBuf是Runtime的核心、
           关于SideTableBuf本人的其他博客有介绍欢迎翻阅。
         */
        arr_init();
    }

    if (hCount > 0) {
        /*Alex注释:
         * 经过以上处理将mach_header结构体数据转化为了header_info结构体数据,接着苹果将header_info的处理转交给了_read_images方法去实现。
         */
        _read_images(hList, hCount, totalClasses, unoptimizedTotalClasses);
    }
    firstTime = NO;
}
```

苹果对于_read_images的注释如下:
     Perform initial processing of the headers in the linked 
     list beginning with headerList. 
     Called by: map_images_nolock
Google翻译之后大意为: 对链接中的header_info列表执行初始化处理，由map_images_nolock调用

_read_images主要做了如下这些事情：
- 1、初始化指针代码混淆器。
- 2、初始化全局变量gdb_objc_realized_classes(NXMapTable)，保存不在共享缓存中的命名类，
   无论是否实现。
- 3、初始化静态全局变量allocatedClasses(NXMapTable)，缓存已经被objc_allocateClassPair
   初始化的类和元类。
- 4、从header_info中读取classref_t列表，如果类未被初始化、从future_named_class_map读取
   类进行初始化，如果是非元类将其保存到静态全局变量nonmeta_class_map中，如果是元类将其保存到全局变量gdb_objc_realized_classes中，最后将初始化的类或者元类保存到allocatedClasses全局Map中。
- 5、读取header_info中的SEL并保存到静态的全局变量namedSelectors中。
- 6、读取header_info中的protocol并保存到静态变量protocol_map中。

接下来通过源码逐行来了解苹果是如何处理header_info列表的,

_read_images(有精简)源码实现如下:

```
void _read_images(header_info **hList, uint32_t hCount, int totalClasses, int unoptimizedTotalClasses)
{
    header_info *hi;
    uint32_t hIndex;
    size_t count;
    size_t i;
    Class *resolvedFutureClasses = nil;
    size_t resolvedFutureClassCount = 0;
    static bool doneOnce;
    runtimeLock.assertLocked();

    //Alex注释：首次加载初始化
    if (!doneOnce) {
        doneOnce = YES;

    //Alex注释:ISA适配
#if SUPPORT_NONPOINTER_ISA

# if SUPPORT_INDEXED_ISA
        //Alex注释: SwiftVersion3之前的版本不支持NonpointerIsa
        for (EACH_HEADER) {
            if (hi->info()->containsSwift()  &&
                hi->info()->swiftVersion() < objc_image_info::SwiftVersion3)
            {
                DisableNonpointerIsa = true;
                break;
            }
        }
# endif

# if TARGET_OS_OSX
        //Alex注释: OS X 10.11之前的版本不支持NonpointerIsa
        if (dyld_get_program_sdk_version() < DYLD_MACOSX_VERSION_10_11) {
            DisableNonpointerIsa = true;
        }
        //Alex注释: 有__DATA,__objc_rawisa段的不支持NonpointerIsa
        for (EACH_HEADER) {
            if (hi->mhdr()->filetype != MH_EXECUTE) continue;
            unsigned long size;
            if (getsectiondata(hi->mhdr(), "__DATA", "__objc_rawisa", &size)) {
                DisableNonpointerIsa = true;
            }
            break;  
        }
# endif

#endif
        //Alex注释: 是否禁用TaggedPointers
        if (DisableTaggedPointers) {
            disableTaggedPointers();
        }
        //Alex注释: 初始化混淆器(随机产生)，保护代码安全。
        initializeTaggedPointerObfuscator();

        int namedClassesSize = 
            (isPreoptimized() ? unoptimizedTotalClasses : totalClasses) * 4 / 3;
        /*Alex注释：
         * gdb_objc_realized_classes 保存不在动态共享缓存(dyld shared cache)中的 命名类  -> hash表
         * allocatedClasses 缓存已经被objc_allocateClassPair方法初始化的类和元类  -> hash表
         */
        gdb_objc_realized_classes =
            NXCreateMapTable(NXStrValueMapPrototype, namedClassesSize);
        allocatedClasses = NXCreateHashTable(NXPtrPrototype, 0, nil);
    }

    for (EACH_HEADER) {
        //Alex注释: 加载每个header_info结构体的Classref_t结构体列表
        classref_t *classlist = _getObjc2ClassList(hi, &count);
        
        if (! mustReadClasses(hi)) {
            continue;
        }

        bool headerIsBundle = hi->isBundle();
        bool headerIsPreoptimized = hi->isPreoptimized();

        for (i = 0; i < count; i++) {
            Class cls = (Class)classlist[i];
               /*Alex注释：
                * readClass主要工作如下:
                * 1、从future_named_class_map(NXMapTable)
                *    全局对象中查找未实现的newCls,取出该Class的(class_rw_t)data数据
                *    (rwNew)，将cls数据拷贝到newCls,将拷贝后的newCls的
                *    (class_rw_t)data强转为class_ro_t并赋值到rwNew的ro成员变量，
                *    最后将rwNew赋值给newCls的(class_rw_t)data
                * 2、将newCls添加到全局变量(NXMapTable)gdb_objc_realized_classes的
                *    MapTable中，gdb_objc_realized_classes保存不在动态共享缓存中的类
                *    无论是否实现
                * 3、将newCls添加到全局变量(NXMapTable)allocatedClasses的
                *    MapTable中，allocatedClasses表保存已经Allocated的类
                */
            Class newCls = readClass(cls, headerIsBundle, headerIsPreoptimized);

            if (newCls != cls  &&  newCls) {
                //Alex注释: newCls添加进数组
                resolvedFutureClasses = (Class *)
                    realloc(resolvedFutureClasses, 
                            (resolvedFutureClassCount+1) * sizeof(Class));
                resolvedFutureClasses[resolvedFutureClassCount++] = newCls;
            }
        }
    }
*******************************需要仔细阅读源码
    //Alex注释: noClassesRemapped创建remapped_class_map静态变量(NXMapTable)，
    if (!noClassesRemapped()) {
        for (EACH_HEADER) {
            Class *classrefs = _getObjc2ClassRefs(hi, &count);
            for (i = 0; i < count; i++) {
                //Alex注释: 返回实时的类指针，该指针可能指向已经重新分配内存的类结构
                remapClassRef(&classrefs[i]);
            }
            classrefs = _getObjc2SuperRefs(hi, &count);
            for (i = 0; i < count; i++) {
                remapClassRef(&classrefs[i]);
            }
        }
    }
    //Alex注释: 将SEL保存到全局的namedSelectors(NXMapTable表)中
    static size_t UnfixedSelectors;
    {
        mutex_locker_t lock(selLock);
        for (EACH_HEADER) {
            if (hi->isPreoptimized()) continue;
            
            bool isBundle = hi->isBundle();
            SEL *sels = _getObjc2SelectorRefs(hi, &count);
            UnfixedSelectors += count;
            for (i = 0; i < count; i++) {
                const char *name = sel_cname(sels[i]);
                sels[i] = sel_registerNameNoLock(name, isBundle);
            }
        }
    }

    //Alex注释: 加载protocols，并保存到静态变量(NXMapTable)protocol_map中
    for (EACH_HEADER) {
        extern objc_class OBJC_CLASS_$_Protocol;
        Class cls = (Class)&OBJC_CLASS_$_Protocol;
        assert(cls);
        NXMapTable *protocol_map = protocols();
        bool isPreoptimized = hi->isPreoptimized();
        bool isBundle = hi->isBundle();

        protocol_t **protolist = _getObjc2ProtocolList(hi, &count);
        for (i = 0; i < count; i++) {
            readProtocol(protolist[i], cls, protocol_map, 
                         isPreoptimized, isBundle);
        }
    }

    //Alex注释: 加载ProtocolRefs，并保存到静态变量(NXMapTable)protocol_map中
    for (EACH_HEADER) {
        protocol_t **protolist = _getObjc2ProtocolRefs(hi, &count);
        for (i = 0; i < count; i++) {
            remapProtocolRef(&protolist[i]);
        }
    }

    //Alex注释: 加载非惰性类，用于+load和静态的instance
    for (EACH_HEADER) {
        classref_t *classlist = 
            _getObjc2NonlazyClassList(hi, &count);
        for (i = 0; i < count; i++) {
            Class cls = remapClass(classlist[i]);
            if (!cls) continue;

#if TARGET_OS_SIMULATOR
            if (cls->cache._buckets == (void*)&_objc_empty_cache  &&  
                (cls->cache._mask  ||  cls->cache._occupied)) 
            {
                cls->cache._mask = 0;
                cls->cache._occupied = 0;
            }
            if (cls->ISA()->cache._buckets == (void*)&_objc_empty_cache  &&  
                (cls->ISA()->cache._mask  ||  cls->ISA()->cache._occupied)) 
            {
                cls->ISA()->cache._mask = 0;
                cls->ISA()->cache._occupied = 0;
            }
#endif
*******************************敲重点：：：需要仔细阅读源码
            //Alex注释:将cls添加到allocatedClasses全局map中
            addClassTableEntry(cls);
            /* Alex注释: 加载非惰性类、执行初始化
            *  递归布局class、superClass、metaClass的树形结构
            *  reconcileInstanceVariables方法协调实例变量的偏移量，内存布局
            */
            realizeClass(cls);
        }
    }
    //Alex注释:实现新的未解决的未来类？？？？？？？？？
    if (resolvedFutureClasses) {
        for (i = 0; i < resolvedFutureClassCount; i++) {
            realizeClass(resolvedFutureClasses[i]);
            resolvedFutureClasses[i]->setInstancesRequireRawIsa(false/*inherited*/);
        }
        free(resolvedFutureClasses);
    }    

    //Alex注释:加载类别(categories)列表
    for (EACH_HEADER) {
        category_t **catlist = 
            _getObjc2CategoryList(hi, &count);
        bool hasClassProperties = hi->info()->hasCategoryClassProperties();

        for (i = 0; i < count; i++) {
            category_t *cat = catlist[i];
            Class cls = remapClass(cat->cls);

            if (!cls) {
                catlist[i] = nil;
                continue;
            }

            bool classExists = NO;
            //Alex注释:判断是否有类的实例方法、协议、实例属性
            if (cat->instanceMethods ||  cat->protocols  
                ||  cat->instanceProperties) 
            {
                //Alex注释:category存放在静态的MapTable(静态变量category_map)中,
                 * 苹果是将category和header_info封装成locstamped_category_t结构体, 
                 * 之后将(locstamped_category_t){category, header_info} 添加到全局
                 * 变量category_map中category->cls映射的locstamped_category_t结构
                 * 体数组中
                 */
                addUnattachedCategoryForClass(cat, cls, hi);
                if (cls->isRealized()) {
                    //Alex注释: 如果类已经初始化将category的方法、属性、协议追加到类的
                    * class_rw_t结构体对应的methods、properties、protocols成员
                    * 变量中
                    * 添加到class_rw_t结构体的操作是在attachCategories方法中实现的。
                    */
                    remethodizeClass(cls);
                    classExists = YES;
                }
            }
            //Alex注释:这里处理的是元类的方法、协议、属性
            if (cat->classMethods  ||  cat->protocols  
                ||  (hasClassProperties && cat->_classProperties)) 
            {
                addUnattachedCategoryForClass(cat, cls->ISA(), hi);
                if (cls->ISA()->isRealized()) {
                    remethodizeClass(cls->ISA());
                }
            }
        }
    }
    //Alex注释:调试不易碎的Ivars
    if (DebugNonFragileIvars) {
        //Alex注释:非惰性的实现所有已知的未实现的类
        realizeAllClasses();
    }

    //Alex注释：打印预选的方法列表、优化方法列表、全部的Classes、优化的Classes, DEBUG
    if (PrintPreopt) {
        static unsigned int PreoptTotalMethodLists;
        static unsigned int PreoptOptimizedMethodLists;
        static unsigned int PreoptTotalClasses;
        static unsigned int PreoptOptimizedClasses;

        for (EACH_HEADER) {
            classref_t *classlist = _getObjc2ClassList(hi, &count);
            for (i = 0; i < count; i++) {
                Class cls = remapClass(classlist[i]);
                if (!cls) continue;

                PreoptTotalClasses++;
                if (hi->isPreoptimized()) {
                    PreoptOptimizedClasses++;
                }
                
                const method_list_t *mlist;
                if ((mlist = ((class_ro_t *)cls->data())->baseMethods())) {
                    PreoptTotalMethodLists++;
                    if (mlist->isFixedUp()) {
                        PreoptOptimizedMethodLists++;
                    }
                }
                if ((mlist=((class_ro_t *)cls->ISA()->data())->baseMethods())) {
                    PreoptTotalMethodLists++;
                    if (mlist->isFixedUp()) {
                        PreoptOptimizedMethodLists++;
                    }
                }
            }
        }
    }

#undef EACH_HEADER
}

```

经过以上处理类的装载已经完成、然后执行load_images方法、处理已经被加载的类的+load方法。

### 二、分析load_images实现

苹果对于load_images的注释如下:
    Process +load in the given images which are being mapped in by dyld.
Google翻译后大意是：处理在dyld被mapped的images的+load方法。

load_images的代码并不多，主要是对load_images执行递归锁操作，之后调用prepare_load_methods装载+load方法，最后通过call_load_methods调用Class和Category的+load方法。

```
void load_images(const char *path __unused, const struct mach_header *mh)
{
    if (!hasLoadMethods((const headerType *)mh)) return;
    //Alex注释: loadMethod加锁
    recursive_mutex_locker_t lock(loadMethodLock);
    {
        mutex_locker_t lock2(runtimeLock);
        //Alex注释: 装载+load方法
        prepare_load_methods((const headerType *)mh);
    }
    //Alex注释: 调用+load方法
    call_load_methods();
}
```


我们知道category和class都可以实现+load方法，下面通过源码来细究Category和Class的+load方法是如何实现的。

prepare_load_methods方法实现比较简单、主要是对mhdr中的Class和Category进行遍历、最后将+load的方法处理分发到了schedule_class_load和add_category_to_loadable_list方法、分别用于装载Class的+load和Category的+load方法。

prepare_load_methods源码实现如下:
```
void prepare_load_methods(const headerType *mhdr)
{
    size_t count, i;
    runtimeLock.assertLocked();

    classref_t *classlist = _getObjc2NonlazyClassList(mhdr, &count);
    for (i = 0; i < count; i++) {
        /*Alex注释:
         * 加载类的 +load方法，并将Class的load方法和类保存到全局的
         * (struct loadable_class *)loadable_classes 结构体数组中
         */
        schedule_class_load(remapClass(classlist[i]));
    }

    category_t **categorylist = _getObjc2NonlazyCategoryList(mhdr, &count);
    for (i = 0; i < count; i++) {
        category_t *cat = categorylist[i];
        Class cls = remapClass(cat->cls);
        if (!cls) continue; 
        realizeClass(cls);
        assert(cls->ISA()->isRealized());
        /*Alex注释:
         * 加载category的 +load方法， 并将category的load方法和category存储到 
         * (struct loadable_category *)loadable_categories 结构体数组中
         */
        add_category_to_loadable_list(cat);
    }
}
```

关于类的+load方法的装载、通过源码可以看出苹果是采用递归的方式从父类到子类逐个遍历，最后将具体的装载处理交由add_class_to_loadable_list方法执行。

add_class_to_loadable_list实现就更加简明了，大致分为下面几步: 
- 1、获取Class的+load方法
- 2、确保struct loadable_class *loadable_classes结构体数组有足够空间
- 3、将Class和+load方法保存到loadable_classes结构体数组中

```
static void schedule_class_load(Class cls)
{
    if (!cls) return;
    assert(cls->isRealized());

    if (cls->data()->flags & RW_LOADED) return;

    schedule_class_load(cls->superclass);
    add_class_to_loadable_list(cls);
   <!-- add_class_to_loadable_list 实现 {
        IMP method;

        loadMethodLock.assertLocked();

        method = cls->getLoadMethod();
        if (!method) return; 
        
        if (loadable_classes_used == loadable_classes_allocated) {
            loadable_classes_allocated = loadable_classes_allocated*2 + 16;
            loadable_classes = (struct loadable_class *)
                realloc(loadable_classes,
                                  loadable_classes_allocated *
                                  sizeof(struct loadable_class));
        }
        
        loadable_classes[loadable_classes_used].cls = cls;
        loadable_classes[loadable_classes_used].method = method;
        loadable_classes_used++;
    } -->
    cls->setInfo(RW_LOADED); 
}
```

关于Category的+load方法的装载，其实和类的装载大致相同。只是这里在调用add_category_to_loadable_list方法装载时、如果Class没有realize的话会对Class进行realize,还要注意的是Category的+load方法是被单独保存在一个全局的struct loadable_category *loadable_categories
静态变量中的,他和类的+load方法是分开存放的。

+load方法装载之后苹果通过call_load_methods方法来调用Class和Category的+load方法。Class和Category的+load调用顺序在这里也可以得到想要的答案。

call_load_methods的实现源码如下:
```
void call_load_methods(void)
{
    static bool loading = NO;
    bool more_categories;

    loadMethodLock.assertLocked();

    if (loading) return;
    loading = YES;

    void *pool = objc_autoreleasePoolPush();

    do {
        //Alex注释: 循环调用Class的+load方法
        while (loadable_classes_used > 0) {
            call_class_loads();
        }

        //Alex注释:调用category的+load方法，仅调用一次
        more_categories = call_category_loads();

    } while (loadable_classes_used > 0  ||  more_categories);

    objc_autoreleasePoolPop(pool);

    loading = NO;
}
```

类的+load的调用源码:
```
static void call_class_loads(void)
{
    int i;
    struct loadable_class *classes = loadable_classes;
    int used = loadable_classes_used;
    loadable_classes = nil;
    loadable_classes_allocated = 0;
    loadable_classes_used = 0;
    //Alex注释:遍历调用Class的+load方法
    for (i = 0; i < used; i++) {
        Class cls = classes[i].cls;
        load_method_t load_method = (load_method_t)classes[i].method;
        if (!cls) continue; 
        //Alex注释:调用+load方法
        (*load_method)(cls, SEL_load);
    }
    if (classes) free(classes);
}
```

Category的+load方法的调用源码:
```
static bool call_category_loads(void)
{
    int i, shift;
    bool new_categories_added = NO;
    
    struct loadable_category *cats = loadable_categories;
    int used = loadable_categories_used;
    int allocated = loadable_categories_allocated;
    loadable_categories = nil;
    loadable_categories_allocated = 0;
    loadable_categories_used = 0;

    //Alex注释: 循环调用Category的+load方法
    for (i = 0; i < used; i++) {
        Category cat = cats[i].cat;
        load_method_t load_method = (load_method_t)cats[i].method;
        Class cls;
        if (!cat) continue;

        cls = _category_getClass(cat);
        if (cls  &&  cls->isLoadable()) {
            //Alex注释:调用+load方法
            (*load_method)(cls, SEL_load);
            cats[i].cat = nil;
        }
    }

    shift = 0;
    for (i = 0; i < used; i++) {
        if (cats[i].cat) {
            cats[i-shift] = cats[i];
        } else {
            shift++;
        }
    }
    used -= shift;

    new_categories_added = (loadable_categories_used > 0);
    for (i = 0; i < loadable_categories_used; i++) {
        if (used == allocated) {
            allocated = allocated*2 + 16;
            cats = (struct loadable_category *)
                realloc(cats, allocated *
                                  sizeof(struct loadable_category));
        }
        cats[used++] = loadable_categories[i];
    }

    if (loadable_categories) free(loadable_categories);

    if (used) {
        loadable_categories = cats;
        loadable_categories_used = used;
        loadable_categories_allocated = allocated;
    } else {
        if (cats) free(cats);
        loadable_categories = nil;
        loadable_categories_used = 0;
        loadable_categories_allocated = 0;
    }

    return new_categories_added;
}
```

### 三、分析unmap_image实现

unmap_image对应map_images有map操作就有对应的unmap。
苹果对于unmap的注释如下：
    Process the given image which is about to be unmapped by dyld.
Google翻译大意为:处理由dyld取消映射的image。

```
unmap_image的源码如下:
void  unmap_image(const char *path __unused, const struct mach_header *mh)
{
    recursive_mutex_locker_t lock(loadMethodLock);
    mutex_locker_t lock2(runtimeLock);
    //Alex注释: 加锁将具体的任务转交unmap_image_nolock方法
    unmap_image_nolock(mh);
}
```

从全局的header_info结构体链表中遍历查找Runtime的header_info结构体，调用_unload_image处理header_info结构体、最后将其从链表中移除。
```
void unmap_image_nolock(const struct mach_header *mh)
{
    header_info *hi;
    
    for (hi = FirstHeader; hi != NULL; hi = hi->getNext()) {
        if (hi->mhdr() == (const headerType *)mh) {
            break;
        }
    }

    if (!hi) return;

    _unload_image(hi);

    removeHeader(hi);
    free(hi);
}
```

```
void _unload_image(header_info *hi)
{
    size_t count, i;

    loadMethodLock.assertLocked();
    runtimeLock.assertLocked();
    //Alex注释: 停止附加Category和调用Category的+load方法
    category_t **catlist = _getObjc2CategoryList(hi, &count);
    for (i = 0; i < count; i++) {
        category_t *cat = catlist[i];
        if (!cat) continue; 
        Class cls = remapClass(cat->cls);
        assert(cls); 

        removeUnattachedCategoryForClass(cat, cls);

        remove_category_from_loadable_list(cat);
    }

    NXHashTable *classes = NXCreateHashTable(NXPtrPrototype, 0, nil);
    classref_t *classlist;
    //Alex注释: 从header_info结构体中加载已经remap的Classs
    classlist = _getObjc2ClassList(hi, &count);
    for (i = 0; i < count; i++) {
        Class cls = remapClass(classlist[i]);
        if (cls) NXHashInsert(classes, cls);
    }

    classlist = _getObjc2NonlazyClassList(hi, &count);
    for (i = 0; i < count; i++) {
        Class cls = remapClass(classlist[i]);
        if (cls) NXHashInsert(classes, cls);
    }

    NXHashState hs;
    Class cls;

    hs = NXInitHashState(classes);
    while (NXNextHashState(classes, &hs, (void**)&cls)) {
        //Alex注释: 将loadable表中的类移除
        remove_class_from_loadable_list(cls);
        //Alex注释: 将类、元类从已经初始化的全局静态表中移除
        detach_class(cls->ISA(), YES);
        detach_class(cls, NO);
    }
    hs = NXInitHashState(classes);
    while (NXNextHashState(classes, &hs, (void**)&cls)) {
        //Alex注释: 释放类和元类指针
        free_class(cls->ISA());
        free_class(cls);
    }
    //Alex注释: 释放hashTable
    NXFreeHashTable(classes);
}
```


### 四、总结
   
 这次源码阅读学习到了类是如何从共享缓存、读取、转化、以及初始化这样一个装载过程,学到了类、方法、协议等在内存中存储的形式以及类元类超类之间的关系。在load_images模块也证实了之前关于category和class中+load方法的认知。
 未来可期，加油少年！！








