Title: Python调用C++程序备忘笔记
Date: 2015-09-02
Modified: 2015-09-02
Category: Language
Tags: python, c++
Slug: Python调用C++程序备忘笔记
Author: littlewhite

[TOC]

Python的优点是开发效率高，使用方便，C++则是运行效率高，这两者可以相辅相成，不管是在Python项目中嵌入C++代码，或是在C++项目中用Python实现外围功能，都可能遇到Python调用C++模块的需求，下面列举出集中c++代码导出成Python接口的几种基本方法

## 原生态导出
Python解释器就是用C实现，因此只要我们的C++的数据结构能让Python认识，理论上就是可以被直接调用的。我们实现test1.cpp如下

    #include <Python.h>
    
    int Add(int x, int y)
    {
        return x + y;
    }
    
    int Del(int x, int y)
    {
        return x - y;
    }
    
    PyObject* WrappAdd(PyObject* self, PyObject* args)
    {
        int x, y;
        if (!PyArg_ParseTuple(args, "ii", &x, &y))
        {
            return NULL;
        }
        return Py_BuildValue("i", Add(x, y));
    }
    
    PyObject* WrappDel(PyObject* self, PyObject* args)
    {
        int x, y;
        if (!PyArg_ParseTuple(args, "ii", &x, &y))
        {
            return NULL;
        }
        return Py_BuildValue("i", Del(x, y));
    }
    static PyMethodDef test_methods[] = {
        {"Add", WrappAdd, METH_VARARGS, "something"},
        {"Del", WrappDel, METH_VARARGS, "something"},
        {NULL, NULL}
    };
    
    extern "C"
    void inittest1()
    {
        Py_InitModule("test1", test_methods);
    }
    
编译命令如下

    g++ -fPIC -shared test1.cpp -I/usr/include/python2.6 -o test1.so

运行Python解释器，测试如下

    >>> import test1
    >>> test1.Add(1,2)
    3

这里要注意一下几点

* 如果生成的动态库名字为test1，则源文件里必须有inittest1这个函数，且Py_InitModule的第一个参数必须是“test1”，否则Python导入模块会失败
* 如果是cpp源文件，inittest1函数必须用extern "C"修饰，如果是c源文件，则不需要。原因是Python解释器在导入库时会寻找initxxx这样的函数，而C和C++对函数符号的编码方式不同，C++在对函数符号进行编码时会考虑函数长度和参数类型，具体可以通过`nm test1.so`查看函数符号，c++filt工具可通过符号反解出函数原型

## 通过boost实现
我们使用和上面同样的例子，实现test2.cpp如下

    #include <boost/python/module.hpp>
    #include <boost/python/def.hpp>
    using namespace boost::python;
    
    int Add(const int x, const int y)
    {
        return x + y;
    }
    
    int Del(const int x, const int y)
    {
        return x - y;
    }
    
    BOOST_PYTHON_MODULE(test2)
    {
        def("Add", Add);
        def("Del", Del);
    }

其中BOOST\_PYTHON\_MODULE的参数为要导出的模块名字

编译命令如下

    g++ test2.cpp -fPIC -shared -o test2.so -I/usr/include/python2.6 -I/usr/local/include -L/usr/local/lib -lboost_python

**注意： 编译时需要指定boost头文件和库的路径，我这里分别是/usr/local/include和/usr/local/lib**

或者通过setup.py导出模块

    #!/usr/bin/env python
    from distutils.core import setup
    from distutils.extension import Extension
    
    setup(name="PackageName",
        ext_modules=[
            Extension("test2", ["test2.cpp"],
            libraries = ["boost_python"])
        ])
        
Extension的第一个参数为模块名，第二个参数为文件名

执行如下命令

    python setup.py build

这时会生成build目录，找到里面的test2.so，并进入同一级目录，验证如下

    >>> import test2
    >>> test2.Add(1,2)
    3
    >>> test2.Del(1,2)
    -1
    
## 导出类
test3.cpp实现如下

    #include <boost/python.hpp>
    using namespace boost::python;
    
    class Test
    {
    public:
        int Add(const int x, const int y)
        {
            return x + y;
        }
    
        int Del(const int x, const int y)
        {
            return x - y;
        }
    };
    
    BOOST_PYTHON_MODULE(test3)
    {
        class_<Test>("Test")
            .def("Add", &Test::Add)
            .def("Del", &Test::Del);
    }

注意：BOOST_PYTHON_MODULE里的.def使用方法有点类似Python的语法，等同于
    
    class_<Test>("Test").def("Add", &Test::Add);
    class_<Test>("Test").def("Del", &Test::Del);
    
编译命令如下

    g++ test3.cpp -fPIC -shared -o test3.so -I/usr/include/python2.6 -I/usr/local/include/boost -L/usr/local/lib -lboost_python

测试如下

    >>> import test3
    >>> test = test3.Test()
    >>> test.Add(1,2)
    3
    >>> test.Del(1,2)
    -1

## 导出变参函数
test4.cpp实现如下

    #include <boost/python.hpp>
    using namespace boost::python;
    
    class Test
    {
    public:
        int Add(const int x, const int y, const int z = 100)
        {
            return x + y + z;
        }
    };
    
    int Del(const int x, const int y, const int z = 100)
    {
        return x - y - z;
    }
    
    BOOST_PYTHON_MEMBER_FUNCTION_OVERLOADS(Add_member_overloads, Add, 2, 3)
    BOOST_PYTHON_FUNCTION_OVERLOADS(Del_overloads, Del, 2, 3)
    
    BOOST_PYTHON_MODULE(test4)
    {
        class_<Test>("Test")
            .def("Add", &Test::Add, Add_member_overloads(args("x", "y", "z"), "something"));
        def("Del", Del, Del_overloads(args("x", "y", "z"), "something"));
    }
    
这里Add和Del函数均采用了默认参数，Del为普通函数，Add为类成员函数，这里分别调用了不同的宏，宏的最后两个参数分别代表函数的最少参数个数和最多参数个数

编译命令如下

    g++ test4.cpp -fPIC -shared -o test4.so -I/usr/include/python2.6 -I/usr/local/ include/boost -L/usr/local/lib -lboost_python

测试如下

    >>> import test4
    >>> test = test4.Test()
    >>> print test.Add(1,2)
    103
    >>> print test.Add(1,2,z=3)
    6
    >>> print test4.Del(1,2)
    -1
    >>> print test4.Del(1,2,z=3)
    -1
    
## 导出带Python对象的接口
既然是导出为Python接口，调用者难免会使用Python特有的数据结构，比如tuple,list,dict，由于原生态方法太麻烦，这里只记录boost的使用方法，假设要实现如下的Python函数功能

    def Square(list_a)
    {
        return [x * x for x in list_a]
    }
    
即对传入的list每个元素计算平方，返回list类型的结果

代码如下

    #include <boost/python.hpp>
    
    boost::python::list Square(boost::python::list& data)
    {
        boost::python::list ret;
        for (int i = 0; i < len(data); ++i)
        {
            ret.append(data[i] * data[i]);
        }
    
        return ret;
    }
    
    BOOST_PYTHON_MODULE(test5)
    {
        def("Square", Square);
    }
    
编译命令如下

    g++ test5.cpp -fPIC -shared -o test5.so -I/usr/include/python2.6 -I/usr/local/include/boost -L/usr/local/lib -lboost_python

测试如下

    >>> import test5
    >>> test5.Square([1,2,3])
    [1, 4, 9]
    
boost实现了boost::python::tuple, boost::python::list, boost::python::dict这几个数据类型，使用方法基本和Python保持一致，具体方法可以查看boost头文件里的boost/python/tuple.hpp及其它对应文件

另外比较常用的一个函数是`boost::python::make_tuple()`，使用方法如下

    boost::python::tuple t = boost::python::make_tuple(a, b, c);


