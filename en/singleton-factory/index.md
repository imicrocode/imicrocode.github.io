# 单例工厂模式


**单例工厂模式**确保一个类只有一个实例, 并提供一个全局访问点。

<!--more-->

## 1 饿汉模式
```java
/**
 * 饿汉模式
 * 
 *      优点: 线程安全，调用时效率高 
 *      缺点: 不管是否使用都创建对象，可能造成资源浪费
 */
public class Hungry {

    private Hungry() {
    }

    private static Hungry hungry = new Hungry();

    public static Hungry getInstance() {
        return hungry;
    }

}
```
## 2 普通懒汉模式
```java
/**
 * 懒汉模式
 * 
 *      优点: 调用时才创建对象,不会浪费资源 
 *      缺点: 非线程安全
 */
public class LazyNotSafe {

    private LazyNotSafe() {
    }

    private static LazyNotSafe lazyNotSafe;

    public static LazyNotSafe getInstance() {
        if (lazyNotSafe == null) {
            lazyNotSafe = new LazyNotSafe();
        }
        return lazyNotSafe;
    }

}
```
## 3 加锁懒汉模式
```java
/**
 * 加锁懒汉模式
 * 
 *      优点: 线程安全 
 *      缺点: 方法锁,效率较低
 */
public class LazySync {

    private LazySync() {
    }

    private static LazySync lazySync;

    public static synchronized LazySync getInstance() {
        if (lazySync == null) {
            lazySync = new LazySync();
        }
        return lazySync;
    }

}
```
## 4 双重检查锁
```java
/**
 * 双重检查锁懒汉模式 
 * 
 *      优点: 线程安全,效率较高
 * 
 * 注意：lazyDoubleLock对象需要加volatile关键字,禁止JVM指令重排,否则可能导致返回未创建好的对象
 */
public class LazyDoubleLock {

    private LazyDoubleLock() {
    }

    private static volatile LazyDoubleLock lazyDoubleLock;

    public static LazyDoubleLock getInstance() {
        if (lazyDoubleLock == null) {
            synchronized (LazyDoubleLock.class) {
                if (lazyDoubleLock == null) {
                    lazyDoubleLock = new LazyDoubleLock();
                }
            }
        }
        return lazyDoubleLock;
    }

}
```
## 5 关于以上单例工厂实现的思考
&nbsp;&nbsp;&nbsp;&nbsp;以上单例工厂类或多或少存在部分问题, 主要以下2个问题:
- **反射攻击**
  > 可以禁止通过构造器实例化解决
- **序列化问题**
  > `java.io.ObjectOutputStream`代表对象输出流,它的`writeObject(Object obj)`方法可对参数指定的obj对象进行序列化,把得到的字节序列写到一个目标输出流中。
  >
  > `java.io.ObjectInputStream`代表对象输入流,它的`readObject()`方法从一个源输入流中读取字节序列,再把它们反序列化为一个对象，并将其返回。
### 5.1 ObjectOutputStream源码
```java
/**
 * Underlying writeObject/writeUnshared implementation.
 */
private void writeObject0(Object obj, boolean unshared)
        throws IOException {
        
    //......
    //此处省略

    // remaining cases
    if (obj instanceof String) {
        writeString((String) obj, unshared);
    } else if (cl.isArray()) {
        writeArray(obj, desc, unshared);
    } else if (obj instanceof Enum) {
        writeEnum((Enum<?>) obj, desc, unshared);
    } else if (obj instanceof Serializable) {
        writeOrdinaryObject(obj, desc, unshared);
    } else {
        if (extendedDebugInfo) {
            throw new NotSerializableException(
                    cl.getName() + "\n" + debugInfoStack.toString());
        } else {
            throw new NotSerializableException(cl.getName());
        }
    }

    //......
    //此处省略

}
```
通过以上源码可知:
- String, Array, Enum不用实现Serializable接口, 其他情况下必须实现Serializable接口才能支持序列化;
### 5.2 ObjectInputStream源码
```java
/**
 * Reads and returns "ordinary" (i.e., not a String, Class,
 * ObjectStreamClass, array, or enum constant) object, or null if object's
 * class is unresolvable (in which case a ClassNotFoundException will be
 * associated with object's handle).  Sets passHandle to object's assigned
 * handle.
 */
private Object readOrdinaryObject(boolean unshared) throws IOException {
    if (bin.readByte() != TC_OBJECT) {
        throw new InternalError();
    }

    //......
    //此处省略

    Object obj;
    try {
        obj = desc.isInstantiable() ? desc.newInstance() : null;
    } catch (Exception ex) {
        throw (IOException) new InvalidClassException(
                desc.forClass().getName(),
                "unable to create instance").initCause(ex);
    }

    //......
    //此处省略

    if (obj != null &&
                handles.lookupException(passHandle) == null &&
                desc.hasReadResolveMethod()) {
            Object rep = desc.invokeReadResolve(obj);
        if (unshared && rep.getClass().isArray()) {
            rep = cloneArray(rep);
        }
        if (rep != obj) {
            // Filter the replacement object
            if (rep != null) {
                if (rep.getClass().isArray()) {
                    filterCheck(rep.getClass(), Array.getLength(rep));
                } else {
                    filterCheck(rep.getClass(), -1);
                }
            }
            handles.setObject(passHandle, obj = rep);
        }
    }
    
    //......
    //此处省略

    return obj;

}
```
```java
/**
 * Reads in and returns enum constant, or null if enum type is
 * unresolvable.  Sets passHandle to enum constant's assigned handle.
 */
private Enum<?> readEnum(boolean unshared) throws IOException {
    if (bin.readByte() != TC_ENUM) {
        throw new InternalError();
    }

    //......
    //此处省略

    String name = readString(false);
    Enum<?> result = null;
    Class<?> cl = desc.forClass();
    if (cl != null) {
        try {
            @SuppressWarnings("unchecked")
            Enum<?> en = Enum.valueOf((Class) cl, name);
            result = en;
        } catch (IllegalArgumentException ex) {
            throw (IOException) new InvalidObjectException(
                    "enum constant " + name + " does not exist in " +
                            cl).initCause(ex);
        }
        if (!unshared) {
            handles.setObject(enumHandle, result);
        }
    }

    //......
    //此处省略

    return result;

}
```
通过以上源码可知:
- 对象反序列化时, 通过反射创建
- 对象反序列化时, 如果存在`readResolve()`方法, 则使用该方法返回的对象
- 枚举反序列化时, 本身保持单例
### 5.3 反射实例化源码
```java
public T newInstance(Object ... initargs)
    throws InstantiationException, IllegalAccessException,
            IllegalArgumentException, InvocationTargetException
{
    if (!override) {
        Class<?> caller = Reflection.getCallerClass();
        checkAccess(caller, clazz, clazz, modifiers);
    }
    if ((clazz.getModifiers() & Modifier.ENUM) != 0)
        throw new IllegalArgumentException("Cannot reflectively create enum objects");
    ConstructorAccessor ca = constructorAccessor;   // read volatile
    if (ca == null) {
        ca = acquireConstructorAccessor();
    }
    @SuppressWarnings("unchecked")
    T inst = (T) ca.newInstance(initargs);
    return inst;
}
```
通过以上源码可知:
- 枚举不能通过反射实例化
### 5.4 结论
通过以上分析:
- 类中有`readResolve()`方法可以解决反序列化破坏
- 枚举是最完美的单例工厂模型
## 6 推荐单例工厂处理
### 6.1 静态内部类
```java
/**
 * 静态内部类
 * 
 *      优点:线程安全,效率较高.
 */
public class StaticInner implements Serializable {

    private static final long serialVersionUID = 3676394038652350456L;

    private StaticInner() {
        // 构造器判断,防止反射攻击
        if (InnerInstance.STATIC_INNER != null) {
            throw new IllegalStateException();
        }
    }

    /**
     * 防止序列化攻击
     */
    private Object readResolve() throws ObjectStreamException {
        return InnerInstance.STATIC_INNER;
    }

    public static StaticInner getInstance() {
        return InnerInstance.STATIC_INNER;
    }

    private static class InnerInstance {
        private static final StaticInner STATIC_INNER = new StaticInner();
    }

}
```
### 6.2 枚举
```java
/**
 * 最完美的单例工厂模型
 */
public enum EnumSingleton {

    INSTANCE;

    private final Object instance;

    EnumSingleton() {
        instance = new Object();
    };

    public Object getInstance() {
        return instance;
    }

}
```
## 7 单例工厂模式总结
|                | 饿汉模式 | 懒汉模式 | 加锁懒汉模式 | 双重锁检查 | 静态内部类 | 枚举 |
| -------------- | -------- | -------- | ------------ | ---------- | ---------- | ---- |
| **延迟加载**   | 否       | 是       | 是           | 是         | 是         | 是   |
| **线程安全**   | 是       | 否       | 是           | 是         | 是         | 是   |
| **高效**       | 是       | 是       | 否           | 是         | 是         | 是   |
| **序列化安全** | 否       | 否       | 否           | 否         | 否         | 是   |
| **反射安全**   | 否       | 否       | 否           | 否         | 否         | 是   |
- 如果对空间要求不高，可以用饿汉模式
- 懒汉模式由于非线程安全，加锁懒汉模式由于效率太低，一般不建议使用
- 一般来说，双重检查锁是用得较多的懒汉模式
- 静态内部类效果上跟懒汉模式差不多, 但更常见
- 枚举不管从代码量还是功能上讲，都是目前最推崇的单例模式，只是用习惯的人不多

