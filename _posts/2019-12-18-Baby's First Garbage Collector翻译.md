当我感到有压力并且有太多事情要做时，我就会有这种矛盾的反应，我会想出另一件事来逃避压力。通常它是一个我可以编写和完成的小型独立程序。
有一天早上，我正在写的那本书把我吓坏了，这是我工作不得不做的事情，我正在为一个怪圈的演讲做准备，突然，我想，“我应该写一个垃圾收集器。”
是的，我意识到这段话让我看起来有多疯狂。但我的连接错误是你的免费教程上的一个基本的编程语言实现!在大约100行vanilla C代码中，我成功地设计了一个基本的标记-清除收集器，实际上，你知道，它是用来收集的。
垃圾收集被认为是编程中鲨鱼出没的水域之一，但在这篇文章中，我将为您提供一个很好的儿童池。(里面可能还有鲨鱼，但至少会浅一些。)

# 减少、复用、再循环
垃圾收集背后的基本思想是，语言(在大多数情况下)似乎可以访问无限内存。开发人员只需不断地分配、分配和分配，就像魔术一样，它永远不会失败。
当然，机器没有无限的内存。实现的方式是当它需要分配一些内存时它意识到内存快用完了，它就会收集垃圾。
在这个上下文中，“垃圾”指的是以前分配的、现在不再使用的内存。为了让无限内存的幻觉起作用，语言需要对于“不再被使用”非常安全。如果随机对象在程序试图访问它们时才开始被回收，那就没有什么意思了。
为了可收集，该语言必须确保程序无法再次使用该对象。如果它不能获得对象的引用，那么它显然不能再次使用它。所以“在用”的定义其实很简单:
1. 仍然在作用域内的变量所引用的任何对象都在使用中。
2. 任何被另一个正在使用的对象引用的对象都在使用中。

第二个规则是递归规则。如果对象A是由一个变量引用的，并且它有一些引用对象B的字段，那么就使用B，因为您可以通过A访问它。
最终的结果是一个可访问对象的图——通过从一个变量开始并遍历对象，可以访问世界上的所有对象。任何不在那个可达对象图中的对象对程序来说都是死的，它的内存已经到了可以收割的时候了。

# 标记和清除
有很多不同的方法可以实现查找和回收所有未使用的对象的过程，但是最简单的也是迄今为止发明的第一个算法叫做“标记-清除”。它是由约翰·麦卡锡发明的，这个人发明了Lisp和大胡子，所以你现在执行它就像在和一个上古之神交流，但希望不是以某种洛夫克拉夫式（霍華德·菲利普斯·洛夫克拉夫特是美國恐怖，科幻与奇幻小說作家，尤以其怪奇小說著称）的方式，以你的大脑和视网膜被摧毁的方式结束。
它的工作原理与我们对可达性的定义几乎完全一样:
1. 从根开始，遍历整个对象图。每次你碰到一个物体，在它上面设置一个“标记”位为真。
1. 完成之后，找到所有没有设置标记位的对象并删除它们。

就是这样。我知道，你能想到的，对吧?如果你有，你就会是一篇被引用数百次的论文的作者。这里的教训是，要想在计算机科学领域出名，你不需要想出真正聪明的东西，你只需要先想出愚蠢的东西。

# 一对对象
在我们开始实现这两个步骤之前，让我们先做一些准备工作。我们实际上不会为语言实现解释器——没有解析器、字节码或其他任何愚蠢的东西——但是我们确实需要一些最少的代码来创建一些要收集的垃圾。

让我们假装我们正在为一门小语言编写一个解释器。它是动态类型的，有两种对象的类型:ints和pairs。这里有一个枚举来标识对象的类型:

```c
typedef enum {
  OBJ_INT,
  OBJ_PAIR
} ObjectType;
```

一个pair可以是任何一对，两个ints，一个int和另一个pair。你可以用它走得更远。因为VM中的对象可以是这两种中的一种，所以C语言中实现它的典型方式是使用标记了的union。
我们将这样定义它:

```c
typedef struct sObject {
  ObjectType type;

  union {
    /* OBJ_INT */
    int value;

    /* OBJ_PAIR */
    struct {
      struct sObject* head;
      struct sObject* tail;
    };
  };
} Object;
```

主Object结构有一个type字段，用于标识它是什么类型的值——是int还是pair。然后它有一个union来保存int或pair的数据。如果你的C迟钝了，union就是内存中字段重叠的结构。由于给定的对象只能是一个整型或一对，所以没有理由在一个对象中同时为所有三个字段保留内存。union就是这样做的。绝妙的。

# 最小虚拟机
现在我们可以把它封装到一个虚拟机结构中。它在这个故事中的角色是在当前范围内有一个存储变量的堆栈。大多数语言vm要么是基于堆栈的(如JVM和CLR)，要么是基于寄存器的(如Lua)。在这两种情况下，实际上仍然有一个堆栈。它用于存储表达式中间需要的局部变量和临时变量。
我们将明确和简单地建模如下:

```c
#define STACK_MAX 256
```

```c
typedef struct {
  Object* stack[STACK_MAX];
  int stackSize;
} VM;
```

现在我们已经有了基本的数据结构，让我们拼凑一些代码来创建一些东西。首先，让我们写一个函数，创建和初始化一个VM:

```c
VM* newVM() {
  VM* vm = malloc(sizeof(VM));
  vm->stackSize = 0;
  return vm;
}
```

一旦我们有了一个VM，我们需要能够操作它的堆栈:

```c
void push(VM* vm, Object* value) {
  assert(vm->stackSize < STACK_MAX, "Stack overflow!");
  vm->stack[vm->stackSize++] = value;
}
```

```c
Object* pop(VM* vm) {
  assert(vm->stackSize > 0, "Stack underflow!");
  return vm->stack[--vm->stackSize];
}
```

好了，现在我们可以把东西放到变量里了，我们需要能够实际创建对象。首先是一个小助手功能:

```c
Object* newObject(VM* vm, ObjectType type) {
  Object* object = malloc(sizeof(Object));
  object->type = type;
  return object;
}
```

它执行实际的内存分配并设置类型标记。我们稍后会再讨论这个问题。使用它，我们可以写函数把各种对象推到虚拟机的堆栈:

```c
void pushInt(VM* vm, int intValue) {
  Object* object = newObject(vm, OBJ_INT);
  object->value = intValue;
  push(vm, object);
}
```
```c
Object* pushPair(VM* vm) {
  Object* object = newObject(vm, OBJ_PAIR);
  object->tail = pop(vm);
  object->head = pop(vm);

  push(vm, object);
  return object;
}
```

这就是我们的VM。如果我们有一个解析器和一个解释器来调用这些函数，我们就有了一门真正的语言。如果我们有无限的内存，它甚至可以运行真正的程序。既然我们没有，让我们开始收集一些垃圾吧。

# 标记
第一阶段是标记。我们需要遍历所有可到达的对象并设置它们的标记位。然后我们需要做的第一件事是给对象添加一个标记位:

```c
typedef struct sObject {
  unsigned char marked;
  /* Previous stuff... */
} Object;
```

创建新对象时，将修改newObject()以初始化marked为零。为了标记所有可达的对象，我们从内存中的变量开始，这意味着遍历堆栈。它是这样的:

```c
void markAll(VM* vm)
{
  for (int i = 0; i < vm->stackSize; i++) {
    mark(vm->stack[i]);
  }
}
```

这反过来又叫mark。我们将分阶段构建。第一:

```c
void mark(Object* object) {
  object->marked = 1;
}
```

这是最重要的一点。我们已经将对象本身标记为可访问的，但是请记住我们还需要处理对象中的引用:可访问性是递归的。如果对象是一对，那么它的两个字段也是可达的。处理起来很简单:

```c
void mark(Object* object) {
  object->marked = 1;

  if (object->type == OBJ_PAIR) {
    mark(object->head);
    mark(object->tail);
  }
}
```

但是这里有一个bug。你看到了吗?我们现在在递归，但是我们没有检查循环。如果在一个循环中有一堆相互指向的对，就会溢出堆栈并导致崩溃。
要处理这个，我们只需要跳出如果我们到达一个我们已经处理过的对象。因此，完整的mark()函数是:

```c
void mark(Object* object) {
  /* If already marked, we're done. Check this first
     to avoid recursing on cycles in the object graph. */
  if (object->marked) return;

  object->marked = 1;

  if (object->type == OBJ_PAIR) {
    mark(object->head);
    mark(object->tail);
  }
}
```

现在我们能调用markAll()并且它将正确地标记内存中每个可到达的对象。我们基本上已经完成了!

# 清除
下一个阶段是扫描我们分配的所有对象，并释放所有没有标记的对象。但是这里有一个问题:根据定义，所有未标记的对象都是不可到达的!我们找不到他们!

VM已经实现了对象引用的语言语义:所以我们只将指向对象的指针存储在变量和pair元素中。一旦一个对象不再被其中之一指向，我们就完全丢失了它，实际上是泄漏了内存。

解决这个问题的技巧是，VM可以有自己的对象引用，这些对象与语言用户可见的语义不同。换句话说，我们可以自己跟踪它们。
最简单的方法是维护一个链表，其中包含我们曾经分配过的每个对象。我们将对象本身扩展为列表中的一个节点:

```c
typedef struct sObject {
  /* The next object in the list of all objects. */
  struct sObject* next;

  /* Previous stuff... */
} Object;
```


VM会跟踪列表的头部:

```c
typedef struct {
  /* The first object in the list of all objects. */
  Object* firstObject;

  /* Previous stuff... */
} VM;
```
newVM()内我们将确定初始化firstObject为NULL。
每当我们创建一个对象，我们添加到列表:
```c
Object* newObject(VM* vm, ObjectType type) {
  Object* object = malloc(sizeof(Object));
  object->type = type;
  object->marked = 0;

  /* Insert it into the list of allocated objects. */
  object->next = vm->firstObject;
  vm->firstObject = object;

  return object;
}
```

这样，即使语言找不到对象，语言实现仍然可以找到对象。要清除和删除未标记的对象，我们只需要遍历列表:

```c
void sweep(VM* vm)
{
  Object** object = &vm->firstObject;
  while (*object) {
    if (!(*object)->marked) {
      /* This object wasn't reached, so remove it from the list
         and free it. */
      Object* unreached = *object;

      *object = unreached->next;
      free(unreached);
    } else {
      /* This object was reached, so unmark it (for the next GC)
         and move on to the next. */
      (*object)->marked = 0;
      object = &(*object)->next;
    }
  }
}
```
因为那个指针指向一个指针，所以代码读起来有点复杂，但是如果你仔细看，你会发现它很简单。它只是遍历整个链表。每当它碰到一个没有标记的对象时，它就释放它的内存并将其从列表中移除。完成后，我们将删除所有不可到达的对象。
恭喜你!我们有一个垃圾收集器!这里只缺少一个部分:真正地调用它。首先让我们把这两个阶段放在一起:

```c
void gc(VM* vm) {
  markAll(vm);
  sweep(vm);
}
```

没有比这更明显的标记-清除实现了。最棘手的部分是弄清楚什么时候该调用它。“内存不足”到底是什么意思，尤其是在拥有近乎无限虚拟内存的现代计算机上?
事实证明，这个问题没有绝对的对错之分。这实际上取决于VM的用途以及它运行在什么硬件上。为了保持这个示例的简单性，我们将在分配一定数量的内存之后收集内存。这实际上是一些语言实现的工作方式，而且很容易实现。
我们将扩展VM来跟踪我们创建了多少:

```c
typedef struct {
  /* The total number of currently allocated objects. */
  int numObjects;

  /* The number of objects required to trigger a GC. */
  int maxObjects;

  /* Previous stuff... */
} VM;
```
然后初始化它们:

```c
VM* newVM() {
  /* Previous stuff... */

  vm->numObjects = 0;
  vm->maxObjects = INITIAL_GC_THRESHOLD;
  return vm;
}
```
INITIAL_GC_THRESHOLD将是启动第一个GC时的对象数量。更小的数字在内存方面更保守，更大的数字在垃圾收集上花费的时间更少。调整口味。

每当我们创建一个对象，我们增加numObjects和运行一个集合，如果它达到最大:

```c
Object* newObject(VM* vm, ObjectType type) {
  if (vm->numObjects == vm->maxObjects) gc(vm);

  /* Create object... */

  vm->numObjects++;
  return object;
}
```

我不会显示它，但我们也会微调sweep()来递减numObjects每次它释放一个。最后，修改gc()来更新max:

```c
void gc(VM* vm) {
  int numObjects = vm->numObjects;

  markAll(vm);
  sweep(vm);

  vm->maxObjects = vm->numObjects * 2;
}
```

在每次收集之后，我们根据收集之后剩下的活动对象的数量更新maxObjects。这里的乘数允许堆随着活动对象数量的增加而增长。同样，如果释放了一堆对象，它也会自动收缩。

# 简单的
你成功了!如果您遵循了所有这些，那么您现在已经掌握了一个简单的垃圾收集算法。如果你想看到所有这些，这里是完整的代码。让我在这里强调，虽然这个收集器很简单，但它不是一个玩具。

在此基础上可以进行大量的优化(在GC和编程语言中，优化占了90%)，但是这里的核心代码是真正的GC。它与Ruby和Lua中直到最近才出现的收集器非常相似。您可以发布使用与此完全相同的代码的产品代码。现在去建造一些很棒的东西吧!


[原文](http://journal.stuffwithstuff.com/2013/12/08/babys-first-garbage-collector/ "原文")

