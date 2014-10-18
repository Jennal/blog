# GPP读书笔记之Flyweight模式（二）

原文地址：<http://gameprogrammingpatterns.com/flyweight.html>

## Flyweight模式是什么

Flyweight是一个非常简单的设计模式。它可能也是唯一可以从硬件上获得支持的设计模式。从名字可以看出，这是一个轻量级的设计模式，不仅仅体现在实现上，它也能大大降低内存的消耗，提高执行效率。它常被用在场景的开发中，特别是瓦片场景之类的，包含很多重复出现的子元素。

以瓦片地图（TileMap）举例，简单的说，Flyweight就是把瓦片的属性（成员变量）划分为两类

*   许多瓦片共有的属性，例如贴图、形状等
*   各个瓦片独有的属性，例如位置、旋转角度等

将共有的属性提取出来，每一个共有的属性做一个单例，所有的瓦片共享这些共有属性（保存引用或者指针）。这些共有的属性往往是加载缓慢的资源，那么在渲染时，只需要渲染一次，然后在不同的位置多次重绘，就可以大大降低内存消耗，提高执行效率。

以上是我粗浅的理解，原文给出了2个例子，来看原文的例子

## 简单的例子：森林中的树

作者很会讲故事，他说可以用简单的几句文字就能描述广阔的森林，但是用程序来实现，却不是那么简单的事情。当玩家看到满屏的树，图形程序员看到的是数百万个矩形每1/60秒被GPU渲染一次。

上千棵树，每棵树包含上千个矩形，即使你有足够的内存来保存整个森林的数据，渲染它们需要大量的数据占用从CPU到GPU的总线。

每棵树包含这些元素

*   多边形网格，来描述树的主干、分支以及树叶
*   枝干和树叶的纹理贴图
*   位置和朝向信息
*   其他一些信息，例如大小、色调，使得每棵树看起来都不太一样

它的代码大概是这个样子

<pre lang="cpp">class Tree
{
private:
    Mesh mesh_; //多边形网格
    Texture bark_; //枝干纹理贴图
    Texture leaves_; //树叶纹理贴图
    Vector position_;
    double height_;
    double thickness_;
    Color barkTint_;
    Color leafTint_;
};
</pre>

它包含了很多数据，特别是多边形网格和贴图信息。这些数据太大了，以至于无法在一帧里面全部丢到GPU里面去渲染。幸运的是，老前辈们已经帮我们想到了解决的方案。

最关键的问题在于，虽然有几千棵树，但是他们大部分看起来是一样的。他们可是使用相同的多边形网格信息和相同的纹理来渲染。这样一来，所有的树可以共享相同的实例来进行渲染。

先来看看之前的对象结构

![第一版森林结构图][1]

我们可以把上面的类一分为二，提取出共有的属性到一个独立的类

<pre lang="cpp">class TreeModel
{
private:
    Mesh mesh_;
    Texture bark_;
    Texture leaves_;
};
</pre>

游戏中只需要包含一个TreeModel对象的实例，没理由在内存中保存相同的多边形和纹理几千次吧。所有的Tree的实例都可以引用这个TreeModel实例。

<pre lang="cpp">class Tree
{
private:
    TreeModel* model_;

    Vector position_;
    double height_;
    double thickness_;
    Color barkTint_;
    Color leafTint_;
};
</pre>

现在的对象结构看起来就像这样

![第二版森林结构图][2]

虽然这个结构看起来好了很多，我们大大减小了内存的开销。但这么做并没有给渲染带来什么好处，我们需要让GPU理解我们的共享资源，才能提高渲染速度。

### 数以千计的对象

为了减少GPU渲染的次数，我们需要只把TreeModel发送给GPU一次。然后，分别发送每棵树的其他信息（位置、颜色、大小）。最后，我们告诉GPU，“用那一个TreeModel的数据去渲染每一棵树”。

幸运的是，现在的显卡真的提供了这样的API支持。实现的细节超过了本书的范围，但是Direct3D和OpenGL都支持[实例化绘制（Instanced Rendering）][3]。

这两个API，都需要程序员提供两组数据，一组是共用的多边形/纹理等信息，用来告诉GPU绘制什么；另一组是位置、大小等的信息列表，用来告诉GPU在哪里以及如何绘制第一组数据。最后，只要调用一次绘制，整片森林就轻松带微笑地出现了。

## Flyweight模式小结

现在大家已经看到了一个Flyweight应用的具体例子。这个例子简单到令人发指，也许有人会说，这根本就不是什么设计模式嘛，只是共享一个对象。没错，上面的例子，确实只共享了一个对象，那是因为这个例子足够简单、清晰、易懂。但是实际生产环境，想找到这样的例子，或许有些困难。后面会介绍稍微复杂的例子，还请大家稍安勿躁。

Flyweight解决的问题是，有太多的对象，却只有太少的内存。核心思想是，拆分对象，分解成共有的属性和特有的属性，把共有的属性独立成类的实例，让每个对象去共享这些实例。

## 复杂的例子：森林扎根的地方（地面）

地面将使用瓦片地图（tile-based）的方式来实现，有很多种类型的瓦片，例如：草地、土地、山、河、湖等等。

每个瓦片包含了以下的属性

*   移动代价，决定角色从这块瓦片上经过的移动速度
*   标记瓦片是否是水，决定这里是走人还是走船
*   渲染使用的纹理

游戏程序员特别在乎执行效率，所以他们认为没有必要把每一个瓦片的所有信息都放到内存里面，一般会用枚举来实现

<pre lang="cpp">enum Terrain
{
    TERRAIN_GRASS,
    TERRAIN_HILL,
    TERRAIN_RIVER
    // Other terrains...
};
</pre>

世界地图的平面将由一个二维数组来组织

<pre lang="cpp">class World
{
private:
    Terrain tiles_[WIDTH][HEIGHT];
};
</pre>

实际使用瓦片的时候的代码是这样的

<pre lang="cpp">int World::getMovementCost(int x, int y)
{
    switch (tiles_[x][y])
    {
        case TERRAIN_GRASS: return 1;
        case TERRAIN_HILL:  return 3;
        case TERRAIN_RIVER: return 2;
        // Other terrains...
    }
}

bool World::isWater(int x, int y)
{
    switch (tiles_[x][y])
    {
        case TERRAIN_GRASS: return false;
        case TERRAIN_HILL:  return false;
        case TERRAIN_RIVER: return true;
        // Other terrains...
    }
}
</pre>

很显然，这样写可以运行的很好，但是却是很恶心的代码。移动代价（MovementCost）、是否是水（isWater），这些都应该属于是瓦片的数据，却被硬编码在了函数里面。应该把这些属性合并成一个瓦片类（Terrain），这不就是类（objects）设计的意义么？

<pre lang="cpp">class Terrain
{
public:
    Terrain(int movementCost,
            bool isWater,
            Texture texture)
    : movementCost_(movementCost),
      isWater_(isWater),
      texture_(texture)
    {}

    int getMovementCost() const { return movementCost_; }
    bool isWater() const { return isWater_; }
    const Texture& getTexture() const { return texture_; }

private:
    int movementCost_;
    bool isWater_;
    Texture texture_;
};
</pre>

地图上需要成百上千的瓦片（Terrain），但我们并不希望创建这么多的实例。你应该察觉到了，上面的类，没有包含位置信息，这也是唯一的，每个瓦片（Terrain）不同的数据。所以整个瓦片（Terrain）都是上下文无关的，可以当做共有属性来使用。

因此，我们只需要保存每一种类型的瓦片（Terrain）一个实例就可以了。

<pre lang="cpp">class World
{
private:
    Terrain* tiles_[WIDTH][HEIGHT];

    // Other stuff...
};
</pre>

数组中所有相同类型的Terrain指针，都将指向同一个实例。

![Terrain指针示意图][4]

当这么多Terrain的指针指向相同的实例时，如果动态创建Terrain对象的话，它们的生存周期就变得难以管理。简单的处理方法是，把它们放到World类里面

<pre lang="cpp">class World
{
public:
    World()
    : grassTerrain_(1, false, GRASS_TEXTURE),
      hillTerrain_(3, false, HILL_TEXTURE),
      riverTerrain_(2, true, RIVER_TEXTURE)
    {}

private:
    Terrain grassTerrain_;
    Terrain hillTerrain_;
    Terrain riverTerrain_;

    // Other stuff...
};
</pre>

绘制的代码是这样的

<pre lang="cpp">void World::generateTerrain()
{
    // Fill the ground with grass.
    for (int x = 0; x &lt; WIDTH; x++)
    {
        for (int y = 0; y &lt; HEIGHT; y++)
        {
            // Sprinkle some hills.
            if (random(10) == 0)
            {
                tiles_[x][y] = &hillTerrain_;
            }
            else
            {
                tiles_[x][y] = &grassTerrain_;
            }
        }
    }

    // Lay a river.
    int x = random(WIDTH);
    for (int y = 0; y &lt; HEIGHT; y++) {
        tiles_[x][y] = &riverTerrain_;
    }
}
</pre>

为了不让外部直接访问World的Terrain实例，我们还需要提供一个方法

<pre lang="cpp">const Terrain& World::getTile(int x, int y) const
{
    return *tiles_[x][y];
}
</pre>

这样一来，World就不需要知道Terrain的细节，如果你需要得到一个Terrain的属性，可以这么做

<pre lang="cpp">int cost = world.getTile(2, 3).getMovementCost();
</pre>

这么做，我们就可以简单调用对象的API来获得它的属性。从枚举改写为类，几乎没有什么代价，毕竟指针比枚举大不了多少。

## 性能问题

有些人可能会关心，从枚举到指针的转变，会带来多大的性能损耗。答案是，几乎没有损耗，可能还有性能提升。

想知道详情的话，建议[阅读原文][5]。我只想做一个简单的总结，作者有句话说的非常好：优化性能的黄金法则，就是先做性能测试。没有任何测试，光靠自己想象，是很难模拟真实情况的。因为现代硬件上的优化，已经大大超乎普通人类的想象了。要在脑子里还原程序在硬件上运行的情况，几乎是不可能的事情。游戏的性能问题，也很可能不再是某个单一原因引起的。

作者实测的结果是，使用Flyweight修改后的代码，要比之前用枚举写的，跑得更快，但这取决于其他数据在内存中的结构。

在这里也给各位同学提个醒，千万别看到某段代码，就轻易下结论说，“这里肯定有性能问题”或者“这么写性能肯定没问题”，实践才是检验真理的唯一标准。

 [1]: http://jennal.com/wp-content/uploads/2014/08/flyweight-trees.png
 [2]: http://jennal.com/wp-content/uploads/2014/08/flyweight-tree-model.png
 [3]: http://en.wikipedia.org/wiki/Geometry_instancing
 [4]: http://jennal.com/wp-content/uploads/2014/08/flyweight-tiles.png
 [5]: http://gameprogrammingpatterns.com/flyweight.html#what-about-performance
