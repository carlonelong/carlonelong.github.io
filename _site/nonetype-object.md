# None object分析

“管中窥豹，可见一斑” --刘义庆《世说新语》

## Before we start
开始之前需要知道一点：Python中所有的变量都是对象。包括None、bool变量、各种内置类型以及用户定义的类和类生成的实例，以及Python解释器中code的静态和动态反映。
## Where to start?
我记得高中数学做选择题的时候，如果有几个选项拿不准又懒得算，一个比较好的办法就是用几个特例来验证。

至于Python, 最简单的对象无非就是None和True、False三个对象。至于为什么说这是三个对象，可以简单通过以下代码验证：
```
a = None # or True/False
b = None
assert(a is b)
```
即所有的None/True/False分别都是同一个对象，原因会在后文提到。
So, let's get started with the simplest.
## 定位到具体代码
首先，通过分析源码的结构，我们知道对象都是放在objects目录下的，遗憾的是这里并没有对应的none*.c/.h文件。
我们尝试通过None的类型信息来获得一些帮助。
`type(None)`
得到`<class 'NoneType'>`。
通过搜索这个NoneType我们可以定位到object.c文件。此外还发现了一些有意思的内容，比如这个字符串是一个叫做 _PyNone_Type 的第二个属性。
```
PyTypeObject _PyNone_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "NoneType",
    0,
    0,
    none_dealloc,       /*tp_dealloc*/ /*never called*/
    0,                  /*tp_print*/
    0,                  /*tp_getattr*/
    0,                  /*tp_setattr*/
    0,                  /*tp_reserved*/
    none_repr,          /*tp_repr*/
    &none_as_number,    /*tp_as_number*/
    0,                  /*tp_as_sequence*/
    0,                  /*tp_as_mapping*/
    0,                  /*tp_hash */
    0,                  /*tp_call */
    0,                  /*tp_str */
    0,                  /*tp_getattro */
    0,                  /*tp_setattro */
    0,                  /*tp_as_buffer */
    Py_TPFLAGS_DEFAULT, /*tp_flags */
    0,                  /*tp_doc */
    0,                  /*tp_traverse */
    0,                  /*tp_clear */
    0,                  /*tp_richcompare */
    0,                  /*tp_weaklistoffset */
    0,                  /*tp_iter */
    0,                  /*tp_iternext */
    0,                  /*tp_methods */
    0,                  /*tp_members */
    0,                  /*tp_getset */
    0,                  /*tp_base */
    0,                  /*tp_dict */
    0,                  /*tp_descr_get */
    0,                  /*tp_descr_set */
    0,                  /*tp_dictoffset */
    0,                  /*tp_init */
    0,                  /*tp_alloc */
    none_new,           /*tp_new */
};
```
这就是Node的类型信息，这里我们暂时跳过这个结构体，先找到None真正对应的对象。
同时这个_PyNone_Type还出现在
```
PyObject _Py_NoneStruct = {
  _PyObject_EXTRA_INIT
  1, &_PyNone_Type
};
```

在 object.h中
```
PyAPI_DATA(PyObject) _Py_NoneStruct; /* Don't use this directly */
#define Py_None (&_Py_NoneStruct)

/* Macro for returning Py_None from a function */
#define Py_RETURN_NONE return Py_INCREF(Py_None), Py_None
```
PyAPI_DATA是一个跟系统相关的宏，用来声明一个extern变量。而这个Py_None 就是None在C语言中的体现了，其本质就是一个经过层层包装的<strong>全局</strong>_Py_NoneStruct。也就是说，所有python中的None实际上都是这个全局的结构体。
### PyObject
Python中所有变量都是对象,并且所有对象（除了None)都直接或间接继承于object。目前我们还没有找到这个object在C代码中对应的结构体，但是肯定会跟PyObject这个结构体有莫大的关系。在object.h中可以看到PyObject的定义
```
typedef struct _object {
    _PyObject_HEAD_EXTRA
    Py_ssize_t ob_refcnt;
    struct _typeobject *ob_type;
} PyObject;
```
同时还有一段注释
```
/* Nothing is actually declared to be a PyObject, but every pointer to
 * a Python object can be cast to a PyObject*.  This is inheritance built
 * by hand.  Similarly every pointer to a variable-size Python object can,
 * in addition, be cast to PyVarObject*.
 */
```
可见没有任何对象是直接对应于PyObject对象的，但是任意对象都可以转成PyObject的指针。所以这些对象在内存分布上一定遵循统一的规则。

从PyObject的定义可以看到，其结构非常简单，所有对象都含有三部分。_PyObject_HEAD_EXTRA是内存相关的属性（实际上是指向在对上分配的另外的对象的两个指针），ob_refcnt为引用计数，ob_type为这个对象的类型对象的指针，比如_Py_NoneStruct的ob_type就指向_PyNone_Type。我们暂时跳过内存相关的内容，因此只关注这个ob_type域。因为在PyObject中并没有任何跟类型相关的信息（比如None和True在这里没有差别），所以每个类型的不同表现都肯定反映在ob_type中。
### PyTypeObject分析
在typestruct.h（根据编译条件的不同也可能是object.h，由宏Py_LIMITED_AP控制）中可以找到PyTypeObject的定义
```
typedef struct _typeobject {
    const char *tp_name; /* For printing, in format "<module>.<name>" */
    Py_ssize_t tp_basicsize, tp_itemsize; /* For allocation */

    /* Methods to implement standard operations */

    destructor tp_dealloc;
    printfunc tp_print;
    getattrfunc tp_getattr;
    setattrfunc tp_setattr;
    PyAsyncMethods *tp_as_async; /* formerly known as tp_compare (Python 2)
                                    or tp_reserved (Python 3) */
    reprfunc tp_repr;

    /* Method suites for standard classes */

    PyNumberMethods *tp_as_number;
    PySequenceMethods *tp_as_sequence;
    PyMappingMethods *tp_as_mapping;

    /* More standard operations (here for binary compatibility) */

    hashfunc tp_hash;
    ternaryfunc tp_call;
    reprfunc tp_str;
    getattrofunc tp_getattro;
    setattrofunc tp_setattro;

    /* Functions to access object as input/output buffer */
    PyBufferProcs *tp_as_buffer;

    /* Flags to define presence of optional/expanded features */
    unsigned long tp_flags;

    const char *tp_doc; /* Documentation string */

    /* call function for all accessible objects */
    traverseproc tp_traverse;

    /* delete references to contained objects */
    inquiry tp_clear;

    /* rich comparisons */
    richcmpfunc tp_richcompare;

    /* weak reference enabler */
    Py_ssize_t tp_weaklistoffset;

    /* Iterators */
    getiterfunc tp_iter;
    iternextfunc tp_iternext;

    /* Attribute descriptor and subclassing stuff */
    struct PyMethodDef *tp_methods;
    struct PyMemberDef *tp_members;
    struct PyGetSetDef *tp_getset;
    struct _typeobject *tp_base;
    PyObject *tp_dict;
    descrgetfunc tp_descr_get;
    descrsetfunc tp_descr_set;
    Py_ssize_t tp_dictoffset;
    initproc tp_init;
    allocfunc tp_alloc;
    newfunc tp_new;
    freefunc tp_free; /* Low-level free-memory routine */
    inquiry tp_is_gc; /* For PyObject_IS_GC */
    PyObject *tp_bases;
    PyObject *tp_mro; /* method resolution order */
    PyObject *tp_cache;
    PyObject *tp_subclasses;
    PyObject *tp_weaklist;
    destructor tp_del;

    /* Type attribute cache version tag. Added in version 2.6 */
    unsigned int tp_version_tag;

    destructor tp_finalize;

} PyTypeObject;
```

这个非常大的结构体(400Bytes)中，有很多函数指针，用于根据不同的类型实现不同功能。比如其中的tp_as_number就是把该对象当做一个数字时应表现出来的行为。从这里我们看出Python的面向对象机制的实现：所有对象最开始的一段内存分布都是相同的（所以才能转成PyObject指针），通过对象的ob_type指针指向某一个PyTypeObject（也遵循前述内存分布，通过PyObject_VAR_HEAD实现）对象，这个对象里面定义了所有支持的行为。而多态机制正是通过覆盖父类的函数指针来实现的。这跟C++非常相似，只不过C++是直接把虚函数表放在了对象里面最开始的位置，而Python是通过type对象这一代理来实现的。Python为什么要怎么设计，我现在也不懂，希望看完代码之后能领悟其中的深意。

这里可能会有一个疑问，对于None, bool, int等变量，PyObject不需要额外存储数据，所以其结构是够用的。但是对于list，dict等可变对象，他们内部的对象又是存放在哪里的呢？答案就是上文提到的PyVarObject。
```
typedef struct {
    PyObject ob_base;
    Py_ssize_t ob_size; /* Number of items in variable part */
} PyVarObject;
```
实际上PyVarObject是在PyObject后面增加了一个计数器，用来表示该可变对象中存储了多少个对象。而这些对象的存储地址则存在子类型（比如PyDictObject）中。

这么多的函数，还是从最简单的_PyNone_Type开始看起吧。
#### _PyNone_Type分析
从前面的代码片段可以看到_PyNone_Type只实现了tp_dealloc, tp_repr, tp_as_number, tp_flags, tp_new几个字段。除开内存分配相关函数tp_flags是一些运行时相关的值，这里先不管。tp_repr就是在Python中调用repr时在C中最后调用的函数，而tp_as_number前面已经提到过了，_PyNone_Type的tp_as_number 是 none_as_number
```
static PyNumberMethods none_as_number = {
    0,                          /* nb_add */
    0,                          /* nb_subtract */
    0,                          /* nb_multiply */
    0,                          /* nb_remainder */
    0,                          /* nb_divmod */
    0,                          /* nb_power */
    0,                          /* nb_negative */
    0,                          /* nb_positive */
    0,                          /* nb_absolute */
    (inquiry)none_bool,         /* nb_bool */
    0,                          /* nb_invert */
    0,                          /* nb_lshift */
    0,                          /* nb_rshift */
    0,                          /* nb_and */
    0,                          /* nb_xor */
    0,                          /* nb_or */
    0,                          /* nb_int */
    0,                          /* nb_reserved */
    0,                          /* nb_float */
    0,                          /* nb_inplace_add */
    0,                          /* nb_inplace_subtract */
    0,                          /* nb_inplace_multiply */
    0,                          /* nb_inplace_remainder */
    0,                          /* nb_inplace_power */
    0,                          /* nb_inplace_lshift */
    0,                          /* nb_inplace_rshift */
    0,                          /* nb_inplace_and */
    0,                          /* nb_inplace_xor */
    0,                          /* nb_inplace_or */
    0,                          /* nb_floor_divide */
    0,                          /* nb_true_divide */
    0,                          /* nb_inplace_floor_divide */
    0,                          /* nb_inplace_true_divide */
    0,                          /* nb_index */
};
```
又是一个不小的结构体
```
typedef struct {
    binaryfunc nb_add;
    binaryfunc nb_subtract;
    binaryfunc nb_multiply;
    binaryfunc nb_remainder;
    binaryfunc nb_divmod;
    ternaryfunc nb_power;
    unaryfunc nb_negative;
    unaryfunc nb_positive;
    unaryfunc nb_absolute;
    inquiry nb_bool;
    unaryfunc nb_invert;
    binaryfunc nb_lshift;
    binaryfunc nb_rshift;
    binaryfunc nb_and;
    binaryfunc nb_xor;
    binaryfunc nb_or;
    unaryfunc nb_int;
    void *nb_reserved;  /* the slot formerly known as nb_long */
    unaryfunc nb_float;
    binaryfunc nb_inplace_add;
    binaryfunc nb_inplace_subtract;
    binaryfunc nb_inplace_multiply;
    binaryfunc nb_inplace_remainder;
    ternaryfunc nb_inplace_power;
    binaryfunc nb_inplace_lshift;
    binaryfunc nb_inplace_rshift;
    binaryfunc nb_inplace_and;
    binaryfunc nb_inplace_xor;
    binaryfunc nb_inplace_or;
    binaryfunc nb_floor_divide;
    binaryfunc nb_true_divide;
    binaryfunc nb_inplace_floor_divide;
    binaryfunc nb_inplace_true_divide;
    unaryfunc nb_index;
    binaryfunc nb_matrix_multiply;
    binaryfunc nb_inplace_matrix_multiply;
} PyNumberMethods;
```
好在这些函数的命名都能比较清楚地看出用途。可以看到none_as_number只实现了一个唯一的域nb_bool
```
static int
none_bool(PyObject *v)
{
    return 0;
}
```
所以None始终是False
## 小结
1. Python中所有变量都是对象，在C一层都可以转成PyObject指针。
2. Python采用了相同的内存起始布局+函数指针的方式来实现继承和多态。
3. Python的函数（方法）是按需定制的，比如NoneType只实现了tp_as_number 中的 none_bool。
4. 好吧写这么多确实没多少内容。



