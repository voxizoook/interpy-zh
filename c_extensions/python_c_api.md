# Python/C API

[Python/C API](https://docs.python.org/2/c-api/)可能是被最广泛使用的方法。它不仅简单，而且可以在C代码中操作你的Python对象。

这种方法需要以特定的方式来编写C代码以供Python去调用它。所有的Python对象都被表示为一种叫做PyObject的结构体，并且```Python.h```头文件中提供了各种操作它的函数。例如，如果PyObject表示为PyListType(列表类型)时，那么我们便可以使用```PyList_Size()```函数来获取该结构的长度，类似Python中的```len(list)```函数。大部分对Python原生对象的基础函数和操作在```Python.h```头文件中都能找到。

示例

编写一个C扩展，添加所有元素到一个Python列表(所有元素都是数字)

来看一下我们要实现的效果，这里演示了用Python调用C扩展的代码
```Python
#Though it looks like an ordinary python import, the addList module is implemented in C
import addList

l = [1,2,3,4,5]
print "Sum of List - " + str(l) + " = " +  str(addList.add(l))

```

上面的代码和普通的Python文件并没有什么分别，导入并使用了另一个叫做```addList```的Python模块。唯一差别就是这个模块并不是用Python编写的，而是C。

接下来我们看看如何用C编写```addList```模块，这可能看起来有点让人难以接受，但是一旦你了解了这之中的各种组成，你就可以一往无前了。

```C
//Python.h has all the required function definitions to manipulate the Python objects
#include <python.h>

//This is the function that is called from your python code
static PyObject* addList_add(PyObject* self, PyObject* args){ //M1：addList为module名， add为函数名

    PyObject * listObj;
	long length, elem;
	int i, sum;
    //The input arguments come as a tuple, we parse the args to get the various variables
    //In this case it's only one list variable, which will now be referenced by listObj
    if (! PyArg_ParseTuple( args, "O", &listObj ))
        return NULL;
    
	length= PyList_Size(listObj);

    //iterate over all the elements
	sum=0;
    for (i = 0; i < length; i++) {
        //get an element out of the list - the element is also a python objects
        PyObject* temp = PyList_GetItem(listObj, i);
        //we know that object represents an integer - so convert it into C long
        elem = PyLong_AsLong(temp);
        sum += elem;
    }

    //value returned back to python code - another python object
    //build value here converts the C long to a python integer
    return Py_BuildValue("i", sum);

}

//This is the docstring that corresponds to our 'add' function.
static char addList_docs[] =                //M2：此处addList为module名，需与M1处对应
"add(  ): add all elements of the list\n";

/* This table contains the relavent info mapping -
   <function-name in python module>, <actual-function>,
   <type-of-args the function expects>, <docstring associated with the function>
 */
static PyMethodDef addList_funcs[] = {    ////此处addList为module名，需与M1处对应
    {"add", addList_add, METH_VARARGS, addList_docs},    //此处add为函数名,需与M1处add对应，addList_add与M1处对应
    {NULL, NULL, 0, NULL}       //此行为结束标识

};

static struct PyModuleDef addListModule = {     //此处为M1处的Module名+Module
    PyModuleDef_HEAD_INIT,
    "addList",   /* name of module */
    addList_docs, /* module documentation, may be NULL */
    -1,       /* size of per-interpreter state of the module,
                 or -1 if the module keeps state in global variables. */
    addList_funcs
};

/*
   addList is the module name, and this is the initialization block of the module.
   <desired module name>, <the-info-table>, <module's-docstring>
 */

PyMODINIT_FUNC
PyInit_addList(void)
{
    return PyModule_Create(&addListModule);
}

```

逐步解释
- ```Python.h```头文件中包含了所有需要的类型(Python对象类型的表示)和函数定义(对Python对象的操作)
- 接下来我们编写将要在Python调用的函数, 函数传统的命名方式由{模块名}_{函数名}组成，所以我们将其命名为```addList_add```   
- 然后填写想在模块内实现函数的相关信息表，每行一个函数，以空行作为结束
- 最后的模块初始化块签名为```PyMODINIT_FUNC init{模块名}```。

函数```addList_add```接受的参数类型为PyObject类型结构(同时也表示为元组类型，因为Python中万物皆为对象，所以我们先用PyObject来定义)。传入的参数则通过```PyArg_ParseTuple()```来解析。第一个参数是被解析的参数变量。第二个参数是一个字符串，告诉我们如何去解析元组中每一个元素。字符串的第n个字母正是代表着元组中第n个参数的类型。例如，"i"代表整形，"s"代表字符串类型, "O"则代表一个Python对象。接下来的参数都是你想要通过```PyArg_ParseTuple()```函数解析并保存的元素。这样参数的数量和模块中函数期待得到的参数数量就可以保持一致，并保证了位置的完整性。例如，我们想传入一个字符串，一个整数和一个Python列表，可以这样去写
```C
int n;
char *s;
PyObject* list;
PyArg_ParseTuple(args, "siO", &n, &s, &list);

```

在这种情况下，我们只需要提取一个列表对象，并将它存储在```listObj```变量中。然后用列表对象中的```PyList_Size()```函数来获取它的长度。就像Python中调用```len(list)```。

现在我们通过循环列表，使用```PyList_GetItem(list, index)```函数来获取每个元素。这将返回一个```PyObject*```对象。既然Python对象也能表示```PyIntType```，我们只要使用```PyInt_AsLong(PyObj *)```函数便可获得我们所需要的值。我们对每个元素都这样处理，最后再得到它们的总和。

总和将被转化为一个Python对象并通过```Py_BuildValue()```返回给Python代码，这里的i表示我们要返回一个Python整形对象。

现在我们已经编写完C模块了。将下列代码保存为```setup.py```
```Python
#build the modules

from distutils.core import setup, Extension

setup(name='addList', version='1.0',  \
      ext_modules=[Extension('addList', ['adder.c'])])
```

并且运行
```Shell
python setup.py install

```

现在应该已经将我们的C文件编译安装到我们的Python模块中了。

在一番辛苦后，让我们来验证下我们的模块是否有效
```Python
#module that talks to the C code
import addList

l = [1,2,3,4,5]
print "Sum of List - " + str(l) + " = " +  str(addList.add(l))
```

输出结果如下
```
Sum of List - [1, 2, 3, 4, 5] = 15

```

如你所见，我们已经使用Python.h API成功开发出了我们第一个Python C扩展。这种方法看似复杂，但你一旦习惯，它将变的非常有效。

Python调用C代码的另一种方式便是使用[Cython](http://cython.org/)让Python编译的更快。但是Cython和传统的Python比起来可以将它理解为另一种语言，所以我们就不在这里过多描述了。

#Note By WHK
在生成动态文件时需注意以下几点：
1.在VS2010中将项目配置中的配置属性->VC++目录中包含目录改为python3下的include，库目录改为python3下的libs目录（不是Lib目录）；链接器->输入中的附加依赖项修改为libs目录中的python37.lib
可参考：https://www.cnblogs.com/chengxuyuancc/p/6374239.html
2.在VS2010中取消编译头，C/C++->预编译头->不使用预编译头
可参考：https://blog.csdn.net/mj511099781/article/details/12994925
3.如遇到缺少inttypes.h时，可下载之，保存到VS2010下的VC目录下，includes中
4.在编译时出现重复定义的情况可以将重复的部分注释掉
5.生成DLL文件后，将后缀改为.pyd即可在python中import
6.还需要注意的一点就是64位的python需要配合64位的VS使用，两者需保持一致。调整VS2010为x64版本可参考以下链接
可参考：https://jingyan.baidu.com/article/7082dc1c3f3f41e40a89bd81.html
