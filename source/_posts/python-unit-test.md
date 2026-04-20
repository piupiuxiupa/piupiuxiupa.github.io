---
title: Python 单元测试完全指南：unittest 框架从入门到精通
date: 2026-04-20 14:52:16
tags: python
categories: Notes
---

## 目录

- [目录](#目录)
- [1. 单元测试概述](#1-单元测试概述)
  - [1.1 什么是单元测试](#11-什么是单元测试)
  - [1.2 为什么单元测试很重要](#12-为什么单元测试很重要)
  - [1.3 Python 中的测试工具生态](#13-python-中的测试工具生态)
- [2. unittest 框架基础](#2-unittest-框架基础)
  - [2.1 框架核心组件](#21-框架核心组件)
  - [2.2 四大核心概念详解](#22-四大核心概念详解)
  - [2.3 测试执行生命周期](#23-测试执行生命周期)
- [3. 编写第一个测试](#3-编写第一个测试)
  - [3.1 被测代码](#31-被测代码)
  - [3.2 编写测试](#32-编写测试)
  - [3.3 运行测试](#33-运行测试)
  - [3.4 理解测试输出](#34-理解测试输出)
- [4. 断言方法完整列表](#4-断言方法完整列表)
  - [4.1 基本断言](#41-基本断言)
  - [4.2 相等性断言](#42-相等性断言)
  - [4.3 比较断言](#43-比较断言)
  - [4.4 异常断言](#44-异常断言)
  - [4.5 警告断言](#45-警告断言)
  - [4.6 类型与身份断言](#46-类型与身份断言)
  - [4.7 集合断言](#47-集合断言)
  - [4.8 字符串断言](#48-字符串断言)
  - [4.9 浮点数断言](#49-浮点数断言)
  - [4.10 自定义失败消息与日志断言](#410-自定义失败消息与日志断言)
- [5. 测试夹具](#5-测试夹具)
  - [5.1 setUp 和 tearDown](#51-setup-和-teardown)
  - [5.2 setUpClass 和 tearDownClass](#52-setupclass-和-teardownclass)
  - [5.3 setUpModule 和 tearDownModule](#53-setupmodule-和-teardownmodule)
  - [5.4 夹具执行顺序总结](#54-夹具执行顺序总结)
  - [5.5 夹具中的异常处理](#55-夹具中的异常处理)
- [6. 测试套件](#6-测试套件)
  - [6.1 手动构建测试套件](#61-手动构建测试套件)
  - [6.2 使用 TestLoader 加载测试](#62-使用-testloader-加载测试)
  - [6.3 嵌套测试套件](#63-嵌套测试套件)
  - [6.4 自定义 TestRunner](#64-自定义-testrunner)
- [7. 测试发现](#7-测试发现)
  - [7.1 命令行发现](#71-命令行发现)
  - [7.2 编程式发现](#72-编程式发现)
  - [7.3 项目结构最佳实践](#73-项目结构最佳实践)
- [8. 跳过测试和预期失败](#8-跳过测试和预期失败)
  - [8.1 跳过测试的装饰器](#81-跳过测试的装饰器)
  - [8.2 预期失败](#82-预期失败)
  - [8.3 动态跳过](#83-动态跳过)
- [9. 子测试](#9-子测试)
  - [9.1 基本用法](#91-基本用法)
  - [9.2 子测试与参数化](#92-子测试与参数化)
  - [9.3 子测试的注意事项](#93-子测试的注意事项)

---

## 1. 单元测试概述

### 1.1 什么是单元测试

单元测试（Unit Testing）是软件开发中的一种测试方法，它针对程序中最小的可测试单元进行检查和验证。在 Python 中，这个"单元"通常是一个函数、一个方法或者一个类。

单元测试的核心思想是**隔离**：把代码的每一部分独立出来，确认它按照预期工作。就好比造一辆汽车，你需要先单独测试每个零件（螺丝、发动机、刹车片），确认它们各自合格，然后才能组装成整车。

一个典型的单元测试流程如下：

```
编写代码 → 编写测试 → 运行测试 → 检查结果 → 修复问题 → 重新测试
```

来看一个最简单的例子。假设你有这样一个函数：

```python
def add(a, b):
    """两数相加"""
    return a + b
```

对应的单元测试就是：

```python
import unittest


class TestAdd(unittest.TestCase):

    def test_add_positive_numbers(self):
        result = add(2, 3)
        self.assertEqual(result, 5)

    def test_add_negative_numbers(self):
        result = add(-1, -2)
        self.assertEqual(result, -3)

    def test_add_mixed_numbers(self):
        result = add(5, -3)
        self.assertEqual(result, 2)


if __name__ == '__main__':
    unittest.main()
```

### 1.2 为什么单元测试很重要

单元测试不是可选项，它是专业开发流程中不可或缺的一环。以下是它带来的核心价值：

**快速定位 Bug**

当代码修改后引入了错误，单元测试会立刻告诉你哪个函数出了问题，而不是等到整个系统运行时才崩溃。

```python
# 一个有 bug 的函数
def divide(a, b):
    return a / b  # 没有处理 b=0 的情况


# 测试会暴露这个问题
class TestDivide(unittest.TestCase):

    def test_divide_by_zero(self):
        with self.assertRaises(ZeroDivisionError):
            divide(10, 0)
```

**为重构提供安全网**

当你需要重构代码时，已有的测试确保行为不变。你可以大胆地修改内部实现，只要测试全部通过，外部行为就是正确的。

```python
# 重构前
def factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n - 1)


# 重构后（改成迭代实现）
def factorial(n):
    result = 1
    for i in range(2, n + 1):
        result *= i
    return result


# 两种实现对应的测试完全相同
class TestFactorial(unittest.TestCase):

    def test_factorial(self):
        self.assertEqual(factorial(0), 1)
        self.assertEqual(factorial(1), 1)
        self.assertEqual(factorial(5), 120)
        self.assertEqual(factorial(10), 3628800)
```

**充当活的文档**

好的测试用例本身就是最好的使用文档。看一眼测试，就能知道函数在各种输入下的预期行为。

```python
class TestStringProcessor(unittest.TestCase):
    """这些测试本身就是 StringProcessor 的用法说明"""

    def test_strip_and_lower(self):
        """字符串去除空格并转为小写"""
        result = process_string("  Hello World  ")
        self.assertEqual(result, "hello world")

    def test_empty_string(self):
        """空字符串应返回空字符串"""
        result = process_string("")
        self.assertEqual(result, "")

    def test_none_input_raises(self):
        """None 输入应抛出 ValueError"""
        with self.assertRaises(ValueError):
            process_string(None)
```

**提高代码设计质量**

当你发现某个函数很难测试，通常意味着它的设计有问题：耦合太紧、职责太多、或者依赖太复杂。编写测试倒逼你写出更好的代码。

**支持持续集成**

自动化测试是 CI/CD 流水线的基础。每次提交代码自动运行全部测试，保证代码库始终处于可发布状态。

```yaml
# .github/workflows/test.yml 示例
name: Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install -r requirements.txt
      - run: python -m unittest discover -s tests
```

### 1.3 Python 中的测试工具生态

Python 有多种测试框架，各有特色：

| 框架 | 特点 | 适用场景 |
|------|------|----------|
| **unittest** | Python 标准库自带，xUnit 风格 | 标准项目，不需要额外依赖 |
| **pytest** | 语法简洁，插件丰富 | 追求开发效率的项目 |
| **doctest** | 在文档字符串中写测试 | 简单示例验证 |
| **nose2** | unittest 的扩展 | 需要 unittest 增强功能 |
| **robot framework** | 关键字驱动，支持 ATDD | 验收测试 |

本指南专注于 **unittest**，因为它是 Python 内置框架，零依赖，且在企业环境中使用广泛。

---

## 2. unittest 框架基础

### 2.1 框架核心组件

unittest 框架（有时被称为 PyUnit）的灵感来源于 Kent Beck 的 Smalltalk 测试框架和 JUnit。它从 Python 2.1 开始以 `unittest` 模块的形式成为标准库的一部分。

整个框架由以下核心组件构成：

```
unittest 框架架构
├── TestCase        # 测试用例的基类，所有测试类都继承它
├── TestSuite       # 测试套件，组织多个测试用例
├── TestLoader      # 测试加载器，自动发现和加载测试
├── TestRunner      # 测试运行器，执行测试并输出结果
├── TestResult      # 测试结果，记录通过、失败、错误等信息
└── TestFixture     # 测试夹具，提供 setUp/tearDown 等机制
```

### 2.2 四大核心概念详解

**TestCase（测试用例）**

`TestCase` 是最核心的类。你编写的每个测试类都必须继承它。它提供了断言方法、测试夹具钩子等基础设施。

```python
import unittest


class MyFirstTest(unittest.TestCase):
    """所有测试类都继承自 unittest.TestCase"""

    def test_something(self):
        """测试方法必须以 test_ 开头"""
        self.assertTrue(True)

    def test_another_thing(self):
        """每个 test_ 方法都是一个独立的测试用例"""
        self.assertEqual(1 + 1, 2)
```

**TestSuite（测试套件）**

`TestSuite` 用于将多个测试用例组合在一起，可以按逻辑分组运行。

```python
import unittest


class TestLogin(unittest.TestCase):

    def test_login_success(self):
        self.assertTrue(True)


class TestLogout(unittest.TestCase):

    def test_logout_success(self):
        self.assertTrue(True)


# 创建测试套件
suite = unittest.TestSuite()
suite.addTest(TestLogin('test_login_success'))
suite.addTest(TestLogout('test_logout_success'))

# 运行套件
runner = unittest.TextTestRunner(verbosity=2)
runner.run(suite)
```

**TestLoader（测试加载器）**

`TestLoader` 负责根据各种规则自动发现和加载测试用例。

```python
import unittest

# 从单个类加载所有测试
loader = unittest.TestLoader()
suite = loader.loadTestsFromTestCase(TestLogin)

# 从模块加载所有测试
import my_test_module
suite = loader.loadTestsFromModule(my_test_module)

# 按名称加载
suite = loader.loadTestsFromName('my_test_module.TestLogin.test_login_success')

# 自动发现
suite = loader.discover(start_dir='tests', pattern='test_*.py')
```

**TestRunner（测试运行器）**

`TestRunner` 负责执行测试并展示结果。最常用的是 `TextTestRunner`。

```python
import unittest

# 详细输出
runner = unittest.TextTestRunner(verbosity=2)

# 运行测试
result = runner.run(suite)

# 检查结果
print(f"测试总数: {result.testsRun}")
print(f"成功: {result.testsRun - len(result.failures) - len(result.errors)}")
print(f"失败: {len(result.failures)}")
print(f"错误: {len(result.errors)}")
```

### 2.3 测试执行生命周期

理解测试执行的顺序对编写高质量的测试非常重要。

```
程序启动
│
├── setUpModule()          ← 整个模块级别的前置操作
│
│   ┌── setUpClass()       ← 测试类级别的前置操作
│   │
│   │   ┌── setUp()        ← 单个测试方法的前置操作
│   │   ├── test_method_1()
│   │   └── tearDown()     ← 单个测试方法的后置操作
│   │
│   │   ┌── setUp()
│   │   ├── test_method_2()
│   │   └── tearDown()
│   │
│   └── tearDownClass()    ← 测试类级别的后置操作
│
├── tearDownModule()       ← 整个模块级别的后置操作
│
└── 输出结果
```

来看一个完整的生命周期演示：

```python
import unittest


def setUpModule():
    print("\n=== setUpModule：模块初始化 ===")


def tearDownModule():
    print("\n=== tearDownModule：模块清理 ===")


class TestLifecycle(unittest.TestCase):

    @classmethod
    def setUpClass(cls):
        print("\n  --- setUpClass：测试类初始化 ---")

    @classmethod
    def tearDownClass(cls):
        print("\n  --- tearDownClass：测试类清理 ---")

    def setUp(self):
        print("    . setUp：测试方法前置")

    def tearDown(self):
        print("    . tearDown：测试方法后置")

    def test_first(self):
        print("    . 执行 test_first")
        self.assertTrue(True)

    def test_second(self):
        print("    . 执行 test_second")
        self.assertTrue(True)


if __name__ == '__main__':
    unittest.main(verbosity=2)
```

运行后你会看到：

```
=== setUpModule：模块初始化 ===

  --- setUpClass：测试类初始化 ---
    . setUp：测试方法前置
    . 执行 test_first
    . tearDown：测试方法后置
    . setUp：测试方法前置
    . 执行 test_second
    . tearDown：测试方法后置

  --- tearDownClass：测试类清理 ---

=== tearDownModule：模块清理 ===
```

---

## 3. 编写第一个测试

### 3.1 被测代码

假设我们有一个简单的计算器模块 `calculator.py`：

```python
# calculator.py


class Calculator:
    """一个简单的计算器类"""

    def __init__(self):
        self.memory = 0

    def add(self, a, b):
        """加法"""
        return a + b

    def subtract(self, a, b):
        """减法"""
        return a - b

    def multiply(self, a, b):
        """乘法"""
        return a * b

    def divide(self, a, b):
        """除法"""
        if b == 0:
            raise ValueError("除数不能为零")
        return a / b

    def power(self, base, exponent):
        """幂运算"""
        return base ** exponent

    def sqrt(self, n):
        """开平方"""
        if n < 0:
            raise ValueError("不能对负数开平方")
        return n ** 0.5

    def memory_store(self, value):
        """存储到记忆"""
        self.memory = value

    def memory_recall(self):
        """读取记忆"""
        return self.memory

    def memory_clear(self):
        """清除记忆"""
        self.memory = 0
```

### 3.2 编写测试

创建 `test_calculator.py`：

```python
# test_calculator.py

import unittest
from calculator import Calculator


class TestCalculator(unittest.TestCase):
    """Calculator 类的完整测试"""

    def setUp(self):
        """每个测试方法前创建新的 Calculator 实例"""
        self.calc = Calculator()

    def test_add(self):
        """测试加法"""
        self.assertEqual(self.calc.add(2, 3), 5)
        self.assertEqual(self.calc.add(-1, 1), 0)
        self.assertEqual(self.calc.add(0, 0), 0)
        self.assertEqual(self.calc.add(100, 200), 300)

    def test_subtract(self):
        """测试减法"""
        self.assertEqual(self.calc.subtract(5, 3), 2)
        self.assertEqual(self.calc.subtract(3, 5), -2)
        self.assertEqual(self.calc.subtract(0, 0), 0)

    def test_multiply(self):
        """测试乘法"""
        self.assertEqual(self.calc.multiply(3, 4), 12)
        self.assertEqual(self.calc.multiply(-2, 3), -6)
        self.assertEqual(self.calc.multiply(0, 100), 0)

    def test_divide(self):
        """测试除法"""
        self.assertEqual(self.calc.divide(10, 2), 5)
        self.assertEqual(self.calc.divide(7, 2), 3.5)
        self.assertAlmostEqual(self.calc.divide(1, 3), 0.333, places=2)

    def test_divide_by_zero(self):
        """测试除以零应抛出 ValueError"""
        with self.assertRaises(ValueError) as context:
            self.calc.divide(10, 0)
        self.assertEqual(str(context.exception), "除数不能为零")

    def test_power(self):
        """测试幂运算"""
        self.assertEqual(self.calc.power(2, 3), 8)
        self.assertEqual(self.calc.power(5, 0), 1)
        self.assertEqual(self.calc.power(2, -1), 0.5)

    def test_sqrt(self):
        """测试开平方"""
        self.assertEqual(self.calc.sqrt(4), 2)
        self.assertEqual(self.calc.sqrt(0), 0)
        self.assertAlmostEqual(self.calc.sqrt(2), 1.414, places=2)

    def test_sqrt_negative(self):
        """测试对负数开平方应抛出 ValueError"""
        with self.assertRaises(ValueError):
            self.calc.sqrt(-1)

    def test_memory_operations(self):
        """测试记忆功能"""
        self.assertEqual(self.calc.memory_recall(), 0)
        self.calc.memory_store(42)
        self.assertEqual(self.calc.memory_recall(), 42)
        self.calc.memory_clear()
        self.assertEqual(self.calc.memory_recall(), 0)


if __name__ == '__main__':
    unittest.main(verbosity=2)
```

### 3.3 运行测试

有几种运行方式：

```bash
# 方式一：直接运行测试文件
python test_calculator.py

# 方式二：通过 unittest 模块运行
python -m unittest test_calculator

# 方式三：运行指定测试方法
python -m unittest test_calculator.TestCalculator.test_add

# 方式四：详细输出
python -m unittest test_calculator -v
```

### 3.4 理解测试输出

运行测试后，你会看到类似这样的输出：

```
test_add (test_calculator.TestCalculator) ... ok
test_divide (test_calculator.TestCalculator) ... ok
test_divide_by_zero (test_calculator.TestCalculator) ... ok
test_memory_operations (test_calculator.TestCalculator) ... ok
test_multiply (test_calculator.TestCalculator) ... ok
test_power (test_calculator.TestCalculator) ... ok
test_sqrt (test_calculator.TestCalculator) ... ok
test_sqrt_negative (test_calculator.TestCalculator) ... ok
test_subtract (test_calculator.TestCalculator) ... ok

----------------------------------------------------------------------
Ran 9 tests in 0.001s

OK
```

输出符号的含义：

| 符号 | 含义 |
|------|------|
| `.` | 测试通过 |
| `F` | 测试失败（断言不通过） |
| `E` | 测试错误（抛出了意外异常） |
| `s` | 测试被跳过 |
| `x` | 预期失败（确实失败了） |
| `u` | 预期失败但意外通过了 |

当有测试失败时，输出会变成：

```
test_add (test_calculator.TestCalculator) ... FAIL
test_subtract (test_calculator.TestCalculator) ... ok

======================================================================
FAIL: test_add (test_calculator.TestCalculator)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "test_calculator.py", line 15, in test_add
    self.assertEqual(self.calc.add(2, 3), 6)
AssertionError: 5 != 6

----------------------------------------------------------------------
Ran 2 tests in 0.000s

FAILED (failures=1)
```

---

## 4. 断言方法完整列表

断言是测试的核心。`unittest.TestCase` 提供了丰富的断言方法。理解并正确使用它们，能让你的测试既准确又清晰。

### 4.1 基本断言

**`assertTrue(expr, msg=None)`** 和 **`assertFalse(expr, msg=None)`**

验证表达式为真或为假。

```python
import unittest


class TestBoolAssertions(unittest.TestCase):

    def test_assertTrue(self):
        """验证值为 True"""
        self.assertTrue(1 == 1)
        self.assertTrue("hello")
        self.assertTrue([1, 2, 3])  # 非空列表为 True
        self.assertTrue(True)

    def test_assertFalse(self):
        """验证值为 False"""
        self.assertFalse(1 == 2)
        self.assertFalse("")
        self.assertFalse([])  # 空列表为 False
        self.assertFalse(0)
        self.assertFalse(None)
```

### 4.2 相等性断言

**`assertEqual(first, second, msg=None)`** 和 **`assertNotEqual(first, second, msg=None)`**

```python
import unittest


class TestEqualityAssertions(unittest.TestCase):

    def test_assertEqual_numbers(self):
        """数字相等"""
        self.assertEqual(1 + 1, 2)
        self.assertEqual(3.14, 3.14)

    def test_assertEqual_strings(self):
        """字符串相等"""
        self.assertEqual("hello" + " world", "hello world")
        self.assertEqual("你好".upper(), "你好")  # 中文 upper 不变

    def test_assertEqual_lists(self):
        """列表相等"""
        self.assertEqual([1, 2, 3], [1, 2, 3])
        self.assertEqual(sorted([3, 1, 2]), [1, 2, 3])

    def test_assertEqual_dicts(self):
        """字典相等"""
        self.assertEqual({"a": 1}, {"a": 1})

    def test_assertEqual_sets(self):
        """集合相等"""
        self.assertEqual({1, 2, 3}, {3, 2, 1})

    def test_assertEqual_tuples(self):
        """元组相等"""
        self.assertEqual((1, 2), (1, 2))

    def test_assertNotEqual(self):
        """验证不相等"""
        self.assertNotEqual(1, 2)
        self.assertNotEqual("a", "b")
        self.assertNotEqual([1, 2], [2, 1])
```

当比较失败时，`assertEqual` 会根据类型选择合适的比较方式来展示差异：

```python
class TestEqualDiff(unittest.TestCase):

    def test_dict_diff(self):
        """字典比较失败时显示详细差异"""
        expected = {"name": "张三", "age": 25, "city": "北京"}
        actual = {"name": "张三", "age": 26, "city": "上海"}
        # 失败输出会清晰展示哪些键值不同
        # self.assertEqual(expected, actual)
        # 输出示例:
        # AssertionError: {'age': 25, 'city': '北京'} != {'age': 26, 'city': '上海'}
        pass
```

### 4.3 比较断言

```python
import unittest


class TestComparisonAssertions(unittest.TestCase):

    def test_assertGreater(self):
        """验证 first > second"""
        self.assertGreater(10, 5)
        self.assertGreater(3.14, 3.13)
        self.assertGreater("zebra", "apple")  # 字母序比较

    def test_assertGreaterEqual(self):
        """验证 first >= second"""
        self.assertGreaterEqual(10, 10)
        self.assertGreaterEqual(10, 5)

    def test_assertLess(self):
        """验证 first < second"""
        self.assertLess(5, 10)
        self.assertLess(1, 100)

    def test_assertLessEqual(self):
        """验证 first <= second"""
        self.assertLessEqual(5, 5)
        self.assertLessEqual(5, 10)
```

### 4.4 异常断言

**`assertRaises(exc, callable=None, *args, **kwargs)`**

这是处理异常场景的关键断言。

```python
import unittest


def divide(a, b):
    if b == 0:
        raise ValueError("除数不能为零")
    return a / b


def access_item(lst, index):
    return lst[index]


def convert_to_int(value):
    return int(value)


class TestExceptionAssertions(unittest.TestCase):

    def test_assertRaises_context_manager(self):
        """推荐用法：上下文管理器"""
        with self.assertRaises(ValueError):
            divide(10, 0)

    def test_assertRaises_with_message_check(self):
        """检查异常消息的内容"""
        with self.assertRaises(ValueError) as cm:
            divide(10, 0)
        self.assertEqual(str(cm.exception), "除数不能为零")

    def test_assertRaises_callable(self):
        """直接传入可调用对象和参数"""
        self.assertRaises(IndexError, access_item, [1, 2, 3], 10)

    def test_assertRaises_multiple_types(self):
        """检查是否抛出多种异常之一（用元组）"""
        with self.assertRaises((ValueError, TypeError)):
            convert_to_int("abc")

    def test_assertRaises_TypeError(self):
        """类型错误"""
        with self.assertRaises(TypeError):
            divide("a", "b")


class TestExceptionWithRegex(unittest.TestCase):

    def test_assertRaisesRegex(self):
        """检查异常消息是否匹配正则表达式"""
        with self.assertRaisesRegex(ValueError, "除数.*零"):
            divide(10, 0)

    def test_assertRaisesRegex_callable(self):
        """可调用对象形式"""
        self.assertRaisesRegex(
            IndexError,
            r"list index out of range",
            access_item, [1, 2], 100
        )

    def test_assertRaisesRegex_pattern_variations(self):
        """各种正则匹配"""
        with self.assertRaisesRegex(ValueError, r"不能为零"):
            divide(1, 0)

        with self.assertRaisesRegex(ValueError, r"^除数"):
            divide(1, 0)

        with self.assertRaisesRegex(ValueError, r"零$"):
            divide(1, 0)
```

### 4.5 警告断言

```python
import unittest
import warnings


def deprecated_function():
    warnings.warn("此函数已弃用", DeprecationWarning, stacklevel=2)
    return 42


def user_warning_function():
    warnings.warn("注意：数据可能不完整", UserWarning, stacklevel=2)
    return "result"


class TestWarningAssertions(unittest.TestCase):

    def test_assertWarns(self):
        """验证是否产生警告"""
        with self.assertWarns(DeprecationWarning):
            deprecated_function()

    def test_assertWarns_callable(self):
        """可调用对象形式"""
        self.assertWarns(DeprecationWarning, deprecated_function)

    def test_assertWarnsRegex(self):
        """验证警告消息匹配正则"""
        with self.assertWarnsRegex(DeprecationWarning, r"已弃用"):
            deprecated_function()

    def test_assertWarns_checks_message(self):
        """检查警告消息内容"""
        with self.assertWarns(UserWarning) as cm:
            user_warning_function()
        self.assertIn("数据可能不完整", str(cm.warning))

    def test_warnings_context(self):
        """获取警告对象"""
        with self.assertWarns(DeprecationWarning) as cm:
            result = deprecated_function()
        self.assertEqual(result, 42)
        self.assertIn("弃用", str(cm.warning))
```

### 4.6 类型与身份断言

```python
import unittest


class TestIdentityAssertions(unittest.TestCase):

    def test_assertIs(self):
        """验证两个对象是同一个对象（is 关系）"""
        a = [1, 2, 3]
        b = a
        c = [1, 2, 3]
        self.assertIs(a, b)      # 同一对象
        # self.assertIs(a, c)    # 这会失败！值相同但不是同一对象

    def test_assertIsNot(self):
        """验证两个对象不是同一个对象"""
        a = [1, 2, 3]
        b = [1, 2, 3]
        self.assertIsNot(a, b)  # 值相同但不是同一对象

    def test_assertIsNone(self):
        """验证值为 None"""
        self.assertIsNone(None)
        self.assertIsNone(dict().get("不存在的键"))
        result = None
        self.assertIsNone(result)

    def test_assertIsNotNone(self):
        """验证值不为 None"""
        self.assertIsNotNone(0)
        self.assertIsNotNone("")
        self.assertIsNotNone([])
        self.assertIsNotNone(False)

    def test_assertIsInstance(self):
        """验证对象是某类型的实例"""
        self.assertIsInstance(42, int)
        self.assertIsInstance("hello", str)
        self.assertIsInstance([1, 2], list)
        self.assertIsInstance(3.14, (int, float))  # 支持类型元组

    def test_assertNotIsInstance(self):
        """验证对象不是某类型的实例"""
        self.assertNotIsInstance(42, str)
        self.assertNotIsInstance("hello", int)
        self.assertNotIsInstance([], dict)
```

> **重要提示：`assertIs` vs `assertEqual`**
>
> `assertEqual(a, b)` 检查值是否相等（`a == b`），而 `assertIs(a, b)` 检查是否是同一个对象（`a is b`）。对于 `None`、`True`、`False` 等单例，应该用 `assertIsNone`/`assertIsNotNone` 而不是 `assertEqual`。

### 4.7 集合断言

```python
import unittest


class TestCollectionAssertions(unittest.TestCase):

    def test_assertIn(self):
        """验证成员关系"""
        self.assertIn(3, [1, 2, 3, 4, 5])
        self.assertIn("a", {"a": 1, "b": 2})
        self.assertIn("hello", "hello world")
        self.assertIn(42, {42, 43, 44})

    def test_assertNotIn(self):
        """验证不包含"""
        self.assertNotIn(6, [1, 2, 3, 4, 5])
        self.assertNotIn("c", {"a": 1, "b": 2})
        self.assertNotIn("xyz", "hello world")

    def test_assertCountEqual(self):
        """验证两个序列包含相同的元素（不关心顺序）"""
        self.assertCountEqual([1, 2, 3], [3, 2, 1])
        self.assertCountEqual([1, 1, 2], [2, 1, 1])  # 重复元素也会检查
        self.assertCountEqual([1, 2, 3], (3, 2, 1))
        self.assertCountEqual("abc", "cba")

        # 实际应用：检查两个查询结果是否包含相同的记录
        expected_ids = [1, 2, 3, 4, 5]
        actual_ids = [5, 3, 1, 4, 2]
        self.assertCountEqual(expected_ids, actual_ids)

    def test_assertCountEqual_with_duplicates(self):
        """assertCountEqual 会考虑重复次数"""
        self.assertCountEqual([1, 1, 2, 3], [3, 1, 2, 1])
        # self.assertCountEqual([1, 1, 2], [1, 2, 2])  # 失败！

    def test_assertSetEqual(self):
        """验证集合相等"""
        self.assertSetEqual({1, 2, 3}, {3, 2, 1})
        self.assertSetEqual(set(), set())

    def test_assertListEqual(self):
        """验证列表相等（展示逐元素差异）"""
        self.assertListEqual([1, 2, 3], [1, 2, 3])

    def test_assertTupleEqual(self):
        """验证元组相等"""
        self.assertTupleEqual((1, "a"), (1, "a"))

    def test_assertDictEqual(self):
        """验证字典相等（展示键值差异）"""
        self.assertDictEqual({"a": 1, "b": 2}, {"a": 1, "b": 2})
```

### 4.8 字符串断言

```python
import unittest


class TestStringAssertions(unittest.TestCase):

    def test_assertRegex(self):
        """验证字符串匹配正则表达式"""
        text = "订单号：ORD-2024-00123"
        self.assertRegex(text, r"ORD-\d{4}-\d{5}")

        email = "user@example.com"
        self.assertRegex(email, r"^[\w.-]+@[\w.-]+\.\w+$")

        phone = "13812345678"
        self.assertRegex(phone, r"^1[3-9]\d{9}$")

    def test_assertNotRegex(self):
        """验证字符串不匹配正则表达式"""
        text = "hello world"
        self.assertNotRegex(text, r"\d+")

        email = "invalid-email"
        self.assertNotRegex(email, r"^[\w.-]+@[\w.-]+\.\w+$")
```

### 4.9 浮点数断言

```python
import unittest
import math


class TestFloatAssertions(unittest.TestCase):

    def test_assertAlmostEqual(self):
        """验证两个浮点数近似相等"""
        # 默认精确到小数点后 7 位
        self.assertAlmostEqual(0.1 + 0.2, 0.3)
        self.assertAlmostEqual(0.1 + 0.2, 0.3, places=7)

        # 指定精确位数
        self.assertAlmostEqual(3.14159, 3.14160, places=3)

        # 用 delta 参数指定允许的误差范围
        self.assertAlmostEqual(10.0, 10.5, delta=0.6)
        self.assertAlmostEqual(10.0, 10.5, delta=0.5)

    def test_assertNotAlmostEqual(self):
        """验证两个浮点数不相近"""
        self.assertNotAlmostEqual(1.0, 1.1)
        self.assertNotAlmostEqual(3.14, 3.15, places=2)

    def test_almostEqual_practical(self):
        """实际应用场景"""
        # 三角函数
        self.assertAlmostEqual(math.sin(math.pi), 0.0, places=10)
        self.assertAlmostEqual(math.cos(0), 1.0)

        # 货币计算
        total = 19.99 * 3
        self.assertAlmostEqual(total, 59.97, places=2)

        # 科学计算
        result = math.sqrt(2) ** 2
        self.assertAlmostEqual(result, 2.0, places=10)
```

### 4.10 自定义失败消息与日志断言

所有断言方法都支持 `msg` 参数，用于在断言失败时提供更友好的提示。

```python
import unittest


class TestCustomMessages(unittest.TestCase):

    def test_with_custom_message(self):
        """使用自定义消息让失败更容易理解"""
        user_age = 17
        self.assertGreaterEqual(
            user_age, 18,
            f"用户年龄 {user_age} 不满足最低 18 岁的要求"
        )

    def test_fail_method(self):
        """直接标记测试失败"""
        condition = False
        if not condition:
            self.fail("前置条件不满足，测试无法继续")

    def test_assertion_with_context(self):
        """在断言中提供上下文信息"""
        response_code = 404
        response_body = {"error": "Not Found"}
        self.assertEqual(
            response_code, 200,
            f"请求失败。状态码: {response_code}, 响应: {response_body}"
        )
```

还有一个特殊的 `assertLogs` 上下文管理器，用于验证日志输出：

```python
import unittest
import logging

logger = logging.getLogger('my_app')
logger.setLevel(logging.DEBUG)


class TestLogging(unittest.TestCase):

    def test_assertLogs(self):
        """验证日志输出"""
        with self.assertLogs('my_app', level='INFO') as cm:
            logger.info("处理开始")
            logger.warning("发现异常数据")
            logger.info("处理完成")

        self.assertEqual(len(cm.output), 3)
        self.assertIn("处理开始", cm.output[0])
        self.assertIn("异常数据", cm.output[1])

    def test_assertLogs_level(self):
        """指定最低日志级别"""
        with self.assertLogs('my_app', level='WARNING') as cm:
            logger.debug("这条不会出现")
            logger.warning("这条会捕获")
            logger.error("这条也会捕获")

        self.assertEqual(len(cm.output), 2)

    def test_assertNoLogs(self):
        """验证没有产生日志（Python 3.10+）"""
        with self.assertNoLogs('my_app'):
            pass
```

---

## 5. 测试夹具

测试夹具（Test Fixture）提供了测试前准备环境和测试后清理环境的机制。掌握夹具，是写好测试的基本功。

### 5.1 setUp 和 tearDown

`setUp` 在每个测试方法之前运行，`tearDown` 在之后运行。即使测试方法失败或出错，`tearDown` 也会被执行。

```python
import unittest
import tempfile
import os


class TestFileProcessor(unittest.TestCase):
    """文件处理器的测试"""

    def setUp(self):
        """每个测试前创建临时目录和文件"""
        self.temp_dir = tempfile.mkdtemp()
        self.test_file = os.path.join(self.temp_dir, "test_data.txt")
        with open(self.test_file, 'w') as f:
            f.write("第一行\n第二行\n第三行\n")

    def tearDown(self):
        """每个测试后清理临时文件"""
        if os.path.exists(self.test_file):
            os.remove(self.test_file)
        os.rmdir(self.temp_dir)

    def test_read_file(self):
        """测试读取文件"""
        with open(self.test_file, 'r') as f:
            content = f.read()
        self.assertIn("第一行", content)
        self.assertEqual(content.count('\n'), 3)

    def test_append_file(self):
        """测试追加内容"""
        with open(self.test_file, 'a') as f:
            f.write("第四行\n")
        with open(self.test_file, 'r') as f:
            lines = f.readlines()
        self.assertEqual(len(lines), 4)

    def test_file_exists(self):
        """测试文件存在"""
        self.assertTrue(os.path.exists(self.test_file))
```

来看一个数据库连接的场景：

```python
import unittest
import sqlite3


class TestDatabase(unittest.TestCase):
    """数据库操作的测试"""

    def setUp(self):
        """每个测试前创建内存数据库和测试表"""
        self.conn = sqlite3.connect(":memory:")
        self.cursor = self.conn.cursor()
        self.cursor.execute("""
            CREATE TABLE users (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                email TEXT UNIQUE NOT NULL,
                age INTEGER
            )
        """)
        test_data = [
            ("张三", "zhang@example.com", 25),
            ("李四", "li@example.com", 30),
            ("王五", "wang@example.com", 28),
        ]
        self.cursor.executemany(
            "INSERT INTO users (name, email, age) VALUES (?, ?, ?)",
            test_data
        )
        self.conn.commit()

    def tearDown(self):
        """每个测试后关闭连接"""
        self.conn.close()

    def test_query_all_users(self):
        """测试查询所有用户"""
        self.cursor.execute("SELECT * FROM users")
        users = self.cursor.fetchall()
        self.assertEqual(len(users), 3)

    def test_insert_user(self):
        """测试插入新用户"""
        self.cursor.execute(
            "INSERT INTO users (name, email, age) VALUES (?, ?, ?)",
            ("赵六", "zhao@example.com", 22)
        )
        self.conn.commit()
        self.cursor.execute("SELECT COUNT(*) FROM users")
        count = self.cursor.fetchone()[0]
        self.assertEqual(count, 4)

    def test_delete_user(self):
        """测试删除用户"""
        self.cursor.execute("DELETE FROM users WHERE name = ?", ("张三",))
        self.conn.commit()
        self.cursor.execute("SELECT * FROM users WHERE name = ?", ("张三",))
        result = self.cursor.fetchone()
        self.assertIsNone(result)

    def test_update_user(self):
        """测试更新用户"""
        self.cursor.execute(
            "UPDATE users SET age = ? WHERE name = ?",
            (26, "张三")
        )
        self.conn.commit()
        self.cursor.execute(
            "SELECT age FROM users WHERE name = ?",
            ("张三",)
        )
        age = self.cursor.fetchone()[0]
        self.assertEqual(age, 26)
```

### 5.2 setUpClass 和 tearDownClass

`setUpClass` 和 `tearDownClass` 是类方法，在整个测试类之前/之后各运行一次。适合做一些开销较大的初始化工作。

```python
import unittest


class ExpensiveResource:
    """模拟一个初始化成本很高的资源"""
    _instance_count = 0

    def __init__(self):
        ExpensiveResource._instance_count += 1
        self.id = ExpensiveResource._instance_count

    def process(self, data):
        return f"processed: {data}"

    def close(self):
        pass


class TestWithClassFixture(unittest.TestCase):
    """使用类级别夹具"""

    @classmethod
    def setUpClass(cls):
        """整个测试类只执行一次"""
        cls.resource = ExpensiveResource()
        cls.shared_data = {"initialized": True}

    @classmethod
    def tearDownClass(cls):
        """整个测试类只执行一次"""
        cls.resource.close()

    def test_process_data_a(self):
        """测试 A：共享同一个资源实例"""
        result = self.resource.process("data_a")
        self.assertIn("processed", result)

    def test_process_data_b(self):
        """测试 B：共享同一个资源实例"""
        result = self.resource.process("data_b")
        self.assertIn("processed", result)

    def test_shared_data(self):
        """测试共享数据"""
        self.assertTrue(self.shared_data["initialized"])
```

### 5.3 setUpModule 和 tearDownModule

模块级别的夹具函数，在整个测试模块的所有测试类之前/之后执行。

```python
# test_module_fixture.py

import unittest

database_connection = None
test_config = {}


def setUpModule():
    """模块级别前置"""
    global database_connection, test_config
    database_connection = {"connected": True}
    test_config = {
        "host": "localhost",
        "port": 5432,
        "database": "test_db"
    }


def tearDownModule():
    """模块级别后置"""
    global database_connection
    database_connection = None


class TestDatabaseQueryA(unittest.TestCase):

    def test_connection_available(self):
        self.assertIsNotNone(database_connection)
        self.assertTrue(database_connection["connected"])

    def test_config_loaded(self):
        self.assertEqual(test_config["host"], "localhost")


class TestDatabaseQueryB(unittest.TestCase):

    def test_another_query(self):
        self.assertIsNotNone(database_connection)

    def test_config_available(self):
        self.assertIn("database", test_config)


if __name__ == '__main__':
    unittest.main(verbosity=2)
```

### 5.4 夹具执行顺序总结

执行顺序：

```
1.  setUpModule
2.    setUpClass (TestExampleA)
3.      setUp -> test_one -> tearDown
4.    tearDownClass (TestExampleA)
5.    setUpClass (TestExampleB)
6.      setUp -> test_two -> tearDown
7.    tearDownClass (TestExampleB)
8.  tearDownModule
```

### 5.5 夹具中的异常处理

**重要规则：**

1. `setUp` 失败 → 该测试方法不运行，`tearDown` 也不运行
2. 测试方法失败 → `tearDown` 仍然运行
3. `tearDown` 失败 → 测试标记为 ERROR
4. `setUpClass` 失败 → 整个测试类的所有方法都不运行
5. `setUpModule` 失败 → 模块中所有测试都不运行

```python
import unittest


class TestFixtureExceptions(unittest.TestCase):

    def setUp(self):
        self.data = [1, 2, 3]
        # 如果 raise RuntimeError("setUp 失败")
        # -> test_xxx 不会运行，tearDown 不会运行
        # -> 测试标记为 ERROR

    def tearDown(self):
        self.data = None

    def test_normal(self):
        self.assertEqual(len(self.data), 3)
```

---

## 6. 测试套件

### 6.1 手动构建测试套件

测试套件让你可以精确控制运行哪些测试。

```python
import unittest


class TestUserAPI(unittest.TestCase):

    def test_create_user(self):
        self.assertTrue(True)

    def test_get_user(self):
        self.assertTrue(True)

    def test_update_user(self):
        self.assertTrue(True)

    def test_delete_user(self):
        self.assertTrue(True)


class TestProductAPI(unittest.TestCase):

    def test_list_products(self):
        self.assertTrue(True)

    def test_search_products(self):
        self.assertTrue(True)


# 方式一：逐个添加测试方法
suite = unittest.TestSuite()
suite.addTest(TestUserAPI('test_create_user'))
suite.addTest(TestUserAPI('test_get_user'))

# 方式二：一次添加多个测试
tests = [
    TestUserAPI('test_update_user'),
    TestUserAPI('test_delete_user'),
]
suite.addTests(tests)

# 方式三：添加整个测试类
suite.addTests(unittest.TestLoader().loadTestsFromTestCase(TestProductAPI))

# 运行套件
runner = unittest.TextTestRunner(verbosity=2)
result = runner.run(suite)

print(f"\n运行了 {result.testsRun} 个测试")
print(f"成功: {result.wasSuccessful()}")
```

### 6.2 使用 TestLoader 加载测试

```python
import unittest
import sys


class TestLogin(unittest.TestCase):

    def test_login_with_valid_credentials(self):
        self.assertTrue(True)

    def test_login_with_invalid_password(self):
        self.assertTrue(True)


class TestRegistration(unittest.TestCase):

    def test_register_new_user(self):
        self.assertTrue(True)

    def test_register_duplicate_email(self):
        self.assertTrue(True)


loader = unittest.TestLoader()

# 方法一：从测试类加载
suite1 = loader.loadTestsFromTestCase(TestLogin)
print(f"从 TestLogin 加载了 {suite1.countTestCases()} 个测试")

# 方法二：从模块加载
current_module = sys.modules[__name__]
suite2 = loader.loadTestsFromModule(current_module)
print(f"从模块加载了 {suite2.countTestCases()} 个测试")

# 方法三：按名称加载
suite3 = loader.loadTestsFromName('__main__.TestLogin.test_login_with_valid_credentials')

# 方法四：按名称列表加载
suite4 = loader.loadTestsFromNames([
    '__main__.TestLogin.test_login_with_valid_credentials',
    '__main__.TestRegistration.test_register_new_user',
])

runner = unittest.TextTestRunner(verbosity=2)
runner.run(suite2)
```

### 6.3 嵌套测试套件

测试套件可以嵌套，形成分层的测试组织结构。

```python
import unittest


class TestUserCrud(unittest.TestCase):

    def test_create(self):
        self.assertTrue(True)

    def test_read(self):
        self.assertTrue(True)

    def test_update(self):
        self.assertTrue(True)

    def test_delete(self):
        self.assertTrue(True)


class TestPayment(unittest.TestCase):

    def test_charge(self):
        self.assertTrue(True)

    def test_refund(self):
        self.assertTrue(True)


# 构建嵌套套件
loader = unittest.TestLoader()

user_suite = unittest.TestSuite()
user_suite.addTests(loader.loadTestsFromTestCase(TestUserCrud))

finance_suite = unittest.TestSuite()
finance_suite.addTests(loader.loadTestsFromTestCase(TestPayment))

master_suite = unittest.TestSuite([
    user_suite,
    finance_suite,
])

print(f"用户模块: {user_suite.countTestCases()} 个")
print(f"财务模块: {finance_suite.countTestCases()} 个")
print(f"总计: {master_suite.countTestCases()} 个")

runner = unittest.TextTestRunner(verbosity=2)
runner.run(master_suite)
```

### 6.4 自定义 TestRunner

```python
import unittest
import sys
import time


class CustomTestResult(unittest.TestResult):
    """自定义测试结果收集器"""

    def __init__(self, stream=None, descriptions=True, verbosity=1):
        super().__init__(stream, descriptions, verbosity)
        self.stream = stream or sys.stderr
        self.start_time = None

    def startTestRun(self):
        self.start_time = time.time()
        self.stream.write("开始测试运行\n")
        self.stream.write("=" * 60 + "\n")

    def stopTestRun(self):
        elapsed = time.time() - self.start_time
        self.stream.write("=" * 60 + "\n")
        self.stream.write(f"总耗时: {elapsed:.3f} 秒\n")
        self.stream.write(f"测试总数: {self.testsRun}\n")
        passed = self.testsRun - len(self.failures) - len(self.errors)
        self.stream.write(f"通过: {passed}\n")
        self.stream.write(f"失败: {len(self.failures)}\n")
        self.stream.write(f"错误: {len(self.errors)}\n")
        self.stream.write(f"跳过: {len(self.skipped)}\n")

    def addSuccess(self, test):
        super().addSuccess(test)
        self.stream.write(f"  [PASS] {test._testMethodName}\n")

    def addFailure(self, test, err):
        super().addFailure(test, err)
        self.stream.write(f"  [FAIL] {test._testMethodName}\n")

    def addError(self, test, err):
        super().addError(test, err)
        self.stream.write(f"  [ERROR] {test._testMethodName}\n")

    def addSkip(self, test, reason):
        super().addSkip(test, reason)
        self.stream.write(f"  [SKIP] {test._testMethodName}: {reason}\n")


class CustomTestRunner(unittest.TextTestRunner):
    """自定义测试运行器"""
    resultclass = CustomTestResult


# 使用自定义运行器
class TestDemo(unittest.TestCase):

    def test_pass(self):
        self.assertEqual(1 + 1, 2)

    @unittest.skip("演示跳过")
    def test_skip(self):
        pass


if __name__ == '__main__':
    suite = unittest.TestLoader().loadTestsFromTestCase(TestDemo)
    CustomTestRunner().run(suite)
```

---

## 7. 测试发现

### 7.1 命令行发现

测试发现让你无需手动列举每个测试文件，框架会自动查找并运行所有测试。

```bash
# 基本用法：在当前目录递归查找 test*.py
python -m unittest discover

# 指定起始目录
python -m unittest discover -s tests

# 指定文件名匹配模式
python -m unittest discover -p "test_*.py"

# 指定顶层目录（用于处理包导入）
python -m unittest discover -t . -s tests

# 组合参数
python -m unittest discover -s tests -p "test_*.py" -t . -v
```

参数说明：

| 参数 | 缩写 | 说明 | 默认值 |
|------|------|------|--------|
| `--start-directory` | `-s` | 开始搜索的目录 | `.`（当前目录） |
| `--pattern` | `-p` | 文件名匹配模式 | `test*.py` |
| `--top-level-directory` | `-t` | 顶层项目目录 | 与 start-directory 相同 |
| `--verbose` | `-v` | 详细输出 | 关闭 |

### 7.2 编程式发现

```python
import unittest
import os


def discover_and_run_tests(test_dirs, pattern='test_*.py'):
    """在多个目录中发现并运行测试"""
    loader = unittest.TestLoader()
    master_suite = unittest.TestSuite()

    for test_dir in test_dirs:
        if os.path.exists(test_dir):
            suite = loader.discover(
                start_dir=test_dir,
                pattern=pattern,
                top_level_dir='.'
            )
            master_suite.addTests(suite)
            print(f"从 {test_dir} 加载了 {suite.countTestCases()} 个测试")
        else:
            print(f"警告：目录 {test_dir} 不存在")

    runner = unittest.TextTestRunner(verbosity=2)
    return runner.run(master_suite)


if __name__ == '__main__':
    test_dirs = [
        'tests/unit',
        'tests/integration',
        'tests/api',
    ]
    result = discover_and_run_tests(test_dirs)
    exit(0 if result.wasSuccessful() else 1)
```

### 7.3 项目结构最佳实践

一个规范的 Python 项目测试目录结构：

```
my_project/
├── src/
│   └── my_package/
│       ├── __init__.py
│       ├── calculator.py
│       ├── user_service.py
│       └── database.py
├── tests/
│   ├── __init__.py
│   ├── unit/
│   │   ├── __init__.py
│   │   ├── test_calculator.py
│   │   └── test_user_service.py
│   ├── integration/
│   │   ├── __init__.py
│   │   └── test_database.py
├── pyproject.toml
└── README.md
```

关键规范：

1. 测试文件以 `test_` 开头
2. 测试类以 `Test` 开头
3. 测试方法以 `test_` 开头
4. 测试目录有 `__init__.py`（使其成为包）
5. 测试目录与源代码目录分离

发现命令：

```bash
# 运行所有测试
python -m unittest discover -s tests -v

# 只运行单元测试
python -m unittest discover -s tests/unit -v

# 只运行集成测试
python -m unittest discover -s tests/integration -v
```

---

## 8. 跳过测试和预期失败

### 8.1 跳过测试的装饰器

有时候某些测试在特定条件下不应该运行。unittest 提供了一组装饰器来处理这种情况。

```python
import unittest
import sys
import platform


class TestSkipDecorators(unittest.TestCase):

    @unittest.skip("演示无条件跳过")
    def test_skip_unconditionally(self):
        """这个测试永远不会运行"""
        self.fail("不应该执行到这里")

    @unittest.skipIf(sys.version_info < (3, 10), "需要 Python 3.10+")
    def test_skip_if_python_old(self):
        """只在 Python 3.10+ 运行"""
        self.assertTrue(True)

    @unittest.skipUnless(platform.system() == "Linux", "仅在 Linux 上运行")
    def test_skip_unless_linux(self):
        """只在 Linux 上运行"""
        self.assertTrue(True)

    @unittest.skipIf(sys.platform == "win32", "Windows 上不运行")
    def test_skip_on_windows(self):
        """Windows 上跳过"""
        self.assertTrue(True)

    def test_normal(self):
        """普通测试，总会运行"""
        self.assertTrue(True)


if __name__ == '__main__':
    unittest.main(verbosity=2)
```

### 8.2 预期失败

`@expectedFailure` 标记一个预期会失败的测试。如果测试确实失败了，结果记为"预期失败"；如果意外通过了，结果记为"意外通过"。

```python
import unittest


class TestExpectedFailure(unittest.TestCase):

    @unittest.expectedFailure
    def test_expected_to_fail(self):
        """这个测试已知有 bug，预期会失败"""
        result = 1 + 1
        self.assertEqual(result, 3)

    @unittest.expectedFailure
    def test_unexpected_pass(self):
        """如果意外通过，标记为 unexpected success"""
        result = 1 + 1
        self.assertEqual(result, 2)


class TestBugTracking(unittest.TestCase):
    """实际应用：跟踪已知 bug"""

    @unittest.expectedFailure
    def test_issue_123_unicode_handling(self):
        """当 bug 修复后，测试会变成 unexpected success"""
        result = "你好".encode("ascii")
        self.assertIsNotNone(result)
```

### 8.3 动态跳过

在测试方法内部动态决定是否跳过：

```python
import unittest
import os


class TestDynamicSkip(unittest.TestCase):

    def test_require_env_var(self):
        """需要环境变量才能运行的测试"""
        api_key = os.environ.get("API_KEY")
        if not api_key:
            self.skipTest("未设置 API_KEY 环境变量")

        self.assertTrue(len(api_key) > 0)

    def test_require_network(self):
        """需要网络连接的测试"""
        try:
            import socket
            socket.create_connection(("www.google.com", 80), timeout=5)
        except OSError:
            self.skipTest("无法连接网络")

        self.assertTrue(True)

    def test_require_module(self):
        """依赖可选模块的测试"""
        try:
            import numpy  # noqa: F401
        except ImportError:
            self.skipTest("numpy 未安装")

        import numpy as np
        arr = np.array([1, 2, 3])
        self.assertEqual(len(arr), 3)
```

跳过装饰器对比：

| 装饰器/方法 | 场景 | 输出标记 |
|-------------|------|----------|
| `@skip(reason)` | 无条件跳过 | SKIP |
| `@skipIf(condition, reason)` | 条件为真时跳过 | SKIP |
| `@skipUnless(condition, reason)` | 条件为假时跳过 | SKIP |
| `@expectedFailure` | 预期失败 | EXPECTED FAILURE / UNEXPECTED SUCCESS |
| `self.skipTest(reason)` | 运行时动态跳过 | SKIP |

---

## 9. 子测试

### 9.1 基本用法

子测试（subTest）允许在一个测试方法中运行多个相关测试，即使某个子测试失败，其他子测试仍然继续运行。

```python
import unittest


class TestSubTest(unittest.TestCase):

    def test_even_numbers(self):
        """测试多个偶数"""
        for number in [2, 4, 6, 8, 10]:
            with self.subTest(number=number):
                self.assertEqual(number % 2, 0, f"{number} 不是偶数")

    def test_string_upper(self):
        """测试多个字符串的大写转换"""
        test_cases = [
            ("hello", "HELLO"),
            ("world", "WORLD"),
            ("Python", "PYTHON"),
            ("", ""),
        ]
        for input_str, expected in test_cases:
            with self.subTest(input=input_str):
                self.assertEqual(input_str.upper(), expected)
```

### 9.2 子测试与参数化

子测试最常见的用法是实现轻量级的参数化测试：

```python
import unittest


def is_palindrome(s):
    """检查字符串是否为回文"""
    s = s.lower().replace(" ", "")
    return s == s[::-1]


def validate_email(email):
    """简单的邮箱验证"""
    if not email or "@" not in email:
        return False
    parts = email.split("@")
    if len(parts) != 2:
        return False
    return len(parts[0]) > 0 and "." in parts[1]


class TestPalindromeWithSubTest(unittest.TestCase):

    def test_palindromes(self):
        """参数化测试：回文字符串"""
        test_cases = [
            ("radar", True),
            ("level", True),
            ("A man a plan a canal Panama", True),
            ("hello", False),
            ("Python", False),
            ("", True),
            ("a", True),
            ("aa", True),
            ("ab", False),
        ]
        for input_str, expected in test_cases:
            with self.subTest(input=input_str):
                self.assertEqual(is_palindrome(input_str), expected)

    def test_email_validation(self):
        """参数化测试：邮箱验证"""
        test_cases = [
            ("user@example.com", True),
            ("test.email@domain.org", True),
            ("", False),
            ("no-at-sign", False),
            ("@domain.com", False),
            ("user@", False),
            ("user@domain", False),
            ("user.name@sub.domain.com", True),
        ]
        for email, expected in test_cases:
            with self.subTest(email=email):
                self.assertEqual(validate_email(email), expected)
```

### 9.3 子测试的注意事项

```python
import unittest


class TestSubTestBehavior(unittest.TestCase):

    def test_subtest_continues_after_failure(self):
        """子测试失败不会中断其他子测试"""
        for i in range(5):
            with self.subTest(i=i):
                # i=3 会失败，但其他子测试仍然运行
                self.assertNotEqual(i, 3)

    def test_subtest_with_multiple_params(self):
        """子测试可以有多个参数"""
        test_cases = [
            (2, 3, 5),
            (0, 0, 0),
            (-1, 1, 0),
            (100, 200, 300),
        ]
        for a, b, expected in test_cases:
            with self.subTest(a=a, b=b):
                self.assertEqual(a + b, expected)
```

**子测试 vs 独立测试方法：**

| 特性 | 子测试 | 独立方法 |
|------|--------|----------|
| 一个失败是否影响其他 | 不影响 | 不影响 |
| setUp/tearDown 调用次数 | 1次 | 每个方法1次 |
| 测试报告粒度 | 粗（显示子测试参数） | 细（每个方法单独显示） |
| 适用场景 | 大量相似测试 | 差异较大的测试 |
| 性能 | 更快（只初始化一次） | 更慢（每次都初始化） |
