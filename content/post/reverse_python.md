---
title: "Python逆向初体验"
date: 2021-08-22T08:57:42+08:00
ShowToc: true
---

# Cython篇

---

Python逆向主要难度还是在pyd（或者说so）上，以下为Cython生成的可供python导入的模块的逆向手段。



Cython也能产生exe，使用`—embed`命令行参数，然后正常添加好include目录和libs目录和依赖就可以直接编译了。



# 开发的角度

## 项目创建和构建

在Windows上，直接通过setup.py的处理方法，编译出来的pyd没有符号文件产生，而且似乎也没有办法通过设置命令行，设置某些参数之类的方式来生成符号文件。我目前探索比较方便的方案是：

创建VS的动态链接库工程，项目属性里做如下工作：

- （可选）可以在pre event里调用cython生成`.c`



![](/reverse_python/image/image.png "")

- 需要在项目依赖的include目录和lib目录，用到的lib依赖，三个地方都加上python的部分



![](/reverse_python/image/image_1.png "")

![](/reverse_python/image/image_2.png "")

![](/reverse_python/image/image_3.png "")

- 指定生成和本机python环境一样的pyd，比如都是x64的
- 修改生成目标的`dll`后缀设置为`pyd`

然后项目文件的管理：

- 不要把`.py`放项目的`Source Files`（`源文件`）filter下，不然会被当成项目源码干扰编译过程，可以创建一个单独的filter来放`.py`文件，比如我现在的管理方式：

![](/reverse_python/image/image_4.png "")



# 逆向的角度

- 尽可能恢复符号，借用别的有符号的pyd，用bindiff进行恢复
- 导出一部分结构体



`__Pyx_AddTraceback`对逆向有较大辅助作用

- [ ] 确认下还有哪些符号有利于逆向分析

## 代码逆向定位

都在slots里，所以是挨着的，确定了一个就能确定另一个。函数实现也是挨着的。

- `__pyx_pymod_create`
- `__pyx_pymod_exec_modulename`

```c
static PyModuleDef_Slot __pyx_moduledef_slots[] = {
  {Py_mod_create, (void*)},
  {Py_mod_exec, (void*)__pyx_pymod_exec_demo},
  {0, NULL}
};


// 在python的moduleobject.h里有定义
typedef struct PyModuleDef_Slot{
    int slot;
    void *value;
} PyModuleDef_Slot;

```








## pyx和py的区别

其实就算不使用cython的语法特性，同样的代码放在py和pyx里，cython编译出来的结果并不相同。



### `__pyx_pymod_exec_modulename`中的区别

#### 自定义函数的区别

```c
/*--- Execution code ---*/
```


其中自定义的函数部分，py生成的是

```c
__pyx_t_1 = __Pyx_CyFunction_New(&__pyx_mdef_4util_1func, 0, __pyx_n_s_func, NULL, __pyx_n_s_util, __pyx_d, ((PyObject *)__pyx_codeobj__2)); if (unlikely(!__pyx_t_1)) __PYX_ERR(0, 1, __pyx_L1_error)
  __Pyx_GOTREF(__pyx_t_1);
  if (PyDict_SetItem(__pyx_d, __pyx_n_s_func, __pyx_t_1) < 0) __PYX_ERR(0, 1, __pyx_L1_error)
  __Pyx_DECREF(__pyx_t_1); __pyx_t_1 = 0;
```


pyx生成的是

```c
  __pyx_t_1 = PyCFunction_NewEx(&__pyx_mdef_4util_1func, NULL, __pyx_n_s_util); if (unlikely(!__pyx_t_1)) __PYX_ERR(0, 1, __pyx_L1_error)
  __Pyx_GOTREF(__pyx_t_1);
  if (PyDict_SetItem(__pyx_d, __pyx_n_s_func, __pyx_t_1) < 0) __PYX_ERR(0, 1, __pyx_L1_error)
  __Pyx_DECREF(__pyx_t_1); __pyx_t_1 = 0;
```


看下定义

```c
#define PyCFunction_NewEx(ML, SELF, MOD) PyCMethod_New((ML), (SELF), (MOD), NULL)
PyAPI_FUNC(PyObject *) PyCMethod_New(PyMethodDef *, PyObject *,
                                     PyObject *, PyTypeObject *);
```




```c
static PyObject *__Pyx_CyFunction_New(PyMethodDef *ml, int flags, PyObject* qualname,
                                      PyObject *closure, PyObject *module, PyObject* globals, PyObject* code) {
    PyObject *op = __Pyx_CyFunction_Init(
        PyObject_GC_New(__pyx_CyFunctionObject, __pyx_CyFunctionType),
        ml, flags, qualname, closure, module, globals, code
    );
    if (likely(op)) {
        PyObject_GC_Track(op);
    }
    return op;
}
```




#### 其他区别

```c
/* FetchCommonType.proto */
static PyTypeObject* __Pyx_FetchCommonType(PyTypeObject* type);

/* CythonFunctionShared.proto */
#define __Pyx_CyFunction_USED 1
#define __Pyx_CYFUNCTION_STATICMETHOD  0x01
#define __Pyx_CYFUNCTION_CLASSMETHOD   0x02
#define __Pyx_CYFUNCTION_CCLASS        0x04
#define __Pyx_CyFunction_GetClosure(f)\
    (((__pyx_CyFunctionObject *) (f))->func_closure)
#define __Pyx_CyFunction_GetClassObj(f)\
    (((__pyx_CyFunctionObject *) (f))->func_classobj)
#define __Pyx_CyFunction_Defaults(type, f)\
    ((type *)(((__pyx_CyFunctionObject *) (f))->defaults))
#define __Pyx_CyFunction_SetDefaultsGetter(f, g)\
    ((__pyx_CyFunctionObject *) (f))->defaults_getter = (g)
typedef struct {
    PyCFunctionObject func;
#if PY_VERSION_HEX < 0x030500A0
    PyObject *func_weakreflist;
#endif
    PyObject *func_dict;
    PyObject *func_name;
    PyObject *func_qualname;
    PyObject *func_doc;
    PyObject *func_globals;
    PyObject *func_code;
    PyObject *func_closure;
    PyObject *func_classobj;
    void *defaults;
    int defaults_pyobjects;
    size_t defaults_size;  // used by FusedFunction for copying defaults
    int flags;
    PyObject *defaults_tuple;
    PyObject *defaults_kwdict;
    PyObject *(*defaults_getter)(PyObject *);
    PyObject *func_annotations;
} __pyx_CyFunctionObject;
static PyTypeObject *__pyx_CyFunctionType = 0;
#define __Pyx_CyFunction_Check(obj)  (__Pyx_TypeCheck(obj, __pyx_CyFunctionType))
static PyObject *__Pyx_CyFunction_Init(__pyx_CyFunctionObject* op, PyMethodDef *ml,
                                      int flags, PyObject* qualname,
                                      PyObject *self,
                                      PyObject *module, PyObject *globals,
                                      PyObject* code);
static CYTHON_INLINE void *__Pyx_CyFunction_InitDefaults(PyObject *m,
                                                         size_t size,
                                                         int pyobjects);
static CYTHON_INLINE void __Pyx_CyFunction_SetDefaultsTuple(PyObject *m,
                                                            PyObject *tuple);
static CYTHON_INLINE void __Pyx_CyFunction_SetDefaultsKwDict(PyObject *m,
                                                             PyObject *dict);
static CYTHON_INLINE void __Pyx_CyFunction_SetAnnotationsDict(PyObject *m,
                                                              PyObject *dict);
static int __pyx_CyFunction_init(void);

/* CythonFunction.proto */
static PyObject *__Pyx_CyFunction_New(PyMethodDef *ml,
                                      int flags, PyObject* qualname,
                                      PyObject *closure,
                                      PyObject *module, PyObject *globals,
                                      PyObject* code);
```






## __Pyx_InitGlobals —— 字面值初始化

包含大量变量初始化，这个变量呢，在python中其实是作为常量，或者说是字面值。比如python里的一个赋值`a = 1`，就会需要在__Pyx_InitGlobals里创建PyObject* PyInt_FromLong(1)，然后python对象才能被赋值成1。

顺着这些变量的交叉引用，能找到def变成成的c函数。





## __pyx_pymod_exec_模块名(PyObject *__pyx_pyinit_module)

会有个`PyModule_GetDict()`通过这个module获取字典，获得一个`dict`对象`__pyx_d`。

- 各种import，将import的结果保存在这个字典里。而且根据__Pyx_Import参数判断是直接import还是用了as当别名
- 全局变量，直接在这里`PyDict_SetItem()`赋值了。



### 一些例子

```c
// import array as arr
 __pyx_t_1 = __Pyx_Import(__pyx_n_s_array, 0, 0); if (unlikely(!__pyx_t_1)) __PYX_ERR(0, 1, __pyx_L1_error)
  __Pyx_GOTREF(__pyx_t_1);
  if (PyDict_SetItem(__pyx_d, __pyx_n_s_arr, __pyx_t_1) < 0) __PYX_ERR(0, 1, __pyx_L1_error)
  __Pyx_DECREF(__pyx_t_1); __pyx_t_1 = 0;


// 全局变量赋值
  /* "util.py":7
 * DO_NOT_SET_FLAG = 0
 * SET_FLAG = 1
 * RAM_SIZE = 1024             # <<<<<<<<<<<<<<
 * RF_SIZE = 32
 * 
 */
  if (PyDict_SetItem(__pyx_d, __pyx_n_s_RAM_SIZE, __pyx_int_1024) < 0) __PYX_ERR(0, 7, __pyx_L1_error)

 
 // def 创建函数对象
  /* "util.py":11
 * 
 * 
 * def __init_memory():             # <<<<<<<<<<<<<<
 *     global mem
 *     mem = arr.array('I', [])
 */
  __pyx_t_1 = __Pyx_CyFunction_New(&__pyx_mdef_4util_1__init_memory, 0, __pyx_n_s_init_memory, NULL, __pyx_n_s_util, __pyx_d, ((PyObject *)__pyx_codeobj__15)); if (unlikely(!__pyx_t_1)) __PYX_ERR(0, 11, __pyx_L1_error)
  __Pyx_GOTREF(__pyx_t_1);
  if (PyDict_SetItem(__pyx_d, __pyx_n_s_init_memory, __pyx_t_1) < 0) __PYX_ERR(0, 11, __pyx_L1_error)
  __Pyx_DECREF(__pyx_t_1); __pyx_t_1 = 0; 

 
 // 全局会执行的代码
   /* "util.py":609
 * 
 * print('[-- Loaded --]')             # <<<<<<<<<<<<<<
 */
  __pyx_t_1 = __Pyx_PyObject_Call(__pyx_builtin_print, __pyx_tuple__95, NULL); if (unlikely(!__pyx_t_1)) __PYX_ERR(0, 609, __pyx_L1_error)
  __Pyx_GOTREF(__pyx_t_1);
  __Pyx_DECREF(__pyx_t_1); __pyx_t_1 = 0; 
```






## 从指针获取对象

`memoryview()`对于支持缓冲区协议的数据，可以允许Python代码访问。

[python - Is it possible to dereference variable id's? - Stack Overflow](https://stackoverflow.com/questions/15011674/is-it-possible-to-dereference-variable-ids)

介绍了`gc.get_objects`，`_ctypes.PyObj_FromPtr(obj_id)`之类的方法。





# PYC篇

---

## 常规逆向思路

先利其器：

[https://github.com/rocky/python-uncompyle6](https://github.com/rocky/python-uncompyle6)

[https://github.com/SycloverTeam/pycdc/](https://github.com/SycloverTeam/pycdc/)（pycdc环境配置简单，可自行编译）

010Editor



pyc有时候可以直接import，import了之后导入了一些内容，可以进行侧信道攻击。

比如VNCTF2022的BabyMaze，本意应该是看dis内容，还原`_map`，以及`wsad`控制在迷宫中移动。



但我们可以直接import

![](/reverse_python/image/image_5.png "")



可以进行个patch

![](/reverse_python/image/image_6.png "")



可以偷窥下`_map`:

![](/reverse_python/image/image_7.png "")

很大程度上能减轻逆向工作量



## 混淆

`.pyc`文件，可以加混淆，以及其他保护手段，加大逆向难度。

- [ ] pyc去掉符号名称（换成空格），样本：

[py10.pyc](https://www.wolai.com/2QAe1xSxUMeqyq6u2oXa4f)



# Opcode的处理

主要是dis模块和marshal模块



