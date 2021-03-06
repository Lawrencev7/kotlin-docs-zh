# 类型安全构建器

使用命名良好的函数作为构建器，再加上带接收者的函数字面量，就有可能创建类型安全（type-safe）、静态类型（statically-typed）的构建器。

类型安全构建器允许我们创建基于 Kotlin 的领域特定语言（DSL），这种 DSL 可以以一种半声明（semi-declarative）的方式来构建层级化的数据结构（hierarchical data structures）。使用场景有：

- 用 Kotlin 代码生成标记内容，例如 HTML 或 XMl；
- 用编程方式来布局 UI 组件：Anko；
- 为 web 服务配置路由：Ktor。

## 一个类型安全构建器的例子

考虑如下代码：

```kotlin
import com.example.html.* // see declarations below

fun result(args: Array<String>) =
    html {
        head {
            title {+"XML encoding with Kotlin"}
        }
        body {
            h1 {+"XML encoding with Kotlin"}
            p  {+"this format can be used as an alternative markup to XML"}

            // an element with attributes and text content
            a(href = "http://kotlinlang.org") {+"Kotlin"}

            // mixed content
            p {
                +"This is some"
                b {+"mixed"}
                +"text. For more see the"
                a(href = "http://kotlinlang.org") {+"Kotlin"}
                +"project"
            }
            p {+"some text"}

            // content generated by
            p {
                for (arg in args)
                    +arg
            }
        }
    }
```

这是一段完全合法的 Kotlin 代码。

## 工作原理

我们可以简单地过一下 Kotlin 实现类型安全构建器的机制。首先我们要构建所需要的模型，这个 case 需要对 HTML 标签进行建模。它可以很容易地利用一系列类来做到。例如，`HTML` 是一个描述 `<html>` 标签的类，它定义了像 `<head>` 以及 `<body>` 这样的子节点。

现在，我们回忆一下为什么可以写成如下形式：

```kotlin
html {
 // ...
}
```

`html` 的确是一个用 lambda 表达式作为参数的函数调用。这个函数的定义如下：

```kotlin
fun html(init: HTML.() -> Unit): HTML {
    val html = HTML()
    html.init()
    return html
}
```

这个函数有一个名为 `init` 的参数，其本身也是一个函数。函数的类型是 `HTML.() -> Unit`（*带接收者的函数类型*）。这就意味着，我们需要给这个函数传入一个 `HTML` 类型的实例（*接收者*），并且在函数内部我们可以调用这个实例的成员。访问接收者可以使用 `this` 关键词：

```kotlin
html {
    this.head { /* ... */ }
    this.body { /* ... */ }
}
```

（`head` 和 `body` 都是 `HTML` 的成员函数。）

现在，照例可以省略 `this`，一个非常像构建器的东西已经有了：

```kotlin
html {
    head { /* ... */ }
    body { /* ... */ }
}
```

那么，这个调用会做什么呢？我们来看一下 `html` 的函数体。它会创建一个 `HTML` 的实例，然后调用作为参数传递给它的函数进行初始化（在这个例子中，会归结于调用 `HTML` 实例的 `head` 和 `body` 方法），然后返回这个实例。这的确是一个构建器应该做的。

`HTML` 类中 `head` 和 `body` 函数的定义类似于 `html`。唯一的不同点是它们会把构建的实例添加到这个封闭的 `HTML` 实例的 `children` 集合中去。

```kotlin
fun head(init: Head.() -> Unit) : Head {
    val head = Head()
    head.init()
    children.add(head)
    return head
}

fun body(init: Body.() -> Unit) : Body {
    val body = Body()
    body.init()
    children.add(body)
    return body
}
```

实际上，这两个函数做了同样的事情，所以我们便有了一个泛型版本，`initTag`：

```kotlin
protected fun <T : Element> initTag(tag: T, init: T.() -> Unit): T {
    tag.init()
    children.add(tag)
    return tag
}
```

所以，函数会变得非常简单：

```kotlin
fun head(init: Head.() -> Unit) = initTag(Head(), init)

fun body(init: Body.() -> Unit) = initTag(Body(), init)
```

并且，我们可以使用它们来构建 `<head>` 和 `<body>` 标签。

另一个需要讨论的事情是我们如何把文本添加给标签体。上面的例子中我们可以写成如下形式：

```kotlin
html {
    head {
        title {+"XML encoding with Kotlin"}
    }
    // ...
}
```

所以，基本上我们只需要把一个字符串放在标签体内就可以，但是它前面有一个 `+`，所以这是一个调用前缀 `unaryPlus()` 操作的函数调用。这个操作实际上是通过一个扩展函数 `unaryPlus()` 来定义的，这个函数是抽象类 `TagWithText` 的成员（`Title` 的父类）。

```kotlin
operator fun String.unaryPlus() {
    children.add(TextElement(this))
}
```

所以，`+` 前缀在这里所做的事是把一个字符串包裹成 `TextElement` 实例，然后把它添加到 `children` 集合中，所以它就变成了这颗标签树合理的一部分。

所有这些定义在一个 `com.example.html` 的包中，然后在上例构建器的声明之上导入。最后一节有这个包的完整定义。

## 范围控制：@DslMarker（从 1.1 开始支持）
当使用 DSL 时，可能会遇到一种情况：上下文中有太多可被调用的函数。在 lambda 内，因为我们可以调用每个可用的隐式接收者的方法，因而会得到不一致的结果，例如位于 `head` 标签嵌套在另一个 `head` 中：

```kotlin
html {
    head {
        head {} // should be forbidden
    }
}
```

在这个例子中，只有最近的隐式接收者（`this@head`）的成员才可用；`head()` 是外部接收者（`this@html`）的一个成员，所以调用它必须是不合法的。

为了解决这个问题，Kotlin 1.1 引入了一个特殊的机制来控制接收者范围。

为了使编译器能够控制范围，我们只需要给所有 DSL 中用到的接收者类型标记相同的注解即可。例如，可以为 HTML 构建器声明一个注解 `@HTMLTagMarker`：

```kotlin
@DslMarker
annotation class HtmlTagMarker
```

用 `@DslMarker` 标注的注解类叫 DSL 标记器。

在我们的 DSL 中，所有的标签类都继承了同一个超类 - `Tag`。使用 `@HtmlTagMarker` 仅仅标记超类就足够了，之后编译器会认为所有的继承类已经标记过了。

```kotlin
@HtmlTagMarker
abstract class Tag(val name: String) { ... }
```

没有必要用给 `HTML` 或 `Head` 类添加 `@HtmlTagMarker` 注解，因为它们的父类已经加过了：

```kotlin
class HTML() : Tag("html") { ... }
class Head() : Tag("head") { ... }
```

添加了这个注解之后，Kotlin 编译器就能够知道哪些隐式接收者属于同一个 DSL，并且只会允许调用最近的接收者的成员：

```kotlin
html {
    head {
        head { } // error: a member of outer receiver
    }
    // ...
}
```

注意，这里仍然可以调用外部接收者的成员，但是必须显示指明接收者：

```kotlin
html {
    head {
        this@html.head { } // possible
    }
    // ...
}
```


## `com.example.html` 包的完整定义

下面的代码展示了如何定义 `com.example.html` 包（只包含了上例中用到的元素）。它构建了一个 HTML 树。大量使用了扩展函数和带接收者的 lambda。

注意，`@DslMarker` 注解只有在 Kotlin 1.1 之后才可用。

```kotlin
package com.example.html

interface Element {
    fun render(builder: StringBuilder, indent: String)
}

class TextElement(val text: String) : Element {
    override fun render(builder: StringBuilder, indent: String) {
        builder.append("$indent$text\n")
    }
}

@DslMarker
annotation class HtmlTagMarker

@HtmlTagMarker
abstract class Tag(val name: String) : Element {
    val children = arrayListOf<Element>()
    val attributes = hashMapOf<String, String>()

    protected fun <T : Element> initTag(tag: T, init: T.() -> Unit): T {
        tag.init()
        children.add(tag)
        return tag
    }

    override fun render(builder: StringBuilder, indent: String) {
        builder.append("$indent<$name${renderAttributes()}>\n")
        for (c in children) {
            c.render(builder, indent + "  ")
        }
        builder.append("$indent</$name>\n")
    }

    private fun renderAttributes(): String {
        val builder = StringBuilder()
        for ((attr, value) in attributes) {
            builder.append(" $attr=\"$value\"")
        }
        return builder.toString()
    }

    override fun toString(): String {
        val builder = StringBuilder()
        render(builder, "")
        return builder.toString()
    }
}

abstract class TagWithText(name: String) : Tag(name) {
    operator fun String.unaryPlus() {
        children.add(TextElement(this))
    }
}

class HTML : TagWithText("html") {
    fun head(init: Head.() -> Unit) = initTag(Head(), init)

    fun body(init: Body.() -> Unit) = initTag(Body(), init)
}

class Head : TagWithText("head") {
    fun title(init: Title.() -> Unit) = initTag(Title(), init)
}

class Title : TagWithText("title")

abstract class BodyTag(name: String) : TagWithText(name) {
    fun b(init: B.() -> Unit) = initTag(B(), init)
    fun p(init: P.() -> Unit) = initTag(P(), init)
    fun h1(init: H1.() -> Unit) = initTag(H1(), init)
    fun a(href: String, init: A.() -> Unit) {
        val a = initTag(A(), init)
        a.href = href
    }
}

class Body : BodyTag("body")
class B : BodyTag("b")
class P : BodyTag("p")
class H1 : BodyTag("h1")

class A : BodyTag("a") {
    var href: String
        get() = attributes["href"]!!
        set(value) {
            attributes["href"] = value
        }
}

fun html(init: HTML.() -> Unit): HTML {
    val html = HTML()
    html.init()
    return html
}
```
