# TVM-FFI 核心类型系统详解

> 本文档详细介绍 `3rdparty/tvm-ffi` 模块的核心数据类型和类层次结构。

## 目录

1. [概述](#概述)
2. [C API 层结构体](#c-api-层结构体)
3. [C++ API 层类](#c-api-层类)
4. [类型索引系统](#类型索引系统)
5. [完整类图](#完整类图)
6. [内存布局](#内存布局)
7. [类型特征系统](#类型特征系统)
8. [使用示例](#使用示例)

---

## 概述

TVM-FFI 是一个用于机器学习系统的开源 ABI 和 FFI（Foreign Function Interface）框架。其核心设计围绕两个层次：

- **C API 层** - 稳定的 C 结构体和函数，用于跨语言互操作
- **C++ API 层** - 面向对象的封装，提供类型安全和 RAII 管理

---

## C API 层结构体

### 核心结构体关系图

```mermaid
classDiagram
    direction TB
    
    %% ═══════════════════════════════════════════════════════════
    %% 对象头 - 所有堆分配对象的起始部分
    %% ═══════════════════════════════════════════════════════════
    class TVMFFIObject {
        «对象头 24字节»
        「所有堆对象以此开头」
        ────────────────────
        +uint64_t combined_ref_count
        「组合引用计数」
        「低32位=强引用, 高32位=弱引用」
        ────────────────────
        +int32_t type_index
        「运行时类型索引」
        ────────────────────
        +uint32_t __padding
        「对齐填充,必须为0」
        ────────────────────
        +void (*deleter)(void*, int)
        「析构函数指针」
    }
    
    %% ═══════════════════════════════════════════════════════════
    %% 通用值类型 - 16字节紧凑表示
    %% ═══════════════════════════════════════════════════════════
    class TVMFFIAny {
        «通用值 16字节»
        「可存储POD值或对象指针」
        ────────────────────
        +int32_t type_index
        「类型索引,标识值类型」
        ────────────────────
        +union padding_or_len
        「填充或小字符串长度」
        ────────────────────
        +union value
        「值联合体」
    }
    
    %% ═══════════════════════════════════════════════════════════
    %% 字节数组 - 字符串/字节的基础结构
    %% ═══════════════════════════════════════════════════════════
    class TVMFFIByteArray {
        «字符数组视图」
        「不拥有数据所有权」
        ────────────────────
        +const char* data
        「指向字符数据的指针」
        ────────────────────
        +size_t size
        「字符数组长度」
    }
    
    %% ═══════════════════════════════════════════════════════════
    %% 函数单元 - 函数对象的存储结构
    %% ═══════════════════════════════════════════════════════════
    class TVMFFIFunctionCell {
        «函数存储单元」
        「嵌入在FunctionObj中」
        ────────────────────
        +int (*safe_call)(void*, const TVMFFIAny*, int32_t, TVMFFIAny*)
        「C风格安全调用接口」
        ────────────────────
        +void* cpp_call
        「C++优化调用路径」
    }
    
    %% ═══════════════════════════════════════════════════════════
    %% 类型信息 - 运行时反射系统
    %% ═══════════════════════════════════════════════════════════
    class TVMFFITypeInfo {
        «运行时类型信息」
        「支持反射和类型检查」
        ────────────────────
        +int32_t type_index
        「类型索引」
        +int32_t type_depth
        「继承深度」
        +TVMFFIByteArray type_key
        「类型键名字符串」
        +TVMFFIAny* type_ancestors
        「父类型数组」
        +int32_t num_fields
        「字段数量」
        +TVMFFIFieldInfo* fields
        「字段信息数组」
        +int32_t num_methods
        「方法数量」
        +TVMFFIMethodInfo* methods
        「方法信息数组」
    }
    
    %% 关系说明
    TVMFFIAny ..> TVMFFIObject : v_obj 指向
    TVMFFIFunctionCell --* TVMFFIObject : 嵌入在对象中
    TVMFFIByteArray --* TVMFFITypeInfo : 被使用
    
    note for TVMFFIObject "这是C层最核心的结构\n所有Object子类内存布局\n都以TVMFFIObject开头"
    note for TVMFFIAny "16字节紧凑表示\n支持小字符串优化\nPOD类型直接存值\n对象类型存指针"
```

### TVMFFIAny 值联合体详解

```mermaid
classDiagram
    direction LR
    
    class TVMFFIAny {
        «通用值类型»
        ─────────────────────
        <<16 字节紧凑表示>>
        ─────────────────────
        +int32_t type_index
        「类型索引 - 标识存储的值类型」
        ─────────────────────
        +uint32_t zero_padding
        「填充字段 - 必须为0」
        +uint32_t small_str_len
        「小字符串长度 - 仅当type_index=SmallStr时有效」
        ─────────────────────
        Union value 「值联合体 8字节」:
        ─────────────────────
        +int64_t v_int64
        「整数值 - 用于int/bool」
        +double v_float64
        「浮点值 - 用于float/double」
        +void* v_ptr
        「通用指针 - 用于DLTensor*等」
        +TVMFFIObject* v_obj
        「对象指针 - 用于所有Object子类」
        +const char* v_c_str
        「C字符串指针 - 用于const char*参数」
        +DLDataType v_dtype
        「数据类型 - 张量的dtype」
        +DLDevice v_device
        「设备描述 - CPU/GPU等」
        +char v_bytes[8]
        「小字符串存储 - ≤7字节的字符串直接存这里」
    }
    
    note for TVMFFIAny "小字符串优化(SSO):\n≤7字节: 直接存在v_bytes中\n>7字节: 分配StringObj堆对象\n\n这个设计减少了:\n- 堆分配次数\n- 内存碎片\n- 提高了缓存命中率"
```

---

## C++ API 层类

### 核心类层次 - Object 继承体系

```mermaid
classDiagram
    direction TB
    
    %% ═══════════════════════════════════════════════════════════
    %% Object - 所有堆分配对象的基类
    %% ═══════════════════════════════════════════════════════════
    class Object {
        «所有堆对象的基类」
        「不可直接实例化」
        ─────────────────────
        #TVMFFIObject header_
        「C兼容的对象头」
        「包含引用计数和类型信息」
        ─────────────────────
        +IsInstance~T~() bool
        「运行时类型检查」
        +type_index() int32_t
        「获取类型索引」
        +GetTypeKey() string
        「获取类型键名」
        +use_count() uint64_t
        「获取引用计数」
        +unique() bool
        「是否唯一引用」
        ─────────────────────
        -IncRef()
        「增加强引用计数」
        -DecRef()
        「减少强引用计数」
        -IncWeakRef()
        「增加弱引用计数」
        -DecWeakRef()
        「减少弱引用计数」
        ─────────────────────
        +static constexpr _type_key
        「编译时类型键」
        +static constexpr _type_final
        「是否不可继承」
        +static constexpr _type_index
        「编译时类型索引」
    }
    
    %% ═══════════════════════════════════════════════════════════
    %% 具体对象类型 - Object 的直接子类
    %% ═══════════════════════════════════════════════════════════
    
    class StringObj {
        「字符串对象」
        type_index = 65
        type_final = true
        「不可继承」
    }
    
    class BytesObj {
        「字节对象」
        type_index = 66
        type_final = true
    }
    
    class ErrorObj {
        「错误对象」
        type_index = 67
        「用于异常传递」
    }
    
    class FunctionObj {
        「函数对象」
        type_index = 68
        「可调用实体」
        ─────────────────────
        +CallPacked(args, num_args, result)
        「打包参数调用」
    }
    
    class ShapeObj {
        「形状对象」
        type_index = 69
        「张量维度信息」
    }
    
    class TensorObj {
        「张量对象」
        type_index = 70
        「多维数组数据」
    }
    
    class ArrayObj {
        「数组容器」
        type_index = 71
        type_final = true
        ─────────────────────
        +size() size_t
        「元素数量」
        +at(i) Any&
        「访问第i个元素」
        +operator[](i) Any&
        「数组访问运算符」
    }
    
    class MapObj {
        「映射容器」
        type_index = 72
        「键值对存储」
    }
    
    class ModuleObj {
        「模块对象」
        type_index = 73
        「运行时模块」
    }
    
    %% 继承关系
    Object <|-- StringObj : 继承
    Object <|-- BytesObj : 继承
    Object <|-- ErrorObj : 继承
    Object <|-- FunctionObj : 继承
    Object <|-- ShapeObj : 继承
    Object <|-- TensorObj : 继承
    Object <|-- ArrayObj : 继承
    Object <|-- MapObj : 继承
    Object <|-- ModuleObj : 继承
    
    note for Object "24字节对象头结构:\n────────────────\ncombined_ref_count (8B)\n  低32位 = 强引用计数\n  高32位 = 弱引用计数\n────────────────\ntype_index (4B)\n  运行时类型索引\n────────────────\ndeleter (8B)\n  析构函数指针\n\n延迟析构策略:\n强引用归零 → 调用析构函数\n弱引用归零 → 释放内存"
```

### 引用类型层次

```mermaid
classDiagram
    direction TB
    
    class ObjectRef {
        <<base reference>>
        #ObjectPtr~Object~ data_
        +get() const Object*
        +defined() bool
        +as~T~() const T*
        +same_as(other) bool
        +static ContainerType
        +static _type_is_nullable = true
    }
    
    class String {
        -BytesBaseCell data_
        +data() const char*
        +size() size_t
        +operator std::string()
    }
    
    class Bytes {
        -BytesBaseCell data_
        +data() const char*
        +size() size_t
    }
    
    class Function {
        +operator()(Args...) Any
        +CallPacked(args, num_args, result)
        +static FromPacked(callable)
        +static FromTyped(callable)
        +static GetGlobal(name)
        +static SetGlobal(name, func)
    }
    
    class Array~T~ {
        +at(i) T
        +size() size_t
        +empty() bool
        +begin() iterator
        +end() iterator
    }
    
    class Map~K,V~ {
        +at(key) V
        +size() size_t
        +find(key) iterator
    }
    
    class Shape {
        +size() size_t
        +operator[](i) int64_t
    }
    
    class Tensor {
        +data() void*
        +dtype() DLDataType
        +shape() Shape
    }
    
    ObjectRef <|-- String
    ObjectRef <|-- Bytes
    ObjectRef <|-- Function
    ObjectRef <|-- Array
    ObjectRef <|-- Map
    ObjectRef <|-- Shape
    ObjectRef <|-- Tensor
```

### 智能指针类型

```mermaid
classDiagram
    direction LR
    
    class ObjectPtr~T~ {
        <<strong reference>>
        -Object* data_
        +get() T*
        +reset()
        +use_count() int
        +unique() bool
        +operator*()
        +operator->()
    }
    
    class WeakObjectPtr~T~ {
        <<weak reference>>
        -Object* data_
        +lock() ObjectPtr~T~
        +expired() bool
        +reset()
    }
    
    ObjectPtr "1" --> "1" Object : manages
    WeakObjectPtr "1" --> "1" Object : observes
```

### Any 和 AnyView

```mermaid
classDiagram
    direction TB
    
    class AnyView {
        <<non-owning view>>
        <<16 bytes>>
        #TVMFFIAny data_
        +as~T~() optional~T~
        +cast~T~() T
        +type_index() int32_t
        +GetTypeKey() string
    }
    
    class Any {
        <<owning value>>
        <<16 bytes>>
        +Any()
        +~Any()
        +Any(T value)
        +reset()
    }
    
    AnyView <|-- Any : extends
    note for AnyView "非拥有引用\n可安全复制\n用于函数参数"
    note for Any "拥有所有权\n析构时释放引用\n用于返回值和存储"
```

---

## 类型索引系统

### 类型索引分布

```mermaid
graph TB
    subgraph "POD Types [0, 64)"
        N[kTVMFFINone = 0]
        I[kTVMFFIInt = 1]
        B[kTVMFFIBool = 2]
        F[kTVMFFIFloat = 3]
        P[kTVMFFIOpaquePtr = 4]
        DT[kTVMFFIDataType = 5]
        DV[kTVMFFIDevice = 6]
        SS[kTVMFFISmallStr = 11]
        SB[kTVMFFISmallBytes = 12]
    end
    
    subgraph "Static Object Types [64, 128)"
        OBJ[kTVMFFIObject = 64]
        STR[kTVMFFIStr = 65]
        BYT[kTVMFFIBytes = 66]
        ERR[kTVMFFIError = 67]
        FUN[kTVMFFIFunction = 68]
        SHP[kTVMFFIShape = 69]
        TNS[kTVMFFITensor = 70]
        ARR[kTVMFFIArray = 71]
        MAP[kTVMFFIMap = 72]
        MOD[kTVMFFIModule = 73]
    end
    
    subgraph "Dynamic Object Types [128, +∞)"
        DYN[kTVMFFIDynObjectBegin = 128]
        USR[User-defined types...]
    end
    
    N --> I --> B --> F --> P --> DT --> DV --> SS --> SB
    OBJ --> STR --> BYT --> ERR --> FUN --> SHP --> TNS --> ARR --> MAP --> MOD
    DYN --> USR
```

---

## 完整类图

### 完整类型系统概览

```mermaid
flowchart TB
    subgraph "C API Layer"
        CO[TVMFFIObject]
        CA[TVMFFIAny]
        CBA[TVMFFIByteArray]
        CFC[TVMFFIFunctionCell]
        CTI[TVMFFITypeInfo]
    end
    
    subgraph "C++ Core Classes"
        OBJ[Object]
        OREF[ObjectRef]
        OPTR[ObjectPtr T]
        WOPTR[WeakObjectPtr T]
        ANY[Any]
        ANYV[AnyView]
    end
    
    subgraph "Concrete Object Types"
        SO[StringObj]
        BO[BytesObj]
        FO[FunctionObj]
        EO[ErrorObj]
        AO[ArrayObj]
        MO[MapObj]
        SHO[ShapeObj]
        TO[TensorObj]
        MDO[ModuleObj]
    end
    
    subgraph "Reference Types"
        S[String]
        B[Bytes]
        FN[Function]
        AR[Array T]
        MP[Map K V]
        SH[Shape]
        TN[Tensor]
    end
    
    CO --> OBJ : header
    CA --> ANYV : data
    ANYV --> ANY : extends
    
    OBJ --> SO
    OBJ --> BO
    OBJ --> FO
    OBJ --> EO
    OBJ --> AO
    OBJ --> MO
    OBJ --> SHO
    OBJ --> TO
    OBJ --> MDO
    
    OREF --> S
    OREF --> B
    OREF --> FN
    OREF --> AR
    OREF --> MP
    OREF --> SH
    OREF --> TN
    
    OPTR --> OREF : contains
    WOPTR --> OBJ : observes
```

### 对象创建和内存管理流程

```mermaid
sequenceDiagram
    participant User
    participant make_object
    participant Allocator
    participant ObjectPtr
    participant Object
    participant TVMFFIObject
    
    User->>make_object: make_object&lt;T&gt;(args...)
    make_object->>Allocator: SimpleObjAllocator
    Allocator->>Allocator: AlignedAlloc(size)
    Allocator->>Object: placement new T(args...)
    Allocator->>TVMFFIObject: Set header fields
    Note over TVMFFIObject: combined_ref_count = 0x100000001<br/>(strong=1, weak=1)
    Allocator->>ObjectPtr: Create from raw pointer
    ObjectPtr-->>User: Return ObjectPtr&lt;T&gt;
    
    User->>ObjectPtr: reset() / destructor
    ObjectPtr->>Object: DecRef()
    Object->>TVMFFIObject: --strong_ref
    alt strong_ref == 0
        Object->>Object: destructor()
        Object->>TVMFFIObject: --weak_ref
        alt weak_ref == 0
            Object->>Allocator: AlignedFree()
        end
    end
```

---

## 内存布局

### TVMFFIObject 内存布局

```mermaid
block-beta
    columns 4
    
    block:header:4
        A["combined_ref_count (8B)"]
    end
    
    block:header2:4
        B["type_index (4B)"]
        C["__padding (4B)"]
    end
    
    block:header3:4
        D["deleter (8B)"]
    end
    
    E["Total: 24 bytes, 8-byte aligned"]
```

### TVMFFIAny 内存布局

```mermaid
block-beta
    columns 4
    
    block:any:4
        A["type_index (4B)"]
        B["zero_padding / small_str_len (4B)"]
    end
    
    block:value:4
        C["value union (8B)"]
        C["v_int64 / v_float64 / v_ptr / v_obj / v_c_str / v_dtype / v_device / v_bytes[8]"]
    end
    
    D["Total: 16 bytes"]
```

### StringObj 内存布局

```mermaid
block-beta
    columns 6
    
    block:obj1:6
        A["TVMFFIObject header (24B)"]
    end
    
    block:obj2:6
        B["TVMFFIByteArray.data (8B)"]
        C["TVMFFIByteArray.size (8B)"]
    end
    
    block:obj3:6
        D["char data[] (variable)"]
        D["char data[] (variable)"]
        D["char data[] (variable)"]
        D["..."]
        D["'\\0'"]
        E["padding"]
    end
```

### ArrayObj 内存布局

```mermaid
block-beta
    columns 6
    
    block:arr1:6
        A["TVMFFIObject header (24B)"]
    end
    
    block:arr2:6
        B["capacity_ (8B)"]
        C["size_ (8B)"]
    end
    
    block:arr3:6
        D["data_ (8B)"]
        E["data_deleter_ (8B)"]
    end
    
    block:arr4:6
        F["TVMFFIAny[0] (16B)"]
        G["TVMFFIAny[1] (16B)"]
        H["..."]
        I["TVMFFIAny[n-1] (16B)"]
    end
```

---

## 类型特征系统

### TypeTraits 模板

```mermaid
classDiagram
    direction TB
    
    class TypeTraits~T~ {
        <<template>>
        +static constexpr bool convert_enabled
        +static constexpr bool storage_enabled
        +static constexpr int32_t field_static_type_index
        +static CopyToAnyView(T, TVMFFIAny*)
        +static MoveToAny(T, TVMFFIAny*)
        +static CheckAnyStrict(TVMFFIAny*) bool
        +static CopyFromAnyViewAfterCheck(TVMFFIAny*) T
        +static MoveFromAnyAfterCheck(TVMFFIAny*) T
        +static TryCastFromAnyView(TVMFFIAny*) optional~T~
        +static GetMismatchTypeInfo(TVMFFIAny*) string
        +static TypeStr() string
        +static TypeSchema() string
    }
    
    class TypeTraitsBase {
        +static constexpr bool convert_enabled = true
        +static constexpr bool storage_enabled = true
        +static constexpr field_static_type_index
        +static GetMismatchTypeInfo()
    }
    
    class ObjectRefTypeTraitsBase~TObjRef~ {
        +static CopyToAnyView()
        +static MoveToAny()
        +static CheckAnyStrict()
        +static CopyFromAnyViewAfterCheck()
        +static MoveFromAnyAfterCheck()
        +static TryCastFromAnyView()
    }
    
    class FallbackOnlyTraitsBase~T,FallbackTypes~ {
        +static TryCastFromAnyView()
        +static TryFallbackTypes()
    }
    
    TypeTraits <|-- TypeTraitsBase
    TypeTraitsBase <|-- ObjectRefTypeTraitsBase
    TypeTraitsBase <|-- FallbackOnlyTraitsBase
```

### 类型特化示例

```mermaid
graph TB
    subgraph "POD Type Specializations"
        TT_INT[TypeTraits&lt;int&gt;]
        TT_BOOL[TypeTraits&lt;bool&gt;]
        TT_FLOAT[TypeTraits&lt;float&gt;]
        TT_PTR[TypeTraits&lt;void*&gt;]
        TT_DEV[TypeTraits&lt;DLDevice&gt;]
    end
    
    subgraph "String/Bytes Specializations"
        TT_STR[TypeTraits&lt;String&gt;]
        TT_BYTES[TypeTraits&lt;Bytes&gt;]
        TT_CSTR[TypeTraits&lt;const char*&gt;]
        TT_STDSTR[TypeTraits&lt;std::string&gt;]
    end
    
    subgraph "ObjectRef Specializations"
        TT_OREF[TypeTraits&lt;ObjectRef&gt;]
        TT_FUNC[TypeTraits&lt;Function&gt;]
        TT_ARR[TypeTraits&lt;Array&lt;T&gt;&gt;]
    end
    
    subgraph "Optional Specialization"
        TT_OPT[TypeTraits&lt;Optional&lt;T&gt;&gt;]
    end
```

---

## 使用示例

### 创建和使用对象

```cpp
#include <tvm/ffi/memory.h>
#include <tvm/ffi/string.h>
#include <tvm/ffi/container/array.h>

// 创建字符串
tvm::ffi::String str = "Hello, TVM-FFI!";

// 创建数组
tvm::ffi::Array<tvm::ffi::Any> arr = {1, 2.0, "three"};

// 使用 Any
tvm::ffi::Any value = 42;
int64_t int_val = value.cast<int64_t>();

// 使用 Optional
tvm::ffi::Optional<tvm::ffi::String> opt_str = tvm::ffi::String("optional");
```

### 创建函数

```cpp
#include <tvm/ffi/function.h>

// 从 lambda 创建函数
auto add = [](int a, int b) -> int { return a + b; };
tvm::ffi::Function func = tvm::ffi::Function::FromTyped(add, "add");

// 注册全局函数
tvm::ffi::Function::SetGlobal("add", func);

// 调用函数
tvm::ffi::Function add_func = tvm::ffi::Function::GetGlobalRequired("add");
tvm::ffi::Any result = add_func(1, 2);
```

### 自定义对象类型

```cpp
#include <tvm/ffi/object.h>

class MyObject : public tvm::ffi::Object {
 public:
  int value;
  
  explicit MyObject(int v) : value(v) {}
  
  static constexpr const char* _type_key = "MyObject";
  static constexpr bool _type_final = true;
  TVM_FFI_DECLARE_OBJECT_INFO_STATIC("MyObject", MyObject, Object);
};

class MyObjectRef : public tvm::ffi::ObjectRef {
 public:
  TVM_FFI_DEFINE_OBJECT_REF_METHODS(MyObjectRef, ObjectRef, MyObject);
};

// 使用
MyObjectRef ref = tvm::ffi::make_object<MyObject>(42);
```

---

## 参考资料

- 源码位置: `3rdparty/tvm-ffi/include/tvm/ffi/`
- 官方文档: https://tvm.apache.org/
- GitHub: https://github.com/apache/tvm-ffi
