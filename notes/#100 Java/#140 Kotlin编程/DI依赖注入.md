reference: https://juejin.cn/post/7294965012749320218?searchId=202402181722590B857C96F369A05D7787
reference: https://cloud.tencent.com/developer/article/1753272

## 注入概念

-------------
Dependency Injection 简称DI
为什么引入：
1. 解耦
2. 消除模板代码

## 如何注入接口
```kotlin
@HiltAndroidApp
class MyApp : Application() {
    @Inject
    lateinit var software: ISoftware

    override fun onCreate() {
        super.onCreate()
        println("inject result:${software.printName()}")
    }
}
```
```kotlin
interface ISoftware {
    fun printName()
}

class SoftwareImpl @Inject constructor(): ISoftware{
    override fun printName() {
        println("name is fish")
    }
}

@Module
@InstallIn(SingletonComponent::class)
abstract class SoftwareModule {
    @Binds
    abstract fun bindSoftware(impl: SoftwareImpl):ISoftware
}
```

Dagger1 -> Dagger2 -> Hilt
那么Dagger2和Dagger1不同的地方在哪里呢？最重要的不同点在于，实现方式完全发生了变化。刚才我们已经知道，Dagger1是基于Java反射实现的， 并
且列举了它的一些弊端。而Google开发的Dagger2是基于Java注解实现的，这样就把反射的那些弊端全部解决了。

通过注解，Dagger2会在编译时期自动生成用于依赖注入的代码，所以不会增加任何运行耗时。另外，Dagger2会在编译时期检查开发者的依赖注入用法是
否正确，如果不正确的话则会直接编译失败，这样就能将问题尽可能早地抛出。也就是说，只要你的项目正常编译通过，基本也就说明你的依赖注入用法没
什么问题了。