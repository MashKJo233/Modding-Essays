> 有鉴于1.12.2 Forge文档的中文翻译版中，这一部分的内容，大部分都未被翻译，且已经被翻译的内容也是一股机翻的味道，读起来不甚顺畅，因此这里便将“高级模型（Advanced Models）”这一部分进行翻译。翻译会穿插译注，且不会完全照搬全文，会根据中文的语言习惯进行补充词汇、省略词汇、调整语句顺序，且类名、方法名不会被翻译。

简单的模型和方块状态文件格式或许很好用，然而它们并不是动态的。举个例子：Forge提供的万能桶可以持有任何一种流体，流体完全有可能是由其他Mod所添加的。它得动态地把空桶和流体的模型组装在一起。这是怎么做到的？请见`IModel`。

为了理解这一套系统是如何工作的，让我们深入到模型系统的内里。通过这一部分的学习，你或许会对“有关动态模型的这一切都是怎么发生的”建立起清晰的认知。如果你没做到这一点，也是正常的。你可能并不会理解所有东西，但当你点开这一部分时，你学到的东西应该会越来越多，直到你明了了一切知识。

> 重要：如果这是你第一次阅读这个部分，不要跳过任何文字！你应该按照顺序阅读所有内容，以建构起一个全面的认知！同样，在你初学时，不要点进去某些引导你进入更深层次内容的链接！

<ol>

<li>一个ModelResourceLocation组成的集合会被标识为待被ModelLoader加载的模型。</li>
<ul>
<li>对于物品而言，它们的模型必须经过ModelLoader.registerItemVariants的标识，才可被加载（ModelLoader.setCustomModelResourceLocation已经自动完成了这一步）。</li>
<li>对于方块而言，它们的方块状态映射（译注：IStateMapper）会产生一个Map<IBlockState, ModelResourceLocation>。所有方块都会被遍历，Map的值会被加载。</li>
</ul>

<li>所有IModel会从每个ModelResourceLocaion中被加载，并被缓存在一个Map<ModelResourceLocation, IModel>之中。</li>
<ul>
<li>一个IModel会被一个唯一的接受它的ICustomModelLoader加载（多个ICustomModelLoader尝试加载一个相同的模型会抛出一个LoaderException）。如果未找到模型，且传入的ResourceLocation实际上是一个ModelResourceLocation（这说明这不是一个普通的模型，而实际上是一个方块状态的variant），它会被方块状态加载器（VariantLoader）所加载；否则，模型会被认为是一个原版格式的JSON模型，会被以原版的方式加载（VanillaLoader）。</li>
<ul>
<li>译注：方块和Mod物品基本都是通过VariantLoader加载的，方块的IStateMapper就是填充一个Map<IBlockState, ModelResourceLocation>的对象，Mod物品的模型位置的指定也需要一个ModelResourceLocation——因此物品的模型位置会首先从其方块形式（如果有）的inventory这一variant寻找，如果找不到，则会传递给VanillaLoader，即——在路径assets/[modid]/models/item路径下寻找。直接通过ResourceLocation来加载IModel，通常是特殊的ICustomModelLoader的做法。</li>
</ul>
<li>一个原版的JSON模型（models/item/*.json或者models/block/*.json）在被加载时，是一个ModelBlock（是的，甚至对于物品的情形也是如此），这是一个原版的类，和IModel接口并无任何联系。为了纠正这一点，它被包装进了一个名为VanillaModelWrapper的类，该类则实现了接口IModel。</li>
<li>一个原版/Forge（译注：Forge BlockState V1）的方块状态格式的variant在被加载时，先会加载它所来自的整个blockstate JSON文件。这个JSON文件会被反序列化为一个ModelBlockDefinition，这里面缓存着该JSON的路径。接着一个variant定义的列表会从ModelBlockDefinition中被导出，置入一个WeightedRandomModel中。</li>
<li>当加载一个原版的物品JSON模型时（models/item/*.json），模型会从一个带有inventory这一variant的ModelResourceLocation那里得来（例如：泥土方块的物品形式的模型是minecraft:dirt#inventory）；因此模型会被VariantLoader加载。当VariantLoader从blockstate JSON那里加载失败后，便传递给VanillaLoader加载。</li>
<ul>
<li>这个过程中最重大的副作用是如果有一个错误出现在VariantLoader中，它会尝试用VanillaLoader来加载该模型。如果VanillaLoader也不能成功加载模型，导致的异常会产生2个堆栈追踪信息。第一个是VanillaLoader的，第二个则来自VariantLoader。当你对模型加载错误进行debug的时候，你需要先确认你分析的是那个正确的堆栈信息。</li>
</ul>
<li>一个IModel可以被加载自一个ResourceLocation，或者通过调用ModelLoaderRegistry.getModel从缓存中获取或作为其中一个异常处理备选方案。</li>
</ul>

<li>所有加载的模型依赖的纹理会被加载并放入精灵图集。精灵图集是一个巨型的纹理，它把所有模型依赖的纹理粘在一起。当一个模型要渲染一个纹理时，会加上一个额外的UV坐标偏移，来正确映射到图集中的对应的纹理。</li>

<li>所有模型会通过调用model.bake(model.getDefaultState(), ...)被烘焙。烘焙而成的IBakedModel会被缓存到一个Map<ModelResourceLocation, IBakedModel>中。</li>

<li>这个Map接着会被存储在ModelManager中。ModelManager是一个游戏中的单例且以Minecraft::modelManager字段的形式存在，它是private修饰的，没有getter可用。</li>
<ul>
<li>但ModelManager是能被获取到的，不用反射或者Access Transformers——通过Minecraft.getMinecraft().getRenderItem().getItemModelMesher().getModelManager()或者Minecraft().getMinecraft().getBlockRenderDispatcher().getBlockModelShapes().getModelManager()。不同于它们迥异的名称，二者是等价的。</li>
<li>你可以通过ModelManager中的缓存（没有正在加载和/或烘焙的模型，只有已经完成这些步骤的IBakedModel）来获取对应的IBakedModel——通过调用ModelManager::getModel。</li>
</ul>

<li>最终，一个IBakedModel会被渲染出来。这个过程通过调用IBakedModel::getQuads做到。该方法的返回值是一个BakedQuad（Quad的意思是四边形——带有四个顶点的多边形）的列表（List）。它们能被稍后直接送往GPU进行渲染。物品和方块在这里有些许的不同，但逻辑相对简单。</li>
<ul>
<li>原版的物品有属性覆盖（Property Override）。为了解决这些，IBakedModel定义了getOverrides方法，它返回一个ItemOverrideList。ItemOverrideList定义了handleItemState方法（译注：实际上这个方法是Forge patch进来的），带有原本的IBakedModel、实体（Entity）、World、ItemStack这些参数，返回最终的IBakedModel。物品的overrides会被应用在其他针对模型的操作之前，包括getQuads方法。因为IBlockState参数并不适用于物品，getQuads方法中这一参数在渲染物品时，传入的是null。</li>

<li>译注：简而言之：物品的IBakedModel不会直接调用getQuads，而是先通过对应的ItemOverrideList::handleItemState获取最终的IBakedModel。</li>

<li>方块有方块状态，当一个方块的IBakedModel被渲染时，方块状态（IBlockState）会被直接传递给getQuads方法。仅在模型这一上下文中，方块状态可以有其他一些额外的属性，被称作Unlisted Properties。</li>
</ul>
</ol>
