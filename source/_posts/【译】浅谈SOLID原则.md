---
title: 【译】浅谈SOLID原则
date: 2019-12-04 22:45:11
tags: 技术杂谈
---

SOLID原则是一种编码的标准，为了避免不良设计，所有的软件开发人员都应该清楚这些原则。SOLID原则是由Robert C Martin推广并被广泛引用于面向对象编程中。正确使用这些规范将提升你的代码的可扩展性、逻辑性和可读性。<!-- more -->

当开发人员按照不好的设计来开发软件时，代码将失去灵活性和健壮性。任何一点点小的修改都非常容易引起bug。因此，我们应该遵循SOLID原则。

首先我们需要花一些时间来了解SOLID原则，当你能够理解这些原则并正确使用时，你的代码质量将会得到大幅的提高。同时，它可以帮助你更好的理解一些优秀软件的设计。

为了理解SOLID原则，你必须清楚接口的用法，如果你还不理解接口的概念，建议你先读一读这篇[文章](https://medium.com/better-programming/understanding-use-of-interface-and-abstract-class-9a82f5f15837)。

下面我将用简单易懂的方式为你描述SOLID原则，希望能帮助你对这些原则有个初步的理解。

### 单一责任原则

> 一个类只能因为一个理由被修改。
>
> *A class should have one, and only one, reason to change.*

一个类应该只为一个目标服务。并不是说每个类都只能有一个方法，但它们都应该与类的责任有直接关系。所有的方法和属性都应该努力做好同一类事情。当一个类具有多个目标或职责时，就应该创建一个新的类出来。

我们来看一下这段代码：

``` java
public class OrdersReportService {

    public List<OrderVO> getOrdersInfo(Date startDate, Date endDate) {
        List<OrderDO> orders = queryDBForOrders(startDate, endDate);

        return transform(orders);
    }

    private List<OrderDO> queryDBForOrders(Date startDate, Date endDate) {
        // select * from order where date >= startDate and date < endDate;
    }

    private List<OrderVO> transform(List<OrderDO> orderDOList) {
        //transform DO to VO
    }
}
```

这段代码就违反了单一责任原则。为什么会在这个类中执行sql语句？这样的操作应该放到持久化层，持久化层负责处理数据的持久化的相关操作，包括从数据库中存储或查询数据。所以这个职责不应该属于这个类。

transform方法同样不应该属于这个类，因为我们可能需要很多种类型的转换。

因此我们需要对代码进行重构，重构之后的代码如下（为了节省篇幅）：

``` java
public class OrdersReportService {

    @Autowired
    private OrdersReportDao ordersReportDao;
    @Autowired
    private Formatter formatter;
    public List<OrderVO> getOrdersInfo(Date startDate, Date endDate) {
        List<OrderDO> orders = ordersReportDao.queryDBForOrders(startDate, endDate);

        return formatter.transform(orders);
    }
}

public class OrdersReportDao {
    
    public List<OrderDO> queryDBForOrders(Date startDate, Date endDate) {}
}

public class Formatter {
    
    private List<OrderVO> transform(List<OrderDO> orderDOList) {}
}
```

### 开闭原则

> 对扩展开放，对修改关闭。
>
> *Entities should be open for extension, but closed for modification.*

软件实体（包括类、模块、函数等）都应该可扩展，而不用因为扩展而修改实体的内容。如果我们严格遵循这个原则，就可以做到修改代码行为时，不需要改动任何原始代码。

我们还是以一段代码为例：

``` java
class Rectangle extends Shape {
    private int width;
    private int height;

    public Rectangle(int width, int height) {
        this.width = width;
        this.height = height;
    }
}
class Circle extends Shape {
    private int radius;

    public Circle(int radius) {
        this.radius = radius;
    }
}
class CostManager {
    public double calculate(Shape shape) {
        double costPerUnit = 1.5;
        double area;
        if (shape instanceof Rectangle) {
            area = shape.getWidth() * shape.getHeight();
        } else {
            area = shape.getRadius() * shape.getRadius() * pi();
        }

        return costPerUnit * area;
    }
}
```

如果你想要计算正方形的面积，那么我们就需要修改calculate方法的代码。这就破坏了开闭原则。根据这个原则，我们不能修改原有代码，但是我们可以进行扩展。

所以我们可以把计算面积的方法放到Shape类中，再由每个继承它的子类自己去实现自己的计算方法。这样就不用修改原有的代码了。

### 里氏替换原则

里氏替换原则是由[Barbara Liskov](https://en.wikipedia.org/wiki/Barbara_Liskov)在1987年的“数据抽象“大会上提出的。Barbara Liskov和[Jeannette Wing](https://en.wikipedia.org/wiki/Jeannette_Wing)在1994年发表了论文对这一原则进行阐述：

> 如果φ(x)是类型T的属性，并且S是T的子类型，那么φ(y)就是S的属性。
>
> *Let φ(x) be a property provable about objects x of type T. Then φ(y) should be true for objects y of type S where S is a subtype of T.*

Barbara Liskov给出了易于理解的版本，但是这一版本更依赖于类型系统：

> *1. Preconditions cannot be strengthened in a subtype.*
> *2. Postconditions cannot be weakened in a subtype.*
> *3. Invariants of the supertype must be preserved in a subtype.*

Robert Martin在1996年提出了更加简洁、通顺的定义：

> 使用指向基类指针的函数也可以使用子类。
>
> *Functions that use pointers of references to base classes must be able to use objects of derived classes without knowing it.*

更简单一点讲就是子类可以替代父类。

根据里氏替换原则，我们可以在接受抽象类（接口）的任何地方用它的子类（实现类）来替代它们。基本上，我们应该注意在编程时不能只关注接口的输入参数，还需要保证接口实现类的返回值都是同一类型的。

下面这段代码就违反了里氏替换原则：

``` php
<?php
interface LessonRepositoryInterface
{
    /**
     * Fetch all records.
     *
     * @return array
     */
    public function getAll();
}
class FileLessonRepository implements LessonRepositoryInterface
{
    public function getAll()
    {
        // return through file system
        return [];
    }
}
class DbLessonRepository implements LessonRepositoryInterface
{
    public function getAll()
    {
        /*
            Violates LSP because:
              - the return type is different
              - the consumer of this subclass and FileLessonRepository won't work identically
         */
        // return Lesson::all();
        // to fix this
        return Lesson::all()->toArray();
    }
}
```

译者注：这里没想到Java应该怎么实现，因此直接用了作者的代码，大家理解就好

### 接口隔离原则

> 不能强制客户端实现它不使用的接口。
>
> *A client should not be forced to implement an interface that it doesn’t use.*

这个规则告诉我们，应该把接口拆的尽可能小。这样才能更好的满足客户的确切需求。

与单一责任原则类似，接口隔离原则也是通过将软件拆分为多个独立的部分来最大程度的减少副作用和重复代码。

我们来看一个例子：

``` java
public interface WorkerInterface {

    void work();
    void sleep();
}

public class HumanWorker implements WorkerInterface {

    public void work() {
        System.out.println("work");
    }
    public void sleep() {
        System.out.println("sleep");
    }
}

public class RobotWorker implements WorkerInterface {

    public void work() {
        System.out.println("work");
    }
    public void sleep() {
        // No need
    }
}
```

在上面这段代码中，我们很容易发现问题所在，机器人不需要睡觉，但是由于实现了WorkerInterface接口，它不得不实现sleep方法。这就违背了接口隔离的原则，下面我们一起修复一下这段代码：

``` java
public interface WorkAbleInterface {

    void work();
}

public interface SleepAbleInterface {

    void sleep();
}

public class HumanWorker implements WorkAbleInterface, SleepAbleInterface {

    public void work() {
        System.out.println("work");
    }
    public void sleep() {
        System.out.println("sleep");
    }
}

public class RobotWorker implements WorkerInterface {

    public void work() {
        System.out.println("work");
    }
}
```

### 依赖倒置原则

> 高层模块不应该依赖于低层的模块，它们都应该依赖于抽象。
>
> 抽象不应该依赖于细节，细节应该依赖于抽象。
>
> High-level modules should not depend on low-level modules. Both should depend on abstractions.
>
> Abstractions should not depend on details. Details should depend on abstractions.

简单来讲就是：抽象不依赖于细节，而细节依赖于抽象。

通过应用依赖倒置模块，只需要修改依赖模块，其他模块就可以轻松得到修改。同时，低层模块的修改是不会影响到高层模块修改的。

我们来看这段代码：

``` java
public class MySQLConnection {

    public void connect() {
        System.out.println("MYSQL Connection");
    }
}

public class PasswordReminder {

    private MySQLConnection mySQLConnection;

    public PasswordReminder(MySQLConnection mySQLConnection) {
        this.mySQLConnection = mySQLConnection;
    }
}
```

有一种常见的误解是，依赖倒置只是依赖注入的另一种表达方式，实际上两者并不相同。

在上面这段代码中，尽管将MySQLConnection类注入了PasswordReminder类，但它依赖于MySQLConnection。而高层模块PasswordReminder是不应该依赖于低层模块MySQLConnection的。因此这不符合依赖倒置原则。

如果你想要把MySQLConnection改成MongoConnection，那就要在PasswordReminder中更改硬编码的构造函数注入。

要想符合依赖倒置原则，PasswordReminder就要依赖于抽象类（接口）而不是细节。那么应该怎么改这段代码呢？我们一起来看一下：

``` java
public interface ConnectionInterface {

    void connect();
}

public class MySQLConnection implements ConnectionInterface {

    public void connect() {
        System.out.println("MYSQL Connection");
    }
}

public class PasswordReminder {

    private ConnectionInterface connection;

    public PasswordReminder(ConnectionInterface connection) {
        this.connection = connection;
    }
}
```

修改后的代码中，如果我们想要将MySQLConnection改成MongoConnection，就不需要修改PasswordReminder类的构造函数注入，因为这里PasswordReminder类依赖于抽象而非细节。

感谢阅读！

### 原文地址

https://medium.com/better-programming/solid-principles-simple-and-easy-explanation-f57d86c47a7f

### 译者点评

作者对于SOLID原则介绍的还是比较清楚的，但是里氏原则那里我认为说得还不是很明白，举的例子似乎也不是很明确。我理解的里氏替换原则是：子类可以扩展父类的功能，但不能修改父类方法。因此里氏替换原则可以说是开闭原则的一种实现。当然，这篇文章也只是大概介绍了SOLID的每个原则，大家可以通过查资料来进行更详细的了解。我相信理解了这些设计原则之后，你对程序设计就会有更加深入的认识。后面我也会继续推送一些关于设计原则的文章，欢迎关注。