categories:
- 技术

tags:
- Test

title: 测试金字塔
---

## 一、自动化测试的重要性

**公理**：要正确交付软件代码，测试是必不可少的。

### 1. 测试分类

- 手动测试
- 自动化测试

### 2. 何时采用何种测试？

- 一次性使用的代码
    + *示例*：临时脚本或工具
    + 采用手工测试
- 需要维护的代码
    + *示例*：业务代码
        - **警惕陷阱**：这个需求后面不会改，先手动测试吧
    + 采用自动化测试
        - 手工测试低效、重复、乏味
        - 自动化测试可以充当代码变更的保护网


## 二、测试金字塔

![Test-Pyramid](https://upload.wikimedia.org/wikipedia/commons/5/54/The_test_automation_pyramid.png)

### 1. 核心原则

- 编写不同粒度的测试
- 层次越高，编写的测试应该越少

### 2. 反模式

- [测试冰淇淋][1]


## 三、单元测试

### 1. Why

为什么要多写单元测试？

- 规模小（关注点聚焦，更容易编写）
- 执行快（最小化依赖，运行成本低）

### 2. What

何为一个单元？

- 面向过程/函数式：一个函数
- 面向对象：一个方法、一个类

群居和独居

- 独居：隔离所有外部依赖
- 群居：只隔离那些执行慢或者副作用大的依赖（比如数据库、外部服务等）

### 3. How

测试什么？

- 至少应该测试类的公共方法
- 如果类复杂到需要测试私有方法，考虑从设计上拆分类，把这些私有方法变成另外一个类的公共方法

测试结构（[Arrange-Act-Assert](https://xp123.com/articles/3a-arrange-act-assert/) 或者 [Given-When-Then](https://martinfowler.com/bliki/GivenWhenThen.html)）

- 准备测试数据
- 调用被测方法
- 断言返回的是期待的结果


## 四、集成测试

测试应用和所有外部依赖的集成。

常见的外部依赖有：

- 数据库（如 MySQL/Redis/Elasticsearch）
- 消息队列（如 Kafka）
- 外部服务（如 AWS S3)


## 五、契约测试

### 1. 不适用场景

- 对外服务的提供者和消费者之间

### 2. 适用场景

- 内部微服务的提供者和消费者之间

### 3. 契约测试的特征

- 由消费者编写
- 由提供者执行
- 提供者和消费者之间的通信方式，常见的有：
    + REST 接口
    + gRPC 接口
    + 异步事件


## 六、UI 测试

前后端分离的架构下，UI 测试可以是：

- 纯前端 UI 测试
- 后端 API 集成测试（cURL 和 Postman 测试的自动化版本）


## 七、避免重复测试

### 1. 测试成本

- 编写和维护测试要花时间
- 阅读和理解他人的测试也要花时间
- 执行这些测试也要花时间

### 2. 基本法则

- 如果一个更高层级的测试发现了一个错误，并且底层测试全都通过了，那么应该编写一个低层级测试去覆盖这个错误
- 竭尽所能把测试往金字塔下层赶
- 删掉那些已经被低层级测试覆盖完全的高层级测试
    + **警惕陷阱**：[沉没成本][3]（不忍删除花了时间精力编写的测试）


## 八、相关阅读

- [测试金字塔实战][2]（[英文版][4]）
- [Why bother writing tests at all?][5]
- [Introducing the Software Testing Cupcake (Anti-Pattern)][1]


[1]: https://www.thoughtworks.com/insights/blog/introducing-software-testing-cupcake-anti-pattern
[2]: https://insights.thoughtworks.cn/practical-test-pyramid/
[3]: https://zh.wikipedia.org/wiki/%E6%B2%89%E6%B2%A1%E6%88%90%E6%9C%AC
[4]: https://martinfowler.com/articles/practical-test-pyramid.html
[5]: https://dave.cheney.net/2019/05/14/why-bother-writing-tests-at-all
