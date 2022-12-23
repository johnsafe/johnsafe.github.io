## 运用增强学习算法提升推荐效果

文/单艺

>关于推荐系统，业界关注较多的是经典的推荐算法（如协同推荐、矩阵分解、排序学习等）和架构。在实际应用中，用户、事物和环境时刻变化，会使得原有模型失效。而增强学习（Reinforcement Learning），作为机器学习中的一个分支，强调如何基于环境而行动，进而取得最大化的预期利益。它长于在线规划，在探索（在未知的领域）和利用（现有知识）之间找到平衡。对于上述挑战，它能够提供切实有效的方法。本文结合猎聘网的需求与实践，介绍运用 Multi-Armed Bandits 帮助解决推荐系统的冷启动、在线策略实验和调优等重要问题，并展望增强学习的更多运用及未来发展。

### 猎聘推荐匹配策略与挑战

先简单介绍一下猎聘网在推荐系统方面的应用背景。

猎聘网是一个求职招聘平台。人岗匹配推荐系统“伯乐”的功能包括给求职者推荐职位和给企业推荐人才，要求比较高的准确率。如图1所示，我们做了很多细小的匹配模型和策略，不仅从地区、行业、职能、社交等不同维度来把个人和企业的职位进行匹配，也用到了很多用户行为方面的数据，如个人的浏览行为和投递行为，企业 HR 的下载行为，以及表示满意或者不满意的行为。我们都把这些数据用到匹配的策略里面去，生成不同的结果，经过融合和过滤，分别推送给个人用户和企业。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a9874d80b0f.png" alt="图1 猎聘伯乐人岗匹配推荐系统" title="图1 猎聘伯乐人岗匹配推荐系统" />

图1 猎聘伯乐人岗匹配推荐系统

此外，猎聘还有职业社交推荐的功能，满足的是用户找职场人脉的重要需求。如图2所示，匹配策略除了用户的行业和职能，还根据用户提供的履历推算他的社交图谱，例如同事、同学关系，用来产生出候选集，再经过融合、过滤，最后得到推荐结果。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a9873a2ede1.png" alt="图2 猎聘同道人脉推荐系统" title="图2 猎聘同道人脉推荐系统" />

图2 猎聘同道人脉推荐系统

职业推荐工作的一个特点是每天都面对很多新事物：新的用户，新的职位、UI 也在不断优化改进，人的心理都在发生变化，要做不同模型或策略的实验。对新的事物一无所知的时候，我们需要去探索，收集更多的信息，但探索是有成本的。比如我们新上一个模型，如果它表现不好，对用户体验会形成一定的负面效果，这个时候我们的 KPI 就会受到影响。另外，如果我们知道在什么情况下用什么样的策略效果最好，我们就要尽量利用这个已知最优的策略。但是探索和利用是矛盾的，我们在一个时间点只能做一件事情，必须要进行取舍。

解决这个问题常用的方法是 A/B 测试。例如注册页面 A/B 测试有两个版本，通过实验测试，看哪个版本的效果更好。传统的做法，是平均切流量过去，或者切小部分流量过去进行测试，针对转化率不佳的部分，再想办法解决问题。这会带来一些不必要的潜在损失。

### 增强学习的应用思路

A/B 测试问题类似赌场老虎机——赌场中有很多的老虎机，你去试哪台机器可以赢钱，希望收益最大化。在这方面，研究人员做了很多的研究，最早的论文在1933年就出来了。经典的 Multi-Armed Bandits Problem（MAB），假设有 M 个相互独立的老虎机，每个老虎机 a 产生一个奖励值r是服从概率分布 P[r|a] 的， P[r|a] 是非平稳的，会随着时间变化。我们的目标是最大化累计奖励值（Reward），即尽可能多赢钱，同时也是最小化累计后悔值（Regret）。

Multi-Armed Bandits 是增强学习领域一个比较简单的问题，增强学习则是更大范围的问题。如图3所示，假设那只老鼠是智能的，它用它的经验和知识进行决策，做某个动作，也许得到一些奖励，也许什么都没有，也许被老鼠夹打一下。奖励值是能够量化的，聪明的老鼠能够在这个环境下自主去行动，去最优化自己的收获。收益里往往有一个“无风险利率”，它是大于0小于1的，所以尽可能让近期收益比较大会好一些。这就是增强学习的原理。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a9872a8d4f4.png" alt="图3 增强学习模型示例" title="图3 增强学习模型示例" />

图3 增强学习模型示例

回到老虎机问题，随机选择或者简单尝试之后，拍脑袋选择一个认为最“好”的，都是懒惰的想法。增强学习里面有一种很简单但有效的办法，是留一个相当小的比例去实验，这就是 ε-Greedy Algorithm。如图4所示，留很少一部分的流量（如1%-5%）去做实验，随机从所有的策略（N 个 Arm）中选择，看它的返回结果怎么样。然后基于多次实验产生的知识，在余下的大部分流量中选择已知的最佳策略。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a9871d11a3c.png" alt="图4 ε-Greedy算法" title="图4 ε-Greedy算法" />

图4 ε-Greedy 算法

εGreedy Algorithm 很简单，如果用 Python 来写也就十几行代码（Source: https://gist.github.com/sergeyf/63a37a5db18f8cebf962），里面用一个随机数产生器来选择是进行探索还是利用。

```
def epsilon_greedy(observed_data,epsilon=0.01):
   totals = observed_data.sum(1) # totals
   successes = observed_data[:,0] # successes
   estimated_means = successes/totals # the current means
   best_mean = np.argmax(estimated_means) # the best mean
   be_exporatory = np.random.rand() < epsilon # should we explore?
   if be_exporatory: # totally random, excluding the best_mean
     other_choice = np.random.randint(0,len(observed_data))
     while other_choice == best_mean:
       other_choice = np.random.randint(0,len(observed_data))
     return other_choice
   else: # take the best mean
     return best_mean
```

更高级的方法，就是用 Upper Confidence Bound（UCB）。如图5所示，针对各个 Arm 的回报样本进行统计，算出它的均值，再算出它的置信区间，用乐观的估计即 Upper Confidence Bound；图中用红线代表均值，盒子顶部是 Upper Confidence Bound，最终选 Upper Confidence Bound 最高的策略（Arm）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a987078a203.png" alt="图5 Upper Confidence Bound算法" title="图5 Upper Confidence Bound算法" />

图5 Upper Confidence Bound

Upper Confidence Bound算法的Python例子代码如下（Source: https://gist.github.com/sergeyf/63a37a5db18f8cebf962）：def UCB_normal(observed_data):

```
totals = observed_data.sum(1) # totals
   successes = observed_data[:,0] # successes
   estimated_means = successes/totals # sample mean
   estimated_variances = (estimated_means - estimated_means**2)/totals
   UCB = estimated_means + 1.96*np.sqrt(estimated_variances)
   return np.argmax(UCB)
```

这是一个用正态分布去逼近一个伯努利分布的算法，里面有一个常见参数1.96，即取95%的置信度。它总是选择当前最大的 Upper Bound。

还有一个 Thompson Sampling Algorithm，跟贝叶斯理论有关系的。其思路是奖励值是一个随机变量，服从一个分布，我们可以通过学习、探测知道它后验的分布，然后更新我们的模型、参数，每次根据最新的后验参数得出来的分布进行采样，最后也是选取一个收益最大的。即使是比较差的策略，奖励分布的采样结果也有可能出现高的值，所以可以保证一定程度的探索。图6的例子里面有三个臂（Arm），在实验启动后会越来越倾向于选用最佳的蓝线代表的 Arm，但它还是会在其他几个臂做探索，只不过探索的比例比较小。最后我们会看到各条线的方差越来越小，越来越确定。这是因为我们对它们了解越来越多。到了后来，大量情况下都会集中选最优的。当分布发生变化的时候它也能自适应地去改变，这也是 Thompson Sampling 的一个优点。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a986f822c05.png" alt="图6 Thompson Sampling算法" title="图6 Thompson Sampling算法" />

图6 Thompson Sampling 算法

如果假设 Reward 服从伯努利分布，Thompson Sampling 算法就是一行 Python 代码，是用伯努利分布的共轭分布——Beta 分布来实现。算法的实现真的很简单（Source: https://gist.github.com/sergeyf/63a37a5db18f8cebf962）。

```
def thompson_sampling(observed_data): 
   return np.argmax( np.random.beta(observed_data[:,0], observed_data[:,1]) )
```

通过这些方法可以比较有效地平衡探索和利用。用上面这些代码做模拟实验就能看到效果，如图7所示，不同的线代表不同的模型算法，纵轴是后悔值，越低越好，横轴是实验的次数。可以看到，UCB 和 Thompson Sampling 能很快就找到比较优的策略，其他几种方法是次优的。所以，UCB 和 Thompson Sampling 的实际使用也是很普遍的。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a986e60d0b0.png" alt="图7 不同算法的效果比较" title="图7 不同算法的效果比较" />

图7 不同算法的效果比较

### 实践效果

基于上述算法，我们做了一个 Hawk 实验平台，其架构如图8所示：输入进来以后，有一个 E&E Engine 分配策略，通过允许随时更新的配置文件，可以随时进行新的实验，Log Processor 定期进行日志的离线分析，生成新的模型。我们主要用的是 Thompson Sampling 算法。它的好处是不要求实时更新数据，因为它是一个概率模型。对于 UCB，如果我们不更新实验数据，它是确定性的，总是固定地选择 Bound 最高的策略，即在一定时间内只能有一个策略被使用。所以它要求实现系统能够快速更新，这意味着付出更多的开发成本。而 Thompson Sampling 作为概率模型，即使我们不更新数据，它仍然可以按照学到的后验概率进行有效的随机化的策略选择，所以不存在类似的问题。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a986db526d1.png" alt="图8 Hawk实验平台架构" title="图8 Hawk实验平台架构" />

图8 Hawk 实验平台架构

第二个用途是推荐系统中常见的冷启动（Cold Start）问题。新用户来了，我们不知道他的兴趣在哪里，这个时候可以利用这些增强学习算法来做。方法是：选择一些优质的 Item，根据这些算法分配给新的用户，收集新用户的反馈（点击了或者是收藏了）。有了这些反馈数据之后，我们就可以对类别的兴趣进行打分，分数估计可以用 UCB 或者 Thompson Sampling 来做。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a986d18f785.png" alt="图9 加入context的问题框架" title="图9 加入context的问题框架" />

图9 加入 context 的问题框架

上面阐述了的这些场景里面没有考虑用户特征。但实际上，我们或多或少都能知道用户的一些信息，比如他/她的访问时间、地理位置、所用浏览器和浏览记录等等，这些信息是有用的，可以被用来实现更好的推荐效果。如图9所示是一个例子，里面有一些常见的上下文环境信息（context）。而前述的简单算法无法处理这些上下文。还好，研究人员已经研究了一些这方面的算法，可以利用上下文信息进行更好的 Bandit 的选择。对于有上下文的 Bandit Problem，业界比较常用的模型是两个。其中一个是线性的 Linear Contextual Bandits 模型。它假设一个臂（Arm）的回报是 context 的线性函数，θ是不同因素的权重，每个 Arm 有不同的θ。关键是求出θ使得后悔值最小。线性模型下，看到的回报可以当做是连续值。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a98662ad0a9.png" alt="公式1" title="公式1" />

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a986738cc32.png" alt="公式2" title="公式2" />

一个用来解这个问题的算法是 LinUCB。它估计出来一个 Reward，根据现在学的参数算出一个 Confidence，然后用乐观的方式，选择 Upper Bound 最高的臂就可以。这和前述 UCB 类似。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a986c4aa1b9.png" alt="图10 LinUCB算法" title="图10 LinUCB算法" />

图10 LinUCB 算法

另外一个思路是基于 Thompson Sampling。如图11所示，其关键思想是参数θ是一个估计值，是随机变量。假设参数服从某个分布，每次对参数做抽样（而不是对 Arm 抽样），用抽出来的参数做一个估计，计算每一个 Arm 的回报并取最大值。收到反馈之后再更新参数的分布。这和之前的思想是一致的。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a986b968633.png" alt="图11 Thompson Sampling for Contextual Bandits" title="图11 Thompson Sampling for Contextual Bandits" />

图11 Thompson Sampling for Contextual Bandits

### 总结

增强学习提供了一套自适应学习系统的理论框架，Multi-Armed Bandits 模型虽然简单，但对于推荐系统是非常实用的框架，在推荐策略试验、用户兴趣探测、内容试验等方面都能很好地发挥作用。而如果有更多的信息，利用 Contextual Bandits 模型会获得更好的推荐性能。

与常用监督学习不同，增强学习不需要预先了解所有的信息，与我们面临的很多实际情况吻合，所以它在实践中有非常多的用途，包括智能控制、智能机器人、调度优化、互联网广告、在线游戏等，如 Google 利用深度增强学习来实现用于游戏的机器智能。

至于 Multi-Armed Bandits 用于推荐系统的最新方向，包括Bootstrap Bandit Algorithm 和 Collaborative Filtering with Bandits，猎聘网也在探索，期待有机会和大家进行更多的分享交流。
