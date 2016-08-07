---
title: 使用Python调用C函数
layout: post
---

> 脚本语言经常会作为一种“胶水语言”来使用，来完成对各种外来组建的整合控制功能，如此能够将外部的功能简单的进行导入成为了一种脚本语言是否好用的关键。之前一直使用Lua做类似的工作，其优点就是轻量级，可以直接使用直接编译的标准so，缺点主要是功能过于有限，不适合完成比较复杂的工作。而由于Raspberry PI主要使用Python作为核心编程语言，这里就尝试使用Python来完成整合任务，其中关键的步骤就是使用Python调用C的函数。以下的方法仅适用于Python2。

#### 1. 直接调用C的库函数

> 直接使用ctypes可以直接引入C函数库，然后直接调用即可，代码如下
>
```python
from ctypes import *;
libc = cdll.LoadLibrary('/usr/lib/libc.dylib');
libc.printf("Hello World!\n");
```

#### 2. 编写可用Python调用的C函数

> 与编写Lua可调用的C函数类似，Python可调用的C函数也为特定的模式，并通过编译为so的方式提供给Python使用，以下为简单的一个例子
>
```c++
#include <Python.h>
>
PyObject* wrap_demo_func(PyObject* self, PyObject* args) {
    int a, b;
    if (!PyArg_ParseTuple(args, "ii", &a, &b)) {
        return NULL;
    }
    return Py_BuildValue("i", a + b);
}
>
static PyMethodDef demoMethods[] = {
    {"demo_func", wrap_demo_func, METH_VARARGS, "demo func"},
    {NULL, NULL}
};
>
extern "C" void initdemo() {
    Py_InitModule("demo", demoMethods);
}
```
> 编译的命令如下
>
```bash
g++ -g -Wall -I /usr/include/python2.7/ -fpic --shared demo.cc -o demo.so
```
> 运行的Python脚本如下
>
```python
import demo;
print demo.demo_func(1, 2);
```
> 运行后会输出3

#### 3. 编写Python调用的C函数中的一些处理

> 调用一个C++类中的方法
>
```c++
#include <Python.h>
class Demo {
public:
    void func() {
        printf("Hello World!\n");
    }
};
static void del_demo(PyObject *obj) {
    delete (Demo*) PyCapsule_GetPointer(obj, "Demo");
}
PyObject* wrap_create_demo(PyObject* self, PyObject* args) {
    Demo* demo = new Demo();
    return PyCapsule_New(demo, "Demo", del_demo);
}
>
PyObject* wrap_demo_func(PyObject* self, PyObject* args) {
    Demo* demo;
    PyObject* p_demo;
    if (!PyArg_ParseTuple(args,"O", &p_demo)) {
        return NULL;
    }
    if (!(demo = (Demo*) PyCapsule_GetPointer(p_demo, "Demo"))) {
        return NULL;
    }
    demo->func();
    return Py_BuildValue("");
}
>
static PyMethodDef demoMethods[] = {
    {"create_demo", wrap_create_demo, METH_VARARGS, "create demo"},
    {"demo_func", wrap_demo_func, METH_VARARGS, "demo func"},
    {NULL, NULL}
};
>
extern "C" void initdemo() {
    Py_InitModule("demo", demoMethods);
}
```
> 运行的Python脚本如下
>
```python
import demo;
d = demo.create_demo();
demo.demo_func(d);
```
> 运行后输出Hello World!
