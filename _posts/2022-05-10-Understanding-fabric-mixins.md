# Understanding fabric mixins
Fabric's mixins have always been sort of a mystery for me, so I'm writing this to document how they work.

## How mixin even begins to do what it does
Mixin first looks at your mixin config, and picks out all of the classes you gave to it. Then, it tries to make sense of the mixin, parsing the @Mixin annotation to find out which class to modify.

Mixin then looks at all the methods you want to modify, and this is where the mixin types come to play
## The mixin types
Mixin has several modifications you can do to a method out of the box. The most basic one is @Inject. It allows you to add code to a method whereever you want. To do this, it takes 2 mandatory arguments: `method` and `at`

`method` is the target method's descriptor, with a few quirks. First off, you do not have to specify the full descriptor, if there's only one method with that name. If there's several with different descriptors, you will have to specify the entire name. An example descriptor would look something like this:

`repeatString(Ljava/lang/String;I)Ljava/lang/String;`

This might seem a bit confusing, but here is what this means:

| Example | Method name | Method arguments | Method return type |
| ------- | ----------- | ---------------- | ------------------ |
| repeatString(Ljava/lang/String;I)Ljava/lang/String; | repeatString | Ljava/lang/String;I | Ljava/lang/String; |
| somethingWithTwoDoublesThatReturnsAFloat(DD)F | somethingWithTwoDoublesThatReturnsAFloat | DD | F |
| somethingThatReturnsABoolean()Z | somethingThatReturnsABoolean | | Z |

You might ask what this `Z` or `I` or even `Ljava/lang/String;` means. These are the types. Single characters are primitives, here's a list of them:

| Java name | Representation | Description |
| - | - | - |
| byte | B | A signed byte | 
| char | C | A single character |
| double | D | A double-precision floating point integer |
| float | F | A single-precision floating point integer |
| int | I | A full integer |
| long | J | A long integer |
| short | S | A signed short |
| boolean | Z | A boolean |
| | [ | An array indicator, makes the type an array (example: `[I` -> `int[]`, `[[Z` -> `boolean[][]`) |
| | V | Void, nothing is returned (only applies to the return value) |

"But 0x! There's one missing! What about L(something);?"

L is a special one. It's not a single character, and thus deserves its own section. The L(something); is a class representation. The (something) is just the class name, but with slashes instead of dots. Don't forget to end it with the semicolon tho, that's important.

| Example | Class name |
| - | - |
| Ljava/lang/String; | java.lang.String |
| Ljava/io/File; | java.io.File |
| Ljava/lang/management/ManagementFactory; | java.lang.management.ManagementFactory |

You can also use $ signs to target an inner class, like `Ljava/lang/String$1;`, but that's more advanced, and won't be covered here.

# The at
`@At` is so advanced that it requires its own section to understand. The basic syntax for `@At` is just `@At("HEAD/TAIL/RETURN")`, and that's most of what you will need.

## Complex syntax
Of course, `@At` can also inject into a very specific point. Take this, as example: `@At(value="INVOKE", target="Ljava/util/ArrayList;clear()V")`. Do you know what that means? Yeah, me neither. Let's break it down.

`value` is just `INVOKE`, which means "this method is getting invoked". In the `target`, we tell mixin which method we want to target. We can see the two topics we discussed earlier: L(something); and the method name.

If you apply the concepts described above, we are suddenly targetting ArrayList#clear(), which might look like this if you visualize it:

```java
ArrayList<String> someList = new ArrayList<>();
someList.add("Some text");
someList.clear(); // <-- Our target
// <-- WILL INJECT HERE
someOtherCode(someList); // method continues with its normal stuff
```

## Shift
At's shift allows us to change where it will point. This is especially useful in `@Inject`, for example. The default is up to the mixin type, but you can make it shift `BEFORE` or `AFTER` the target. For `@Inject`, the default is `AFTER`. But you can also make it inject `BEFORE`.

# The mixin types
## `@Inject`
Allows you to inject code into a specific part of a method

### Examples
- `@Inject(method="someMethod", at=@At("HEAD"))` - will inject into the start of the method
   
    ```java
    void someMethod() {
        // <-- WILL INJECT HERE
        someStuff();
        someOtherStuff();
        if (someStuff) return;
    }
    ```
- `@Inject(method="someMethod", at=@At("TAIL"))` - will inject into the end of the method
   
    ```java
    void someMethod() {
        someStuff();
        someOtherStuff();
        if (someStuff) return;
        // <-- WILL INJECT HERE
    }
    ```
- `@Inject(method="someMethod", at=@At("RETURN"))` - will inject into every return statement of the method (including the tail)
   
    ```java
    void someMethod() {
        someStuff();
        if (someOtherStuff()) {
            someOtherOtherStuff();
            // <-- WILL INJECT HERE
            return;
        }
        if (someStuff) {
            // <-- WILL INJECT HERE
            return;
        }
        // <-- WILL INJECT HERE
    }
    ```
- `@Inject(method="someMethod", at=@At(value="INVOKE", target="Ljava/util/ArrayList;clear()V"))` - Will inject after every ArrayList#clear() call
   
    ```java
    List<String> someMethod() {
        ArrayList<String> strings = new ArrayList<>();
        strings.add("abc");
        strings.add("abcd123");
        strings.clear();
        // <-- WILL INJECT HERE
        strings.add("abcdefg");
        strings.clear();
        // <-- WILL INJECT HERE
        return strings;
    }
    ```

## `@Redirect`
Replaces a method call entirely

Here, the `@At` specifies which method call to replace. `HEAD` or `TAIL` do not apply here.
### Examples
- `@Redirect(method="someMethod", at=@At(value="INVOKE", target="Ljava/util/ArrayList;clear()V"))` - Will replace every ArrayList#clear() call

    Redirector method body:

    ```java
    void replaceStuff(ArrayList strings) {
        System.out.println("Cancelled clear");
    }
    ```

    Target:


    ```java
    void someMethod() {
        ArrayList<String> strings = new ArrayList<>();
        strings.add("abc");
        strings.add("abcd123");
        strings.clear(); // -> System.out.println("Cancelled clear");
    }
    ```
    
    Above turns into this:

    ```java
    void someMethod() {
        ArrayList<String> strings = new ArrayList<>();
        strings.add("abc");
        strings.add("abcd123");
        redirect$someid$replaceStuff(strings); // Method is autogenerated by mixin, it's content is System.out.println("Cancelled clear");
    }
    ```
- `@Redirect(method="someMethod", at=@At(value="FIELD", target="Lthe/Target;someField:Ljava/lang/String;"))` - Replaces field access

    Redirector method body:

    ```java
    String replaceStuff(Target targetClass) {
        return "another value";
    }
    ```

    Target:

    ```java
    void someMethod() {
        String fieldCopy = this.someField; // <-- REPLACES THIS
        System.out.println(fieldCopy); // Would print value of someField
    }
    ```
    
    Above turns into this:

    ```java
    void someMethod() {
        String fieldCopy = redirect$someid$replaceStuff(this); // Method is auto generated, content is seen above
        System.out.println(fieldCopy); // Will now print value of whatever we return above
    }
    ```

# Conclusion
Mixin is a tremendously powerful tool, but these 2 mixin types are really the ones you will need the most. More examples can be found [here](https://github.com/2xsaiko/mixin-cheatsheet), and the mixin wiki is [here](https://github.com/SpongePowered/Mixin/wiki)