# Немного о типах данных в Python

## Intro  V

Python - высокоуровневый язык программирования, один из самых популярных по
данным разных рейтингов, очень лёгкий, обладает простым и понятным синтаксисом.
Язык имеет динамическую типизацию, большую стандартную библиотеку и много
применений. Всё это позволяет говорить, что Python - лучший язык программирования
для новичков.

И в этой статье мы рассмотрим, как устроены стандартные питоновские типы, которые
язык поставляет по умолчанию.

Так как это статья о питоне, то её нельзя было начинать как-то иначе. А то
нарушится мировой баланс сил, из-за чего копирайтеры перестанут бесконечно
переводить одни и те же посты из индусских блогов для вайтишников.

Идея посмотреть на исходники питона пришла, когда шарпист спросил меня, сколько
байтов в питоне занимает `int`. Я знал, что инты ради переполнения реализованы
через массив `C`ишных интов, но в деталях особо не знал, как там и что. Поэтому
и стало интересно нормально разобраться в реализации, а заодно начёркать статью
для тех, кому тоже интересно, но лень смотреть. Здесь небольшие заметки о типах
и интересные места реализации, которые я выписывал, пока прыгал по C-коду.

! Смотрел на исходники `CPython` версии `3.12.0 alpha 0` ([вот это вот](https://github.com/python/cpython/tree/d8537580921b2e02f477ff1a8dedcf82c24ef0c2)).
Си почти не знаю, поэтому могу здесь нести какую-то фигню относительно 
реализации, неправильно называя/понимая какие-то сишные выражения. Если так, то
жду поправлений.

## builtins  V

    Смотреть:
    Python/bltnmodule.c (1)
    Python/initconfig.c (2)

Все типы описаны в питоновском модуле `builtins`. 

В файлике `(1):3046` описаны стандартные функции (`abs`, `all`, `any`, ...). В
основном это просто перевод на реализованные где-то глубже функции, а здесь
нужны просто как обёртка для интерпретатора. В самом конце файла по такому же
принципу добавляются все стандартные классы:

```C
    SETBUILTIN("None",                  Py_None);
    SETBUILTIN("Ellipsis",              Py_Ellipsis);
    SETBUILTIN("NotImplemented",        Py_NotImplemented);
    SETBUILTIN("False",                 Py_False);
    SETBUILTIN("True",                  Py_True);
    SETBUILTIN("bool",                  &PyBool_Type);
    SETBUILTIN("memoryview",        &PyMemoryView_Type);
    SETBUILTIN("bytearray",             &PyByteArray_Type);
    SETBUILTIN("bytes",                 &PyBytes_Type);
    SETBUILTIN("classmethod",           &PyClassMethod_Type);
    SETBUILTIN("complex",               &PyComplex_Type);
    SETBUILTIN("dict",                  &PyDict_Type);
    SETBUILTIN("enumerate",             &PyEnum_Type);
    SETBUILTIN("filter",                &PyFilter_Type);
    SETBUILTIN("float",                 &PyFloat_Type);
    SETBUILTIN("frozenset",             &PyFrozenSet_Type);
    SETBUILTIN("property",              &PyProperty_Type);
    SETBUILTIN("int",                   &PyLong_Type);
    SETBUILTIN("list",                  &PyList_Type);
    SETBUILTIN("map",                   &PyMap_Type);
    SETBUILTIN("object",                &PyBaseObject_Type);
    SETBUILTIN("range",                 &PyRange_Type);
    SETBUILTIN("reversed",              &PyReversed_Type);
    SETBUILTIN("set",                   &PySet_Type);
    SETBUILTIN("slice",                 &PySlice_Type);
    SETBUILTIN("staticmethod",          &PyStaticMethod_Type);
    SETBUILTIN("str",                   &PyUnicode_Type);
    SETBUILTIN("super",                 &PySuper_Type);
    SETBUILTIN("tuple",                 &PyTuple_Type);
    SETBUILTIN("type",                  &PyType_Type);
    SETBUILTIN("zip",                   &PyZip_Type);
```

Из-за того, что первые 5 инициализаций (`None`, `Ellipsis`, `NotImplemented`,
`False`, `True`) являются объектами, а не классами, то и устанавливаются не
ссылки на них, а сами созданные объекты. Тут же на строчку ниже определяется и
устанавливается переменная `__debug__`:

```C
    debug = PyBool_FromLong(config->optimization_level == 0);
    if (PyDict_SetItemString(dict, "__debug__", debug) < 0) {
        Py_DECREF(debug);
        return NULL;
    }
    Py_DECREF(debug);
```

Парсятся флаги запуска (втч дебага) в `(2):2253` функцией `config_parse_cmdline`.

## type / PyTypeObject

    Смотреть:
    Include/object.h (3)
    Include/cpython/object.h (4)

Поскольку для любого класса верно `isinstance(ANY.__class__, type) == True`, то
и начать стоит с него.  Вместе с `object` и `super` класс `type` инициализируется
в файле `(3):271`. Все три класса имеют тип `PyTypeObject`. Логично, да:

```C
PyAPI_DATA(PyTypeObject) PyType_Type; /* built-in 'type' */
PyAPI_DATA(PyTypeObject) PyBaseObject_Type; /* built-in 'object' */
PyAPI_DATA(PyTypeObject) PySuper_Type; /* built-in 'super' */
```

Вот что говорит про этот тип [питоновская дока](https://docs.python.org/3/c-api/typeobj.html):

```text

```


Структура `PyTypeObject` (переопределяется в файле `Include/pytypedefs.h:20` из
стуктуры `_typeobject`) описана в `(4):148`:

<details>
  <summary>Варнинг, мультилеттер</summary>

```C
struct _typeobject {
    PyObject_VAR_HEAD
    const char *tp_name; /* For printing, in format "<module>.<name>" */
    Py_ssize_t tp_basicsize, tp_itemsize; /* For allocation */

    /* Methods to implement standard operations */

    destructor tp_dealloc;
    Py_ssize_t tp_vectorcall_offset;
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

    /* Assigned meaning in release 2.0 */
    /* call function for all accessible objects */
    traverseproc tp_traverse;

    /* delete references to contained objects */
    inquiry tp_clear;

    /* Assigned meaning in release 2.1 */
    /* rich comparisons */
    richcmpfunc tp_richcompare;

    /* weak reference enabler */
    Py_ssize_t tp_weaklistoffset;

    /* Iterators */
    getiterfunc tp_iter;
    iternextfunc tp_iternext;

    /* Attribute descriptor and subclassing stuff */
    PyMethodDef *tp_methods;
    PyMemberDef *tp_members;
    PyGetSetDef *tp_getset;
    // Strong reference on a heap type, borrowed reference on a static type
    PyTypeObject *tp_base;
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
    vectorcallfunc tp_vectorcall;
};
```
</details>

Здесь есть много непонятных типов, вот так выглядит (на моём компьютере с моей
осью) без множества `typedef` (комментарии - то, для чего нужно):

```C
struct _object {  // позже переименовывается в `PyObject`;
    long int ob_refcnt;
    _typeobject *ob_type;
};

typedef struct {
    _object ob_base;
    long int ob_size;
} PyVarObject;

struct _typeobject {
    PyVarObject ob_base;
    const char *tp_name;
    long int tp_basicsize, tp_itemsize;
    void function(_object *) tp_dealloc;
    long int tp_vectorcall_offset;
    getattrfunc tp_getattr;
    setattrfunc tp_setattr;
    PyAsyncMethods *tp_as_async;
    reprfunc tp_repr;
    PyNumberMethods *tp_as_number;
    PySequenceMethods *tp_as_sequence;
    PyMappingMethods *tp_as_mapping;
    hashfunc tp_hash;
    ternaryfunc tp_call;
    reprfunc tp_str;
    getattrofunc tp_getattro;
    setattrofunc tp_setattro;ъ
    PyBufferProcs *tp_as_buffer;
    unsigned long tp_flags;
    const char *tp_doc;
    traverseproc tp_traverse;
    inquiry tp_clear;
    richcmpfunc tp_richcompare;
    long int tp_weaklistoffset;
    getiterfunc tp_iter;
    iternextfunc tp_iternext;
    PyMethodDef *tp_methods;
    PyMemberDef *tp_members;
    PyGetSetDef *tp_getset;
    PyTypeObject *tp_base;
    _object *tp_dict;
    descrgetfunc tp_descr_get;
    descrsetfunc tp_descr_set;
    long int tp_dictoffset;
    initproc tp_init;
    allocfunc tp_alloc;
    newfunc tp_new;
    freefunc tp_free;
    inquiry tp_is_gc;
    _object *tp_bases;
    _object *tp_mro;
    _object *tp_cache;
    _object *tp_subclasses;
    _object *tp_weaklist;
    void function(_object *) tp_del;
    unsigned int tp_version_tag;
    void function(_object *) tp_finalize;
    vectorcallfunc tp_vectorcall;
};
```




```C
long int Py_ssize_t;
```

## object

## int

    Objects/abstract.c (x3)
    Objects/longobject.c (x0)
    Include/cpython/longintrepr.h (x1)

хранит 

Перевод в `int` возможен в 2 случаях:
- если у аргумента есть один из методов `.__int__(self)`, `.__index__(self)`,
  `.__trunc__(self)` (проверяет в этом порядке)
- если аргумент `bytes`/`bytearray`/`str` - переводится в `str`, а из него в `int`

Первый случай (хотя он описан в документации, не знал про него, кто ж документацию
читает) как раз переводит `float` и `decimal` в инты через `__trunc__`. Метод
`__int__` у них тоже есть, но только как обёртка и переводит на `__trunc__`.

Во втором случае происходит перевод в инты из строк, а точнее, из набора 8-битных
чисел (`unsigned char`), функцией `PyLong_FromString` (`(x0):2239`). Во время
перевода вычисляется база числа, если её нет (по началу строк типа `0b/0o/0x`),
а затем по базе переводится, собственно, в число. Если база еть степень двойки,
то перевод идёт более быстрой функцией, которая просто переводит значения
символов в двоичный вид и по очереди складывает биты значений символа в массив
значений результирующего инта.

Кстати, угадайте, что это?

```C
unsigned char _PyLong_DigitValue[256] = {
    37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37,
    37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37,
    37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37,
    0,  1,  2,  3,  4,  5,  6,  7,  8,  9,  37, 37, 37, 37, 37, 37,
    37, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24,
    25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 37, 37, 37, 37, 37,
    37, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24,
    25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 37, 37, 37, 37, 37,
    37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37,
    37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37,
    37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37,
    37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37,
    37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37,
    37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37,
    37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37,
    37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37, 37,
};
```

Да, это матрица соответствие кода символа и цифры (`(x0):2114`). Все вылеты за
допустимые символы возвращают `37`: так как максимальная база перевода есть `36`,
то при неизвестном символе число будет больше базы и вылетит ошибка.

Но в случае, если база будет недвоичной, то вычисление будущего размера массива
значений и сам перевод числа гораздо сложнее. Уже нельзя просто переложить биты
из одного массива в другой, нужно сначала предположить максимальный размер
будущего массива чисел, а затем произвести сам перевод числа.

Размер вычисляется с помощью формулы:

```
n >= N * log(B)/log(b)  , где
n - количество необходимых цифр в нужной базе
N - количество цифр во входном числе
B - база числа на входе
b - требуемая база
```


Квадратичная скорость обычных баз, линейная бинарных.

Высчитывает примерный размер будущего инта и выделяет память
по очереди переводит цифры из строки в двоичный вид, много возни с подчёркиваниями
подставляет данные (интересно, но логично, что в зависимости от знака цикл в разные стороны)


В коде перевода строк очень много проверок на символ `"_"`, что не повторяется
больше одного раза подряд и на него не заканчивается число. Как по мне, странно,
можно было оставить такую кучу проверок, работало бы капельку быстрее, а разницы
особо нет. Но это я так считаю, а я не контрибьютор питона.





## bool

## False

## True

## float

## None

## Ellipsis

## NotImplemented

## memoryview

## bytearray
## bytes
## classmethod
## complex
## dict
## enumerate
## filter
## frozenset
## property
## list
## map
## range
## reversed
## set
## slice
## staticmethod
## str
## super
## tuple
## zip

