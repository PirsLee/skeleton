# 总体目标

一个dbus对象可以实现多个接口。传统做法是手动调用sd_bus_add_object_vtable多次，并手动管理InterfacesAdded/Removed信号。

sdbuplus希望通过C++模板机制，只需要指定所需要的接口类，就能够自动完成：

- 接口注册

- 对象添加/移除信号的自动发射

- 支持接口级信号InterfacesAdded和对象级信号ObjectAdded

# 结构

```
sdbusplus
    server
        object
            details
        object::details
```

# 核心类

```C++
template <class... Args>
struct object:
    details::compose<Args...>,                      //多重继承所有接口类
    private sdbusplus::bus::details::bus_friend     //友元，获取内部bus指针
```

# 组合引擎details::compose<Args...>

本质就是编译期的多重继承生成器

工作过程

```C++
sdbusplus::bus_t bus = /*...*/;
object_t<IfaceA, IfaceB> obj(bus, "/my/object", 
                              action::emit_interface_added);

struct object<IfaceA, IfaceB> : 
    details::compose<IfaceA, IfaceB>, 
    private bus_friend

// details::compose<IfaceA, IfaceB>
template<>
struct compose<IfaceA, IfaceB>
    : compose_inherit_t<IfaceA>
      compose_inherit_t<IfaceB>
{
    compose(bus_t& bus, const char* path)
        : IfaceA(bus, path), IfaceB(bus, path) {}   // 包扩展展开为逗号分隔的基类构造列表

    void maybe_emit_iface_added()
    {
        (try_emit<IfaceA>(), try_emit<IfaceB>());   // C++ 17 fold expression
    }
}

```

# 模板使用场景

> 当“一个对象由哪些能力组成”在编译期就确定了，而且这些能力彼此正交，需要零运行时开销，就用编译期模板组合

理解

