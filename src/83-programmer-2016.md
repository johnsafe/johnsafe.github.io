## Google AlphaGo 技术解读——MCTS+DCNN

文/李理

>2016年1月，Google DeepMind 的文章在《自然》杂志发表，AlphaGo 击败欧洲围棋冠军樊麾并且将要在3月份挑战李世石。围棋和深度学习再次大规模高热度地出现在世人眼前。

围棋的估值函数极不平滑，差一个子局面就可能大不相同，同时状态空间大，也没有全局的结构。这使得计算机只能采用穷举法并且因此进展缓慢。Google 的 AlphaGo 通过 MCTS 和深度学习的结合实现了围棋人工智能的突破，已经击败人类职业选手。本文分析这两种技术在围棋 AI 的应用以及 AlphaGo 的创新。

### MCTS

#### MCTS（Monte Carlo Tree Search）和 UCT(Upper Confidence Bound for Trees)

MCTS 算法就是从根节点（当前待评估局面）开始不断构建搜索树的过程。具体可以分成4个步骤，如图1所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56e7781476fa5.jpg" alt="图1 from A Survey of Monte Carlo Tree Search Methods" title="图1 from A Survey of Monte Carlo Tree Search Methods" />

图1 from A Survey of Monte Carlo Tree Search Methods

- Selection

使用一种 Policy 从根节点开始，选择一个最“紧急”（urgent）的需要展开（expand）可展开（expandable）的节点。可展开的节点是非叶子节点（非游戏结束局面），而且至少还有一个孩子没有搜索过。比如图1展示的选择过程，最终选择到的节点不一定是叶子节点（只要它还有未搜索过的孩子就行）。

- Expansion

选中节点的一个或者多个孩子展开并加入搜索树，图1的 Expansion 部分展示了展开一个孩子并加入搜索树的过程。

- Simulation

从这个展开的孩子开始模拟对局到结束，模拟使用的策略叫 Default Policy。

- 游戏结束有个得分（胜负），这个得分从展开的孩子往上回溯到根节点，更新这些节点的统计量，这些统计量会影响下一次迭代算法的 Selection 和 Expansion。

经过足够多次迭代之后（或者时间用完），我们根据某种策略选择根节点最好的一个孩子（比如访问次数最多，得分最高等）。

上面的算法有两个 Policy：Tree Policy 和 Default Policy。Tree Policy 决定怎么选择节点以及展开；而 Default Policy 用于 Simulation（也有文献叫 playout，就是玩下去直到游戏结束）。

MCTS 着重搜索更 urgent 的孩子，当然偶尔也要看看那些不那么 promising 的孩子，说不定是潜力股。这其实就是 exploit 和 explore 平衡的问题。另外，MCTS 直接 Simulation 到对局结束，“回避”了局面评估的难题。

既然是 exploit 和 explore 的问题，把 UCB 用到 MCTS 就是 UCT 算法（MCTS 是一族算法的统称，不同的 Tree Policy 和 Default Policy 就是不同的 MCTS 算法）。

#### UCT 算法

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d659c9e67d5.png" alt="公式1" title="公式1" />

Selection 和 Expansion 的公式如上，和 UCB 很类似，只是多了一个常量 Cp，用来平衡 exploit 和 explore。

每个节点 v 都有两个值，N(v) 和 Q(v)，代表 v 访问过的次数（每次迭代都是从 root 到结束状态的一条 Path，这条 Path 上的每个点都被 visit 一次）和 v 的回报，如果这次 Simulation 己方获胜，则 Q(v)+1，否则 Q(v)-1。（1、3、5...层代表自己，2、4、6...代表对手，如果最终我方获胜，1、3、5都要加一而2、4、6都要减一）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d659e3be0bc.png" alt="公式2" title="公式2" />

具体的计算公式如上，每次选孩子时都用上面的公式计算得分。第一项就是这个节点的平均回报（exploit term）和第二项就是 explore term，访问次数少的节点更应该被 explore。当 N(v)=0 时第二项值为无穷大，所以如果某个节点有未展开的孩子，总是优先展开，然后才可能重复展开回报高的孩子。

#### UCT 算法的特点

- Aheuristic

不需要估值函数，因此也就不需要各种启发式规则、领域知识，从而“回避”了围棋估值函数难的问题。

- Anytime

可以任何时候结束，迭代的次数越多就越准确

- Asymmetric

和人类的搜索类似，搜索树是不对称的，不是固定层次的，而是“重点”搜索 promising 的子树。

#### UCT 在围棋领域的变种和改进

All Moves As First（AMAF） 和 Rapid Action 

Value Estimation（RAVE）

这是很多开源 MCTS 围棋使用的一个变种。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d65a29c5157.png" alt="图2 AMAF" title="图2 AMAF" />

图2 AMAF

如图2我们首先通过 UCT 的 Tree Policy 选择了 C2 和 A1，然后 Default Policy 选择了 B1、A3、C3，最终黑棋获胜。

普通的 MCTS 会更新 C2 和 A1 的 N(v) 和 Q(v)，而 AMAF 认为：既然在 Simulation 的过程中黑方选择了 B1 和 C3，在 root 节点时也可以选择 B1 和 C3，那么这次模拟其实也可以认为 B1 和 C3 对获胜是有帮助的，root节点的 B1 和 C3 也有贡献（标志为*），也应该更新统计量，让下次选择它的概率大一点。同理白棋在 simulation 的时候选择了 A3，在 C2 也可以选择 A3 （有*的那个），那么 C2对于白棋失败也是有责任的，那么下次在 C2 的时候白棋应该避免选择 A3。这样一次迭代就能多更新一些节点（那些没有直接 Selection 的也有一些统计量）。

这个想法对于围棋似乎有些道理（因为不管哪个顺序很可能导致一样的局面，前提是没有吃子），而且实际效果也不错。但是在别的地方是否可以使用就需要看情况了。

这里有两类统计量：直接 selection 的节点统计量(A1,C2)和间接 selection 的节点(B1,C3，A3)。这两种权重应该是不一样的。

所以比较直观的想法是给它们不同的权重 αA+(1-α)U，这就是 α-AMAF。

这个权重 α 是固定的，RAVE 认为随着模拟次数的增加 α 应该减少才对（没有真的模拟是可以用这些间接的统计量，如果有了真的更准确的估计，这些间接的统计量就应该降低权重，这个想法也很自然）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d65a52d3291.png" alt="公式3" title="公式3" />

RAVE 使用上面的公式动态调整α，随着 v(n) 的增加，α 的值不断下降。

**Simulation 的改进**

默认的 Default Policy 随机的选择走法模拟到结束，这在没有任何先验知识的时候是可以的。但是人类在“探索”未知局面时不是随机“探索”的，也是用一个估值函数指导的，去探索那些更 promising 的局面。

具体方法很多，比如 Contextual Monte Carlo  Search，

Fill the Board 以及各种基于 Pattern 的方法。细节就不再展开讨论了。

**MCTS 的并行搜索**

**Leaf Parallelisation**

最简单的是 Leaf Parallelisation，一个叶子用多个线程进行多次 Simulation，完全不改变之前的算法，把原来一次 Simulation 的统计量用多次来代替，这样理论上应该准确不少。但这种并行的问题是需要等待最慢的那个结束才能更新统计量，而且搜索的路径数没有增多。

**Root Parallelisation**

多个线程各自搜索各自的 UCT 树，最后投票。

**Tree Parallelisation**

这是真正的并行搜索，用多个线程同时搜索 UCT 树。当然统计量的更新需要考虑多线程的问题，比如要加锁。

另外一个问题就是多个线程很可能同时走一样的路径（因为大家都选择目前看起来 Promising 的孩子），一种方法就是临时的修改 virtual loss，比如线程1在搜索孩子a，那么就给它的 Q(v) 减一个很大的数，这样其它线程就不太可能选择它了。当然线程1搜索完了之后要记得改回来。

《A Lock-free Multithreaded Monte-Carlo Tree Search Algorithm》使用了一种 lock-free 的算法，这种方法比加锁的方法要快很多，AlphaGo 也用了这个方法。

Segal 研究了为什么多机的MCTS算法很难，并且实验得出结论使用virtual loss 的多线程版本能比较完美地 scale 到64个线程（当然这是单机一个进程的多线程程序）。这也将是 AlphaGo 的 scalability 的瓶颈。

使用了 UCT 算法之后，计算机围棋能提高到 KGS 2d 的水平。

### CNN 和 Move Prediction

MCTS 回避了局面估值的问题，对于不同局面的细微区别，机器能否不需要领域的专家手工编写特征或者规则，自动学习来实现估值函数呢？目前看来，深度学习解决 feature 的自动学习是最 promising 的方法之一。由 Yann LeCun 提出用来解决图像识别问题的 CNN （卷积神经网络），通过卷积来发现位置无关的 feature，而且这些 feature 的参数是相同的，从而与全连接的神经网络相比大大减少了参数的数量。因此，CNN 非常适合围棋这种 feature 很难提取的问题，用 DCNN （多层次的 CNN 模型）来尝试围棋的局面评估似乎也是很自然的想法。

围棋搜索如果不到游戏结束，深的局面并不比浅的容易评估，所以不需要展开搜索树，可以直接评估一个局面下不同走法的好坏。这样做的好处是很容易获得训练数据。我们有大量人类围棋高手的对局（海量中等水平的对局），每一个局面下“好”的走法直接就能够从高手对局库里得到。如果能知道一个局面下最好的走法（或者几个走法），那么对弈时就直接可以选择这个走法。

#### 田渊栋和朱彦的工作（2015）

和 Google DeepMind 进行围棋人工智能竞赛的主要就是 Facebook  田渊栋团队了。Google 文章在《自然》发表的前一天，他们在 arxiv 上发表了自己的工作（《Better Computer Go Player with Neural Network and Long-Term Prediction》）。

他们使用的 feature：

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d65abaadd03.png" alt="图feature" title="图feature" />

除了使用之前工作的标准 feature 之外，他们增加了一些 feature，比如是否边界、距离中心的远近、是否靠近自己与对手的领土（不清楚怎么定义领土的归属）。此外对于之前的 feature 也进行了压缩，之前都把特征分成黑棋或者白棋，现在直接变成己方和对手，这样把模型从两个变成了一个（之前需要给黑棋和白棋分别训练一个模型）。此外的一个不同地方就是类似于 Multi-task 的 learning，同时预测未来3步棋的走法（而不是1步棋走法）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d65b2f4f8e3.png" alt="图feature 2" title="图feature 2" />

为了与前人的工作比较，这里只用了标准的 features，比较的也是预测未来1步棋的准确率，可以发现这个方法还是有效的。

只使用 DCNN 的围棋软件（不用 MCTS 搜索）

darkforest: 标准的 feature，一步的预测，使用 KGS 数据。

darkforest1：扩展的 feature，三步预测，使用 GoGoD 数据。

darkforest2：基于 darkforest1，fine-tuning 了一下参数。

把它们放到 KGS 上比赛，darkforest 能到 1k-1d 的水平，darkforest1 能到 2d 的水平，darkforest2 能到 3d 的水平（注：KGS 的 3d 应该到不了实际的业余3段），下面是具体的情况。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d65b530a502.png" alt="图KGS" title="图KGS" />

因此作者认为加入3步预测的训练是有效的。

#### MCTS+DCNN

Tree Policy：走法首先通过 DCNN 排序，然后按顺序选择，除非累计的概率超过0.8或者超过一定次数的 top 走法。Expansion 使用的 UCT 算法。

Default Policy：参考的 Pachi 的 tree policy，有3*3的 pattern，对手打吃的点（opponent atari point），点眼的检测（detection of nakade points）等。

这个版本的软件叫 darkforest3，在 KGS 上能到 5d 的水平。

#### 弱点

- DCNN 预测的 top3/5 的走法可能不包含局部战役的一个关键点，所以它的局部作战能力还比较弱。

- 对于一些打劫点，即使没用，DCNN 还是会给高分。

- 当局面不好的情况下，它会越走越差（这是 MCTS 的弱点，因为没有好的走法，模拟出来都是输棋，一些比较顽强的抵抗走不出来）。

从上面的分析可以看出：DCNN 给出的走法大局观还是不错的，这正是传统方法很难解决的问题。局部的作战更多靠计算，MCTS 会有帮助。但我个人觉得 MCTS 搜索到结束，没有必要。一个局部的计算也许可以用传统的 alpha-beta 搜索来解决，比如征子的计算，要看6线有没有对手的棋子，另外即使有对手的棋子，也要看位置的高低，DCNN 是没法解决的，需要靠计算。

#### AlphaGo 的创新

**Policy Network & Value Network**

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d65ba498c02.png" alt="图3 AlphaGo的特点" title="图3 AlphaGo的特点" />

图3 AlphaGo 的特点

图3是 AlphaGo 所使用的两个网络以及训练过程，包括 Policy Network 和 Value Network。

Policy Network 的作用是 Tree Policy 时的 Node Selection。（rollout 阶段不能使用 Policy Network，因为 DCNN 的计算速度相对于 Simulation 来说太慢，所以 AlphaGo 又训练了一个简单的 Rollout Policy，它基于一些 local 的 pattern 之类的 feature 训练了一个线性的 softmax）。

Value Network 的作用是解决之前说的很多工作都回避的局面评估问题，AlphaGo 使用深度强化学习（Deep Reinforcment Learning）学习出来局面的估值函数，而不是像象棋程序那样是人工提取的 feature 甚至手工调整权重。MCTS 如果盲目搜索（使用随机的 Default Policy 去 rollout/playout）肯定不好，使用各种领域知识来缩小 rollout 的范围就非常重要。

#### SL Policy Network & Rollout Policy 的训练

AlphaGo 相比之前多了 Rollout Policy，之前的 Rollout Policy 大多是使用手工编制的 pattern，而 AlphaGo 用和训练 Policy Network 相同的数据训练了一个简单的模型来做 rollout。

训练数据来自3千万的 KGS 的数据，使用了13层的 CNN，预测准确率是57%，这和之前田渊栋等人的工作是差不多的。

#### RL Policy Network & Value Network 的训练

之前训练的 SL Policy Network 优化的目标是预测走法，作者认为人类会在很多 promising 的走法里选择，这不一定能提高 AlphaGo 的下棋水平。这可能是一个局面（尤其是优势）的情况下有很多走法，有保守一点但是保证能赢一点点的走法，也有激进但需要算度准确的但能赢很多的走法。这取决于个人的能力（比如官子能力怎么样）和当时的情况（包括时间是否宽裕等等）。

所以 AlphaGo 使用强化学习通过自己跟自己对弈来调整参数，学习更适合自己的 Policy。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d65bf3dc8fa.png" alt="公式4" title="公式4" />

具体的做法是当前版本跟之前的某一个版本（把之前所有版本都保留，不用最近的一个版本可以避免 overfitting）对弈，对弈的走法是根据 Policy Network 来选择的，然后根据结果调整参数。这个公式用自然语言来描述就是最终得分 z_t （获胜或者失败），在 t 时刻局面是 s_t 选择了走法 a_t，P(a_t|s_t)表示局面 s_t 时选择走法 a_t 的概率，就像神经网络的反向传播算法一样，损失 z_t (或者收益)是要由这个走法来负责的。我们调整参数的目的就是让这个概率变小。再通俗一点说就是，比如第一步我们的模型说必须走马（概率是1），如果最终输棋，我们复盘时可能会觉得下次走马的概率应该少一点，所以我们调整参数让走马的概率小一点（就是这个梯度）。

RL Policy Network 的初始参数就是 SL Policy Network 的参数。最后学到的 RL Policy Network 与 SL Policy Network 对弈，胜率超过80%。

另外 RL Policy Network 与开源的 Pachi 对弈（这个能到 2d，也就是业余两段的水平），Pachi 每步做100,000次 Simulation，RL Policy Network 的胜率超过85%，这说明不用搜索只用Move Prediction 能超过 2d 的水平。这和田渊栋等人的工作结论一致，他们的 darkforest2 也只用 Move Prediction 在 KGS 上也能到 3d 的水平。

#### Value Network 的强化学习训练

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d65c25af6ac.png" alt="公式5" title="公式5" />

一个局面在 Policy P 下的估值公式。用通俗的话说就是：在 t 时刻的局面是 s，然后我们用 P 来下棋直到游戏结束，我们重复很多次，然后求平均的得分。当然，最理想的情况是我们能知道双方都是最优策略下的得分，可惜我们并不知道，所以只能用我们之前学到的 SL Policy Network 或者 RL Policy Network 来估计一个局面的得分，然后训练一个 Value Network V(s)。前面我们也讨论过了，RL Policy Network 胜率更高，而我们学出来的 Value Network 是用于 rollout 阶段作为先验概率的，所以 AlphaGo 使用了 RL Policy Network 的局面评估来训练 V(s)。

V(s) 的输入是一个局面，输出是一个局面的好坏得分，这是一个回归问题。AlphaGo 使用了和 Policy Network 相同的参数，不过输出是一个值而不是361个值（用 Softmax 归一化成概率）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d65c4816ba5.png" alt="公式6" title="公式6" />

上面的公式说明：V(s) 的参数 theta 就是简单地用梯度下降来训练。

不过用一盘对局的所有(s,v(s))训练是有问题的，因为同一盘对局的相邻的局面是非常相关的，相邻的局面只差一个棋子，所以非常容易 overfitting，导致模型“记住”了局面而不是学习到重要的 feature。作者用这样的数据训练了一个模型，在训练数据上的 MSE 只有0.19，而在测试数据上是0.37，这明显 overfitting 了。为了解决这个问题，作者用 RL Policy Network 跟自己对局了3千万次，然后每个对局随机选择一个局面，这样得到的模型在训练数据和测试数据上的MSE是0.226和0.234，从而解决了 overfitting 的问题。

#### MCTS + Policy & Value Networks

上面花了大力气训练了 SL Policy Network、Rollout Policy 和 Value Network，那么怎么把它们融合到MCTS中呢？

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d65c738ad5d.png" alt="图4 一次MCTS的Simulation" title="图4 一次MCTS的Simulation" />

图4 一次 MCTS 的 Simulation

一次 MCTS 的 Simulation 可以用图4来说明，下文加黑的地方是这三个模型被用到的地方。

首先每个节点表示一个局面，每一条边表示局面+一个合法的走法(s,a)。每条边保存 Q(s,a)，表示 MCTS 当前累计的 reward，N(s,a) 表示这条边的访问次数，P(s,a) 表示先验概率。

**Selection**

每次 Simulation 使用如下的公式从根节点开始一直选择边直到叶子节点（也就是这条边对应的局面还没有 expand）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d65cd91b982.png" alt="公式7" title="公式7" />

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d6622617d3a.jpg" alt="公式8-1" title="公式8-1" />

Q(s_t,a) 就是 exploit term，而 u(s_t,a) 就是 exploreterm，而且是于先验概率 P(s,a) 相关的，优先探索 SL Policy Network 认为好的走法。

**Evaluation**

对于叶子节点，AlphaGo 不仅使用 Rollout （使用 Rollout Policy）计算得分，而且也使用 Value Network 打分，最终把两个分数融合起来：

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d65d3330741.png" alt="公式9" title="公式9" />

### Backup

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d65d5972269.png" alt="公式10" title="公式10" />

n 次 Simulation 之后更新统计量（从而影响Selection），为什么是 n 次，这涉及到多线程并行搜索以及运行与 GPU 的 Policy Network与 Value Network 与 CPU 主搜索线程通信的问题。

**Expansion**

一个边的访问次数超过一定阈值后展开这个边对应的下一个局面。阈值会动态调整以是的 CPU 和 GPU 的速度能匹配。

####  AlphaGo 的水平

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d65d86371cb.png" alt="图5 AlphaGo的水平" title="图5 AlphaGo的水平" />

图5 AlphaGo 的水平

图5a 是用分布式的 AlphaGo，单机版的 AlphaGo 和 CrazyStone 等主流围棋软件进行比赛，然后使用 Elo Rating 打分。

作者认为 AlphaGo 的水平超过了樊麾（2p)，因此 AlphaGo 的水平应该达到了 2p （不过很多人认为目前樊麾的水平可能到不了 2p）

图5b 说明了 Policy Network Value Network 和 Rollout 的作用，做了一些实验，去掉一些情况下棋力的变化，结论当然是三个都很重要。

图5c 说明了搜索线程数以及分布式搜索对棋力的提升。

### AlphaGo 的实现细节

#### Search Algorithm 

和之前类似，搜索树的每个状态是 s，它包含了所有合法走法(s,a)，每条边包含如下的一些统计量：

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d65dc46662a.png" alt="公式11" title="公式11" />

P(s,a) 是局面 s 下走 a 的先验概率。Wv(s,a) 是 Simulation 时 Value Network 的打分，Wr(s,a) 是 Simulation 时 rollout 的打分。Nv(s,a) 和 Nr(s,a) 分别是 Simulation 时 Value Network 和 rollout 经过边(s,a)的次数。Q(s,a) 是最终融合了 Value Network 打分和 rollout 打分的最终得分。

AlphaGo 没有像之前的工作那样，除了对称的问题，对于 APV-MCTS（Asynchronous Policy and Value MCTS）算法，每次经过一个需要 rollout的(s,a) 时，会随机选择8个对称方向中的一个，然后计算 P(a|s)，因此需要平均这些 value。计算 Policy Network 的机器会缓存这些值，所以 Nv(s,a) 应该小于等于8。

**Selection(图4a)**

从根节点开始使用下面的公式选择 a 直到叶子节点。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d65df1a44dd.png" alt="公式12" title="公式12" />

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d65e01c7caa.png" alt="公式13" title="公式13" />

Q(s,a) 初始值为0，后面 Backup 部分会讲怎么更新 Q(s,a)。

现在我们先看这个公式，第一部分 Q(s,a) 是 exploit term，第二部分是 explore term。这个公式开始会同时考虑 value 高的和探索次数少的走法，但随着 N(s,a) 的增加而更倾向于 value 高的走法。

**Evaluation(图4c)**

叶子节点 sL 被加到一个队列中等到 Value Network 计算得分（异步的），然后从 sL 开始使用 Rollout Policy 模拟对局到游戏结束。

**Backup(图4d)**

在 Simulation 开始之前，把从根一直到 sL 的所有的(s,a)增加 virtual loss，这样可以防止（准确说应该是尽量不要，原文用的词语是 discourage，当然如果其它走法也都有线程在模拟，那也是可以的）其它搜索线程探索相同的路径。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d65e375a410.png" alt="公式14" title="公式14" />

给(s,a)增加 virtual loss，那么根据上面选择的公式，就不太会选中它了。

当模拟结束了，需要把这个 virtual loss 去掉，同时加上这次 Simulation 的得分。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d65edac7191.jpg" alt="公式15" title="公式15" />

此外，当 GPU 算完 value 的得分后也要更新：最终算出 Q(s,a):

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d65f00c7ee2.png" alt="公式16" title="公式16" />

**Expansion(图4b)**

当一条边(s,a)的访问次数 Nr(s,a) 超过一个阈值 Nthr 时会把这条边的局面（其实就是走一下这个走法） s'=f(s,a) 加到搜索树里。

初始化统计量：Nv(s',a)=0，Nr(s',a)=0，Wv(s',a)=0，Wr(s',a)=0，P(s',a)=P(a|s')

由于计算 P(a|s') 需要在 GPU 中利用 SL Policy Network 计算，比较慢，所以先给它一个 place-holder 的值，等到 GPU 那边算完了再更新。

这个 place-holder 的值是使用和 Rollout Policy 类似的一个 Tree Policy 计算出来的（用的模型与 rollout policy 一样，不过特征稍微丰富一些，后面会在表格中看到），在 GPU 算出真的 P(a|s') 之前的 Selection 都是先用这个 place-holder 值，所以也不能估计的太差。因此 AlphaGo 用了一个比 rollout feature 多一些的模型。

Expansion 的阈值 Nthr 会动态调整，目的是使得计算 Policy Network 的 GPU 能够跟上 CPU 的速度。

#### Distributed APV-MCTS 算法

一台 Master 机器执行主搜索（搜索树的部分），一个 CPU 集群进行 rollout 的异步计算，一个 GPU 集群进行 Policy 和 Value Network 的异步计算。

整个搜索树都存在 Master 上，它只负责 Selection 和 Place-Holder 先验的计算以及各种统计量的更新。叶子节点发到 CPU 集群进行 rollout 计算，发到 GPU 集群进行 Policy 和 Value Network 的计算。

最终，AlphaGo 选择访问次数最多的走法而不是得分最高的，因为后者对野点(outlier)比较敏感。走完一步之后，之前搜索过的这部分的子树的统计量直接用到下一轮的搜索中，不属于这步走法的子树直接扔掉。另外 AlphaGo 也实现了 Ponder，也就是对手在思考的时候它也进行思考。它思考选择的走法是比较“可疑”的点——最大访问次数不是最高得分的走法。AlphaGo 的时间控制会把思考时间尽量留在中局，此外 AlphaGo 也会投降——当它发现赢的概率低于10%，也就是 MAXaQ(s,a) < -0.8。

#### Rollout Policy

使用了两大类 pattern，一种是 response 的 pattern，也就是上一步走法附近的 pattern（一般围棋很多走法都是为了“应付”对手的走子）；另一种就是非 response 的 pattern，也就是将要走的那个走法附近的 pattern。具体使用的特征见下表。Rollout Policy 比较简单，每个 CPU 线程每秒可以从空的局面（开局）模拟1000个对局。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d65f683a7b9.png" alt="图pattern" title="图pattern" />

横线之上的 feature 用来 rollout，所有的 feature 用来计算 place-holder 先验概率。

#### Symmetries

前面在讲 Search Algorithm 部分时讲过了。

#### SL Policy Network

SL Policy Network 使用了2940万局面来训练，这些局面来自 KGS 6d-9d 的16万个对局。使用了前100万用来测试，后面的2840万用来训练。此外进行了旋转和镜像，把一个局面变成8个局面。使用随机梯度下降算法训练，训练的 mini-batch 大小是16。使用了50个 GPU 的 DistBelief （并没有使用最新的 TensorFlow），花了3周的时间来训练了3.4亿次训练步骤，估计每个 mini-batch 算一个步骤。

#### RL Policy Network

每次用并行的进行 n 个游戏，使用当前版本（参数）的 Policy Network 和之前的某一个版本的 Policy Network。当前版本的初始值来自 SL Policy Network。然后用 Policy Gradient 来更新参数，这算一次迭代，经过500次迭代之后，就认为得到一个新的版本把它加到 Pool 里用来和当前版本对弈。使用这种方法训练，使用50个 GPU，n=128，10,000次对弈，一天可以训练完成 RL Policy Network。

#### Value Network

前面说了，训练的关键是要自己模拟对弈然后随机选择局面，而不是直接使用 KGS 的对局库来避免 overfitting。

AlphaGo 生成了3千万局面，也就是3千万次模拟对弈，模拟的方法如下：

- 随机选择一个 time-step U~unif{1,450}

- 根据SL Policy Network 走1，2，... ，U-1步棋

- 然后第 U 步棋从合法的走法中随机选择

- 然后用 RL Policy Network 模拟对弈到游戏结束

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d66214d12b1.jpg" alt="公式17-1" title="公式17-1" />

被作为一个训练数据加到训练集合里这个数据是

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d65fe61ceb9.png" alt="公式18" title="公式18" />

的一个无偏估计。

最后这个 Value Network 使用了50个 GPU 训练了一周，使用的 mini-batch 大小是32。

####  Policy/Value Network 使用的 Features

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d6603d94e98.png" alt="图Features" title="图Features" />

其实和前面田渊栋的差不太多，多了两个征子相关的 feature，另外增加了一个常量1和常量0的 plane。

最后一个 feature 是 Value Network 用的，因为判断局面得分时要知道是谁走的，这个很关键。

#### 神经网络结构

**Policy Network**

13层从 CNN，输入时19*19*48，第一个 hidden 层把输入用零把输入 padding 成23*23，然后用 k 个5*5的 filter，stride 是1。

2到12层首先用零把输入 padding 成21*21，然后使用 k 个5*5的 filter，stride 依然是1。

最后一层用一个1*1的 filter，然后用一个 Softmax。

比赛用的 k=192，文章也做了一些实验对比 k=128,256,384的情况。

**Value Network**

14层的 CNN，前面12层和 Policy Network 一样，第13层是一个filter 的卷积层，第14层是全连接的 ReLU 激活，然后输出层是全连接的 tanh 单元。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d66095d5bf4.png" alt="图Elo rating" title="图Elo rating" />

上表为不同分布式版本的水平比较，使用的是 Elo rating 标准。

### 总结

神经网络的训练其实用的时间和机器不多，真正费资源的还是在搜索阶段。最强的 AlphaGo 使用了64个搜索线程，1920个 CPU 的集群和280个 GPU 的机器（二十多台机器）。

MCTS 很难在多机上并行，所以AlphaGo还是在一台机器上实现 LockFree 的多线程并行，不过 rollout 和神经网络计算是在 CPU 和 GPU 集群上进行的。所以分布式 MCTS 的搜索是最大的瓶颈。如果这方面能突破，把机器堆到成百上千台应该能提高不少棋力，但估计该架构到3月与李世石对弈时还很难有突破。

可以增强的是 RL Policy 的自对弈学习，不过这个提升也有限，否则不会只训练一天就停止了，估计也收敛差不多了。

所以，个人认为，3月份 AlphaGo 要战胜李世石的难度还比较大。