---
layout: post
title: "高效的 Json 解析框架 Kotlinx.serialization"
date:  2023-12-11 17:58:12 +0800
categories: ["技术", "编程", "Kotlin"]
tag: ["Kotlin", "Json", "serialization"]
---

## 一、引出问题
你是否有在使用 Gson 序列化对象时，见到如下异常：

```
Abstract classes can't be instantiated! Register an InstanceCreator or a TypeAdapter for this type.
```

什么时候会出现如此异常。下面举个栗子：

```
import com.google.gson.Gson
import com.google.gson.reflect.TypeToken

sealed class Gender
object Male: Gender()
object Female: Gender()

data class Student(
    val id: Int,
    val name: String,
    val gender: Gender
)

fun main() {
    val list1 = listOf(
        Student(1001, "Jimy", Male),
        Student(1002, "Lucy", Female),
        Student(1003, "HanMeimei", Female),
        Student(1004, "LiLei", Male)
    )
    println("list1: $list1")
    val jsonString = Gson().toJson(list1)
    println("jsonString: $jsonString")
    try {
        val typeToken = object : TypeToken<List<Student>>() {}.type
        val list2: List<Student> = Gson().fromJson(jsonString, typeToken)
        println("list2: $list2")
    } catch (ex: Exception) {
        println("catch: ${ex.message}")
    }
}
```

上面的代码，执行结果如下：

```
list1: [Student(id=1001, name=Jimy, gender=serialize.gson.Male@79fc0f2f), Student(id=1002, name=Lucy, gender=serialize.gson.Female@50040f0c), Student(id=1003, name=HanMeimei, gender=serialize.gson.Female@50040f0c), Student(id=1004, name=LiLei, gender=serialize.gson.Male@79fc0f2f)]
jsonString: [{"id":1001,"name":"Jimy","gender":{}},{"id":1002,"name":"Lucy","gender":{}},{"id":1003,"name":"HanMeimei","gender":{}},{"id":1004,"name":"LiLei","gender":{}}]
catch: Abstract classes can't be instantiated! Register an InstanceCreator or a TypeAdapter for this type. Class name: serialize.gson.Gender
```

从这个输出结果，我们可以看到两个问题：

1. `list1` 经过序列化，得到的 `jsonString` 中， `gender` 属性是空。
2. `jsonString` 反序列化过程中发生了异常。

## 二、解决问题
异常信息已经指明了问题的解决方案

```
Abstract classes can't be instantiated! Register an InstanceCreator or a TypeAdapter for this type.
```

>抽象类无法实例化！为此类型注册 InstanceCreator 或 TypeAdapter。

其实也很好理解。 `sealed class`、`abstract class`、`interface` 都是抽象的，不能直接被实例化。对于抽象类的子类或者接口的实现类，应该明确制定序列化和反序列化的规则。由于我们没有注册 TypeAdapter， 默认的 TypeAdapter ，将 Gender 属性序列化为了空对象。在进行反序列化时，空对象不知道应该如何反序列化，所以抛出了如下的异常。

解决办法之一，在序列化和反序列化时，需要使用 Gson 的 `registerTypeAdapter` 或 `registerTypeHierarchyAdapter` 方法来处理密封类的子类。

首先为抽象类/接口创建一个 TypeAdapter

```
class GenderTypeAdapter: TypeAdapter<Gender>() {
    override fun write(out: JsonWriter?, value: Gender?) {
        out?.value(value?.javaClass?.name)
    }

    override fun read(`in`: JsonReader?): Gender {
        return when(val className = `in`?.nextString()) {
            Male::class.java.name -> Male
            Female::class.java.name -> Female
            else -> throw IllegalArgumentException("Unknown class name: $className")
        }
    }
}
```

然后为 Gson 对象注册该 typeAdapter

```
fun main() {
    val list1 = listOf(
        Student(1001, "Jimy", Male),
        Student(1002, "Lucy", Female),
        Student(1003, "HanMeimei", Female),
        Student(1004, "LiLei", Male)
    )
    println("list1: $list1")

    // I'm here
    val jsonString = GsonBuilder().registerTypeAdapter(Gender::class.java, GenderTypeAdapter()).create().toJson(list1)
    
    println("jsonString: $jsonString")
    try {
        val typeToken = object : TypeToken<List<Student>>() {}.type

        // I'm here
        val list2: List<Student> = GsonBuilder().registerTypeAdapter(Gender::class.java, GenderTypeAdapter()).create().fromJson(jsonString, typeToken)
        
        println("list2: $list2")
    } catch (ex: Exception) {
        println("catch: ${ex.message}")
    }
}
```

此时执行结果如下：

```
list1: [Student(id=1001, name=Jimy, gender=serialize.gson.Male@79fc0f2f), Student(id=1002, name=Lucy, gender=serialize.gson.Female@50040f0c), Student(id=1003, name=HanMeimei, gender=serialize.gson.Female@50040f0c), Student(id=1004, name=LiLei, gender=serialize.gson.Male@79fc0f2f)]
jsonString: [{"id":1001,"name":"Jimy","gender":"serialize.gson.Male"},{"id":1002,"name":"Lucy","gender":"serialize.gson.Female"},{"id":1003,"name":"HanMeimei","gender":"serialize.gson.Female"},{"id":1004,"name":"LiLei","gender":"serialize.gson.Male"}]
list2: [Student(id=1001, name=Jimy, gender=serialize.gson.Male@79fc0f2f), Student(id=1002, name=Lucy, gender=serialize.gson.Female@50040f0c), Student(id=1003, name=HanMeimei, gender=serialize.gson.Female@50040f0c), Student(id=1004, name=LiLei, gender=serialize.gson.Male@79fc0f2f)]
```

Ok, 没有问题。
那... `registerTypeAdapter` 和 `registerTypeHierarchyAdapter` 两个方法有什么区别呢？

它们的主要区别在于注册对象的范围不同。

- **registerTypeAdapter** 用于为特定的 Java 对象或类型注册自定义的序列化和反序列化逻辑。使用 TypeAdapter，可以在 Gson 序列化或反序列化特定对象或类型时，对其进行自定义处理。TypeAdapter 只会被应用于所注册的对象或类型。

- **registerTypeHierarchyAdapter** 方法则是用于为特定类及其子类注册自定义的序列化和反序列化逻辑。使用 registerTypeHierarchyAdapter 方法，可以为一个类及其子类注册自定义的序列化和反序列化逻辑，这个逻辑将被应用于该类及其所有子类。这在处理一组类继承结构时非常有用。

- 在使用 `registerTypeHierarchyAdapter` 方法时，需要注意一点，即 Gson 会遍历所有的子类来找到最合适的 TypeAdapter，因此要确保该 TypeAdapter 能够正确处理所有的子类。如果某个子类没有对应的处理逻辑，或者处理逻辑有误，就可能导致序列化或反序列化失败。

因此，如果要为一组类继承结构注册自定义的序列化和反序列化逻辑，可以使用 registerTypeHierarchyAdapter 方法；如果只需要为某个具体的 Java 对象或类型注册自定义的序列化和反序列化逻辑，则可以使用 TypeAdapter。

## 三、用 kotlinx.serialization 进行Kotlin JSON序列化
Gson 是针对 java 对象的序列化框架。基于 Kotlin 对象使用 Gson 框架，会失去 Kotlin 的一些重要特性，比如：

- 非空类型安全。比如 Kotlin 类的属性定义为非空类型时，仍然可以将一个 null 赋值给它创建一个对象。
- 参数默认值没有效果。Kotlin 属性可以赋予默认值。但是当使用 Gson 时，将会失去效果。
修改之前的例子：

```
sealed class Gender
object Male: Gender()
object Female: Gender()

data class Student(
    val id: Int,
    val name: String = "unknown",
    val gender: Gender
)

fun main() {
    val json = """ 
       {
           "id": 1005
       }
    """.trimIndent()
    try {
        val stu = Gson().fromJson(json, Student::class.java)
        println("stu: $stu")
    } catch (ex: Exception) {
        println("catch: ${ex.message}")
    }
}
```

这里我们在定义 Student 类是，给 name 属性指定了一个默认值 `unknown`, 在进行反序列化时，没有指定 name 和 gender， 看看执行结果：

```
stu: Student(id=1005, name=null, gender=null)
```

结果也表明，name 的默认值没有成功，并且 name 和 gender 都赋值为 null 了。

针对上述问题有很多解决办法。但是这里，我要介绍一个新的 Json 框架，Kotlin 团队开发的一个 native 支持的库 `kotlinx.serialization`, 这个库支持JVM，JavaScript，Native所有平台，同时也支持多种格式的序列化——JSON，CBOR，protocol buffers等等。

### 3.1 kotlinx.serialization 的使用
1. plugins 引入：

```
plugins {
    id("org.jetbrains.kotlin.plugin.serialization") version("1.4.30")
}
```

2. dependencies 引入：

```
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.2")
}
```

3. 通过添加 `@Serializable` 注解，给类进行序列化

```
package serialize.ktxSerialization

import kotlinx.serialization.Serializable
import kotlinx.serialization.decodeFromString
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json

@Serializable
sealed class Gender

@Serializable
object Male: Gender()

@Serializable
object Female: Gender()

@Serializable
data class Student(
    val id: Int,
    val name: String = "unknown",
    val gender: Gender
)
```

***注意：所涉及到的抽象类极其子类都需要加上该注解。***

测试代码：

```
fun main() {
    val json = """
       {
         "id": 1005
       }
    """.trimIndent()

    try {
        val stu = Json.decodeFromString<Student>(json)
        println("stu: $stu")
    } catch (ex: Exception) {
        println("catch: ${ex.message}")
    }
}
```

反序列化的关键方法：

```
Json.decodeFromString()
```

执行报错了：

```
catch: Field 'gender' is required for type with serial name 'serialize.ktxSerialization.Student', but it was missing at path: $
```

错误信息指出： gender 属性是必须的。那我们应该如何该如何添加 gender 属性呢？
不急，我们先序列化看看生成的是什么。

```
fun main() {
    val student = Student(1006, "James", Male)
    val jsonString = Json.encodeToString(student)
    println("jsonString: $jsonString")
}
```

执行结果如下：

```
jsonString: {"id":1006,"name":"James","gender":{"type":"serialize.ktxSerialization.Male"}}
```

我们看到，Student 对象序列化之后， `gender` 对应的 value 是
`{"type":"serialize.ktxSerialization.Male"}`
这里是完整的包名类名。

到这里，我们再手动构造验证一下：

```
fun main() {
    val json = """
       {
         "id": 1005,
         "gender": {"type": "serialize.ktxSerialization.Female"}
       }
    """.trimIndent()
    try {
        val stu = Json.decodeFromString<Student>(json)
        println("stu: $stu")
    } catch (ex: Exception) {
        println("catch: ${ex.message}")
    }
}
```

执行结果：

```
stu: Student(id=1005, name=unknown, gender=serialize.ktxSerialization.Female@36d64342)
```

可以看到，反序列化成功，生成的对象，name 属性赋了默认值。

另外需要注意的是：如果在定义 Kotlin 的类中某个属性，没有指定默认值，即便该属性是可空类型，反序列化时也一定要赋值才能执行成功。

修改下例子：

```
package serialize.ktxSerialization

import kotlinx.serialization.Serializable
import kotlinx.serialization.decodeFromString
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json

@Serializable
sealed class Gender

@Serializable
object Male: Gender()

@Serializable
object Female: Gender()

@Serializable
data class Student(
    val id: Int,
    val name: String?,  // 注意这里
    val gender: Gender
)

fun main() {
    val json = """
       {
         "id": 1005,
         "gender": {"type": "serialize.ktxSerialization.Female"}
       }
    """.trimIndent()
    try {
        val stu = Json.decodeFromString<Student>(json)
        println("stu: $stu")
    } catch (ex: Exception) {
        println("catch: ${ex.message}")
    }
}
```

我把 name 设置为可空类型，但是没有默认值。这时反序列化是会失败的：

```
catch: Field 'name' is required for type with serial name 'serialize.ktxSerialization.Student', but it was missing at path: $
```

给 name 属性赋值为 null, 则执行成功

```
fun main() {
    val json = """
       {
         "id": 1005,
         "name", null,
         "gender": {"type": "serialize.ktxSerialization.Female"}
       }
    """.trimIndent()
    try {
        val stu = Json.decodeFromString<Student>(json)
        println("stu: $stu")
    } catch (ex: Exception) {
        println("catch: ${ex.message}")
    }
}
```

结果：

```
stu: Student(id=1005, name=null, gender=serialize.ktxSerialization.Female@340f438e)
```

### 3.2 用 kotlinx.serialization 解决本文开头的问题
对于本文开头引出的问题，如果使用 kotlinx.serialization，则该问题即可轻松解决。
直接上代码：

```
fun main() {
    val list1 = listOf(
        Student(1001, "Jimy", Male),
        Student(1002, "Lucy", Female),
        Student(1003, "HanMeimei", Female),
        Student(1004, "LiLei", Male)
    )
    println("list1: $list1")
    val jsonString = Json.encodeToString(list1)
    println("jsonString: $jsonString")
    try {
        val list2 = Json.decodeFromString<List<Student>>(jsonString)
        println("list2: $list2")
    } catch (ex: Exception) {
        println("catch: ${ex.message}")
    }
}
```

执行结果：

```
list1: [Student(id=1001, name=Jimy, gender=serialize.ktxSerialization.Male@531d72ca), Student(id=1002, name=Lucy, gender=serialize.ktxSerialization.Female@22d8cfe0), Student(id=1003, name=HanMeimei, gender=serialize.ktxSerialization.Female@22d8cfe0), Student(id=1004, name=LiLei, gender=serialize.ktxSerialization.Male@531d72ca)]
jsonString: [{"id":1001,"name":"Jimy","gender":{"type":"serialize.ktxSerialization.Male"}},{"id":1002,"name":"Lucy","gender":{"type":"serialize.ktxSerialization.Female"}},{"id":1003,"name":"HanMeimei","gender":{"type":"serialize.ktxSerialization.Female"}},{"id":1004,"name":"LiLei","gender":{"type":"serialize.ktxSerialization.Male"}}]
list2: [Student(id=1001, name=Jimy, gender=serialize.ktxSerialization.Male@531d72ca), Student(id=1002, name=Lucy, gender=serialize.ktxSerialization.Female@22d8cfe0), Student(id=1003, name=HanMeimei, gender=serialize.ktxSerialization.Female@22d8cfe0), Student(id=1004, name=LiLei, gender=serialize.ktxSerialization.Male@531d72ca)]
```

这里很好理解：
在没有给 Gson 注册 TypeAdapter 的时候，使用默认的 TypeAdapter, 把引用类型序列化为了空。反序列化时才会失败。而是用 kotlinx.serialization ，相当于默认提供了一个序列化和反序列方案。所以直接可以成功。无需我们自己定义序列化和反序列化的规则。

## 四、总结
最后对本文做个总结：

- 在使用 Gson 进行序列化和反序列过程中。要注意多态的情况下。需要自己注册 TypeAdapter。
- 如果使用 Kotlin 开发，优先使用高效的序列化框架：**kotlinx.serialization**。

**kotlinx.serialization** 具有如下特性：

1. 类型安全：满足 Kotlin 的强制类型安全。可处理 Kotlin 的可空类型。
2. 支持属性默认值：解析 JSON 的时候支持 Kotlin 类中属性的默认值。
3. 支持泛型类型：API在序列化和反序列化泛型类型的时候非常简单也非常高效。
4. 序列化字段名：当 json 的 key 和字段名不一致时，可以通过 @SerialName 给字段进行序列化。 同 Gson 中的 `@SerializedName`。
5. 序列化引用对象：当属性的类型是引用类型时，对该类型也需要使用 @Serializable 注解。
6. 数据校验：可以再 json 反序列化时对数据进行校验。
7. 支持 Retrofit 库。详见针对Retrofit 2 Converter.Factory的Kotlin序列化的库。

kotlinx.serialization 有很多优秀的特性。本文算是抛砖引玉。更多特性，请自己手动 Coding 体验。
最后附上 kotlinx.serialization 的官方文档：[https://github.com/Kotlin/kotlinx.serialization](https://github.com/Kotlin/kotlinx.serialization)