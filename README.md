# Concave Intersection
这个开源库基于Unity开发，主要实现了凹多边形之间的相交判断，当然也可以应用于顶点与凹凸包、线段与凹凸包、凸包与凸包之间的相交判断，也就是说，这个方案基本可以解决前面两则文章中的所有多边形判断，效率也非常高，复杂度几乎接近O(n)，下面解释原理。

一、扫线法概念
扫线法其实是一种常用的平面空间算法思路，在很多算法中均有应用，例如我的文章Voronoi图和扫线法中就是应用扫线法来实现Voroni图的计算，从而实现三角剖分。扫线法的典型作用是降维，将原本的二维空间数据，转化为一位空间数据，这样可以大大提升计算效率。



如上图，假设存在多边形ABCDEFG，我们在执行运算时，依照从左至右，从下到上的规则，好像有一条扫描线推遍整个相关平面区域，这样的计算方法，就称为扫线法。
所谓的从左到右不难理解，图中的每个顶点都被从左至右排序，而通过这些顶点的垂线将平面划分成了若干区域，这些被划分的区域我们可以称为不同的扫描阶段，因此以上共有A、G、B、F、D、C、E七个阶段。
那么，从下到上代表什么意思？先看下图：


在每一个扫描阶段，我们都会使用一个区域栈来存储当前阶段范围的空间区域划分情况，如上图中的A阶段，此阶段所能代表的空间区域位于经过A点的垂线与经过G点的垂线之间，此部分空间区域被划分又为三块，AB以下的①号区域，AG与AB之间的②号区域，AG以上的③号区域，我们在执行判断时，一般跟从从下到上的规则进行判断，这里就是依次扫过①、②、③号区域。

二、提出目标问题
首先，给出一个问题，如何判断顶点是否包含于简单的凹多边形中？
这里也可以使用前文所述的射线法来解决，不过我们这里介绍扫线法的使用，因为这在后续的凹包相交判断计算中更有价值。
思考：如上所述，我们可以使用扫线的形式划分整个多边形空间，那只要判断顶点在其中的某个空间内，即可认为包含于凹多边形中。
接下来，我们只须完成两件事：
1、分阶段扫线，构建空间划分的区域栈
2、在区域栈中判断是否遭遇了目标顶点

三、基本结构定义
在深入了解扫描规则之前，我们首先需要一些前置条件：

1、我们需要定义Point结构，以及Region结构
2、我们需要按照逆时针顺序存储凹多边形中的顶点，称为顶点列表(point list)
3、我们需要在每个顶点结构保存邻接顶点的引用，构成双向链表
4、由于表示的区域有内外之分，因此定义enum Inout
5、需要一个有序列表（SortedList），用于水平方向排序被扫描顶点，称为扫描列表(sweep list)
6、需要一个区域栈(RegionList)，虽然类似栈结构，但是不需要应用先进后出的规则，只需用List存储即可
大概主要结构单元如下：

---

public class Point2
{
    public float x;
    public float y;
    public Point2 m_left;
    public Point2 m_right;
}
public enum InOut
{
        In,
        Out,
}
public class SlopeRegion
{
        public Point2 m_start;    //起点
        public Point2 m_end;      //终点
        public float m_slope;     //斜率
        public float m_swY;       //扫描线Y坐标
        public InOut m_inOut;     //区域内外
}

---

四、事件定义与扫描规则
为了更方便执行扫描过程中的处理，我们可以参考Voronoi图和扫线法中的做法，将遭遇的不同类型的扫描结果使用不同的事件来表示，这里我们可以使用三种类型的事件：
1、OpenRegion（开启区域）
在扫描到某个顶点时，发现类似A、D点情况，即它的左右邻接点都位于当前点右侧，则说明遭遇一块界定区域的开启。
2、CloseRegion（关闭区域）
当遭遇类似C、E点的情况，即它的左右邻接点均位于当前点左侧，则说明遭遇了一块界定区域的关闭。
3、TurnRegion（转换区域）
当遭遇类似G、F、B的情况，即它的左右邻接点一左一右分布于当前点两侧，则说明遭遇了一块界定区域的转换。




接下来，我们描述上图的每个扫线阶段时，我们预期的区域栈的变化。
A阶段：为了达成前述的①、②、③号区域划分，因此在区域栈中，我们应该预期依次出现[AB、AG]单元结构，两个区域条目可以将垂直空间划分为三部分。相当于在遭遇开启区域时，需要向栈中插入两个区域单元，各自对应左右邻接点。邻接点靠上的区域单元存放在上方。
G阶段：看图中，形成了三个区域，应该预期出现[GF、AB]结构，也就是说，在遭遇转换区域时，需要删除X方向上靠左的邻接点对应的栈单元AG，再加入靠右的邻接点与当前点构成的栈单元。
B阶段：与G阶段类似，删除AB，插入BC，此时栈结构变成[GF、BC]
F阶段：与G阶段类似，删除GF，插入FE，此时栈结构变成[FE、BC]
D阶段：与A阶段类似，插入DE、DC，此时栈结构变成[FE、DE、DC、BC]
C阶段：此时遭遇关闭区域，代表DC、BC所能界定的区域的终结，因此删除DC、BC单元，栈结构变成[FE、DE]
E阶段:与C阶段类似，删除终结区域FE、DE，栈结构清除，扫描完毕

总结一下扫线过程中遭遇三种类型的事件处理：
1、OpenRegion（开启区域）
向栈中插入两个区域单元，各自对应左右邻接点。邻接点靠上的区域单元存放在上方。
2、CloseRegion（关闭区域）
删除以左右两个邻接点开始的栈单元，也即以当前顶点结束的栈单元。
3、TurnRegion（转换区域）
在遭遇转换区域时，需要删除X方向上靠左的邻接点对应的栈单元，再加入靠右的邻接点与当前点构成的栈单元。

五、顶点与凹多边形相交判定规则
至此，我们可以通过每个扫描阶段，完成空间区域的分割，为了达成最终的判定，我们还需要附加2条附加规则。
1、InOut规则：
如果我们遭遇线段时，是以逆时针顺序分别遭遇两个顶点，那么所界定的区域表示外部区域，反则反之。
如A阶段遭遇了AB，即是逆时针遭遇，所以AB以下的区域代表多边形的外部区域，而同时遭遇的AG，由于是顺时针遭遇，其所代表的区域是内部区域。
2、底部优先规则：
判断一个顶点是否在区域内时，依照从下到上的顺序进行判断，只要被包含在某个区域内部，就表示属于当前区域，比对当前区域的InOut参数表示即可完成判断。如果不属于任意一个区域，则默认属于外部。
至此，我们已经可以完成顶点与凹多边形相交关系的判断。需要额外说明的是，在开始的排序过程中，我们可以将目标顶点与凹包顶点一起纳入扫描列表执行排序，而在每次扫描时先行判断是否是目标顶点，不是，则持续维护扫线堆栈，是则使用当前区域堆栈执行区域判断，完成算法过程。


六、凹多边形之间相交判断
在可以扫线判断顶点的情况下，已经非常接近最后的结果，针对凹多边形之间的处理，我们稍微改变一下规则，即：
1、在开始的排序过程中，我们将凹包C1与凹包C2的顶点一起纳入扫描列表执行排序
2、在每个扫线阶段，如果遭遇的是C1中的顶点，则会将其与C2当前的扫线栈执行区域判断，如果在其中，则完成算法退出，如果不在，则将顶点用作C1堆栈的维护，如果遭遇C2的顶点，则会将其与C1当前的扫线栈执行区域判断，如果在其中，则完成算法退出，如果不在，则将顶点用作C2堆栈的维护。
