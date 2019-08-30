
## Weak的实现及相关数据结构分析

### Weak实现源码

在Runtime的源码文件NSObject.mm内我们可以找到关于SideTable的定义，SideTable只是对weak_table_t的封装、苹果也是通过SideTable来操作内部的weak_table_t对象的。

```
struct SideTable {
spinlock_t slock;
RefcountMap refcnts;
/*Alex注释:
weak对象指针存储表实现
*/
weak_table_t weak_table;

SideTable() {
memset(&weak_table, 0, sizeof(weak_table));
}

~SideTable() {
_objc_fatal("Do not delete SideTable.");
}

void lock() { slock.lock(); }
void unlock() { slock.unlock(); }
void forceReset() { slock.forceReset(); }

// Address-ordered lock discipline for a pair of side tables.

template<HaveOld, HaveNew>
static void lockTwo(SideTable *lock1, SideTable *lock2);
template<HaveOld, HaveNew>
static void unlockTwo(SideTable *lock1, SideTable *lock2);
};

```

SideTable的创建

```

alignas(StripedMap<SideTable>) static uint8_t 
SideTableBuf[sizeof(StripedMap<SideTable>)];

static void SideTableInit() {
new (SideTableBuf) StripedMap<SideTable>();
}
/*Alex注释:
这里 SideTables()全局保存对象和 SideTable之间的映射,对象地址的hash是StripedMap中array的index。
*/
static StripedMap<SideTable>& SideTables() {
/*Alex注释:
reinterpret_cast是C++中与C风格类型转换最接近的类型转换运算符。它让程序员能够将一种对象类型转换为另一种，不管它们是否相关。
*/
return *reinterpret_cast<StripedMap<SideTable>*>(SideTableBuf);
}

// anonymous namespace
};

```

weak_table_t的定义可以在objc-weak.h内找到。

```
/*Alex注释:
weak_entries 存放weak_entry_t结构体对象的数组指针
num_entries  当前entries的数量
mask 最大可存储 entry的数量
max_hash_displacement 最大的hash偏移位数,
*/
struct weak_table_t {
weak_entry_t *weak_entries;
size_t    num_entries;
uintptr_t mask;
uintptr_t max_hash_displacement;
};

```

可以看到weak_table_t内部主要存储的是一个weak_entries的指针数组(其实是一个hash table)，关于weak_entries的结构定义如下:

```
struct weak_entry_t {
DisguisedPtr<objc_object> referent;
union {
struct {
weak_referrer_t *referrers;
uintptr_t        out_of_line_ness : 2;
uintptr_t        num_refs : PTR_MINUS_2;
uintptr_t        mask;
uintptr_t        max_hash_displacement;
};
struct {
// out_of_line_ness field is low bits of inline_referrers[1]
weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
};
};

bool out_of_line() {
return (out_of_line_ness == REFERRERS_OUT_OF_LINE);
}

weak_entry_t& operator=(const weak_entry_t& other) {
memcpy(this, &other, sizeof(other));
return *this;
}

weak_entry_t(objc_object *newReferent, objc_object **newReferrer)
: referent(newReferent)
{
inline_referrers[0] = newReferrer;
for (int i = 1; i < WEAK_INLINE_COUNT; i++) {
inline_referrers[i] = nil;
}
}
};

```

weak_table_t操作相关的方法也在objc-weak.h内可以看到，主要有下面几个方法，这些方法会在Weak对象的操作过程中介绍。

```
/// Adds an (object, weak pointer) pair to the weak table.
id weak_register_no_lock(weak_table_t *weak_table, id referent, 
id *referrer, bool crashIfDeallocating);

/// Removes an (object, weak pointer) pair from the weak table.
void weak_unregister_no_lock(weak_table_t *weak_table, id referent, id *referrer);

/// Called on object destruction. Sets all remaining weak pointers to nil.
void weak_clear_no_lock(weak_table_t *weak_table, id referent);

```

### Weak对象的操作过程

当属性或者成员变量是weak修饰的时候，赋值过程通常会调用下面的方法

```
OBJC_EXPORT id _Nullable 
objc_initWeak(id _Nullable * _Nonnull location, id _Nullable val)
OBJC_AVAILABLE(10.7, 5.0, 9.0, 1.0, 2.0);

// Like objc_storeWeak, but stores nil if the new object is deallocating 
// or the new object's class does not support weak references.
// Returns the value stored (either the new object or nil).
OBJC_EXPORT id _Nullable
objc_storeWeakOrNil(id _Nullable * _Nonnull location, id _Nullable obj)
OBJC_AVAILABLE(10.11, 9.0, 9.0, 1.0, 2.0);

// Like objc_initWeak, but stores nil if the new object is deallocating 
// or the new object's class does not support weak references.
// Returns the value stored (either the new object or nil).
OBJC_EXPORT id _Nullable
objc_initWeakOrNil(id _Nullable * _Nonnull location, id _Nullable val) 
OBJC_AVAILABLE(10.11, 9.0, 9.0, 1.0, 2.0);

```

而以上方法的内部实现是去调用  ` static id storeWeak(id *location, objc_object *newObj) ` 方法，关于storeWeak方法的实现细节如下：

```
static id 
storeWeak(id *location, objc_object *newObj)
{
assert(haveOld  ||  haveNew);
if (!haveNew) assert(newObj == nil);

Class previouslyInitializedClass = nil;
id oldObj;
SideTable *oldTable;
SideTable *newTable;

// Acquire locks for old and new values.
// Order by lock address to prevent lock ordering problems. 
// Retry if the old value changes underneath us.
retry:
if (haveOld) {
oldObj = *location;
oldTable = &SideTables()[oldObj];
} else {
oldTable = nil;
}
if (haveNew) {
newTable = &SideTables()[newObj];
} else {
newTable = nil;
}

SideTable::lockTwo<haveOld, haveNew>(oldTable, newTable);

if (haveOld  &&  *location != oldObj) {
SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
goto retry;
}

// Prevent a deadlock between the weak reference machinery
// and the +initialize machinery by ensuring that no 
// weakly-referenced object has an un-+initialized isa.
if (haveNew  &&  newObj) {
Class cls = newObj->getIsa();
if (cls != previouslyInitializedClass  &&  
!((objc_class *)cls)->isInitialized()) 
{
SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
_class_initialize(_class_getNonMetaClass(cls, (id)newObj));

// If this class is finished with +initialize then we're good.
// If this class is still running +initialize on this thread 
// (i.e. +initialize called storeWeak on an instance of itself)
// then we may proceed but it will appear initializing and 
// not yet initialized to the check above.
// Instead set previouslyInitializedClass to recognize it on retry.
previouslyInitializedClass = cls;

goto retry;
}
}

// Clean up old value, if any.
if (haveOld) {
weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
}

// Assign new value, if any.
if (haveNew) {
newObj = (objc_object *)
weak_register_no_lock(&newTable->weak_table, (id)newObj, location, 
crashIfDeallocating);
// weak_register_no_lock returns nil if weak store should be rejected

// Set is-weakly-referenced bit in refcount table.
if (newObj  &&  !newObj->isTaggedPointer()) {
newObj->setWeaklyReferenced_nolock();
}

// Do not set *location anywhere else. That would introduce a race.
*location = (id)newObj;
}
else {
// No new value. The storage is not changed.
}

SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);

return (id)newObj;
}
```

storeWeak方法其核心是调用了 ` void weak_unregister_no_lock(weak_table_t *weak_table, id referent_id, id *referrer_id) ` 和 ` id weak_register_no_lock(weak_table_t *weak_table, id referent_id, id *referrer_id, bool crashIfDeallocating) ` 方法分别用于清理旧数据、存储新数据。



### Weak对象的释放

在NSObject的` - (void)dealloc; ` 方法背部 苹果通过调用 ` _objc_rootDealloc(self); `来释放对象关联的内存，而 ` void_objc_rootDealloc(id obj) ` 方法内部又调用了obj的 ` inline void objc_object::rootDealloc() ` 方法，该方法内部有调用了 ` id object_dispose(id obj) ` 方法，object_dispose方法通过调用 ` void *objc_destructInstance(id obj) ` 完成对象的销毁，关于objc_destructInstance的方法实现如下：

```
void *objc_destructInstance(id obj) 
{
if (obj) {
// Read all of the flags at once for performance.
bool cxx = obj->hasCxxDtor();
bool assoc = obj->hasAssociatedObjects();

// This order is important.
if (cxx) object_cxxDestruct(obj);//Alex注释 调用 C++析构函数
if (assoc) _object_remove_assocations(obj);//Alex注释 移除关联属性
obj->clearDeallocating();
}

return obj;
}

```

在 ` inline void objc_object::clearDeallocating() ` 方法内部调用 ` void objc_object::sidetable_clearDeallocating() `，而 sidetable_clearDeallocating 方法将具体的清除工作交给了 ` void weak_clear_no_lock(weak_table_t *weak_table, id referent_id) `；weak_clear_no_lock方法的实现如下：

```
void 
weak_clear_no_lock(weak_table_t *weak_table, id referent_id) 
{
objc_object *referent = (objc_object *)referent_id;

weak_entry_t *entry = weak_entry_for_referent(weak_table, referent);
if (entry == nil) {
/// XXX shouldn't happen, but does with mismatched CF/objc
//printf("XXX no entry for clear deallocating %p\n", referent);
return;
}

// zero out references
weak_referrer_t *referrers;
size_t count;

if (entry->out_of_line()) {
referrers = entry->referrers;
count = TABLE_SIZE(entry);
} 
else {
referrers = entry->inline_referrers;
count = WEAK_INLINE_COUNT;
}

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

weak_entry_remove(weak_table, entry);
}

```