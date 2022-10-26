# 1.为什么使用dagger2
- 解决复杂对象依赖关系问题
  - 大项目中依赖关系特别复杂，创建一个类需要很多代码。把这些交给dagger2去处理
- 解耦
  - A依赖B，不需要依赖A的创建，当B的创建过程改变时不需要A改动代码

# 2.dagger2的发展过程
## 2.1. Spring
Spring解决了的问题：
* 流行的依赖注入、控制反转、好莱坞原则
* 解决了初始化顺序问题
* 使用xml配置组合关系

Spring的优缺点：

|优点|缺点|
|---|---|
|解决了初始化顺序|冗长的xml代码|
|实例管理（scoping）|运行时检查配置和图|
||应用流无法追踪|
||map样式的api|

## 2.2. Guice
Guice的优缺点：
|优点|缺点|
|---|---|
|绑定自动发现|运行时检查图|
|极大减少了配置|合成的类|
|配置和被配置的代码相近|应用流无法追踪|
|纯java|map样式的api|

## 2.3. Dagger1
Dagger1的优缺点：
|优点|缺点|
|---|---|
|运行时错误检查|丑陋的代码生成|
|更容易debug，完整的调用堆栈|运行时图混合|
|高效的错误提示|图创建效率低|
||部分trace能力|
||map样式的api|

## 2.4. Dagger2
Dagger2的优点：
- 可调试性
  - 所有代码可追踪（查找引用、打开定义）
  - 代码在调试时，简洁、被很好地构建
- 简洁的API
  - 增加Module
  - 没有更多的maps-api
- 性能
  - 和手写一样迅速
  - 不围绕框架编码

# 三、使用dagger2
dagger不允许多个Component对同一个对象进行注入，这种类似的操作可以用以下方式对多个Component进行组合：
- 继承
- 依赖

## 3.1 继承
继承主要用于继承某个Component的创建对象的能力。

继承都要使用@Subcomponent注解。
在Component中包含Subcomponent的Factory。

需要通过主Component才能访问到Subcomponent。

```kotlin
class Car {
    fun run() {
        println("car run")
    }
}

class Jeep {
    fun run() {
        println("jeep run")
    }
}

@Component(modules = [CarModule::class])
interface CarComponent {
    fun inject(car: ParentTester)
    val phoneFactory: JeepComponent.Factory
}

@Module
class CarModule {

    @Provides
    fun provideCar(): Car {
        return Car()
    }
}

@Subcomponent(modules = [JeepModule::class])
interface JeepComponent {

    fun inject(tester: ChildTester)

    @Subcomponent.Factory
    interface Factory {
        fun create(): JeepComponent
    }
}

@Module
class JeepModule {

    @Provides
    fun provideJeep(): Jeep {
        return Jeep()
    }
}

class Tester {

    companion object {
        @JvmStatic
        fun main(args: Array<String>) {
            ParentTester().test()
            ChildTester().test()
        }
    }
}

class ParentTester {
    @Inject
    lateinit var car: Car

    fun test() {
        println("*** ParentTester ***")
        DaggerCarComponent.create().inject(this)
        car.run()
    }
}

class ChildTester {
    @Inject
    lateinit var jeep: Jeep

    @Inject
    lateinit var car: Car

    fun test() {
        println("*** ChildTester ***")
        DaggerCarComponent.create().phoneFactory.create().inject(this)
        car.run()
        jeep.run()
    }
}
```

输出结果：
```
*** ParentTester ***
wheel scroll
*** ChildTester ***
wheel scroll
phone run
```

此外，在@Module中还可以使用subcomponents指定子组件
```kotlin
@Module(subcomponents = [JeepComponent::class])
class CarModule {

    @Provides
    fun provideCar(): Car {
        return Car()
    }
}
```

继承生成的DaggerXXXComponent文件只有一个。

## 3.2. 依赖
声明要被依赖的Module和Component，Component直接暴露DependencyObject成员给外面：
```kotlin
@Module
class DependencyModule {

    @Provides
    fun provide(): DependencyObject {
        return DependencyObject()
    }
}

@Component(modules = [DependencyModule::class])
interface DependencyComponent {
    val dependencyChildObject: DependencyObject
}
```
声明依赖者的Component，它没有自己的Module，使用dependencies依赖上面的Component：
~~~kotlin
@Component(dependencies = [DependencyComponent::class])
interface DependentComponent {
    fun inject(dependencyTester: DependencyTester)
}
~~~
最终调用，成功注入DependencyObject：
~~~kotlin
class DependencyTester {
    @Inject
    lateinit var dependencyObject: DependencyObject

    fun test() {
        println("*** DependencyTester ***")
        val dependencyComponent = DaggerDependencyComponent.create()
        DaggerDependentComponent.builder().dependencyComponent(dependencyComponent)
            .build().inject(this)
        println("${dependencyObject.hashCode()}")
    }
}
~~~
移除DependencyComponent的成员变量导致编译失败。

感觉上，继承可以继承到父Module的实例创建能力，而依赖貌似只能依赖到被依赖者Component中绑定的变量。

和继承不同，依赖所生成的DaggerXXXComponent文件有多个。


## 3.3. 局部单例
局部单例的意思是，同一个Component，对一个类型的多个对象进行注入时，注入的这个对象是单例的。
如果搞两个Component，那仍然不是单例的。

在module的provide方法上声明@Singleton
~~~kotlin
@Module(subcomponents = [JeepComponent::class])
class CarModule {

    @Singleton
    @Provides
    fun provideCar(): Car {
        return Car()
    }
}
~~~
然后在Component上声明@Singleton，否则会报错
~~~kotlin
@Singleton
@Component(modules = [CarModule::class])
interface CarComponent {
    fun inject(car: ParentTester)
    val phoneFactory: JeepComponent.Factory
}
~~~

- 单例的实现原理是：对Provider执行装饰模式，覆写get方法，使用double-check实现单例。

## 3.4. @Scope及其依赖限制
@Scope表示单例。
@Singleton也使用了@Scope注解。

Component互相依赖的时候，有一些限制：
1. 不同的Component不能有相同的Scope。解决方法就是抄@Singleton的代码，再声明一个Scope，用新声明的Scope去修饰Component，使依赖的时候两个Component的Scope不同
2. 没有Scope的Component不能依赖有Scope的Component

## 3.5. Provider和Lazy的区别
区别是Lazy生成的是单例

## 3.6. @Named使用
同类型变量的多个实例使用@Named标记