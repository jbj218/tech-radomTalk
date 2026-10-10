# 从代理到 AOP

> 代理和 AOP 有什么关系？感觉它们都是在业务逻辑上横切一刀。

## 一、代理和 AOP 的关系

"横切一刀"正是 AOP 的核心思想。不过代理和 AOP 不是并列关系，而是**手段和目的**的关系。

### 1.1 一句话区分

- **AOP**：一种编程思想、一个目标——把横切关注点（日志、事务、权限）从业务代码里抽出来，统一处理。
- **代理**：实现 AOP 的主要技术手段之一——在目标对象外面套一层，调用先经过这层，就能"横切"进去。

AOP 是"想干什么"，代理是"怎么干"。

### 1.2 为什么代理能"横切"

正常调用：`调用方 → 目标对象`

加了代理：`调用方 → 代理对象 → 目标对象`

代理和目标实现同一个接口（或继承同一个类），调用方以为自己在调目标，实际先到代理。代理在这一层就能插入额外逻辑：

```java
// 代理里大致这样
public void save(User u) {
    System.out.println("开启事务");   // 横切逻辑：前置
    target.save(u);                    // 调真正的业务方法
    System.out.println("提交事务");   // 横切逻辑：后置
}
```

所有方法都这么套一层，就相当于在业务逻辑的"竖切"上，横着切了一刀统一处理。这就是"横切一刀"。

### 1.3 Spring AOP 用的两种实现

| 方式             | 原理                           | 要求                  |
| :--------------- | :----------------------------- | :-------------------- |
| **JDK 动态代理** | 运行时生成一个实现相同接口的类 | 目标必须有接口        |
| **CGLIB 代理**   | 运行时生成目标的子类，重写方法 | 目标不能被 final 修饰 |

按老规矩是"有接口用 JDK 代理，没接口用 CGLIB"。但 Spring Boot 2.x 之后 `spring.aop.proxy-target-class` 默认为 `true`，也就是**默认全用 CGLIB**，即使有接口。原因在第五节展开。

下面把这两种实现分别拆开看。重点放在 CGLIB，因为 2.x 以后默认走的就是它。

## 二、JDK 动态代理（基于接口）

**核心要求**：目标类必须实现接口。代理类和目标是"兄弟"关系（都实现同一接口）。

### 2.1 准备目标

```java
public interface UserService {
    void save(String name);
    String find(int id);
}

public class UserServiceImpl implements UserService {
    public void save(String name) {
        System.out.println("保存用户: " + name);
    }
    public String find(int id) {
        return "用户" + id;
    }
}
```

### 2.2 写 InvocationHandler

```java
import java.lang.reflect.*;

public class LogHandler implements InvocationHandler {
    private final Object target;  // 被代理的真实对象

    public LogHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("[前置] 调用方法: " + method.getName());
        Object result = method.invoke(target, args);  // 反射调用真实对象
        System.out.println("[后置] 返回: " + result);
        return result;
    }
}
```

### 2.3 生成代理并调用

```java
public class Main {
    public static void main(String[] args) {
        UserService target = new UserServiceImpl();

        // 关键：用 Proxy.newProxyInstance 生成代理对象
        UserService proxy = (UserService) Proxy.newProxyInstance(
            target.getClass().getClassLoader(),      // 类加载器
            target.getClass().getInterfaces(),       // 代理要实现的接口
            new LogHandler(target)                   // 调用处理器
        );

        proxy.save("张三");
        proxy.find(1);
    }
}
```

输出：

```text
[前置] 调用方法: save
保存用户: 张三
[后置] 返回: null
[前置] 调用方法: find
[后置] 返回: 用户1
```

**要点**：

- 代理对象 `proxy` 实现了 `UserService` 接口，但**不是** `UserServiceImpl`。
- 所有方法调用都被转发到 `invoke`，在 `method.invoke(target, args)` 前后插入逻辑。
- `method.invoke(target, args)` 走的就是反射。

### 2.4 Spring 里的等价写法

```java
@Service
public class UserServiceImpl implements UserService {
    @Transactional  // 这就是"横切逻辑"，Spring 帮你生成代理
    public void save(String name) { ... }
}
```

Spring 检测到 `@Transactional`，发现 `UserServiceImpl` 有接口，就**自动**用 JDK 动态代理生成一个代理对象放进容器。你注入的 `UserService` 其实是代理，不是 `UserServiceImpl`。

------

## 三、CGLIB 代理（基于继承）

**核心要求**：目标类没有接口，或强制用 CGLIB。代理类是目标的**子类**，重写父类方法。

### 3.1 准备目标（无接口）

```java
public class OrderService {
    public void create(String orderNo) {
        System.out.println("创建订单: " + orderNo);
    }
}
```

### 3.2 写 MethodInterceptor

CGLIB 用的是 `MethodInterceptor`，不是 JDK 的 `InvocationHandler`。

```java
import net.sf.cglib.proxy.*;
import java.lang.reflect.Method;

public class LogInterceptor implements MethodInterceptor {
    @Override
    public Object intercept(Object obj, Method method, Object[] args,
                            MethodProxy proxy) throws Throwable {
        System.out.println("[前置] 调用: " + method.getName());
        // 注意：用 proxy.invokeSuper，不是 method.invoke
        Object result = proxy.invokeSuper(obj, args);
        System.out.println("[后置] 返回: " + result);
        return result;
    }
}
```

#### 四个参数的含义

| 参数                | 含义                                                         |
| :------------------ | :----------------------------------------------------------- |
| `Object obj`        | **生成的代理对象本身**（就是 `enhancer.create()` 返回的那个子类实例）。注意不是目标对象，是代理自己。 |
| `Method method`     | 被拦截的**方法**（反射的 `Method` 对象），比如 `OrderService.create`。 |
| `Object[] args`     | 这次调用的**实参**，比如 `{"ORDER-001"}`。                   |
| `MethodProxy proxy` | CGLIB 提供的**方法代理**，用来调用"父类的原始方法"。它是 CGLIB 内部生成的一个辅助对象，**不是**上面那个业务代理。 |

关键区别在最后两个：`method` 是反射的 `Method`，`proxy` 是 CGLIB 的 `MethodProxy`。调父类逻辑时：

```java
proxy.invokeSuper(obj, args);   // 调父类（真实业务逻辑）
// 而不是 method.invoke(target, args)
```

CGLIB 故意用 `MethodProxy.invokeSuper` 而不用反射 `method.invoke`，因为前者是直接字节码调用，快得多。

#### MethodProxy 到底是个什么东西

**`MethodProxy` 没有继承 `obj`**。它和 `obj` 是完全不同的两个对象：`obj` 是代理实例（业务对象），`MethodProxy` 是一个独立的、用于"快速调用原始方法"的工具对象。

`MethodProxy` 本身是一个普通的类，它不继承任何代理类。它的核心作用是通过 `FastClass` 机制，绕开反射，快速定位并调用方法。

它的内部维护了一个 `FastClassInfo`，包含几个关键字段：

- **`f1`**：被代理类（父类，如 `OrderService`）的 `FastClass`。
- **`f2`**：代理类（子类，如 `OrderService$$EnhancerByCGLIB`）的 `FastClass`。
- **`i1`**：被代理类中目标方法的**索引下标**。
- **`i2`**：代理类中"传送门"方法的**索引下标**。

当调用 `invokeSuper(obj, args)` 时，它实际执行的是 `f2.invoke(i2, obj, args)`。这行代码的意思是：通过代理类的 `FastClass`，根据下标 `i2`，在 `obj`（代理实例）上调用那个隐藏的"传送门"方法（如 `CGLIB$sayHello$0`），从而绕过拦截器，直接执行父类的原始逻辑。

**它和代理对象 `obj` 之间是引用关系，不是继承关系。**

在 CGLIB 生成的代理类字节码中，为每个方法都创建了一个静态的 `MethodProxy` 对象：

```java
// 生成的代理类内部（简化）
private static final MethodProxy CGLIB$sayHello$0$Proxy;

static {
    // 创建 MethodProxy 时，把被代理类和代理类的信息传了进去
    CGLIB$sayHello$0$Proxy = MethodProxy.create(
        OrderService.class,                   // 父类（被代理类）
        OrderService$$EnhancerByCGLIB.class,  // 子类（代理类）
        "()V",
        "sayHello",                           // 原始方法名
        "CGLIB$sayHello$0"                    // 代理类中的传送门方法名
    );
}
```

当代理方法被触发时，它会把这个预生成的 `MethodProxy` 实例，连同 `this`（也就是 `obj`）一起传给拦截器：

```java
// 生成的代理类中，被重写的业务方法
public final void sayHello() {
    // 这里传递了 this（obj）和静态的 MethodProxy 实例
    this.CGLIB$CALLBACK_0.intercept(
        this,                    // obj：代理实例自己
        CGLIB$sayHello$0$Method,
        CGLIB$emptyArgs,
        CGLIB$sayHello$0$Proxy   // proxy：工具对象
    );
}
```

所以，`obj` 是"在谁身上调用"，`proxy` 是"用什么工具调用"。`MethodProxy` 内部并不持有 `obj` 的引用，它只是知道自己该去调用哪个类的哪个方法，需要你在调用时通过 `invokeSuper(obj, args)` 把目标对象 `obj` 传给它。

### 3.3 生成代理并调用

```java
import net.sf.cglib.proxy.Enhancer;

public class Main {
    public static void main(String[] args) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(OrderService.class);      // 设置父类
        enhancer.setCallback(new LogInterceptor());      // 设置回调

        OrderService proxy = (OrderService) enhancer.create(); // 生成子类代理
        proxy.create("ORDER-001");
    }
}
```

输出：

```text
[前置] 调用: create
创建订单: ORDER-001
[后置] 返回: null
```

**要点**：

- 代理类是 `OrderService` 的**子类**，所以 `OrderService` 不能被 `final` 修饰，方法也不能 `final`。
- `proxy.invokeSuper(obj, args)` 调的是父类（真实逻辑），不是反射调用目标实例，这是和 JDK 代理最大的区别。
- 因为靠继承，**不需要接口**。

### 3.4 Spring 里的等价写法

```java
@Service
public class OrderService {   // 没有接口
    @Transactional
    public void create(String orderNo) { ... }
}
```

Spring 发现没有接口，自动切换成 CGLIB，生成 `OrderService` 的子类代理。

------

## 四、两种方式对比

|              | JDK 动态代理                          | CGLIB 代理                          |
| :----------- | :------------------------------------ | :---------------------------------- |
| 前提         | 目标必须有接口                        | 无需接口，但不能 final              |
| 代理关系     | 实现同一接口（兄弟）                  | 继承目标（子类）                    |
| 回调接口     | `InvocationHandler`                   | `MethodInterceptor`                 |
| 调用真实逻辑 | `method.invoke(target, args)`（反射） | `proxy.invokeSuper(obj, args)`      |
| 生成方式     | `Proxy.newProxyInstance`              | `Enhancer.create`                   |
| 性能         | 生成快，调用稍慢                      | 生成慢，调用快（现代 JDK 差距很小） |

------

## 五、Spring 里怎么选

默认按"有接口用 JDK，没接口用 CGLIB"走，但 Spring Boot 2.x 起改成了默认强制 CGLIB：

```java
@EnableAspectJAutoProxy(proxyTargetClass = false) // false=JDK优先，true=强制CGLIB
```

对应的配置项是 `spring.aop.proxy-target-class`，从 2.x 开始默认 `true`，也就是**默认全用 CGLIB**，即使有接口。原因是 CGLIB 能代理类本身，注入时不会有"只能按接口注入"的限制。

------

## 六、把几个概念串起来

```java
// JDK 代理的 invoke 里
method.invoke(target, args);
//     ↑ 反射调用真实对象
```

- **AOP** 是目的：把 `@Transactional` 这种横切逻辑抽出来。
- **代理** 是手段：JDK 或 CGLIB 生成一个套壳对象。
- **反射** 是 JDK 代理内部的实现细节：`method.invoke(target, args)`。

三个概念在这一段代码里全部汇合。
