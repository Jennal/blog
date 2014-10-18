# GPP读书笔记之Command模式（一）

最近在读一本[开源的书《Game Programming Patterns》][1]，关于游戏设计模式的，作者叫Bob Nystrom，在EA工作了好多年，整理了他8年来的经验，写成了这本书，最近会出版实体书。作者的文风非常风趣幽默，以至于我一开始读就有点停不下来了，所以打算写一系列读书笔记来表示对作者的敬意。

前面两个章节主要是用来鼓吹这本书有多么多么好（这是作者自己说的），不过还是读到了不少内容，关于程序设计的理解，为什么要做好的设计，代码写出来能运行不就好了吗？这可能是大多数外行人的看法，也许也是码农的看法，但是作为工程师来说，我们要设计结构良好的代码，为的是将来更容易升级和维护。可能开发出一个功能只需要2天时间，但将来可能需要花好几个月来升级和维护这个功能。并且在升级和维护的过程中是非常难去改变原有的设计的，除非全部推倒重写。这对于有多次推倒重写经验的我来说，真是说到心眼里去了。如果你花两天时间写了一坨屎，接下来，你将在粪坑里挣扎几个月。所以，请善待将来的自己，使用良好的设计模式和规范来进行开发。

进入正题之前，再扯点蛋，作者无数次提到[GoF][2]，是Gang of Four的缩写，四人帮？当然不是中国的四人帮。他们是联合撰写了[《Design Patterns》][3]这本书的人。

## Command 模式是什么

[GoF][2] 是这么说的

> Encapsulate a request as an object, thereby letting users parameterize clients with different requests, queue or log requests, and support undoable operations.

我的翻译：

> 把一个请求封装成一个实体，进而让用户可以使用不同的参数来构造不同的请求，把这些请求队列化，或者记录到日志里，甚至支持重做。

作者认为这句话毛病太多了，一方面是client存在歧义，另一方面列举了一堆Command模式能做的事情，但是无法涵盖所有的事情，所以读者如果想做的不在里面，就很可能失去兴趣。所以作者自己总结了一句更简练的话：

> **A command is a reified method call.**

我的翻译：

> Command是被实体化的函数调用。

Command模式很容易让人联想到

*   回调 
*   [第一类函数][4]
*   函数指针
*   闭包
*   partially applied function（因为我不知道这个是什么，就不翻译了）

所以[GoF][2]后来还说

> Commands are an object-oriented replacement for callbacks.

我的翻译：

> Command是回调函数的面向对象实现。

## 代码时间

我简化了一下原文的例子，四个按钮有点多余，两个按钮就能说明问题啦，让我们想象超级玛丽的玩家操作处理代码（不考虑长按B加速的情况，这里只讨论点击操作）。

<pre lang='cpp'>void InputHandler::handleInput()
{
  if (isPressed(BUTTON_A)) jump();
  else if (isPressed(BUTTON_B)) fire();
}
</pre>

这段代码运行在游戏主循环中，已经可以满足一般游戏的需求。

### 可以配置的控制方式

但是这个时候策划说，让我们支持让玩家自己设置按键吧。作为开发，你一定会抓狂。不怕不怕，这种情况就是Command模式派上用场的时候了。

先定义一个基类

<pre lang='cpp'>class Command
{
public:
  virtual ~Command() {}
  virtual void execute() = 0;
};
</pre>

然后是子类

<pre lang='cpp'>class JumpCommand : public Command
{
public:
  virtual void execute() { jump(); }
};

class FireCommand : public Command
{
public:
  virtual void execute() { fireGun(); }
};
</pre>

InputHandler也做一些修改

<pre lang='cpp'>class InputHandler
{
public:
  void handleInput();

  // Methods to bind commands...

private:
  Command* buttonA_;
  Command* buttonB_;
};
</pre>

最后刚刚的处理代码变成了这样

<pre lang='cpp'>void InputHandler::handleInput()
{
  if (isPressed(BUTTON_A)) buttonA_->execute();
  else if (isPressed(BUTTON_B)) buttonB_->execute();
}
</pre>

### 角色引导

原文的标题是Directions for Actors，为什么把Directions翻译成引导，以我的理解，本章节的主要内容是，把jump函数从JumpCommand中取出来，Command不直接调用指定的函数，而是调用角色的成员函数，这样做的好处很明显，可以换角色操作了。所以功能引导从Command转换为角色。

在execute函数加入参数

<pre lang='cpp'>class Command
{
public:
  virtual ~Command() {}
  virtual void execute(GameActor& actor) = 0;
};
</pre>

所以JumpCommand就变成了这样

<pre lang='cpp'>class JumpCommand : public Command
{
public:
  virtual void execute(GameActor& actor)
  {
    actor.jump();
  }
};
</pre>

handleInput明显也不满足需求了，让它不再直接执行Command，而是返回Command

<pre lang='cpp'>Command* InputHandler::handleInput()
{
  if (isPressed(BUTTON_A)) return buttonA_;
  if (isPressed(BUTTON_B)) return buttonB_;

  // Nothing pressed, so do nothing.
  return NULL;
}
</pre>

因为InputHandler明显不知道现在游戏中选择的是什么角色，所以这个时候把Command先返回出来，以供外部调用，这也是一种延迟调用。 所以，我们得再来看看知道现在游戏中选择的是什么角色的某处代码，是怎么使用InputHandler的

<pre lang='cpp'>Command* command = inputHandler.handleInput();
if (command)
{
  command->execute(actor);
}
</pre>

不知道大家有没有发现？这么做了以后，不仅仅是可以换角色了，角色的控制者也可以替换。不仅仅支持玩家通过按键输入来控制角色，还可以利用AI来控制角色。甚至可以使用不同的AI算法来控制不同的角色。还可以用来在某些情况下，使用AI自动控制玩家的角色，比如自动寻路，自动战斗。

下面不得不用一张图来加强理解

![把Command放到队列/流中][5]

如果把Command放到队列/流中，就像上图中间的部分，这样就有效地解耦了生产者（AI/玩家输入）和消费者（角色）。

### 撤销/重做

这是最后一个例子，也是Command模式最知名的应用场景。 如果一个Command对象可以做某件事，那是不是也可以让它撤销呢？撤销操作常用在策略类游戏，工具类应用程序会有更多这类需求，包括自己研发的游戏编辑器，如果你做了一个不能撤销的编辑器给策划用，策划会恨你的。

如果不用Command模式，你想实现撤销操作，会意外地发现那是多么困难的一件事。但是用Command模式来实现的话，就超级简单了。 下面让我们来实现一个回合制的策略类游戏，玩家可以移动自己的单位，也可以撤销移动。

下面是移动操作的代码

<pre lang='cpp'>class MoveUnitCommand : public Command
{
public:
  MoveUnitCommand(Unit* unit, int x, int y)
  : unit_(unit),
    x_(x),
    y_(y)
  {}

  virtual void execute()
  {
    unit_->moveTo(x_, y_);
  }

private:
  Unit* unit_;
  int x_, y_;
};
</pre>

细心的你一定发现了这个类跟之前的不同，之前的角色是从execute传进来的，这个MoveUnitCommand是从构造函数传进来的。这是因为我们后续要做撤销操作，所以肯定要知道要对谁做撤销。这个例子也告诉我们，带参数的Command类，该怎么写。 延续之前的功能，MoveUnitCommand被实例化以后，可以在某一个特定时间点被调用，所以handleInput将这么被改写

<pre lang='cpp'>Command* handleInput()
{
  // Get the selected unit...
  Unit* unit = getSelectedUnit();

  if (isPressed(BUTTON_UP)) {
    // Move the unit up one.
    int destY = unit->y() - 1;
    return new MoveUnitCommand(unit, unit->x(), destY);
  }

  if (isPressed(BUTTON_DOWN)) {
    // Move the unit down one.
    int destY = unit->y() + 1;
    return new MoveUnitCommand(unit, unit->x(), destY);
  }

  // Other moves...

  return NULL;
}
</pre>

可是这么做还是不能撤销啊，别着急，下面告诉你怎么撤销，首先要改写Command基类

<pre lang='cpp'>class Command
{
public:
  virtual ~Command() {}
  virtual void execute() = 0;
  virtual void undo() = 0;
};
</pre>

undo就是execute的反操作，用来撤销execute操作的，那么重做就是重新调用一次execute。 加上undo实现以后的MoveUnitCommand

<pre lang='cpp'>class MoveUnitCommand : public Command
{
public:
  MoveUnitCommand(Unit* unit, int x, int y)
  : unit_(unit),
    xBefore_(0),
    yBefore_(0),
    x_(x),
    y_(y)
  {}

  virtual void execute()
  {
    // Remember the unit's position before the move
    // so we can restore it.
    xBefore_ = unit_->x();
    yBefore_ = unit_->y();

    unit_->moveTo(x_, y_);
  }

  virtual void undo()
  {
    unit_->moveTo(xBefore_, yBefore_);
  }

private:
  Unit* unit_;
  int xBefore_, yBefore_;
  int x_, y_;
};
</pre>

可以看到代码中多了两个变量xBefore_, yBefore_，来保存MoveUnitCommand执行之前的状态（位置）。所以撤销操作，就是移动回之前的位置。当然，在外部要控制玩家的操作，执行完execute，就只能撤销，撤销完，就只能重做，以免产生不必要的误解。 读到这里，对于多个Command的撤销和重做，应该也已经有一些想法了吧。

![多个Command的撤销/重做][6]

这里只给出一张图，不做详细解释了。原文还有一段解释，但我觉得上图已经足够说明一切了。

最后，恭喜你，读完了。

 [1]: http://gameprogrammingpatterns.com/
 [2]: http://c2.com/cgi/wiki?GangOfFour
 [3]: http://en.wikipedia.org/wiki/Design_Patterns
 [4]: http://en.wikipedia.org/wiki/First-class_function
 [5]: http://jennal.com/wp-content/uploads/2014/08/command-stream.png
 [6]: http://jennal.com/wp-content/uploads/2014/08/command-undo.png
