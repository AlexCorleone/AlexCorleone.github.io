## Weak的实现及细节分析


### 一、Weak实现源码(关键词: `SideTable`，`weak_table_t`，`weak_entry_t`)

在 **Runtime** 的源码NSObject.mm文件内，可以找到关于 `SideTable` 的定义；去掉内部关于 `spinlock_t` 的一些方法，`SideTable` 的结构定义如下:

```
struct SideTable {
    spinlock_t slock;//操作SideTable时的锁
    RefcountMap refcnts;
    weak_table_t weak_table;//存放(hash(对象地址))->(weak_entry_t)映射的的弱引用表(hash表)

    SideTable() {//构造函数
        memset(&weak_table, 0, sizeof(weak_table));
    }

    ~SideTable() {//析构函数
        _objc_fatal("Do not delete SideTable.");
    }
}
```

`weak_table_t` 结构体定义在objc-weak.h文件内，其内部是定义了一个 `weak_entry_t` 的结构体指针数组以及操作结构体数组需要用到的变量，`weak_table_t` 的结构定义如下:

```
struct weak_table_t {
    weak_entry_t *weak_entries;//weak_entry_t结构体类型的数组指针
    size_t    num_entries;//当前entries的数量
    uintptr_t mask;//最大可存储entry的数量
    uintptr_t max_hash_displacement;//最大的hash偏移位数
};
```

`weak_entry_t` 是 **对象地址hash值** 对应的实体，其内部包含了全部指向该对象的 **weak** 指针， `weak_entry_t` 的结构定义如下:

```
struct weak_entry_t {
    DisguisedPtr<objc_object> referent;//保存weak指向的对象
    union {
        struct {
        weak_referrer_t *referrers;//referrers存放的是对象的全部Weak指针
        uintptr_t        out_of_line_ness : 2;
        uintptr_t        num_refs : PTR_MINUS_2;
        uintptr_t        mask;
        uintptr_t        max_hash_displacement;
        };
        struct {
        weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    };
};
```

那通过上面的结构分析可以了解到 `SideTable`  -- (持有)>   `weak_table_t`    -- (持有)>    `weak_entry_t *`   -- (持有)>  `weak_referrer_t *` 这样一个关系链，那么 `SideTable` 是怎么创建的呢？它是被直接使用还是像 `weak_table_t` 直接被包裹一层呢？下面这段代码将揭晓答案。

```

//定义了一个SideTableBuf全局静态变量存储StripedMap类型的地址
alignas(StripedMap<SideTable>) static uint8_t SideTableBuf[sizeof(StripedMap<SideTable>)];

static void SideTableInit() {//初始化StripedMap，该map内部存储的是SideTable对象
    new (SideTableBuf) StripedMap<SideTable>();
}

static StripedMap<SideTable>& SideTables() {
    //reinterpret_cast是C++中与C风格类型转换最接近的类型转换运算符。它能够将一种对象类型转换为另一种，不管它们是否相关。
    return *reinterpret_cast<StripedMap<SideTable>*>(SideTableBuf);
}
```
通过上面代码可以看出，苹果是定义了一个SideTableBuf的全局静态变量，而苹果在SideTableInit()方法内部调用了StripedMap<SideTable>()的初始化方法，并将初始化的StripedMap对象地址保存在SideTableBuf全局静态变量内，最后苹果封装了一个方法SideTables()在内部调用了C++的类型强转换将SideTableBuf变量强制转换为<StripedMap<SideTable>*>类型的指针返回给调用者，去操作StripedMap<SideTable>。

关于为什么要这么绕去实现StripedMap的使用苹果也给出了官方解释:由于libc的调用早于C++的运行，所以不能直接使用C++的静态初始化方法；同时也不想使用全局的指针创造额外的开销，所以苹果采用了上述方式进行StripedMap的调用。

在objc-weak.h头文件内也可以看到如下几个方法，这些方法是真实的去处理 `weak_table_t` -> (  `object`  ->  `weak pointer` )之间的添加、删除、以及对象销毁时清除工作的外部接口； 关于这些方法的内部细节实现会在下面介绍，这里只需要记住他们的作用就好。

```
//添加一个(对象，weak指针)对到Weak table.
id weak_register_no_lock(weak_table_t *weak_table, id referent, id *referrer, bool crashIfDeallocating);

//从Weak table删除一个(对象，weak指针)对.
void weak_unregister_no_lock(weak_table_t *weak_table, id referent, id *referrer);

//在对象的销毁方法调用时，设置weak 指针 为nil，
void weak_clear_no_lock(weak_table_t *weak_table, id referent);
```

### 二、Weak对象的赋值分析

当属性或者成员变量是 **weak** 修饰的时候，赋值过程通常会调用下面的方法

```
id objc_initWeak(id _Nullable * _Nonnull location, id _Nullable val);

id objc_storeWeakOrNil(id _Nullable * _Nonnull location, id _Nullable obj);

id objc_initWeakOrNil(id _Nullable * _Nonnull location, id _Nullable val);

id objc_storeWeak(id _Nullable * _Nonnull location, id _Nullable obj);
```

而以上方法的内部是去调用  ```static id storeWeak(id *location, objc_object *newObj)``` 方法，关于 `storeWeak` 方法的实现细节如下：

```
static id storeWeak(id *location, objc_object *newObj)
{
    //如果存储的新对象为nil，触发assert
    if (!haveNew) assert(newObj == nil);

    Class previouslyInitializedClass = nil;
    id oldObj;
    SideTable *oldTable;
    SideTable *newTable;

    retry:
    if (haveOld) {
        //如果有旧值，通过oldObj取出对应的SideTable(oldTable)
        oldObj = *location;
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil;
    }
    if (haveNew) {
    //如果新值之前已经被weak引用(已经创建过对应的SideTable)，通过newObj取出对应的SideTable(newTable)
        newTable = &SideTables()[newObj];
    } else {
        newTable = nil;
    }

    SideTable::lockTwo<haveOld, haveNew>(oldTable, newTable);

    if (haveOld  &&  *location != oldObj) {
        SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
        goto retry;
    }

    if (haveNew  &&  newObj) {
        Class cls = newObj->getIsa();
        if (cls != previouslyInitializedClass  &&  
        !((objc_class *)cls)->isInitialized()) 
        {
            SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
            _class_initialize(_class_getNonMetaClass(cls, (id)newObj));
            previouslyInitializedClass = cls;

            goto retry;
        }
    }

    // 如果有旧值清除旧值
    if (haveOld) {
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }

    // 如果有新值将新值加入到weak_table
    if (haveNew) {
        newObj = (objc_object *)
        weak_register_no_lock(&newTable->weak_table, (id)newObj, location, 
        crashIfDeallocating);

        if (newObj  &&  !newObj->isTaggedPointer()) {
            newObj->setWeaklyReferenced_nolock();
        }
        *location = (id)newObj;
    }
    else {
        // 没有新值，weak_tableb没有变化
    }

    SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);

    return (id)newObj;
}
```

storeWeak方法核心是调用了  ```weak_unregister_no_lock(weak_table_t *weak_table, id referent_id, id *referrer_id)``` 和 ```weak_register_no_lock(weak_table_t *weak_table, id referent_id, id *referrer_id, bool crashIfDeallocating)``` 方法分别用于清除旧值、存储新值。

`weak_unregister_no_lock` 方法的具体实现如下：

```
void weak_unregister_no_lock(weak_table_t *weak_table, id referent_id, id *referrer_id)
{
    objc_object *referent = (objc_object *)referent_id;//weak对象指针
    objc_object **referrer = (objc_object **)referrer_id;//指向对象的weak指针

    weak_entry_t *entry;

    if (!referent) return;

    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        //如果Entry已经存在，移除weak_entry_t内部的特定指针(referrer)
        remove_referrer(entry, referrer);
        bool empty = true;
        if (entry->out_of_line()  &&  entry->num_refs != 0) {
            empty = false;
        }
        else {
            for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
                if (entry->inline_referrers[i]) {
                empty = false; 
                break;
                }
            }
        }
        //如果weak_entry_t内部的指针数组为空，将Entry移除
        if (empty) {
            weak_entry_remove(weak_table, entry);
        }
    }
}
```

`weak_register_no_lock` 方法的具体实现如下：

```
id weak_register_no_lock(weak_table_t *weak_table, id referent_id, 
id *referrer_id, bool crashIfDeallocating)
{
    objc_object *referent = (objc_object *)referent_id;//weak对象指针
    objc_object **referrer = (objc_object **)referrer_id;//指向对象的weak指针

    if (!referent  ||  referent->isTaggedPointer()) return referent_id;

    // ensure that the referenced object is viable
    bool deallocating;
    if (!referent->ISA()->hasCustomRR()) {
        deallocating = referent->rootIsDeallocating();
    }
    else {
        BOOL (*allowsWeakReference)(objc_object *, SEL) = 
        (BOOL(*)(objc_object *, SEL))
        object_getMethodImplementation((id)referent, 
        SEL_allowsWeakReference);
        if ((IMP)allowsWeakReference == _objc_msgForward) {
        return nil;
        }
        deallocating =
        ! (*allowsWeakReference)(referent, SEL_allowsWeakReference);
    }

    if (deallocating) {
        if (crashIfDeallocating) {
            _objc_fatal("Cannot form weak reference to instance (%p) of "
            "class %s. It is possible that this object was "
            "over-released, or is in the process of deallocation.",
            (void*)referent, object_getClassName((id)referent));
        } else {
            return nil;
        }
    }

    weak_entry_t *entry;
    //如果referent对应的weak_entry_t存在，将weak指针附加到weak_entry_t的指针数据上。
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        append_referrer(entry, referrer);
    } 
    else {
        //如果referent对应的weak_entry_t不存在，将
        //1、创建新的weak_entry_t对象，
        //2、查询weak_table是否需要扩容,
        //3、将weak_entry_t插入weak_table内。
        weak_entry_t new_entry(referent, referrer);
        weak_grow_maybe(weak_table);
        weak_entry_insert(weak_table, &new_entry);
    }
    return referent_id;
}
```

### 三、Weak实现的内部功能函数细节分析


weak_entry_insert方法负责将一个entry对象插入weak_table内部的weak_entries数组，实现如下:

```
static void weak_entry_insert(weak_table_t *weak_table, weak_entry_t *new_entry)
{
    //1.获取weak_entry_t数组
    weak_entry_t *weak_entries = weak_table->weak_entries;
    assert(weak_entries != nil);
    //2.通过对象地址的hash获取开始查询的下标
    size_t begin = hash_pointer(new_entry->referent) & (weak_table->mask);
    size_t index = begin;
    size_t hash_displacement = 0;//记录当前hash偏移个数
    //3.从index遍历weak_entries
    while (weak_entries[index].referent != nil) {
        index = (index+1) & weak_table->mask;//保证index在mask范围内(mask是weak_entries的最大存储个数)
        if (index == begin) bad_weak_table(weak_entries);//遍历全部没有找到可以插入new_entry的下标，出现BUG
        hash_displacement++;//hash偏移个数+1
    }
    //4.找到index位，将new_entry存储到index标记位,
    weak_entries[index] = *new_entry;
    weak_table->num_entries++;//entry数量+1

    if (hash_displacement > weak_table->max_hash_displacement) {
    //如果当前的hash偏移个数大于max_hash_displacement,更新最大hash偏移数
        weak_table->max_hash_displacement = hash_displacement;
    }
}
```

weak_entry_remove方法负责将weak_entry_t对象从weak_table_t移除，实现如下：

```
static void weak_entry_remove(weak_table_t *weak_table, weak_entry_t *entry)
{   
    //1.释放referrers空间，释放entry占用内存
    if (entry->out_of_line()) free(entry->referrers);
    bzero(entry, sizeof(*entry));
    
    //2.entry数量-1
    weak_table->num_entries--;
    //3.重新调整weak_table
    weak_compact_maybe(weak_table);
}
```

当weak_entries大小不足时，weak_resize负责weak_entries的扩容，实现如下：

```
static void weak_resize(weak_table_t *weak_table, size_t new_size)
{
    size_t old_size = TABLE_SIZE(weak_table);
    //1.获取旧的entries数据
    weak_entry_t *old_entries = weak_table->weak_entries;
    //2.开辟new_size的新空间
    weak_entry_t *new_entries = (weak_entry_t *)
    calloc(new_size, sizeof(weak_entry_t));
    //3.设置weak_entries及操作weak_entries时的标记
    weak_table->mask = new_size - 1;
    weak_table->weak_entries = new_entries;
    weak_table->max_hash_displacement = 0;
    weak_table->num_entries = 0;  // restored by weak_entry_insert below

    if (old_entries) {
    //4.如果有旧的entries数据遍历插入到新的weak_table->weak_entries数组中
        weak_entry_t *entry;
        weak_entry_t *end = old_entries + old_size;
        for (entry = old_entries; entry < end; entry++) {
            if (entry->referent) {
                weak_entry_insert(weak_table, entry);
            }
        }
        //5.释放old_entries占用的空间
        free(old_entries);
    }
}

```

将weak指针添加到对应的weak_entry_t对象，实现如下：

```
static void append_referrer(weak_entry_t *entry, objc_object **new_referrer)
{
    if (! entry->out_of_line()) {
        for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
            if (entry->inline_referrers[i] == nil) {
                entry->inline_referrers[i] = new_referrer;
                return;
            }
        }

        weak_referrer_t *new_referrers = (weak_referrer_t *)
        calloc(WEAK_INLINE_COUNT, sizeof(weak_referrer_t));
        for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
        new_referrers[i] = entry->inline_referrers[i];
        }
        entry->referrers = new_referrers;
        entry->num_refs = WEAK_INLINE_COUNT;
        entry->out_of_line_ness = REFERRERS_OUT_OF_LINE;
        entry->mask = WEAK_INLINE_COUNT-1;
        entry->max_hash_displacement = 0;
    }

    if (entry->num_refs >= TABLE_SIZE(entry) * 3/4) {
        return grow_refs_and_insert(entry, new_referrer);
    }
    size_t begin = w_hash_pointer(new_referrer) & (entry->mask);
    size_t index = begin;
    size_t hash_displacement = 0;
    while (entry->referrers[index] != nil) {
        hash_displacement++;
        index = (index+1) & entry->mask;
        if (index == begin) bad_weak_table(entry);
    }
    if (hash_displacement > entry->max_hash_displacement) {
        entry->max_hash_displacement = hash_displacement;
    }
    weak_referrer_t &ref = entry->referrers[index];
    ref = new_referrer;
    entry->num_refs++;
}
```

从给定的weak_entry_t移除weak指针，实现如下：

```
static void remove_referrer(weak_entry_t *entry, objc_object **old_referrer)
{
    if (! entry->out_of_line()) {
        for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
            if (entry->inline_referrers[i] == old_referrer) {
                entry->inline_referrers[i] = nil;
                return;
            }
        }
        _objc_inform("Attempted to unregister unknown __weak variable "
        "at %p. This is probably incorrect use of "
        "objc_storeWeak() and objc_loadWeak(). "
        "Break on objc_weak_error to debug.\n", 
        old_referrer);
        objc_weak_error();
        return;
    }

    size_t begin = w_hash_pointer(old_referrer) & (entry->mask);
    size_t index = begin;
    size_t hash_displacement = 0;
    while (entry->referrers[index] != old_referrer) {
        index = (index+1) & entry->mask;
        if (index == begin) bad_weak_table(entry);
        hash_displacement++;
        if (hash_displacement > entry->max_hash_displacement) {
            _objc_inform("Attempted to unregister unknown __weak variable "
            "at %p. This is probably incorrect use of "
            "objc_storeWeak() and objc_loadWeak(). "
            "Break on objc_weak_error to debug.\n", 
            old_referrer);
            objc_weak_error();
            return;
        }
    }
    entry->referrers[index] = nil;
    entry->num_refs--;
}
```

weak实现中hash的实现如下：

```
#if __LP64__
    static inline uint32_t ptr_hash(uint64_t key)
    {
        key ^= key >> 4; //key左移三位异或key
        key *= 0x8a970be7488fda55; 
        key ^= __builtin_bswap64(key);//key 调用__builtin_bswap64()字节翻转 异或上当前key，如 10011011 00111101 11010010  翻转后 11011001 10111100 01001011
        return (uint32_t)key;
    }
#else
    static inline uint32_t ptr_hash(uint32_t key)
    {
        key ^= key >> 4;
        key *= 0x5052acdb;
        key ^= __builtin_bswap32(key);
        return key;
    }
#endif

```

### 四、Weak对象的释放

在NSObject的 ```- (void)dealloc;``` 方法内部苹果通过调用  ```_objc_rootDealloc(self);```  来释放对象内存，而  ```void _objc_rootDealloc(id obj)``` 方法内部又调用了obj的  ```inline void objc_object::rootDealloc()```  方法，该方法内部接着调用了  ```id object_dispose(id obj)``` 方法，```object_dispose``` 方法通过调用 ```void *objc_destructInstance(id obj)``` 完成对象的销毁，关于 ```objc_destructInstance``` 的方法实现如下：

```
void *objc_destructInstance(id obj) 
{
    if (obj) {
        bool cxx = obj->hasCxxDtor();//是否有C++的析构
        bool assoc = obj->hasAssociatedObjects();//是否有对象关联属性

        if (cxx) object_cxxDestruct(obj);//调用C++析构函数
        if (assoc) _object_remove_assocations(obj);//移除对象关联属性
        obj->clearDeallocating();//清除weak对象
    }
    return obj;
}
```

在  ```inline void objc_object::clearDeallocating()```  方法内部调用  ```void objc_object::sidetable_clearDeallocating()``` ，而 ```sidetable_clearDeallocating``` 方法将具体的清除工作交给了  ```void weak_clear_no_lock(weak_table_t *weak_table, id referent_id)```；```weak_clear_no_lock``` 方法的实现如下：

```
void weak_clear_no_lock(weak_table_t *weak_table, id referent_id) 
{
    objc_object *referent = (objc_object *)referent_id;
    //1.查询对象是否有对应的entry实体
    weak_entry_t *entry = weak_entry_for_referent(weak_table, referent);
    if (entry == nil) {
    //如果Entry不存在，直接返回
        return;
    }

    weak_referrer_t *referrers;
    size_t count;
    //2.取出weak指针数组及数组的count
    if (entry->out_of_line()) {
        referrers = entry->referrers;
        count = TABLE_SIZE(entry);
    } 
    else {
        referrers = entry->inline_referrers;
        count = WEAK_INLINE_COUNT;
    }
    //3.循环遍历weak指针数组，将weak指针置nil
    for (size_t i = 0; i < count; ++i) {
        objc_object **referrer = referrers[i];
        if (referrer) {
            if (*referrer == referent) {
                *referrer = nil;
            }
            else if (*referrer) {
                _objc_inform("__weak variable at %p holds %p instead of %p. "
                "This is probably incorrect use of "
                "objc_storeWeak() and objc_loadWeak(). "
                "Break on objc_weak_error to debug.\n", 
                referrer, (void*)*referrer, (void*)referent);
                objc_weak_error();
            }
        }
    }
    //4.调用移除entry的方法将entry移除
    weak_entry_remove(weak_table, entry);
}
```

### 五、总结
weak原理实现相对来说并不复杂，苹果通过一个全局的静态StripedMap存储`SideTable`，通过对象可以查找到对应的`SideTable`，而`SideTable`内部维护了一个`weak_table_t`表,然后通过Hash的形式将对象与`weak_entry_t`形成映射；而对象所有的weak指针存储到`weak_entry_t`内部的referrers结构体数组中，这样在对象被dealloc时便可以通过对象Hash值来找到对应的`weak_entry_t`将指向该对象的Weak指针指向nil。
继续加油！！！！

