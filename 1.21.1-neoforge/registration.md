在正文开始前，先让我们明晰几个概念：
<ul>
<li>游戏对象（Game Object）：玩家在游戏中实际交互的对象，如“16个苹果”“世界上的一个矿车实体”等等</li>
<li>注册对象（Registry Object）：游戏对象的种类，如“16颗苹果”的种类为“苹果”，“世界上的一个矿车实体”对应“矿车实体”</li>
<li>注册项（Registry Entry）：注册对象的种类，如“物品”“方块”“方块实体类型”“实体类型”“生物实体属性”“状态效果”等，上述提到的“苹果”“矿车实体”分别对应了“物品”“实体类型”</li>
<li>注册表（Registry）：用于统筹管理某一注册项下所有注册对象的存在，可以理解为一个键为唯一标识符、值为对应注册对象的一个Map</li>
</ul>

## 唯一标识符：ResourceKey<?>和ResourceLocation
前面提到唯一标识符，它的类型是什么呢？有请`ResourceKey`（资源键）。

观察`ResourceKey`内部的成员变量：
```Java
    //The name of the parent registry of the resource.
    private final ResourceLocation registryName;

    //The location of the resource within the registry.
    private final ResourceLocation location;
```
所以我们先来讲`ResourceLocation`。望文生义，`ResourceLocation`用于指代某个位于资源包中的资源文件的位置，它也的确有这个意义，不过在这里它则同样是作为“唯一标识符”出现的。

`ResourceLocation`分为2个部分：命名空间（namespace）和路径（path），写成字符串形式就是`namespace:path`。命名空间，对于原版的情形是`minecraft`，对于Mod的情形，自然就是你自己的modid哩。

说完`ResourceLocation`，再来看`ResourceKey`。细心的读者可能已经注意到它有一个泛型参数了——泛型参数即代表了——该`ResourceKey`所指代的对象的类型。所谓“`ResourceKey`所指代的对象”并不只能是注册对象，还可以是注册表。没错，注册表也需要一个唯一标识符。

注册表的资源键的`registryName`，Minecraft原版规定其为`minecraft:root`（`Registries.ROOT_REGISTRY_NAME`），代表该资源键代表一个注册表。而`location`起的自然是唯一标识符的作用——没错，给定资源键泛型参数的情况下，起到“唯一标识符”作用的，其实是`ResourceKey`内部的那个`location`。举例：对于物品的注册表（`BuiltInRegistries.ITEM`），其资源键的`location`为`minecraft:item`。

而对于注册对象的资源键，其`registryName`，就是其对应注册表的资源键的`location`，这也印证了注释中的“parent registry of the resource”。至于它本身的`location`？自然就是喜闻乐见的、通常意义上的“注册名（Registry Name）”了（经常写1.12- mod的modder应当深有体会），如，苹果的资源键的`location`，为`minecraft:apple`。

再来看看一个`ResourceKey<?>`对象是怎么产生的：
```Java
    //create方法较为直观的重载，对应两个内部字段：registryName和location。
    ResourceKey<?> key1 = ResourceKey.create(aCertainLocation1, aCertainLocation2);
    //另一个重载，registryName会取传入的这个资源键的location。换言之，这里有些体现“资源键间的继承”这种概念。
    ResourceKey<?> key2 = ResourceKey.create(aCertainResourceKey, aCertainLocation3);
    //专门为创建注册表资源键而生的方法，只需填写location即可。
    ResourceKey<Registry<?>> key3 = ResourceKey.createRegistryKey(aCertainLocation4);
```

所有注册表的资源键都位于类`net.minecraft.core.registries.Registries`中。

## 注册表（Registry）和Holder<?>
前文说过，只需把注册表（`Registry<?>`）看成一个`Map<ResourceKey<T>, T>`即可。但和Map不同的是，你可以通过值来获取该值对应的资源键。换言之，键和值是一一对应的。

注册表有“冻结与否”一说。当游戏初始化到需要注册游戏对象时，注册表会“解冻”，此时便允许原版和Mod向注册表中写入内容；过了该阶段后，注册表便会被“冻住”（`Registry#freeze`）。任何试图在注册表处于冻结状态时对其内部状态进行改变的行为都是不合法的。

另外一个需要介绍的概念是`Holder<?>`，它相当于一个容器，将我们的注册对象包装起来（相当于1.12- 的`IRegistryDelegate`），泛型参数为注册对象对应的基类类型，如对于包装了物品对象的`Holder`，其泛型参数类型应总为`net.minecraft.world.item.Item`。换言之，`Holder`泛型参数类型和包装的对象的实际类型可能并不完全相同，而完全可能是有继承关系的两个类。

`Holder`是个接口，定义了`value`方法来取出包装的对象，其有2个实现类：`Holder.Direct`和`Holder.Reference`。对于后者而言，在注册尚未完成的情况下访问其内部值是不合法且很危险的——具体地说，会抛出一个异常：“Trying to access unbound value.”

## 固有注册表和可写注册表
> 游戏内有几十种注册表，它们分别都有不同的作用。这些注册表中，可以分为2类：
>> 固有注册表（Built-in Registry）：游戏硬编码的注册表，内部数据无法通过任何方式修改。这些注册表在各个世界中都通用。
>> 可写注册表（Writable Registry）：游戏读取世界中的数据包获得这些注册表的信息，游戏代码内部并不存在这些注册表的数据。这些注册表与世界绑定，根据世界不同数据也有可能不同。
（以上这段文字来源于[注册表 - 中文 Minecraft Wiki](https://zh.minecraft.wiki/w/注册表)）

显而易见，我们在代码层面能直接操作并添加新元素的，就是固有注册表；至于可写注册表（又称为数据包注册表），我们应该以数据包的形式添加新元素。

所有固有注册表都能在`BuiltInRegistries`类中找到；而对于可写注册表，由于它会随着世界的不同而变动，因此你需要一个`Level`上下文，通过该`Level`对象获取一个`RegistryAccess`：

<ul>
<li>对于服务端世界，请调用ServerLevel#registryAccess</li>
<li>对于客户端世界，请调用Minecraft.getInstance().getConnection().registryAccess()</li>
<ul>
<li>请注意：只有当你的客户端实际进入了一个世界（本地存档/远程服务器）的时候，它才会返回一个有效的值，否则会直接报NPE</li>
<li>而且不同于固有注册表，并非所有可写注册表都会同步所有数据至客户端，请格外注意这一点</li>
</ul>
</ul>

之后，我们就可以调用方法`#registryOrThrow`来获取我们想要的注册表了。

## 注册的过程
对于可写注册表，我们只需写写数据包就好了，交由Minecraft数据包机制处理即可。

而对于固有注册表呢？原版的做法是将所有注册对象都写在诸如`Items`、`Blocks`这种类当中，并在静态初始化块或bootstrap方法中调用`Registry#register`来注册这些对象。

遗憾的是，Mod要注册注册对象的话，用这一套方法是行不通的。究其原因，原版的类会被ClassLoader很早地加载到，Mod类则不会；且由于注册项之间可能会有相互依赖的关系——如方块的物品形式也是物品，需要像普通物品一样被注册，因此方块的注册必然会位于物品之前，这就导致Minecraft原版不同注册项的注册顺序是受到严格把控的，Modder如果弄错了该有的顺序，后果不堪设想。

NeoForge提供了2种方法来注册：监听`RegisterEvent`，和延迟注册（`DeferredRegister`）机制。考虑到上述所提到的注册项注册顺序的问题，推荐使用后者。

### DeferredRegister
```Java
public class RegistryHandler {
    //这里T即为你的注册项类型，可以为Item、Block等等。
    //DeferredRegister字段的命名一般习惯上是这样的：把注册项类型名称全大写，再加S，如DeferredRegister<Item>一般会被命名为ITEMS。
    //create方法的第一个参数可以是ResourceKey<Registry<T>>、ResourceLocation或Registry<T>。
    public static final DeferredRegister<T> DEFERRED_REGISTER_T = DeferredRegister.create(BuiltInRegistries.xxx, MODID);

    //第一个参数为该注册对象的注册名，第二个参数可以是一个Function<ResourceLocation, ? extends S>，也可以是一个Supplier<? extends S>。
    //一般而言填一个Supplier更为常见。
    //这个lambda就是DeferredRegister的精髓所在了：实现游戏对象的惰性初始化，或许这个字段被加载得较早，但是lambda里的方法要到RegisterEvent被发布才被调用，这个DeferredHolder才真正可用。因此各个注册项之间的注册先后顺序就不用Modder来操心了。
    public static final DeferredHolder<T, S extends T> YOUR_REGISTRY_OBJECT = DEFERRED_REGISTER_T.register("example_name", () -> ...);

    ...

    //将我们的DeferredRegister注册进MOD总线，一般是在Mod主类构造器中调用它。
    public static void register(IEventBus bus) {
        DEFERRED_REGISTER_T.register(bus);
    }
}
```

`DeferredHolder`是一种特殊的`Holder`，它用于弥补`Holder`的不足——`Holder`的泛型参数只能是该种注册项的上限类，如对于物品的`Holder`，泛型参数只能是`Item`而不能是`PickaxeItem`这种子类。我们看看`DeferredHolder`的类定义就知道：`public class DeferredHolder<R, T extends R> implements Holder<R>, Supplier<T>`。因此一个`DeferredHolder`同时也是一个`Supplier`。

上述介绍的是注册注册对象的一般步骤，但实际上NeoForge考虑到物品（`Item`）、方块（`Block`）和物品堆叠组件类型（`DataComponentType<?>`）的注册非常常见，因此提供了`DeferredRegister`的三个子类：`DeferredRegister.Items`、`DeferredRegister.Blocks`和`DeferredRegister.DataComponents`，以及`DeferredHolder`的两个子类：`DeferredItem`和`DeferredBlock`。并以此为基础提供了很多便捷的方法。

但是注意，这些`DeferredRegister`子类的某些注册方法会把构建新对象时的某一部分移到lambda外边（以此来满足使用方法引用的条件），因此如果你处理得不好，还是可能导致一些问题。不过NeoForge也为这些情况的涉及的某些方法，提供了把直接的对象引用换成`Supplier`的重载（如`IItemPropertiesExtensions#component`），这是为了保证惰性加载，你应该尽量用含`Supplier`的重载。
