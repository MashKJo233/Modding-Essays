Capability，是Forge于1.8.9加入的系统，主要作用是：允许你方便地给已有的或你自己的游戏对象附加数据，同时提供统一的交互和序列化标准，降低代码间的耦合程度。

Capability直译过来就是“能力”（但作者认为这个直译用在本文中太过别扭，因此接下来大部分时候就使用英文原名Capability来指代），那我们不禁会想：如果我们要给游戏对象附加某种“能力”，让其类实现特定接口不行吗？的确可以，但也仅限于你自己的东西，面对原版或其他Mod的游戏对象我们便无能为力了。更何况，如果你打算和其他Mod做联动呢？Capability系统的出现就是为了解决这一切。

## 关键概念
一个Capability大致上由这些部分组成：
<ul>
<li>Capability接口：暴露出来的接口，只负责描述该Capability行为，不负责序列化</li>
<ul><li>Capability接口的默认实现类：用于默认情况下实例化Capability接口</li></ul>
<li>Capability Token（Capability<?>）：用于代表“Capability的种类”，泛型参数即为该种类对应的Capability接口，作用类似于唯一标识符</li>
<li>用于序列化的类：这一部分有些复杂，Capability系统虽然有一个Capability.IStorage<?>（泛型参数为Capability接口）用于定义序列化标准，但具体怎么调用它、何时调用它没有定式，要具体情况具体对待</li>
<li>Capability提供者（ICapabilityProvider）：实际提供Capability的对象，可以是游戏对象本身（ItemStack、TileEntity等），也可以是第三方的ICapabilityProvider</li>
</ul>

如果读者是第一次接触这几个概念，估计已经被绕晕了，也正常，Capability的系统设计得很复杂，读者可以先往后看，研究示例代码，再来看这一部分，便会有新的认识的。

## 有哪些种类的游戏对象支持Capability系统？
那得看ICapabilityProvider的实现类都有哪些。在1.12.2版本，它们有：ItemStack、TileEntity、Entity、Village、World、Chunk。其中，前三者的Capability被使用得最多。

## 操作Capability
我们可以通过`ICapabilityProvider#hasCapability`来判断某个ICapabilityProvider是否支持某种Capability，也可以通过`ICapabilityProvider#getCapability`来获取对应Capability种类的接口实例。

这两个方法的形参列表是一样的：`(Capability<?> capability, EnumFacing facing)`。EnumFacing只对部分方块实体的Capability有意义，我们暂时不去理会它；第一个参数是Capability Token，这个就是判断返回何值的依据了。

## 为游戏对象附加Capability数据
对于已有的游戏对象，我们只需覆写ICapabilityProvider的这两个方法就可以了。
```Java
    private final IExampleCapability instance;
    
    ...

    @Override
    public boolean hasCapability(@Nonnull Capability<?> capability, @Nullable EnumFacing facing) {
        return capability == aCertainCapability || super.hasCapability(capability, facing);
    }

    @Override
    public <T> T getCapability(@Nonnull Capability<T> capability, @Nullable EnumFacing facing) {
        if(capability == aCertainCapability) {
            return aCertainCapability.cast(instance);
        }
        return super.getCapability(capability, facing);
    }
```
那个cast是因为Capability接口的具体类型匹配不上那个`T`类型，所以我们要用Forge给我们提供的这个方法，避免unchecked cast（当然你要真的当unchecked领域大蛇也不是不行）。

看不懂调用父类方法是啥意思？不要紧，我们继续往下看。

对于给已有的游戏对象附加Capability（实质是附加第三方ICapabilityProvider），我们需要监听事件：`AttachCapabilitiesEvent<? extends ICapabilityProvider>`。

注意：泛型参数只能是TileEntity、Entity这种基类，而不能是如TileEntityBeacon、EntityPlayer这种子类！要给他们的子类的实例附加Capability，请在方法体内进行instanceof判断。

```Java
    //这个事件会在对应的ICapabilityProvider被实例化时触发。
    @SubscribeEvent
    public static void onAttachCapabilities(AttachCapabilitiesEvent<TileEntity> event) {
        ... //如果有必要的话，检查一下当前所在的逻辑端。

        event.addCapability(new ResourceLocation(...), new ICapabilityProvider() {
            ...
            //实现hasCapability和getCapability，思路和上述差不多。
        });
        //留个心眼，实际上这里一般都不会直接实现ICapabilityProvider，原因看后面。
    }
```

很显然，既然这是监听事件，那么肯定存在：不止一个Mod监听该事件给同一类ICapabilityProvider附加第三方ICapabilityProvider（组合模式）的情况。Capability系统力求为Mod间交互构建桥梁，自然不可能用“覆盖”的方式处理这一问题。

Forge的解决方案是：给所有ICapabilityProvider的实现类基类添加一个CapabilityDispatcher字段，这是一个特殊的ICapabilityProvider，负责收集所有的第三方ICapabilityProvider。

读者现在应该能理解为什么上面覆写ICapabilityProvider的那俩方法需要再调用父类的实现方法了——因为父类的方法中很可能囊括了子类判断中没考虑到的Capability类型，如果一个Capability Token不符合子类的判断标准，则传递给其父类，若最终传递给了基类，若基类判断也不符合（非第三方ICapabilityProvider即它自身，不支持该类型的Capability），则会调用CapabilityDispatcher判断，CapabilityDispatcher会依次调用它收集到的所有第三方ICapabilityProvider进行判断的。这实际上体现了责任链设计模式。

## Capability数据的序列化和反序列化
每一种类型的Capability都需要注册（详见后文），其中要求一个Capability.IStorage<?>的实现，看起来它就是用来定义序列化标准的接口了？实际上并不完全对，它确实是规范，但并不强制你在序列化时使用它。换言之，这很可能就是一个设计失误。

根据ICapabilityProvider是否为第三方的，Capability数据的序列化方式还有一定的区别。ItemStack的ICapabilityProvider一定是第三方提供（毕竟ItemStack是final class），Village、World、Chunk也几乎是这样（当然也有Mod继承它们，但非常少，而且说句题外话，这样通常会造成兼容性问题），这里我们只讨论TileEntity和Entity的情形——那当然是覆写它们的有关数据持久化的方法啊，至于到底是直接手写，还是代理给什么其他方法，其实无所谓。

对于第三方Capability（通过监听`AttachCapabilitiesEvent<? extends ICapabilityProvider>`或覆写 `Item#initCapabilities`），CapabilityDispatcher会检测第三方ICapabilityProvider是否还实现了INBTSerializable<? extends NBTBase>，并以此为依据进行序列化和反序列化。泛型参数为序列化后的实际NBT类型。

特别地，有一个复合接口：ICapabilitySerializable，同时继承了ICapabilityProvider和INBTSerializable，所以我们在构造ICapabilityProvider的匿名实现类时，一般都该用ICapabilitySerializable（虽说它的本意是减少Forge的patch文本量，毕竟维护patch不易，会消耗不小的时间精力）。

我们可以不遵循Forge定的标准，把序列化逻辑写在其他地方（比如Capability接口实现类中，甚至弄个Utility class存放静态方法），但假定我们正确实现了IStorage接口，那么这个时候，一般会在ICapabilitySerializable的实现中，调用IStorage规定的那俩方法。

## 特例：玩家数据的持久化
实体对象死亡时，Capability数据会清空，这种逻辑对于一般的实体没什么问题，但是对于玩家而言问题就大了——玩家死亡时本质是复制了一个EntityPlayer对象出来，再把旧的EntityPlayer设成Dead状态，让GC来回收它。这个过程Capability数据默认也是丢失的，甚至玩家对象复制，在玩家进入末地传送门时也会发生——而在死亡或从末地回主世界时丢失Capability数据，一般都不是Modder的预期目的。

因此我们需要监听`PlayerEvent.Clone`事件，手动将Capability数据复制过来。

## Capability数据的双端同步
读者应该知道，上述的Capability的序列化和反序列化，都只发生在服务端，而客户端对此是毫不知情的。如果我们要有诸如“在屏幕上显示Capability数据信息”这种客户端层面的需求，我们就需要进行网络通信了。

ItemStack的NBT是默认全部同步至客户端的（严格来说，ItemStack大部分时候都是依附于方块实体、Container、实体等的，因此同步ItemStack实质上是对这三种游戏对象进行同步，原版和绝大多数Mod的对它们的同步实现都会默认同步全部ItemStack NBT），其他ICapabilityProvider则不是如此。原版的网络系统比较完善，但设计得太蠢——INetHandler及其子接口把一系列handle方法都硬编码死了，导致INetHandler只能处理原版的那些数据包，我们没有自定义数据包的空间。

幸好Forge提供了SimpleImpl，由此我们就能同步我们的Capability数据了。

## 获取Capability Token
我们在之前忽略了一个很重要的问题：Capability Token怎么拿到？那可是调用getCapability及hasCapability的关键啊。而Capability<?>的构造方法好像又是package-private的，这怎么办？

Forge提供了@CapabilityInject注解（其实就是依赖注入）：
```Java
    @CapabilityInject(IExampleCapability.class)
    public static final Capability<IExampleCapability> EXAMPLE_CAPABILITY = null;
    //值设为null是为了规避IDE对final字段的初始化检验，实际上Forge通过反射给其赋值，是无所谓final与否的。
    //这其实就是Capability系统降低代码耦合度的秘诀了——注解中的Class对象会被FML读取，FML会用try-catch保证该Class不存在时不崩游戏，而Capability的泛型参数在运行时会被擦除，具体使用该Capability Token的时候先判空，由此，即可安全地使用其他Mod提供的Capability。
    //所以，除非该Capability是由Forge提供的，否则你必须在你的代码中通过@CapabilityInject获取对应的Capability Token，再用，不能直接引用其他Mod代码中的Capability Token。
```

## 自定义Capability
前文提到，所有类型的Capability必须先注册：
```Java
    //第一个参数是Capability接口的Class对象。
    //第二个参数是Capability.IStorage<?>的实现。
    //第三个参数是Callable<? extends IExampleCapability>，lambda方法是用于提供该接口实现类的工厂方法。
    CapabilityManager.INSTANCE.register(IExampleCapability.class, new Capability.IStorage<>() {
        ...
    }, () -> new IExampleCapability() {
        ...
    });
```

## Forge内建的Capability
没错，Forge的一大票好用的功能都是建基于Capability系统之上的。Forge提供了以下四类Capability：
<ul>
<li>物品容器：IItemHandler</li>
<li>流体容器：IFluidHandler和IFluidHandlerItem</li>
<li>能量容器：IEnergyStorage</li>
<li>动画状态机：IAnimationStateMachine</li>
</ul>
