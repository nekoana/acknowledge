> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7047680282463305735)

本文简要介绍 Android Runtime 虚拟机里的一些细节点，主要包括 dex file, oat file, mirror::Class, ArtField, ArtMethod, DexCache, ClassTable 等。

了解这些细节，在后面学习类查找等原理时会轻松很多，所以先讲一下。

dex2oat 触发场景
------------

[cs.android.com/android/pla…](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fmaster%3Aart%2Fdex2oat%2Fdex2oat.cc "https://cs.android.com/android/platform/superproject/+/master:art/dex2oat/dex2oat.cc")

dex2oat 的作用：对 dex 文件进行编译，根据参数，生成 oat vdex art 文件。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a6f23b4495842738d6558f3e0a637de~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image) ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6def6d8fad584857a2c9067e45be9e20~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

各种文件
----

### .dex

[cs.android.com/android/pla…](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fmaster%3Aart%2Flibdexfile%2Fdex%2Fdex_file.h "https://cs.android.com/android/platform/superproject/+/master:art/libdexfile/dex/dex_file.h")

主要看下 class_def，class_def 代表的是类的基本信息，关键内容：

*   class_idx/superclass_idx：string_id 的索引，类名字符串
    
*   interfaces_off：数组，对应的是实现的接口类型 id
    
    *   type_list -> type_item -> type_idx
*   class_data_off：所有成员变量和成员函数信息
    
    *   class_data_item
        *   定义、继承和实现的函数
            *   direct_methods
                *   static, private, constructor
            *   virtual_methods
                *   除了 direct_methods 以外的
*   code_item 是什么？
    
    *   code_item 存储的是 dex 中的字节码，用解释器来执行

DexFile:

```
DexFile(const uint8_t* base,
          size_t size,
          const uint8_t* data_begin,
          size_t data_size,
          const std::string& location,
          uint32_t location_checksum,
          const OatDexFile* oat_dex_file,
          std::unique_ptr<DexFileContainer> container,
          bool is_compact_dex);
  const Header* const header_;
  const dex::StringId* const string_ids_;
  const dex::TypeId* const type_ids_;
  const dex::FieldId* const field_ids_;
  const dex::MethodId* const method_ids_;
  const dex::ProtoId* const proto_ids_;
  const dex::ClassDef* const class_defs_;

  // If this dex file was loaded from an oat file, oat_dex_file_ contains a
  // pointer to the OatDexFile it was loaded from. Otherwise oat_dex_file_ is
  // null.
  mutable const OatDexFile* oat_dex_file_;
};
复制代码
```

如果该 dex 是从一个 oat 文件里获取的，DexFile 中还包括一个 oat_dex_file 的指针，指向对于的 oat file。后面 loadClass 时会用到这个指针。

Dex 文件里保存的是符号引用，需要经过一次解析才能拿到最终信息，比如获取类的名称，需要通过 string_id 去 string_data 里找一下才知道。

DexCache 的存在就是为了避免重复解析。

### .odex

DVM 上使用。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/947da63f65744e56a880ebfa582e7fe5~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

.odex 在 dex 文件前增加了 header 信息，后面增加了其他 dex 的依赖和一些辅助信息。

### .oat

[cs.android.com/android/pla…](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fmaster%3Aart%2Fruntime%2Foat_file.h "https://cs.android.com/android/platform/superproject/+/master:art/runtime/oat_file.h")

ART 上使用。

Oat 文件是一种特殊的 ELF 文件格式，它包含 dex 文件编译得到的机器指令，在 8.0 以下包括原始的 dex 内容，8.0 之后 raw dex 在 quicken 化之后是在 .vdex 里。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0b1bbabec8c34a4881bceb4d99dc2361~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

*   oat data section 对应的是 dex 文件相关信息（8.0 之后在 .vdex 文件中）
*   oat exec section 对应的是 dex 编译生成的机器指令

### .vdex

[cs.android.com/android/pla…](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fmaster%3Aart%2Fruntime%2Fvdex_file.h "https://cs.android.com/android/platform/superproject/+/master:art/runtime/vdex_file.h") ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bc6730f15be44fe1bd0bf514758f704e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

*   VerifierDeps 用于快速校验 dex 里 method 合法性

8.0 增加，目的是减少 dex2oat 时间（[android-review.googlesource.com/c/platform/…](https://link.juejin.cn?target=https%3A%2F%2Fandroid-review.googlesource.com%2Fc%2Fplatform%2Fart%2F%2B%2F264514%25EF%25BC%2589%25E3%2580%2582 "https://android-review.googlesource.com/c/platform/art/+/264514%EF%BC%89%E3%80%82")

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cc7cee94aa1f493abac3e751e487fd0d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

> 从 vdex 里提取 dex:[github.com/anestisb/vd…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fanestisb%2FvdexExtractor "https://github.com/anestisb/vdexExtractor")

dex2oat::Setup():

```
// No need to verify the dex file when we have a vdex file, which means it was already
        // verified.
        const bool verify =
            (input_vdex_file_ == nullptr) && !compiler_options_->AssumeDexFilesAreVerified();
        if (!oat_writers_[i]->WriteAndOpenDexFiles(
            vdex_files_[i].get(),
            verify,
            update_input_vdex_,
            copy_dex_files_,
            &opened_dex_files_map,
            &opened_dex_files)) {
          return dex2oat::ReturnCode::kOther;
        }
复制代码
```

如果之前做过 dex2oat，有 vdex 文件，下次执行 dex2oat 时（比如系统 OTA）就可以省去重新 verify dex 的过程。

类信息
---

### mirror::Class

[cs.android.com/android/pla…](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fmaster%3Aart%2Fruntime%2Fmirror%2Fclass.h "https://cs.android.com/android/platform/superproject/+/master:art/runtime/mirror/class.h")

```
// C++ mirror of java.lang.Class
class MANAGED Class final : public Object {
  // Defining class loader, or null for the "bootstrap" system loader.
  HeapReference<ClassLoader> class_loader_;

  // 数组元素的类型
  // (for String[][][], this will be String[][]). null for non-array classes.
  HeapReference<Class> component_type_;

  // 这个类对应的 DexCache 对象，虚拟机直接创建的类没有这个值（数组、基本类型）
  HeapReference<DexCache> dex_cache_;

  //接口表，包括自己实现的和继承的
  HeapReference<IfTable> iftable_;

  // 类名，"java.lang.Class" or "[C"
  HeapReference<String> name_;

  HeapReference<Class> super_class_;

  //虚函数表，invoke-virtual 调用的函数，包括父类的和当前类的
  HeapReference<PointerArray> vtable_;

  //本类定义的非静态成员，不包括父类的。
  uint64_t ifields_;

  /* [0,virtual_methods_offset_):本类的direct函数
     [virtual_methods_offset_,copied_methods_offset_):本类的virtual函数
     [copied_methods_offset_, ...) 诸如miranda函数等  */
  uint64_t methods_;

  // Static fields length-prefixed array.
  uint64_t sfields_;

  uint32_t access_flags_;
  uint32_t class_flags_;

  // Total size of the Class instance; used when allocating storage on gc heap
  uint32_t class_size_;

  // Tid used to check for recursive <clinit> invocation.
  pid_t clinit_thread_id_;
  static_assert(sizeof(pid_t) == sizeof(int32_t), "java.lang.Class.clinitThreadId size check");

  // ClassDef index in dex file, -1 if no class definition such as an array.
  int32_t dex_class_def_idx_;

  // Type index in dex file.
  int32_t dex_type_idx_;
};
复制代码
```

Class 成员变量比较多，重点关注这几个：

*   iftable_:
    *   保存该类直接实现或间接实现（继承）的接口信息
    *   接口信息包含两个部分
        *   接口类所对应的 Class 对象
        *   该接口类中的方法。
*   vtable_:
    *   保存该类直接定义或间接定义的 virtual 方法
    *   比如 Object 类中的 wait、notify、toString 等方法
*   methods_:
    *   只包含本类直接定义的 direct、virtual 方法和 Miranda 方法
    *   一般 vtable_ 包含内容会多于 methods_
*   sfields_ 静态变量
*   ifields_ 实例变量
    *   ClassLinker::LoadClass 阶段分配内存和设置数据

### ArtField

[cs.android.com/android/pla…](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fmaster%3Aart%2Fruntime%2Fart_field.h "https://cs.android.com/android/platform/superproject/+/master:art/runtime/art_field.h")

```
class ArtField {
  GcRoot<mirror::Class> declaring_class_;
  uint32_t access_flags_ = 0;

  // 在 dex 中 field_ids 数组中的索引
  uint32_t field_dex_idx_ = 0；
 //成员变量的offset  
  uint32_t offset_ = 0;
}
复制代码
```

一个 ArtField 对象代表类中的一个成员变量。

offset_ 含义：

*   如果是静态成员变量，offset_ 代表变量的存储空间在 **Class 对象**的内存布局里的起始位置
*   如果是非静态成员变量，offset_ 代表在 **Object 对象**的内存布局里的起始位置

> 手动水印：[shixin.blog.csdn.net/](https://link.juejin.cn?target=https%3A%2F%2Fshixin.blog.csdn.net%2F "https://shixin.blog.csdn.net/")

### ArtMethod

[cs.android.com/android/pla…](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fmaster%3Aart%2Fruntime%2Fart_method.h "https://cs.android.com/android/platform/superproject/+/master:art/runtime/art_method.h")

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/56b8f60966174bc89e4def383b36f29d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

ArtMethod 代表一个运行在 Android Runtime 中的 Java 侧的方法，主要结构：

```
class ArtMethod {

 protected:
  GcRoot<mirror::Class> declaring_class_;

  std::atomic<std::uint32_t> access_flags_;

  //在 dex file 中的位置
  // Offset to the CodeItem. 
  uint32_t dex_code_item_offset_;
  //在 dex 中 method_id 的 index，通过它获取名称等信息
  uint32_t dex_method_index_;

  /* End of dex file fields. */

  // static/direct method -> declaringClass.directMethods
  // virtual method -> vtable
  // interface method -> ifTable
  uint16_t method_index_;

  // 调用一次加一，超过阈值可能会被编译成本地方法
  uint16_t hotness_count_;

  // Fake padding field gets inserted here.

  // Must be the last fields in the method.
  struct PtrSizedFields {
    //方法入口地址
    void* entry_point_from_quick_compiled_code_;
  } ptr_sized_fields_;
}
复制代码
```

这个 entry_point 是在 ClassLinker#LinkCode 时设置的入口，后面执行这个方法时，不论是解释执行还是以本地机器指令执行，都通过 ArtMethod 的 GetEntryPointFromCompiledCode 获取入口点。

缓存
--

### ClassTable

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/21a6fd66964b4ffa96c90ecf94fdb174~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image) 每个 ClassLoader 有一个 class_table_，它的成员主要是一个 ClassSet vector：

```
ClassTable:
  // Lock to guard inserting and removing.
  mutable ReaderWriterMutex lock_;
  // We have a vector to help prevent dirty pages after the zygote forks by calling FreezeSnapshot.
  std::vector<ClassSet> classes_ GUARDED_BY(lock_);

  // Hash set that hashes class descriptor, and compares descriptors and class loaders. Results
  // should be compared for a matching class descriptor and class loader.
  typedef HashSet<TableSlot,
                  TableSlotEmptyFn,
                  ClassDescriptorHashEquals,
                  ClassDescriptorHashEquals,
                  TrackingAllocator<TableSlot, kAllocatorTagClassTable>> ClassSet;
复制代码
```

通过 ClassLinker::InsertClass 插入到 ClassTable 中

*   ClassLinker::InsertClassTableForClassLoader
    *   ClassLinker::RegisterClassLoader 创建 ClassTable

```
void ClassLinker::RegisterClassLoader(ObjPtr<mirror::ClassLoader> class_loader) {
  CHECK(class_loader->GetAllocator() == nullptr);
  CHECK(class_loader->GetClassTable() == nullptr);
  Thread* const self = Thread::Current();
  ClassLoaderData data;
  data.weak_root = self->GetJniEnv()->GetVm()->AddWeakGlobalRef(self, class_loader);
  // Create and set the class table.
  data.class_table = new ClassTable;
  class_loader->SetClassTable(data.class_table);
  // Create and set the linear allocator.
  data.allocator = Runtime::Current()->CreateLinearAlloc();
  class_loader->SetAllocator(data.allocator);
  // Add to the list so that we know to free the data later.
  class_loaders_.push_back(data);
}
复制代码
```

调用处： ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8107b654287a4dadb1f63c532e78ca2d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image) FindClass 时会调用 LookupClass 查询：

```
ObjPtr<mirror::Class> ClassLinker::LookupClass(Thread* self,
                                               const char* descriptor,
                                               size_t hash,
                                               ObjPtr<mirror::ClassLoader> class_loader) {
  ReaderMutexLock mu(self, *Locks::classlinker_classes_lock_);
  ClassTable* const class_table = ClassTableForClassLoader(class_loader);
  if (class_table != nullptr) {
    ObjPtr<mirror::Class> result = class_table->Lookup(descriptor, hash);
    if (result != nullptr) {
      return result;
    }
  }
  return nullptr;
}

复制代码
```

### DexCache

[cs.android.com/android/pla…](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fmaster%3Aart%2Fruntime%2Fmirror%2Fdex_cache.h "https://cs.android.com/android/platform/superproject/+/master:art/runtime/mirror/dex_cache.h")

DexCache 保存的是一个 Dex 里解析后的成员变量、方法、类型、字符串信息。

```
// C++ mirror of java.lang.DexCache.
class MANAGED DexCache final : public Object {
  HeapReference<ClassLoader> class_loader_;
  // 对应的 dex 文件路径
  HeapReference<String> location_;

  uint64_t dex_file_;                // const DexFile*
  uint64_t preresolved_strings_;     // GcRoot<mirror::String*> array
                                    
  uint64_t resolved_call_sites_;     // GcRoot<CallSite>* array
  
  //field_idx                               
  uint64_t resolved_fields_;         // std::atomic<FieldDexCachePair>*
  uint64_t resolved_method_types_;   // std::atomic<MethodTypeDexCachePair>*
  uint64_t resolved_methods_;        // ArtMethod*,
  uint64_t resolved_types_;          // TypeDexCacheType*
  uint64_t strings_;                 // std::atomic<StringDexCachePair>*

  uint32_t num_preresolved_strings_;   
  uint32_t num_resolved_call_sites_;   
  uint32_t num_resolved_fields_;       
  uint32_t num_resolved_method_types_;  
  uint32_t num_resolved_methods_;      
  uint32_t num_resolved_types_;      
  uint32_t num_strings_;               

}
复制代码
```

什么时候创建和读取呢？

*   在 ART 中每当一个类被加载时，ART 运行时都会检查该类所属的 DEX 文件是否已经关联有一个 Dex Cache。如果还没有关联，那么就会创建一个 Dex Cache，并且建立好关联关系。

DefineClass:

```
ObjPtr<mirror::DexCache> dex_cache = RegisterDexFile(*new_dex_file, class_loader.Get());
  if (dex_cache == nullptr) {
    self->AssertPendingException();
    return sdc.Finish(nullptr);
  }
  klass->SetDexCache(dex_cache);
复制代码
```