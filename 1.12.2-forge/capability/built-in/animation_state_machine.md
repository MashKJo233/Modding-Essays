动画状态机（Animation State Machine），是Forge引入的一套动画API，通过它Modder可以方便地通过JSON格式来描述并创建方块或物品的动画效果，而无需调用底层渲染方法。该说不说，在扁平化之前的、硬编码游戏内容占主流的低版本，能用这样一套高度数据驱动的设计，确实较为超前。

虽说要使用该API，需要让相关的游戏对象提供动画Capability Token（`CapabilityAnimation.ANIMATION_CAPABILITY`），但不得不说这套API的画风和另外三个Capability格格不入，对Capability接口的操作几乎完全是交给Forge内部代码；同时它也比较冷门，在高版本便销声匿迹乃至彻底消失。

本篇教程部分内容参考自Forge官方文档，及Forge论坛上的一篇动画化TileEntity的教程：(https://forums.minecraftforge.net/topic/65310-tileentity-animation-system-explanation/)。

## 适用范围
并非所有ICapabilityProvider都支持动画Capability，Forge的官方文档中有说，ItemStack、TileEntity和Entity均支持动画Capability，实则在1.12的Forge中，实体的动画支持尚未完成，因此只有ItemStack和TileEntity可用。

## JSON文件格式
### Armature JSON
Armature JSON用于定义“物品或方块模型中，参与动画的部分及其可行的空间变换（见`IModelState`及`TRSRTransformation`）”，它必须存放在路径`assets/[modid]/armatures/item`或`assets/[modid]/armatures/block`文件夹下，名称与对应的物品/方块模型的名称相同。

首当其冲的是`joints`，该部分阐明了模型中参与动画的部分，并允许你给其分组：

```JSON
{
    "joints": {
        "group1": {
            "0": [1.0]
        },
        "group2": {
            "1": [1.0],
            "2": [1.0]
        },
        ...
    },
    ...
}
```

`joints`中的节点名称可以任取，和模型文件里定义的elements节点名称并无关系。这里的这些节点，如示例中所见，是要给模型的各个部分分组，每个组便是一个joint。

组内节点名称必须是字符串形式的自然数，实际就是模型文件里定义的element的序数，从0开始计数。

一般而言有动画的方块，模型都是用Blockbench之类的建模软件弄的，导出JSON的时候你就能看到里面定义的element了。对于物品以及简单方块的模型，它们都是继承自预定义好的模板JSON，你还需要看看这些模板JSON定义element的顺序。

然后就是序数对应的值了——数组？莫要奇怪，说是数组，其实填一个数就可以了，它的取值是0 - 1的浮点数，表示的是“每个joint定义的空间变换对最终动画的贡献，或者说比重”。

接着是`clips`，包含了若干clip，clip就是对每个joint可行的空间变换以及触发的事件（`net.minecraftforge.common.animation.Event`）的定义。

```JSON
{
    "clips": {
        "default": {
            "loop": true/false,
            "joint_clips": {
                "group1": [
                    {
                        "variables": ...,
                        "type": "uniform",
                        "interpolation": "linear"/"nearest",
                        "samples": [...]
                    },
                    ...
                ],
                "group2": [
                    ...
                ]
            },
            "events": {
                "<time_value_1>": "one_event",
                "<time_value_2>": "another_event"
            }
        },
        "another_clip": {
            ...
        }
    }
}
```

一般而言，clips中会定义一个名为default的clip。

clip（亦称为armature-clip）的定义格式较为复杂：
<ul>
<li>loop：用于决定该clip定义的空间变换是否是循环的</li>
<li>joint_clips：这里面的各个节点名就完全取决于前面所定义的joints了，每个节点接受一个数组：元素即为该joint可行的空间变换。空间变换的格式如下：</li>
<ul>
<li>variables：用于决定空间变换的类型——无外乎就是平移、伸缩和旋转这三种：</li>
<ul>
<li>平移：`offset_x`、`offset_y`和`offset_z`</li>
<li>伸缩：`scale`、`scale_x`、`scale_y`和`scale_z`</li>
<li>旋转：</li>
<ul>
<li>不动点：`origin_x`、`origin_y`和`origin_z`</li>
<li>旋转轴：`axis_x`、`axis_y`和`axis_z`</li>
<li>旋转角度：`angle`</li>
</ul>
</ul>
<li>type：只能是`uniform`，推测这可能是阐明“动画的渲染作用域”，目前仅“uniform（全局）”可用</li>
<li>interpolation：插值方式，支持`nearset`（最近邻插值）和`linear`（线性插值）两种（实质是根据当前partialTick补全非整tick时渲染状况）</li>
<li>samples：数组，长度取决于variables的类型</li>
</ul>
<li>events：定义动画过程中特定时间点触发的事件——键值对中，键负责定义“触发该事件的时间点”（取值为0 - 1的浮点数），值则为一个字符串，即“事件的名称”。如果没有事件需要被触发，留空即可。event在代码内的体现为`net.minecraftforge.common.animation.Event`（注意不是我们通常所说的`net.minecraftforge.fml.common.eventhandler.Event`）</li>
</ul>

### ASM JSON
ASM JSON用于定义动画的各个状态以及它们之间的相互转化规律，路径并没有强制要求，但推荐存放在`assets/[modid]/asms/item`或`assets/[modid]/asms/block`路径下，名称同Armature JSON。

```JSON
{
    "parameters": {
        ...
    },
    "clips": {
        ...
    },
    "states": [
        ...
    ],
    "transitions": {
        ...
    },
    "start_state": "..."
}
```
`states`，用于定义“动画有哪些状态”，是字符串数组；`start_state`用于声明动画开始时状态机所处的状态，值类型为字符串。

`transitions`则定义了：动画状态机各个状态之间的相互转化法则，键值对中，键为待转化的状态，值为单字符串或字符串数组，代表“可转化到的状态”。

`parameters`和`clips`则有些复杂：

parameter和asm-clip在代码中的形式为接口`ITimeValue`以及`IClip`。ITimeValue接受一个float值的输入——一般为当前的客户端游戏时间（可以理解为`currentTicks + partialTick`），经过一定的处理后返回另外一个浮点值，用于标识动画进行的进度。

IClip有很多种，其中最简单的就是引用armatures中的clip（armature-clip），即`ModelClip`；还有接受一个ITimeValue的输出作为输入的`TimeClip`，等等。简而言之，一个asm-clip定义了：在渲染一个armature-clip时，做出的补充行为——可以是根据当前游戏时间对armature-clip中的空间变换进行调整，或触发事件，等等。

限于篇幅，这里不会赘述所有`ITimeValue`和`IClip`的实现各自的用法，请自行在开发环境中和Forge官方文档上查阅。

## 使用
### @SidedProxy和ASM的加载
由于这是“动画状态机”，司掌客户端的动画渲染效果，因此动画状态机只在客户端有意义。但ItemStack和TileEntity的动画特性由Capability系统提供支持——Capability系统是通用于双端的，因此动画状态机对应的接口`IAnimationStateMachine`也未曾打上`@SideOnly(Side.CLIENT)`。

Forge因此约定：在服务端，IAnimationStateMachine的实例为null；在客户端则相反。

不同于其他内建Capability和九成九的第三方Capability，我们无需额外操作IAnimationStateMachine的实例，要获取对应的实例，只需这么做：
```Java
public class CommonProxy {
    ...
    public IAnimationStateMachine loadASM(ResourceLocation location, ImmutableMap<String, ITimeValue> parameters) {
        return null;
    }
}
```

```Java
public class ClientProxy extends CommonProxy {
    ...
    @Override
    public IAnimationStateMachine loadASM(ResourceLocation location, ImmutableMap<String, ITimeValue> parameters) {
        return ModelLoaderRegistry.loadASM(location, parameters);
    }
}
```

然后我们就可以在我们的ICapabilityProvider中初始化IAnimationStateMachine字段了：
```Java
    private final IAnimationStateMachine asm = proxy.loadASM(new ResourceLocation(MODID, "asms/[block or item]/xxx.json"), ImmutableMap.of(...));
```
第一个参数是一个ResourceLocation，是ASM JSON的完整路径（最完整的ResourceLocation版本，要求写出文件扩展名）；第二个参数则是允许你在代码层面指定`parameters`，或者说是`ITimeValue`。

### 应用于TileEntity
方块实体要拥有特殊的动画，唯一的途径就是注册一个方块实体特殊渲染器（TileEntitySpecialRenderer，TESR），因此支持动画状态机的方块实体要有一个AnimationTESR。

```Java
public class MyAnimatedTileEntity extends TileEntity {
    private final IAnimationStateMachine asm = proxy.loadASM(...);
    ...

    //因为AnimationTESR继承自FastTESR，因此我们需要覆盖这个方法。
    @Override
    public boolean hasFastRenderer() {
        return true;
    }

    //需要提供动画Capability（CapabilityAnimation.ANIMATION_CAPABILITY）。
    @Override
    public boolean hasCapability(Capability<?> capability, EnumFacing facing) {
        return capability == CapabilityAnimation.ANIMATION_CAPABILITY || super.hasCapability(capability, facing);
    }

    @Override
    public <T> T getCapability(Capability<T> capability, EnumFacing facing) {
        if(capability == CapabilityAnimation.ANIMATION_CAPABILITY) {
            return CapabilityAnimation.ANIMATION_CAPABILITY.cast(asm);
        }
        return super.getCapability(capability, facing);
    }
}
```

然后是对应的方块类。首先方块状态容器中必须有`Properties.AnimationProperty`这一`IUnlistedProperty`。

接着，如果你的方块并非一直有动画，那么你需要在你的方块状态容器中提供`Properties.StaticProperty`，这是一个`PropertyBool`，当它的值为true时，动画状态机并不会起作用，方块会像普通方块一样被渲染（但实际上它还是和普通方块不太一样，如它被破坏不会导致区块重绘）；否则，你就必须覆盖`Block#getRenderType`方法，返回`ENTITYBLOCK_ANIMATED`。

最后是绑定TileEntity和对应的AnimationTESR：
```Java
    ClientRegistry.bindTileEntitySpecialRenderer(MyAnimatedTileEntity.class, new AnimationTESR<MyAnimatedTileEntity>()
    {
        @Override
        public void handleEvents(MyAnimatedTileEntity te, float time, Iterable<Event> pastEvents)
        {
            te.handleEvents(time, pastEvents);
        }
    });
```
还记得之前定义的events吗？现在在这里我们就能对这些动画事件进行一定的处理了。

### 应用于ItemStack
ItemStack的动画是借由一个自定义的ItemOverrideList实现的。其ICapabilityProvider只可能来自第三方，我们通过覆写`Item#initCapabilities`或监听`AttachCapabilitiesEvent<ItemStack>`事件来给ItemStack附加动画的ICapabilityProvider，写法和方块实体的类似。

ItemStack的动画中的事件并不能被监听、追踪到。

### 应用于Entity
前文说过，实体的动画支持尚未做完（实际上实体的硬编码模型和JSON格式就格格不入，做不出也是情理之中了）。
