# JDK 动态代理 vs CGLIB

## 核心区别

| 对比 | JDK 动态代理 | CGLIB |
| --- | --- | --- |
| 实现方式 | 基于接口 | 基于继承 |
| 是否要求接口 | 是 | 否 |
| 代理对象 | 实现同一接口的代理类 | 目标类的子类 |
| 核心组件 | `Proxy`、`InvocationHandler` | `MethodInterceptor` |
| `final` 类 | 不受“继承目标类”限制 | 不能代理 |
| `final` 方法 | 可通过接口调用代理 | 不能通过重写增强 |
| `private` 方法 | 不能作为接口代理入口 | 不能通过子类重写增强 |

---

## JDK 动态代理

目标类需要实现接口：

```java
interface UserService {
    void save();
}

class UserServiceImpl implements UserService {
    public void save() {
        System.out.println("save");
    }
}
```

代理关系：

```text
        UserService
        /         \
       ↓           ↓
UserServiceImpl   Proxy
```

调用流程：

```text
方法调用
→ Proxy
→ InvocationHandler.invoke()
→ 目标方法
```

特点：

- 基于接口实现代理。
- 代理类和目标类实现同一个接口。
- 核心是 `Proxy + InvocationHandler`。

---

## CGLIB

CGLIB 不要求目标类实现接口，而是生成目标类的子类。

```text
UserService
    ↑
    │ extends
CGLIB Proxy
```

调用流程：

```text
方法调用
→ 代理子类
→ MethodInterceptor
→ 目标方法
```

特点：

- 基于继承实现代理。
- 通过生成子类、重写方法完成增强。
- `final` 类不能被继承，因此无法使用这种方式代理。
- `final`、`private` 方法不能被子类重写，因此无法通过 CGLIB 重写增强。

---

## Spring AOP 中的使用

Spring AOP 的调用本质：

```text
Caller
  ↓
Proxy
  ↓
增强逻辑
  ↓
Target
```

Spring 会根据目标类型和代理配置选择 JDK 动态代理或基于类的代理。

不要死记：

```text
有接口 → 永远 JDK
没接口 → 永远 CGLIB
```

因为代理方式可以配置。

---

## 性能

早期 JDK 动态代理常通过反射调用目标方法，因此部分场景下 CGLIB 性能更好。

现代 JVM 已对反射和方法调用做了大量优化，两者性能差距通常不是主要选型因素。

实际更应该关注：

- 是否有接口。
- 是否需要基于类进行代理。
- 是否存在 `final` 限制。
- 代理对象需要暴露什么类型。

---

## 速记

```text
JDK
→ 基于接口
→ Proxy + InvocationHandler

CGLIB
→ 基于继承
→ 生成目标类子类
→ MethodInterceptor
→ 受 final 限制
```

一句话：

> **JDK 动态代理是“我和你实现同一个接口”，CGLIB 是“我直接继承你”。**
