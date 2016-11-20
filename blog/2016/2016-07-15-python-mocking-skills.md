categories:
- 技术

tags:
- Python
- 测试

title: Python Mocking 技巧
---


## Mock 全局符号

这里的 `符号` 包括：模块、类、类的实例、函数等。

代码：

```python
# example.py

import the_module
from module import TheClass, the_instance, the_func


def foo():
    return the_module.constant


def bar():
    return TheClass.class_method()


def wow():
    return the_instance.attribute


def sigh():
    return the_func()
```

测试：

```python
>>> from mock import MagicMock, patch
>>> from example import foo, bar, wow, sigh
>>> @patch('example.the_module', constant='Foo')
... def test_foo(mock_the_module):
...     assert foo() == 'Foo'
...
>>> test_foo()
>>> @patch('example.TheClass', class_method=MagicMock(return_value='Bar'))
... def test_bar(mock_the_class):
...     assert bar() == 'Bar'
...     mock_the_class.method.assert_called_once()
...
>>> test_bar()
>>> @patch('example.the_instance', attribute='Wow')
... def test_wow(mock_the_instance):
...     assert wow() == 'Wow'
...
>>> test_wow()
>>> @patch('example.the_func', return_value='Sigh')
... def test_sigh(mock_the_func):
...     assert sigh() == 'Sigh'
...     mock_the_func.assert_called_once()
...
>>> test_sigh()
```


## Mock 局部实例化的类

代码：

```python
# example.py

from module import TheClass


def foo():
    return TheClass()
```

测试：

```python
>>> from mock import patch
>>> from example import foo
>>> @patch('example.TheClass', return_value='Foo')
... def test(mock_class):
...     assert foo() == 'Foo'
...
>>> test()
```


## Mock 局部实例化的类的属性

代码：

```python
# example.py

from module import TheClass


def foo():
    the_class = TheClass()
    return the_class.method()
```

测试：

```python
>>> from mock import MagicMock, patch
>>> from example import foo
>>> @patch('example.TheClass', return_value=MagicMock(method=MagicMock(return_value='Foo')))
... def test(mock_class):
...     assert foo() == 'Foo'
...
>>> test()
```

如果 `method` 的返回值比较复杂，这样写可读性更高：

```python
>>> from mock import MagicMock, patch
>>> from example import foo
>>> @patch('example.TheClass')
... def test(mock_class):
...     method = MagicMock(return_value={'value': 'Foo'})
...     mock_class.return_value = MagicMock(method=method)
...     result = foo()
...     assert result['value'] == 'Foo'
...
>>> test()
```


## Mock 类的属性

代码：

```python
# example.py

class TheClass(object):

    def foo(self):
       	return self.bar()

    def bar(self):
        return 'thing'
```

测试：

```python
>>> from mock import patch
>>> from example import TheClass
>>> @patch.object(TheClass, 'bar', return_value='Foo')
... def test(mock_bar):
...     the_class = TheClass()
...     assert the_class.foo() == 'Foo'
...     mock_bar.assert_called_once()
...
>>> test()
```


## Mock 局部导入的模块

代码：

```python
# example.py

def foo():
    from module import constant
    return constant


def bar():
    import module
    return module.constant
```

测试：

```python
>>> from mock import MagicMock, patch
>>> from example import foo, bar
>>> @patch.dict('sys.modules', module=MagicMock(constant='Foo'))
... def test():
...     assert foo() == 'Foo'
...     assert bar() == 'Foo'
...
>>> test()
```


## Mock 具有特殊属性的对象

通常在 Mock 一个简单对象的时候，我们会使用 `类 Mock`（或者它的 `子类 MagicMock`）。但是 `类 Mock` 本身带有一些 [可选参数][1]，如果待 Mock 对象恰好具有一个属性，该属性与某个可选参数同名，我们在此定义这样的属性为 `特殊属性`。

具有上述 `特殊属性` 的对象，无法通过 `类 Mock` 直接创建出来。例如：

```python
>>> from mock import Mock
>>> m = Mock(name='foo', value='bar')
>>> m.name
<Mock name='foo.name' id='4554622624'>
>>> m.value
'bar'
```

可以看出，name 符合上述对 `特殊属性` 的定义，创建对象 m 时：

1. name 并没有被当做 m 的属性，因此 m.name 的值并不是预期的 foo
2. value 被当做了 m 的属性，因此 m.value 的值是预期的 bar

想要 name 被当做 m 的属性，有两种方式：

1. 直接赋值覆盖

    ```python
    >>> from mock import Mock
    >>> m = Mock(value='bar')
    >>> m.name = 'foo'
    >>> m.name
    'foo'
    >>> m.value
    'bar'
    ```

2. 借助 [configure_mock 方法][2]（个人更倾向于这种方式）

    ```python
    >>> m = Mock()
    >>> m.configure_mock(name='foo', value='bar')
    >>> m.name
    'foo'
    >>> m.value
    'bar'
    ```


[1]: https://docs.python.org/3/library/unittest.mock.html#unittest.mock.Mock
[2]: https://docs.python.org/3/library/unittest.mock.html#unittest.mock.Mock.configure_mock
