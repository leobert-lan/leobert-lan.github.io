# Groovy：MOP一文打尽

## 写在最前

Groovy已经不再是一门新出现的语言，而笔者是在2013年左右接触到它的，并且在2017年时，有机会尝试使用它编写了SpringBoot项目。

但说来惭愧，在很长的一段时间里，我都没有系统的学习它。并且时至今日，我也 `不推荐` 大家再去 `系统的学习` 它，毕竟 `使用它的机会越发地少了`，
但是我依旧认为大家有必要花费一些零碎的时间，快餐式的了解它。

### 如何产生编写Groovy系列的想法

* **Android从业者** 或者 **对新技术比较敏感的朋友** 都知道：Jetbrain 的 Compose 项目即将 *甚至已经* 掀起热潮，而Google 借着从 `Kotlin语言`
传承而来的友好合作关系，也大搞特稿 `Jetpack Compose`，并且 Android势必会成为重要战场。
* Compose必须结合Kotlin，无巧不巧，*据笔者目测盲目统计*，绝大多数的项目均使用了 `Kotlin Script` 进行Gradle项目配置
* Gradle项目的源码中，Groovy代码量第一
* 使用 `Kotlin Script` 并使用 `Kotlin-dsl` 时，往往需要对Gradle本身的内容有深入的了解
* **无意冒犯**，大家对 `Gradle本身的内容` 、 `Groovy语言` 、甚至是原先的 `Gradle Script` 可能都没有较深的了解

在一连串的背景下，痛定思痛，还是先简单学习下 `Groovy语言的高级特效`，这样就可以：

* 理解原先的 `Gradle Script` 内容含义
* 具有快速阅读 `Gradle` 源码的能力
* 明白使用 `Kotlin Script` 时需要对哪些问题对症下药

## MetaObject Protocol 元对象协议

本文直接从MOP开始，忽略掉Groovy的大量基础部分，因为这些基础部分，基本和Java一致。

MOP的目标在于：`运行期进行实时变化`，听起来有点像Java的反射，但是要比Java的反射更加强大。可以：

* 修改方法名、属性名，
* 动态增加类的方法、属性 等等

### 神奇的方法分配-- `invokeMethod` 和 `methodMissing`

首先我们简单了解一下Groovy的方法分配机制：

<div style="background-color: white">
<img src="https://upload-images.jianshu.io/upload_images/6168671-819123f425c9c5bc.png" alt="invokemethod_methodmissing" />
</div>

*图片来自网络*

**本节内容，结果上看起来很神奇，但内容比较枯燥，并且对非语言使用者帮助不大，可以泛读，有个印象即可**

#### invokeMethod 拦截示例

我们先看一个简单的例子：
```groovy
class Demo1 {

    def foo() {
        println 'foo'
    }

    def invokeMethod(String name, Object args) {
        return "unknown method $name(${args.join(',')})"
    }

    static void main(String[] args) {
        def demo = new Demo1()
        demo.foo()
        println demo.bar("A", "B")
    }

}
```

如果是Java或者Kotlin，很明显，`Demo1` 中并没有 `bar` 方法，编译不能通过，但Groovy可以通过编译，得到运行结果：

```shell
> Task :Demo1.main()
foo
unknown method bar(A,B)
```

这就是所谓的 `invokeMethod`，Groovy中设计了一个顶层接口：`GroovyObject`

```groovy
public interface GroovyObject {
    Object invokeMethod(String var1, Object var2);

    Object getProperty(String var1);

    void setProperty(String var1, Object var2);

    MetaClass getMetaClass();

    void setMetaClass(MetaClass var1);
}
```
编译结果：

```groovy
package osp.leobert.groovyworkshop.mop;

import groovy.lang.GroovyObject;
import groovy.lang.MetaClass;
import groovy.transform.Generated;
import org.codehaus.groovy.runtime.GStringImpl;
import org.codehaus.groovy.runtime.callsite.CallSite;

public class Demo1 implements GroovyObject {
    @Generated
    public Demo1() {
        CallSite[] var1 = $getCallSiteArray();
        super();
        MetaClass var2 = this.$getStaticMetaClass();
        this.metaClass = var2;
    }

    public Object foo() {
        CallSite[] var1 = $getCallSiteArray();
        return var1[0].callCurrent(this, "foo");
    }

    public Object invokeMethod(String name, Object args) {
        CallSite[] var3 = $getCallSiteArray();
        return new GStringImpl(new Object[]{name, var3[1].call(args, ",")}, new String[]{"unknown method ", "(", ")"});
    }

    public static void main(String... args) {
        CallSite[] var1 = $getCallSiteArray();
        Object demo = var1[2].callConstructor(Demo1.class);
        var1[3].call(demo);
        var1[4].callStatic(Demo1.class, var1[5].call(demo, "A", "B"));
    }
}

```

我们发现，调用 `foo()` 方法时，并未执行 `invokeMethod` 中的逻辑，而Groovy中还有一个特殊的接口：`GroovyInterceptable`，如果实现它的话：

```groovy
class Demo2 implements GroovyInterceptable {

    def foo(String s) {
        return "foo:$s"
    }

    def invokeMethod(String name, Object args) {
        return "unknown method $name(${args.join(',')})"
    }

    static void main(String[] args) {
        def demo = new Demo2()
        println demo.foo("a")
        println demo.bar("A", "B")
    }
}
```

我们将得到如下结果：

```shell
> Task :Demo2.main()
unknown method foo(a)
unknown method bar(A,B)
```

`foo()` 方法并未被分配!!, 这个场景很容易让我们联想到 `Java的动态代理` 并 `拦截、自行分配方法执行`

#### methodMissing示例

```groovy
class Demo3 {

    def foo(String s) {
        return "foo:$s"
    }

    def invokeMethod(String name, Object args) {
        return "unknown method $name(${args.join(',')})"
    }

    def methodMissing(String name, Object args) {
        return "methodMissing $name(${args.join(',')})"
    }

    static void main(String[] args) {
        def demo = new Demo3()
        println demo.foo("a")
        println demo.bar("A", "B")
    }
}
```

按照前文的分派流程图，我们猜到结果为：
```shell
> Task :Demo3.main()
foo:a
methodMissing bar(A,B)
```
而实现了 `GroovyInterceptable` 的情况下，就无法再利用methodMissing机制拦截，而是按照 `GroovyInterceptable` 走 invokeMethod拦截
```groovy
class Demo4 implements GroovyInterceptable {

    def foo(String s) {
        return "foo:$s"
    }

    def invokeMethod(String name, Object args) {
        return "unknown method $name(${args.join(',')})"
    }

    def methodMissing(String name, Object args) {
        return "methodMissing $name(${args.join(',')})"
    }

    static void main(String[] args) {
        def demo = new Demo4()
        println demo.foo("a")
        println demo.bar("A", "B")
    }
}
```

```shell
> Task :Demo4.main()
unknown method foo(a)
unknown method bar(A,B)
```

### 动态处理类的属性

TODO ：gpath ，类比kotlin

TODO：运行期来增加属性

TODO：`getProperty(String property)`， `setProperty(String property,Object value)` 结合集合来批量增加属性

### 特殊的Expando类


### 利用ExpandoMetaClass实现Mixin机制

Mixin 即 Mix In，混合
