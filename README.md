# Java Reflection in IntelliJ IDEA — A Hands‑On Course

Welcome! This is a practical, step‑by‑step course that teaches Java Reflection **inside IntelliJ IDEA**. Each lesson includes what you’ll learn, concrete IntelliJ steps, runnable code, exercises, and a short quiz. Work through them in order.

---

## Prerequisites

* IntelliJ IDEA (Community or Ultimate)
* JDK 17 or newer installed and configured in IntelliJ

> Tip: The default IntelliJ project (without `module-info.java`) uses the **unnamed module**, which makes reflection examples easier. If you add modules later, see Lesson 7.

---

## Project Setup (once)

1. **Create project** → *New Project* → *Java* → Name: `reflection-labs` → Language level: 17+ → Finish.
2. Right‑click `src/main/java` → *New* → *Package* → `org.example.reflection`.
3. Right‑click the package → *New* → *Java Class* → `Person`.
4. Add a `Main` (or one per lesson as shown below). Run via the green ▶️ gutter icon.

Optional Gradle build (if you prefer):

```groovy
plugins { id 'java' }
java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }
repositories { mavenCentral() }
```

---

# Lesson 1 — Class Objects & Introspection Basics

**Goal:** Obtain `Class` objects at runtime, list fields/methods/constructors, instantiate objects reflectively, and invoke a public method.

### Steps in IntelliJ

1. In package `org.example.reflection`, create `Person` and `L1Basics`.

**`Person.java`**

```java
package org.example.reflection;

public class Person {
    private String name;

    public Person() { }
    public Person(String name) { this.name = name; }

    public String greet(String prefix) {
        return prefix + " " + name;
    }
}
```

**`L1Basics.java`**

```java
package org.example.reflection;

import java.lang.reflect.*;

public class L1Basics {
    public static void main(String[] args) throws Exception {
        // 1) Get Class in three ways
        Class<Person> viaLiteral = Person.class;
        Class<?> viaForName = Class.forName("org.example.reflection.Person");
        Person p0 = new Person("Salar");
        Class<?> viaInstance = p0.getClass();
        System.out.println(viaLiteral == viaForName && viaForName == viaInstance); // true

        // 2) List members
        System.out.println("Fields:");
        for (Field f : viaLiteral.getDeclaredFields()) System.out.println("  " + f);

        System.out.println("Constructors:");
        for (Constructor<?> c : viaLiteral.getDeclaredConstructors()) System.out.println("  " + c);

        System.out.println("Methods:");
        for (Method m : viaLiteral.getDeclaredMethods()) System.out.println("  " + m);

        // 3) Instantiate and call a public method
        Constructor<?> ctor = viaLiteral.getConstructor(String.class);
        Object person = ctor.newInstance("Salar");
        Method greet = viaLiteral.getMethod("greet", String.class);
        Object result = greet.invoke(person, "Hi");
        System.out.println(result); // Hi Salar
    }
}
```

### Try it

* Run `L1Basics`. Read the printed signatures; note return types, modifiers, and parameter types.

### Mini‑exercise

* Add an `age` field and a `getAge()` method to `Person`. Re‑run and observe listings.

### Quick quiz

1. What are three ways to obtain a `Class` object? 2) What’s the difference between `getMethods()` and `getDeclaredMethods()`?

---

# Lesson 2 — Accessing Private Members (Fields & Methods)

**Goal:** Read/write private fields and call private methods using reflection. Understand `setAccessible(true)` and when it works.

### Steps in IntelliJ

1. Create `Secrets.java` and `L2Access.java`.

**`Secrets.java`**

```java
package org.example.reflection;

class Secrets {
    private String token = "xyz-123";
    private void rotate(int salt) { this.token = token + ":" + salt; }
}
```

**`L2Access.java`**

```java
package org.example.reflection;

import java.lang.reflect.*;

public class L2Access {
    public static void main(String[] args) throws Exception {
        Secrets s = new Secrets();
        Class<?> c = s.getClass();

        Field token = c.getDeclaredField("token");
        token.setAccessible(true); // works in unnamed module projects
        System.out.println("before: " + token.get(s));
        token.set(s, "abc-999");
        System.out.println("after:  " + token.get(s));

        Method rotate = c.getDeclaredMethod("rotate", int.class);
        rotate.setAccessible(true);
        rotate.invoke(s, 42);
        System.out.println("rotated: " + token.get(s));
    }
}
```

### Notes

* In default IntelliJ projects (no `module-info.java`), you’re in the **unnamed module** and `setAccessible(true)` generally works.
* If you later use modules and hit `InaccessibleObjectException`, pass VM option `--add-opens your.module/org.example.reflection=ALL-UNNAMED` in **Run/Debug Configurations → VM options** (see Lesson 7).

### Mini‑exercise

* Add a second private field and set both fields via reflection in one go.

### Quick quiz

* What’s the difference between `getField` vs `getDeclaredField`? When do you need `setAccessible(true)`?

---

# Lesson 3 — Arrays, Enums, and Constructors

**Goal:** Create and manipulate arrays reflectively; inspect enum constants; choose and call specific constructors.

**`L3ArraysEnums.java`**

```java
package org.example.reflection;

import java.lang.reflect.*;
import java.util.Arrays;

public class L3ArraysEnums {
    enum Color { RED, GREEN, BLUE }

    public static void main(String[] args) throws Exception {
        // Arrays
        Class<?> intArrayClass = int[].class;
        Object dyn = Array.newInstance(int.class, 5);
        Array.setInt(dyn, 0, 7);
        Array.setInt(dyn, 1, 9);
        System.out.println("Array length: " + Array.getLength(dyn));
        System.out.println("Array as list: " + Arrays.toString((int[]) dyn));

        // Enums
        Class<Color> ec = Color.class;
        Object[] constants = ec.getEnumConstants();
        System.out.println("Enum constants: " + Arrays.toString(constants));

        // Constructors
        Class<StringBuilder> sbc = StringBuilder.class;
        for (Constructor<?> ctor : sbc.getDeclaredConstructors()) System.out.println("CTOR: " + ctor);
        StringBuilder sb = sbc.getConstructor(CharSequence.class).newInstance("hello");
        System.out.println(sb.toString());
    }
}
```

### Mini‑exercise

* Use reflection to create a multi‑dimensional array `int[3][2]` and set some elements.

### Quick quiz

* How do you obtain enum constants reflectively? What does `Array.getLength()` return for a multi‑dimensional array?

---

# Lesson 4 — Annotations & Parameter Metadata

**Goal:** Define custom annotations, read them at runtime, and inspect parameter names/types (including the `-parameters` compiler flag).

### Steps in IntelliJ

1. Create `Loggable.java` (annotation), `Greeter.java`, and `L4Annotations.java`.

**`Loggable.java`**

```java
package org.example.reflection;

import java.lang.annotation.*;

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.TYPE})
public @interface Loggable {
    String value() default "INFO";
}
```

**`Greeter.java`**

```java
package org.example.reflection;

@Loggable("DEBUG")
public class Greeter {
    @Loggable("TRACE")
    public String greet(String name, int times) {
        return ("hi ".repeat(Math.max(0, times))).trim() + ", " + name;
    }
}
```

**`L4Annotations.java`**

```java
package org.example.reflection;

import java.lang.reflect.*;
import java.util.Arrays;

public class L4Annotations {
    public static void main(String[] args) throws Exception {
        Class<Greeter> c = Greeter.class;

        // Class-level annotation
        Loggable classAnno = c.getAnnotation(Loggable.class);
        System.out.println("Class @Loggable: " + (classAnno != null ? classAnno.value() : "<none>"));

        // Method-level annotations & parameter metadata
        for (Method m : c.getDeclaredMethods()) {
            Loggable mAnno = m.getAnnotation(Loggable.class);
            System.out.println("Method " + m.getName() + " @Loggable=" + (mAnno != null ? mAnno.value() : "<none>"));

            // Parameter names (requires -parameters to be accurate)
            for (Parameter p : m.getParameters()) {
                System.out.println("  param: name=" + p.getName() + ", type=" + p.getType());
            }
        }

        // Invoke and print result
        Object g = c.getConstructor().newInstance();
        Method greet = c.getMethod("greet", String.class, int.class);
        System.out.println(greet.invoke(g, "Salar", 3));
    }
}
```

### Keep real parameter names

In IntelliJ: **Settings → Build, Execution, Deployment → Compiler → Java Compiler** → check **Store information about method parameters** (or add `-parameters` to *Additional command line parameters*). Rebuild and rerun to see real parameter names.

### Mini‑exercise

* Add an element `tags()` to `@Loggable` as `String[]` and print them for class and method.

### Quick quiz

* What `RetentionPolicy` do you need to read annotations at runtime? How do you get a method’s annotations?

---

# Lesson 5 — Dynamic Proxies (Interfaces)

**Goal:** Create runtime proxies for interfaces to add cross‑cutting behavior (logging, timing, auth) without modifying implementations.

**`Calculator.java`**

```java
package org.example.reflection;

public interface Calculator {
    int add(int a, int b);
    int mul(int a, int b);
}
```

**`SimpleCalculator.java`**

```java
package org.example.reflection;

public class SimpleCalculator implements Calculator {
    public int add(int a, int b) { return a + b; }
    public int mul(int a, int b) { return a * b; }
}
```

**`L5Proxy.java`**

```java
package org.example.reflection;

import java.lang.reflect.*;
import java.util.Arrays;

public class L5Proxy {
    @SuppressWarnings("unchecked")
    public static <T> T timingProxy(T target, Class<T> iface) {
        return (T) Proxy.newProxyInstance(
                iface.getClassLoader(), new Class<?>[]{iface},
                (proxy, method, args) -> {
                    long t0 = System.nanoTime();
                    try { return method.invoke(target, args); }
                    finally {
                        long dt = System.nanoTime() - t0;
                        System.out.println(method.getName() + Arrays.toString(args) + " took " + dt/1_000_000.0 + " ms");
                    }
                });
    }

    public static void main(String[] args) {
        Calculator calc = new SimpleCalculator();
        Calculator proxied = timingProxy(calc, Calculator.class);
        System.out.println(proxied.add(2, 3));
        System.out.println(proxied.mul(4, 5));
    }
}
```

### Mini‑exercise

* Extend the proxy to read `@Loggable` on the **interface methods** and only time those marked with a certain level.

### Quick quiz

* What are the three arguments to `Proxy.newProxyInstance`? What must your target type be to use JDK dynamic proxies?

---

# Lesson 6 — MethodHandles vs Reflection (Performance & Safety)

**Goal:** Use `java.lang.invoke` MethodHandles as a faster, type‑safe alternative for repeated calls.

**`L6MethodHandles.java`**

```java
package org.example.reflection;

import java.lang.invoke.*;

public class L6MethodHandles {
    public static class MathOps { public static int inc(int x) { return x + 1; } }

    public static void main(String[] args) throws Throwable {
        MethodHandles.Lookup lookup = MethodHandles.lookup();
        MethodType type = MethodType.methodType(int.class, int.class);
        MethodHandle inc = lookup.findStatic(MathOps.class, "inc", type);
        int r = (int) inc.invokeExact(41);
        System.out.println(r); // 42
    }
}
```

### Notes

* Reflective `Method.invoke` is flexible but slower; `MethodHandle` is faster and more type‑safe.

### Mini‑exercise

* Create a `MethodHandle` for `String::substring` and call it.

### Quick quiz

* When should you prefer `MethodHandle` over `Method`?

---

# Lesson 7 — Modules & Accessibility (Java 9+)

**Goal:** Understand why reflection may fail under the module system and how to open packages.

### Two options

1. **Open via VM options** (quick): in **Run/Debug Configurations → VM options**, add:

```
--add-opens java.base/java.lang=ALL-UNNAMED
--add-opens your.module/org.example.reflection=ALL-UNNAMED
```

2. **Open via `module-info.java`**:

```java
module your.module {
    opens org.example.reflection; // allow deep reflection into this package
}
```

### Mini‑exercise

* Trigger and fix an `InaccessibleObjectException` by accessing a private JDK class member, then add the appropriate `--add-opens`.

---

# Lesson 8 — ClassLoaders & Loading Types Dynamically

**Goal:** Discover how classes are found and loaded; load a class by name at runtime from the classpath.

**`L8ClassLoading.java`**

```java
package org.example.reflection;

public class L8ClassLoading {
    public static void main(String[] args) throws Exception {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        System.out.println("CL: " + cl);
        Class<?> listClass = cl.loadClass("java.util.ArrayList");
        System.out.println("Loaded: " + listClass);
    }
}
```

### Mini‑exercise

* Print the classloader hierarchy for `Greeter.class`.

---

# Lesson 9 — Scanning, Discovery & Simple Plugin Pattern

**Goal:** Find implementations of an interface and instantiate them. (Simple approach without external libs.)

**Idea:** Define an interface `Plugin` and manually maintain a registry list. Then instantiate by name using reflection.

**`Plugin.java`**

```java
package org.example.reflection;

public interface Plugin { String name(); void run(); }
```

**`HelloPlugin.java`**

```java
package org.example.reflection;

public class HelloPlugin implements Plugin {
    public String name() { return "hello"; }
    public void run() { System.out.println("Hello from plugin!"); }
}
```

**`L9Plugins.java`**

```java
package org.example.reflection;

import java.util.*;

public class L9Plugins {
    static Map<String, String> registry = Map.of(
            "hello", "org.example.reflection.HelloPlugin"
    );

    public static void main(String[] args) throws Exception {
        String key = args.length > 0 ? args[0] : "hello";
        String cn = registry.get(key);
        Class<?> clazz = Class.forName(cn);
        Object plugin = clazz.getConstructor().newInstance();
        clazz.getMethod("run").invoke(plugin);
    }
}
```

### Mini‑exercise

* Add `TimePlugin` printing the current time and run `L9Plugins time` via **Run Configurations → Program arguments**.

---

# Lesson 10 — Best Practices, Pitfalls & Security

**Topics:**

* Prefer public APIs; reflection is a last resort.
* Cache reflective lookups (`Field`, `Method`) when called repeatedly.
* Respect encapsulation and module boundaries; avoid breaking JDK internals.
* Validate user input when using reflection on names to avoid loading arbitrary classes.
* Consider `ServiceLoader` for plugins and `MethodHandles` for performance‑critical paths.

### Capstone Challenge

Build a tiny “mini‑DI” (dependency injection) container:

* Annotate fields with `@Inject`.
* Reflectively instantiate dependencies and set fields.
* Support singleton scope with a simple cache.

---

## Cheatsheet

* `Class.forName("cn")`, `obj.getClass()`, `Type.class`
* Members: `getFields/DeclaredFields`, `getMethods/DeclaredMethods`, `getConstructors/DeclaredConstructors`
* `setAccessible(true)` for private access (unnamed module or opened packages)
* Arrays: `Array.newInstance`, `Array.getLength`, `Array.set`
* Annotations: `getAnnotation`, `getAnnotationsByType`, `isAnnotationPresent`
* Dynamic proxies: `Proxy.newProxyInstance(loader, interfaces, handler)`
* MethodHandles: `MethodHandles.lookup().findXxx`
* Modules: `--add-opens` or `module-info.java` `opens`

---

### How to use this course

1. Open each `L#*.java` in order and run.
2. Complete the mini‑exercise before moving on.
3. Answer the quick quiz to cement ideas.

When you’re ready, proceed to **Lesson 1** and run `L1Basics`. Happy hacking! 🎯
