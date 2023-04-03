---
date: 2023-03-03
authors:
  - bytez
categories:
  - Nova
  - Bytecode
  - JVM
  - Minecraft
---

# Runtime patching in Nova

## Introduction

Patching is a pretty common practice in most modding communities. Sometimes it's even the only way to add specific features to a game. Minecraft is no exception here. Most of the time, modding frameworks use 1 of 2 main approaches to modify Minecraft's code. The first way is decompiling Minecraft, applying predefined patch files to the decompiled code and finally, recompiling the entire server. These patch files are usually using git's diff format.  
As an example, here's one of the patches applied when running [BuildTools](https://www.spigotmc.org/wiki/buildtools/) to ensure the correct `CommandDispatcher` is used:

<!-- more -->

```diff
--- a/net/minecraft/server/CustomFunctionData.java
+++ b/net/minecraft/server/CustomFunctionData.java
@@ -42,7 +42,7 @@
     }
 
     public CommandDispatcher<CommandListenerWrapper> getDispatcher() {
-        return this.server.getCommands().getDispatcher();
+        return this.server.vanillaCommandDispatcher.getDispatcher(); // CraftBukkit
     }
 
     public void tick() {
```

Paper and its forks use a similar approach. But instead of forcing users to build the server themselves via a tool like BuildTools, they apply the patches and generate a [binary diff](https://www.daemonology.net/bsdiff/) between the vanilla Minecraft server and their patched version. This binary diff is then used to generate Paper's server jar using [PaperClip](https://github.com/PaperMC/Paperclip).

And then there's client- and serverside modding frameworks like Forge, Fabric and Sponge. These frameworks use [Mixins](https://github.com/SpongePowered/Mixin/wiki) and also allow the framework uses to add their own. Unlike Bukkit's approach, Mixins are written in code which already enhances the qol for devs. They allow you to inject code or even add fields and interface implementations to any class. They're also applied directly before startup instead of generating a patched server jar.  
Here's a Mixin that prevents oversized books from being created:

```java
@Mixin(WritableBookItem.class)
public abstract class WritableBookItemMixin_LimitBookSize {

    @Redirect(
            method = "makeSureTagIsValid",
            at = @At(value = "INVOKE", remap = false, target = "Ljava/lang/String;length()I")
    )
    private static int impl$useByteLength(final String s) {
        return s.getBytes(StandardCharsets.UTF_8).length;
    }

    @ModifyConstant(
            method = "makeSureTagIsValid",
            constant = @Constant(intValue = 32767)
    )
    private static int impl$useMaxBookPageSizeFromConfig(final int maxBookPageSize) {
        return SpongeConfigs.getCommon().get().exploits.maxBookPageSize;
    }

    @Inject(
            method = "makeSureTagIsValid",
            cancellable = true,
            at = @At(value = "RETURN")
    )
    private static void impl$useMaxBookSizeFromConfig(final CompoundTag p_150930_0_, final CallbackInfoReturnable<Boolean> cir) {
        if (cir.getReturnValue()) {
            final ListTag listnbt = p_150930_0_.getList("pages", 8);
            final int size = IntStream.range(0, listnbt.size()).mapToObj(listnbt::getString).mapToInt(s -> s.getBytes(StandardCharsets.UTF_8).length).sum();
            if (size > SpongeConfigs.getCommon().get().exploits.maxBookSize) {
                cir.setReturnValue(false);
            }
        }
    }

}
```

So why do none of these approaches work for Nova?

??? question "What's Nova?"

    [Nova](https://www.spigotmc.org/resources/nova-modding-framework-1-19-3.93648/) ([GitHub](https://github.com/xenondevs/Nova)) is a modding framework we're currently working on that allows developers to add custom items, blocks, GUIs and more to Spigot servers via resource packs. Check out our [Spigot thread](https://www.spigotmc.org/threads/advanced-resourcepack-mechanics-how-to-create-custom-items-blocks-guis-and-more.520187/) if you want to learn more about the resource pack stuff.

## The problem

Unlike all previously mentioned modding frameworks, Nova isn't built into the server. Instead, it's a Spigot plugin which causes a few problems. Most classes we need to patch are already loaded by a `ClassLoader` before Nova is even loaded. Sadly, The JVM [will not let you define the same class again](https://github.com/openjdk/jdk/blob/54603aa1b72bfbdd04d69f0f0bf5dcfeb9dcda92/src/hotspot/share/classfile/systemDictionary.cpp#L1752) (even when using `Unsafe#defineClass`). Getting rid of the existing definition is also hard since the JVM uses a so-called [system dictionary](https://github.com/openjdk/jdk/blob/54603aa1b72bfbdd04d69f0f0bf5dcfeb9dcda92/src/hotspot/share/classfile/systemDictionary.cpp) internally which maintains a mapping between class names and their corresponding class definitions. This dictionary makes removing an existing class definition pretty much impossible without writing native code that directly edits memory sections (which would obviously be a bad idea in itself).

## Instrumentation

Luckily, the designers of the JVM thought about this problem and built a solution for it. Most JVM implementations support a feature called [Instrumentation](https://docs.oracle.com/en/java/javase/17/docs/api/java.instrument/java/lang/instrument/package-summary.html) :tada:. `Instrumentation` allows us to modify the bytecode of classes that are already loaded by the JVM. This is exactly what we need to patch Minecraft's classes. But getting an `Instrumentation` instance is not easy. The constructor of the `InstrumentationImpl` class has a `long nativeAgent` parameter, which is used to [pass a pointer of the JPLISAgent](https://github.com/openjdk/jdk/blob/54603aa1b72bfbdd04d69f0f0bf5dcfeb9dcda92/src/java.instrument/share/native/libinstrument/JPLISAgent.c#L509). So it's not possible to get an instance without attaching an agent to the current JVM. But what even is a Java agent?  
A Java agent is a program that can be loaded into the JVM at runtime to provide additional functionality. It is mostly used for debugging or performance monitoring, but can also be used to modify the behavior of a running Java application. Java agents can access the JVM's internal data structure (i.e. the previously mentioned system dictionary) and modify it. Normally, java agents are loaded by the JVM during startup via the `-javaagent` command line argument. Attaching an agent to a running JVM is also possible. The runtime attaching mechanism can also get a bit complicated when supporting multiple JDK distributions, but luckily [byte-buddy](https://github.com/raphw/byte-buddy) exists. Byte-buddy is a library that allows us to attach a java agent without having to write any native code. It's normally used to create dynamic proxies at runtime via `Instrumentation`. However, the agent attachment mechanism is located in a [different module](https://github.com/raphw/byte-buddy/tree/master/byte-buddy-agent), so we don't have to include the entire library and can just call `ByteBuddyAgent.install()` to attach an agent and get an `Instrumentation` instance.

## Patching

Now that we have an `Instrumentation` instance, we can start patching classes. In Nova, we created a simple `Transformer` interface:

```kotlin
internal sealed interface Transformer {
    
    val classes: Set<KClass<*>>
    
    val computeFrames: Boolean
    
    fun transform()
    
    fun shouldTransform(): Boolean = true
    
}
```

This interface contains a `classes` property to specify which classes will be updated by this Transformer (and thus should be redefined) and a `computeFrames` property that states whether ASM should recompute the [stack frames](https://docs.oracle.com/javase/specs/jvms/se17/html/jvms-2.html#jvms-2.6) of the newly updated class.  
We also added a couple of abstract implementations of the `Transformer` interface like `ClassTransformer`, `MethodTransformer`, `MultiTransformer` or even `ReversibleClassTransformer`. The names should be pretty self-explanatory.

## Classloader issue

When writing our first patch, we already ran into our first problem. We wanted to patch the `NoteBlock` class to override the sound logic since the note is no longer stored in the blockstate (See our [spigot thread](https://www.spigotmc.org/threads/advanced-resourcepack-mechanics-how-to-create-custom-items-blocks-guis-and-more.520187/) if you're not familiar with the resource pack tricks). We'd obviously need to check our own data on the current block to determine which sound to play, or if a sound should be played at all. However, we can't access Nova classes from the `NoteBlock` class, since they're loaded via Bukkits `PluginClassLoader` and therefore aren't visible to the `NoteBlock` class. So we had to find a way to squeeze Nova's `ClassLoader` into Minecraft's class loading hierarchy. So the obvious first step was to look at Java's `ClassLoader` class, where we quickly noticed the following field:

```java
// The parent class loader for delegation
private final ClassLoader parent;
```

This field is used to delegate class loading to the parent `ClassLoader` if the current one can't find a requested class. So we just need to replace this field with a custom `ClassLoader` that first checks the original `ClassLoader` and then delegates to Nova's `ClassLoader`:

```kotlin
internal class PatchedClassLoader : ClassLoader(NOVA.loader.javaClass.classLoader.parent.parent) {
    
    private val novaClassLoader = NOVA.javaClass.classLoader as NovaClassLoader
    
    override fun loadClass(name: String, resolve: Boolean): Class<*> {
        // Check if class is already loaded
        var c: Class<*>? = synchronized(getClassLoadingLock(name)) { findLoadedClass(name) }
        
        // Load class from parent (ApplicationClassLoader)
        if (c == null) {
            c = runCatching { parent.loadClass(name) }.getOrNull()
        }
        
        // Load class from Nova & libraries
        if (c == null && checkNonRecursiveStackTrace()) {
            // Restarts the class loading process at the NovaClassLoader
            return novaClassLoader.loadClass(name, resolve)
        }
        
        if (c == null) {
            throw ClassNotFoundException(name)
        }
        
        // Resolve class
        if (resolve) {
            synchronized(getClassLoadingLock(name)) { resolveClass(c) }
        }
        
        return c
    }
    
    /**
     * Checks that a class load was not caused by Nova or another plugin.
     */
    private fun checkNonRecursiveStackTrace(): Boolean {
        return Thread.currentThread().stackTrace
            .none {
                it.className == "org.bukkit.plugin.java.PluginClassLoader" || it.className == "xyz.xenondevs.nova.loader.NovaClassLoader"
            }
    }
    
}
```

As you can see, we also had to add an extra check to prevent recursion in class loading, which would lead to a deadlock.  
With this `ClassLoader` in place, the new class loader hierarchy looks like this:


<p class="text-center">
  <img src="https://i.imgur.com/MVBmFQ2.png" width="80%" alt="New classloading hierarchy"/>
</p>

But of course, things are never as easy as they seem... Firstly, it's normally not possible to get the reflection `Field` instance of the `parent` field because of a hard-coded filter in the JVM. Whenever getting fields of a class via reflection, a call to `Reflection#filterFields` is made.

```java
private Field[] privateGetDeclaredFields(boolean publicOnly) {
    Field[] res;
    /* ... */
    res = Reflection.filterFields(this, getDeclaredFields0(publicOnly));
    /* ... */
    return res;
}
```

This method exists to prevent access to certain fields that are meant to only be accessible inside internal code. Apparently, all fields of the `ClassLoader` class fit into this category, as shown in the `fieldFilterMap` used by the `Reflection#filterFields` method:

```java
fieldFilterMap = Map.of(
    /* ... */
    ClassLoader.class, ALL_MEMBERS,
    /* ... */
);
```

So we had to write a patch that always returns the same list instead of filtering anything (and reverse it after we're done to prevent incompatibility with other plugins):

```kotlin
internal object FieldFilterPatch : ReversibleMethodTransformer(Reflection::filterFields.javaMethod!!) {
    
    override fun transform() {
        methodNode.instructions = buildInsnList {
            aLoad(1) // Field[] fields parameter
            areturn() // Return fields without applying any changes
        }
    }
    
}
```

Now we just need to redefine a few modules to add Nova's module to the opens list:

```kotlin
private val extraOpens = setOf("java.lang", "java.lang.reflect", "java.util", "jdk.internal.misc", "jdk.internal.reflect")

// ...

private fun redefineModule() {
    val novaModule = setOf(Nova::class.java.module)
    val javaBase = Field::class.java.module
    
    INSTRUMENTATION.redefineModule(
        javaBase,
        emptySet(),
        emptyMap(),
        extraOpens.associateWith { novaModule },
        emptySet(),
        emptyMap()
    )
}
```

And finally, replace the `parent` field using `Unsafe` (since the field is final):

```kotlin
val spigotLoader = NOVA.loader.javaClass.classLoader.parent
ReflectionUtils.setFinalField(classLoaderParentField, spigotLoader, PatchedClassLoader())

// ReflectionUtils code:

internal fun setFinalField(field: Field, obj: Any, value: Any?) {
    val unsafe = Unsafe.getUnsafe()
    val offset = unsafe.objectFieldOffset(field)
    unsafe.putReference(obj, offset, value)
}
```

<video playsinline autoplay muted loop style="display: block; margin: 0 auto;">
  <source src="https://i.imgur.com/bKxw3Ah.mp4" type="video/mp4">
</video>
<p class="text-center"><small class="text-center">NoteBlock block updates with an injected <code>println</code></small></p>

## Limitations and the future of our patching system

There is still a pretty big limitation with runtime patching that doesn't exist when patching before the class is loaded. You can only change the instructions of methods. You can't add/remove methods, add interfaces, make fields public, etc. However, it has still allowed us to add a lot of features to Nova including our upcoming world generation support which would've been impossible without runtime patching. But there are ways to get around these limitations by proxying calls to added fields/methods, creating runtime wrapper classes for adding interfaces, etc. I'm also not a fan of the whole writing bytecode by hand thing. It's pretty error-prone and often hard to debug. That's why I'm currently working on a patching framework built into my bytecode library [ByteBase](https://github.com/ByteZ1337/ByteBase). The syntax will be similar to [mixins](https://github.com/SpongePowered/Mixin) and will autogenerate stuff like the proxy calls I mentioned earlier. I'll probably write a blog post about it once it's finished.