# 使用方法句柄（MethodHandle）和变量句柄（VarHandle）以编程方式访问方法和字段

### 方法句柄（MethodHandle）

方法句柄是Java 7 中引入的一种新机制，提供了一种直接、高效且类型安全的方法来调用方法。与反射相比，MethodHandle 的调用速度更快，因为它们不需要在运行时解析方法签名。

使用示例：

要使用方法句柄，首先需要获取一个方法的句柄。这可以通过`MethodHandles.lookup()`方法实现，该方法返回一个`Lookup`对象，用于查找方法句柄。然后，可以使用`findStatic`、`findVirtual`等方法获取特定方法的句柄。

```java
import java.lang.invoke.MethodHandle; 
import java.lang.invoke.MethodHandles; 
import java.lang.invoke.MethodType; 
public class MethodHandleExample { 
    public static void sayHello(String name) { 
       System.out.println("Hello, " + name + "!"); 
    } 
    public static void main(String[] args) throws Throwable { 
        // 获取方法句柄 
        MethodHandle handle = MethodHandles.lookup().findStatic( MethodHandleExample.class, 
        "sayHello", 
        MethodType.methodType(void.class, String.class) ); 
        // 调用方法 handle.invoke("World"); 
    } 
}
```

### 变量句柄（VarHandle）

变量句柄是 Java 9 中引入的一种机制，用于直接访问类的字段。与反射相比，变量句柄提供了更高效的字段访问，并且同样具有类型安全性。

使用示例：

要使用变量句柄，首先需要获取一个字段的句柄。这可以通过`MethodHandles.lookup()`方法与`VarHandle`工厂方法结合使用来实现。然后，可以使用`get`、`set`等方法来读取或写入字段的值。

```java
import java.lang.invoke.MethodHandles; 
import java.lang.invoke.VarHandle; 
public class VarHandleExample { 
    private static int counter = 0; 
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException { 
    // 获取字段句柄 
    VarHandle handle = MethodHandles.lookup().findStaticVarHandle( VarHandleExample.class, "counter", int.class ); 
    // 读取字段值 
    int value = (int) handle.get(); 
    System.out.println("Initial counter value: " + value); // 更新字段值 handle.set(value + 1); value = (int) handle.get(); System.out.println("Updated counter value: " + value); 
    } 
}
```

目前，有一个区分资深 Java 开发者的一个标准是他们对反射及其现代替代方案的熟悉程度。 虽然反射赋予了你强大的能力，但它同时也显得笨拙、容易出错，并且存在性能瓶颈。现代Java 正致力于用标准化的选项来取代反射，其中包括 `MethodHandle和VarHandle`。与反射类似，这些类也让你能够访问对象上的方法和字段，并且更加简洁。

## 方法句柄\\变量句柄 Vs. 反射

为了真正理解 `MethodHandles`和`VarHandles` 它们的作用以及为什么有用，首先我们应该了解Java中的反射机制。这对理解句柄是很有帮助的。同时，这将有助于你理解为什么反射会演变成这些新的 API。

首先有一个基本问题是：反射、方法句柄、变量句柄这些技术满足了什么需求？为什么我们明明可以直接实例化一个对象、调用其公共方法并访问其公共成员，却还要以编程方式来做这些事情呢？

在很多情况下，你实际上无法通过公共方法访问到你需要的内容，因此你必须绕开常规途径。这种情况通常发生在你编写像框架代码这样的程序时，这些代码需要对一系列类进行操作，并对它们执行一些非标准的操作。

举一个持久化框架的例子。假设你需要将类与表之间进行映射，那么你需要获取这些类以了解它们有哪些字段和方法。这种场景在应用代码中也会出现，特别是当你需要访问一个旧版库中其他方式无法访问的部分时。

那么现在决定使用哪种技术需要了解实际需求。如果你可以通过正常的Java调用来解决问题，那么这就是最好的方法。如果你需要更复杂的东西，推荐先看看标准 API，比如`MethodHandles`和`VarHandles`。只有当这些都无法满足需求时，你才应该退回到反射。

下面通过一些例子可以帮助理解。我们先从一个反射的例子开始。

## 使用反射来访问方法

```java
public class MyClass {
  private String name;
  public MyClass(String name) {
    this.name = name;
  }
  public String getName() { 
    return name;
  }
}
```

上面代码是一个非常简单的例子：一个包含字符串名称以及getter和setter方法的类。为了创建这个类，我们可以使用正常的实例化方法：

```java
MyClass objectInstance = new MyClass("John Doe");
```

使用反射访问方法的方式：

```java
Class<?> clazz = objectInstance.getClass();
Method method = clazz3.getDeclaredMethod("getName");
String value = (String) method.invoke(objectInstance);
System.out.println(value); // prints "John Doe"
```

## 使用 MethodHandles 访问方法的方式

方法句柄为我们提供了与反射相同的能力，但语法更加安全。

```java
Class<?> clazz = objectInstance.getClass();  
MethodHandle handle = MethodHandles.lookup().findVirtual(clazz, "getName", methodType(String.class));
String value = (String) handle.invoke(objectInstance);
System.out.println(value);  // Prints “John Doe”
```

我们以相同的方式开始，通过实例获取其对应的类。然后，我们使用`MethodHandles`上的`lookup().findVirtual()`方法。这种方式相比反射，提供了一种更清洁，且JDK认可的方法来访问方法。

## 直接访问字段

现在假设我们之前的类`MyClass`中有一个`name`字段，但是没有提供访问器（即getter方法）。我们现在需要一种更强大的方法来访问它，因为我们要直接访问这个私有成员。以下是使用标准反射来实现这一点的方法：

```java
Class<?> clazz = objectInstance.getClass(); 
Field field = clazz.getDeclaredField("name"); 
field.setAccessible(true); 
String value = (String) field.get(objectInstance);
System.out.println(value); // prints “John Doe”
```

注意，我们再次直接与对象的元数据（如其类和字段）进行交互。我们可以通过`setAccessible` 方法来操纵字段的可访问性。

现在，让我们使用变量句柄来完成同样的事情：

```java
Class<?&gtl clazz = objectInstance.getClass();
VarHandle handle = MethodHandles.privateLookupIn(clazz,   
  MethodHandles.lookup()).findVarHandle(clazz, "name", String.class);
String value = (String) handle.get(objectInstance);
System.out.println(value4); // prints "John Doe"
```

在这里，我们使用`privateLookupIn`是因为字段被标记为私有。

尽管上述代码可以工作，但出于性能考虑，建议将句柄本身静态地实例化，如下所示：

```java
private static VarHandle HANDLE;
  static {
    try {
      HANDLE = MethodHandles.privateLookupIn(MyClass.class, MethodHandles.lookup()).findVarHandle(MyClass.class, "name", String.class);
    } catch (Throwable t){
      throw new RuntimeException(t);
    }
  }

// …

System.out.println("static: " + HANDLE.get(objectInstance));
```

在这里，我们已经静态地实例化了`HANDLE`变量，并在后续的正常代码流程中使用了它。这也强调了这样一个事实：句柄本身是为类型（`MyClass`）定义的，然后为实例（`objectInstance`）重用。

请注意，直接实例化句柄需要你知道类的名称。如果你不知道类的名称，比如你正在使用一个字符串来反射性地实例化那个类，然后访问它的字段，那么你就不能使用这种方法。

## 方法句柄和变量句柄的限制

不过如前所述，方法句柄和变量句柄不支持类的实例化，这在某些场景下带来了限制。

## 结尾

在基准测试中，像我们之前那样静态地声明句柄可以显著提高性能。这是因为 JVM 可以在编译时将这些信息内联。但是，正如前面提到的，这样做并不总是可能的。例如，如果你在编译时不知道类的名称。

