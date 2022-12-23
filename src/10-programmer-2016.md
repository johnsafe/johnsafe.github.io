## Peter Norvig：人工智能将是软件工程的重要部分



文/Peter Norvig

Peter Norvig 是誉满全球的人工智能专家，目前担任 Google 研究总监（Director of Research），他同时也是经典书籍《人工智能编程范式：Common Lisp 案例研究》（*Paradigms of AI Programming: Case Studies in Common Lisp*）和《人工智能：一种现代方法》（*Artificial Intelligence: A Modern Approach*）的作者/合著者。在本文中，我们将看到 Peter Norvig 对人工智能目前进展和未来发展的思考，对人工智能技术在 Google 应用的解读，以及对最新软件工程师在人工智能时代的成长的观点。

### Peter Nervig 眼中的人工智能

问：人工智能领域在哪些方面发生了您未曾预料的演变？

Peter Norvig：在1980年我开始从事人工智能研究时人工智能意味着：一位研究生用说明性语言写下事实，然后拨弄这些事实和推理机制，直到从精心挑选的样本上得到不错的结果，然后写一篇关于它的论文。

虽然我接受并遵循这种工作模式，在我获得博士学位的过程中，我发现了这种方法的三个问题：

- 写下事实太慢了。
- 我们没有处理异常情况或模糊状态的良好方法。
- 这个过程不科学——即使在选定的样本上它能工作，但是在其他样本上工作效果会如何呢？

整个领域的演变回答了这三个问题：

- 我们依靠机器学习，而不是研究生付出的辛苦努力。
- 我们使用概率推理，而不是布尔逻辑。
- 我们希望使用科学严格的方式；我们有训练数据和测试数据的概念，而且我们也有比较不同系统处理标准问题所得到的结果。

1950年，阿兰图灵写道：“我们只能看到未来很短的一段距离，但是我们很清楚还有什么需要完成。”自从1950年，我们已经得到许多发展并实现了许多目标，但图灵的话仍然成立。

问：对于机器学习研究，工业界与学术界有何不同呢？

Peter Norvig：我认为，在教育机构、商业机构还是政府机构并不是很重要——我曾经在这三种机构都学到很多东西。

我建议你在有着一群出色同事和有趣问题的环境下工作。可以是工业界、学术界、政府或者非营利企业，甚至开源社区。在这些领域里，工业界往往有更多的资源（人、计算能力和数据），但如今有很多公开可用的数据供你使用，一个小团队，一台笔记本电脑，或者一个小而廉价的 GPU 集群，或者在云计算服务上租赁或捐献时间。

问：您对深度学习有什么看法？

Peter Norvig：我清楚地记得80年代初的那一天，Geoff Hinton 来到伯克利进行了关于玻尔兹曼机的讲座。对我来说，这是个了不起的视角——他不赞同符号主义人工智能很强大很有用，而我了解到了一种机制，有三件令人兴奋的新（对我而言）事情：根据大脑模型得出的认知合理性；从经验而不是手工编码中学习的模型；还有表示是连续的，而不是布尔值，因此可以避免传统符号专家系统的一些脆弱问题。

事实证明，玻尔兹曼机在那个时代并没有广泛普及，相反，Hinton、LeCun、Bengio、Olshausen、Osindero、Sutskever、Courville、Ng 以及其他人设计的架构得到很好的普及。是什么造成了这种不同呢？是一次一层的训练技术吗？是 ReLU 激活函数？是需要更多的数据？还是使用 GPU 集群可以更快地训练？我不敢肯定，我希望持续的分析可以给我们带来更好的了解。但我可以说，在语音识别、计算机视觉识别物体、围棋和其他领域，这一差距是巨大的：使用深度学习可以降低错误率，这两个领域在过去几年都发生了彻底变化：基本上所有的团队都选择了深度学习，因为它管用。

许多问题依然存在。在计算机视觉里，我们好奇深度网络实际上在做什么：我们可以在一个级别上确定线条识别器，在更高层次确定眼睛和鼻子识别器，然后就是脸部识别器，最终就是整个人的识别器。但在其他领域，一直很难了解网络在做什么。是因为我们没有正确的分析和可视化工具吗？还是因为实际上表示不一致？

在有许多数据的时候，深度学习在各种应用中表现不错，但对于一次性或零次学习，需要将一个领域的知识转移并适应到当前领域又如何呢？深度网络形成了什么样的抽象，我们可以如何解释这些抽象并结合它们？网络会被对抗性输入愚弄；我们如何预防这些，它们代表了根本缺陷还是不相干的把戏？

我们如何处理一个领域中的结构？我们有循环网络（Recurrent Networks）来处理时间，递归网络（Recrsive Networks）来处理嵌套结构，但这些是否已经足够，现在讨论还为时过早。

我对深度学习感到兴奋，因为很多长期存在的领域也是如此。而且我有兴趣了解更多，因为还有许多剩余问题，而且这些问题的答案不仅会告诉我们更多关于深度学习的东西，还可以帮助我们大体理解学习、推理和表示。

问：在深度学习最近取得的成就之后，符号主义人工智能是否还有意义？

Peter Norvig：是的。我们围绕着符号主义人工智能开发了许多强大的原理：逻辑预测、约束满足问题、规划问题、自然语言处理，乃至概率预测。因为这些算法的出色表现，我们处理问题的能力比原来提升了几个数量级。放弃这一切是件可耻的事。我认为其中一个有意识的研究方向是回过头看每一种方法，探索非原子式符号被原子式符号取代的这个过程究竟发生了什么，诸如 Word2Vec 产生的 Word Embedding 之类的原理。

下面是一些例子。假设你有这些逻辑“事实”：

1. 人会说话；
2. 除人以外的动物不会说话；
3. 卡通人物角色会说话；
4. 鱼会游泳；
5. 鱼是除人以外的动物；
6. Nemo是一个卡通人物；
7. Nemo是一条鱼；
8. 那么我们要问了：
9. Nemo会说话吗？
10. Nemo会游泳吗？

用逻辑来表述和解释这个场景的时候遇到了两个大问题。首先，这些事实都有例外，但是用逻辑很难穷举这些例外情况，而且当你逻辑出错的时候预测就会出问题了。其次，在相互矛盾的情况下则逻辑无能为力，就像这里的 Nemo 既会说话又不会说话。也许我们可以用 Word Embedding 技术来解决这些问题。我们还需要 Modus Ponens Embedding（分离规则，一种数学演绎推理规则）吗？不学习“如果 A 且 A 暗示 B，则 B”这样一种抽象的规则，我们是否可以学习何时应用这种规则是恰当的？我觉得这是一个重要的研究领域。

再说一点：许多所谓的符号主意人工智能技术实际上还是优秀的计算机科学算法。举个例子，搜索算法，无论 A*或是蚁群优化，或是其它任何东西，都是一种关键的算法，永远都会非常有用。即使是基于深度学习的 AlphaGo，也包含了搜索模块。

问：我们哪儿做错了？为什么 Common Lisp 不能治愈世界？

Peter Norvig：我认为 Common Lisp 的思想确实能治愈这个世界。如果你回到1981年，Lisp 被视作是另类，因为它所具有的下面这些特性还不被 C 语言程序员所知：

1. 垃圾回收机制；
2. 丰富的容器类型及相应的操作；
3. 强大的对象系统，伴随着各种继承和原生函数；
4. 定义测试例子的亚语言（sublanguage）（并不属于官方版本的一部分，但我自己配置了一套）；
5. 有交互式的读入-运算-打印循环；
6. 敏捷的、增量式的开发模式，而不是一步到位的模式；
7. 运行时对象和函数的自省；
8. 能自定义领域特定语言的宏。

如今，除了宏之外的所有这些特性都在主流编程语言里非常常见。所以说它的思想取胜了，而 Common Lisp 的实现却没有 —— 也许是因为 CL 还遗留了不少1958年编程语言的陋习；也许只是因为一些人不喜欢用大括号。
至于说宏，我也希望它能流行起来，但当用到宏的时候，你成为了一名语言设计者，而许多开发团队喜欢保持底层语言的稳定性，尤其是那些大团队。我想最好有一套使用宏的实用指南，而不是把它们全部抛弃（或是在C语言里严格限制的宏）。

问：在未来10年里，有没有哪些情况下软件工程师不需要学习人工智能或机器学习的，还是每个人都需要学习？

Peter Norvig：机器学习将会是（或许已经是）软件工程的一个重要部分，每个人都必须知道它的运用场景。但就像数据库管理员或用户界面设计一样，并不意味着每个工程师都必须成为机器学习专家——和这个领域的专家共事也是可以的。但是你知道的机器学习知识越多，在构建解决方案方面的能力就越好。

我也认为机器学习专家和软件工程师聚在一起进行机器学习系统软件开发最佳实践将会很重要。目前我们有一套软件测试体制，你可以定义单元测试并在其中调用方法，比如 assertTrue 或者 assertEquals。我们还需要新的测试过程，包括运行试验、分析结果、对比今天和历史结果来查看偏移、决定这种偏移是随机变化还是数据不平稳等。这是一个伟大的领域，软件工程师和机器学习人员一同协作，创建新的、更好的东西。

问：我想从软件工程师转行成为人工智能研究员，应该如何训练自己呢？

Peter Norvig：我认为这不是转行，而是一种技能上的提升。人工智能的关键点在于搭建系统，这正是你手头上的工作。所以你在处理系统复杂性和选择合适的抽象关系方面都有经验，参与过完整的设计、开发和测试流程；这些对于 AI 研究员和软件工程师来说都是基本要求。有句老话这样说，当一项人工智能技术成功之后，它就不再属于人工智能，而是成为了软件工程的一部分。人工智能工作者抱怨上述观点的意思就是他们的工作永远离成功有一步之遥，但你可以认为这表明你只是需要在已知的基础上再添加一些新概念和新技术。

### 人工智能在 Google

问：Google“没有更好的算法，只是多了点数据而已”这种说法是真的吗？

Peter Norvig：我曾引用微软研究院 Michele Banko 和 Eric Brill 发表的一篇关于分析词性辨析算法的论文，他们发现增加训练数据得到的效果提升比更换算法更明显。我说过有些问题确实如此，而另一些问题则不见得。你可以认为这篇论文是“大数据”的功劳，但要注意，在这个领域十亿个单词规模的训练数据集就能看出效果 —— 在笔记本电脑的处理范围内 —— 还不到数据中心的量级。所以，如果你用不了数据中心，不必担心 —— 你拥有的计算资源和数据量几乎完胜任何一个上一代的人，你可以有更多的新发现。

所以没错，大量与任务相契合的高质量数据必然会有帮助。然而真正有挑战的工作在于发明新学习系统的研究和让其真正落实到产品中的工程实现。这个工作正是大多数机器学习成功案例的驱动力。正如 Pat Winston 所说：“人工智能就像葡萄干面包里的葡萄干，葡萄干面包的主要成分还是面包，人工智能软件主体也是常规的软件工程和产品开发。”

问：成为一家“AI-first”公司对 Google 意味着什么？

Peter Norvig：“传统”的 Google 是一个信息检索公司：你提供一个查询，我们快速返回10个相关网页结果，然后你负责找到与查询词相关的返回结果。“现代”的 Google，CEO Sundar Pichai 设定了愿景，它不仅基于相关信息建议，还基于通知和助理。通知，意味着当你需要时，我们提供你需要的信息。例如，Google Now 告诉你该去赴约了，或者你目前在一家杂货店，之前你设定了提醒要买牛奶。助理意味着帮助你实施行动——如规划行程、预定房间。你在互联网上可以做的任何事情，Google都 应该可以帮你实现。

对于信息检索，80%以上的召回率和准确率是非常不错的——不需要所有建议都完美，因为用户可以忽略坏的建议。对于助理，门槛就高了许多，你不会使用20%甚至2%的情形下都预定错房间的服务。所以助理必须更加精准，从而要求更智能、更了解情况。这就是我们所说的“AI-first”。

### Peter Nervig 在 Google

问：你的职业生涯如何起步？

Peter Nervig：我很幸运地进入了一所既有计算机编程又有语言课程的高中（在马萨诸塞州牛顿县）。这激发了我将两者结合起来学习的兴趣。在高中阶段无法实现这个想法，但是到了大学我主修应用数学专业，得以研究这方面（当时，我们学校并没有真正的计算机专业。我开始是主修数学，但很快发现自己并不擅长数学证明，反而在编程方面如鱼得水）。

大学毕业后，我当了两年的程序员，不过仍旧一直在思考这些想法，最后还是申请了研究生回来继续从事科研（我过了四年才厌倦大学生活，而两年就厌倦了工作状态，所以我觉得我对学校的热爱是对工作的两倍）。研究生阶段为我学术生涯打下了基础，而我却迷上了今天所谓的“大数据”（当时还没有这种叫法），我意识到在工业界更容易获得所需要的资源，因此放弃了高校里的职位。我感到幸运的是每个阶段都有优秀的合作伙伴和新的挑战。

问：你在 Google 具体做什么？

Peter Norvig：在 Google 最棒的事情之一就是总有新鲜事；你不会陷入例行公事之中。在快节奏的世界中每周都是如此，当我角色改变之后，每年更是如此。我管理的人员从两人变成了两百人，这意味着我有时候能深入到所参与项目的技术细节中，有时候因为管理的团队太大，只能提一些高层次的笼统看法，并且我相信我的团队正在做的事情是正确的。在那些项目里，我扮演的角色更多的是沟通者和媒介——试图解释公司的发展方向，一个项目具体如何展开，将项目团队介绍给合适的合作伙伴、制造商和消费者，让团队制定出如何实现目标的细节。我在 Google 不写代码，但是如果我有一个想法，我可以使用内部工具写代码进行实验，看看这个想法是否值得尝试。我同样会进行代码审查，这样我就可以了解团队生产的代码，而且这也必须有人去做。

还有很多的会议、邮件、文档要处理。与其他我工作过的公司相比，Google 的官僚主义更少，但有时候是不可避免的。我也会花一些时间参加会议、去大学演讲、与客户交流，以及参与 Quora 问答。

问：在加入 Google 之前，你曾担任美国宇航局（NASA）计算科学部门的主要负责人，在美国宇航局的工作与 Google 的工作有何不同？有哪些文化的差异？

Peter Norvig：美国宇航局与 Google 有很多共同之处：它们都有一群优秀敬业并且充满激情的员工，这些人相信它们的工作使命。而且两者都在推动各自技术的上限。因此，他们在特定项目中的文化往往是相似的。

同时也存在一些差异。美国宇航局的 Gene Kranz 曾说过一句名言：“失败不是种选择（Failure is not an option）。”美国宇航局经常会有几亿美元的使命任务，任何一个错误都有可能毁灭一切。因此，需要极其小心。Google 的项目范围往往更接近 Adam Savage 的想法（与 Jeff Dean 相互呼应）“失败自古至今就是一种选择（Failure is always an option）”。Google 相信，单台计算机可能会发生故障，而设计网络系统可以从故障中恢复。在 Google，有时我们可以在用户看到错误之前进行恢复纠正，而有时当一个错误曝光后，我们可以在简短的时间内纠正它，同时向受到影响的用户致歉，而这在美国宇航局是很少见的。

一方面是因为失败的预期成本存在差异，另一方面是由于空间硬件的成本巨大（参见我在那做的东西），再者就是政府与私人机构的差异，基于这一优势，Google 更容易启动新项目，并在已有的项目中迅速推动新项目的进展。

问：你是如何权衡新功能的开发与旧功能的维护呢？

Peter Norvig：尽你所能将任务做得最好，并且不断改进，这样就会得到提高。

我曾一次次地发现：团队的新员工说“我们为什么不使用 X？”，一位老员工回答说：“我们三年前就试过了 X，结果证明它并不管用”。此时的难题是：你是否接受那位老前辈的回答？或者说，现在的情况已经改变了，是时候重新审视 X了？也许我们有新的数据，或者新的技术，又或者新员工将采取不同的方法，或者说世界改变了，X 将会比以往工作得更好。我无法告诉你该问题的答案，你必须权衡所有证据，并与其他类似问题进行比较。

### 程序员提升之道

问：《人工智能：一种现代方法》还会有新的版本吗？

Peter Norvig：是的，我正在为此努力。但至少还需要一年的时间。

问：我是一名研究生，我的人工智能课程使用《人工智能：一种现代方法》作为参考教材，我如何才能为人工智能编程项目做贡献？

Peter Norvig：现在正是时候：我正在为《人工智能：一种现代方法》这本书的下一个版本的配套代码工作，在<https://github.com/aimacode>上，你可以找到 Java、Python 和 JavaScript 子项目，我们一直在寻找好的贡献者。除了提供书中所有算法的代码实现，我们还希望提供 tutorial 材料和练习。此外，GitHub 上也还有其他好的人工智能项目，都希望有铁杆贡献者。

问：有没有像可汗学院（Khan Academy）和 Udacity 一样的在线资源，可以让人们在不到“十年”就精通一门学科呢？

Peter Norvig：精通可能需要十年，或者是10000个小时，这种时间会因任务、个体以及训练方法的不同而有所差异。但真正的精通并非易事。可汗学院和 Udacity 主要是提供了技术培训，让你不断努力地学习直到你真正地掌握它。在传统的学校教学当中，如果你在考试中获得的成绩是“C”，你就不会再去花更多的时间去学习并掌握它，你会继而专注于下一个学科，因为班集里每个人都是这样做的。在线资源不是万能的，精通它需要加倍努力地学习，而学习需要动力，动力则可以通过人与人之间的联系逐步提升，这在网上是很难学到的。因此，在一个领域，走上正轨，我们需要在社交、动机方面做更多的工作，我们需要对个人需求有针对性地做更多的定制培训，同时我们还需要做更多使实践审慎和有效的工作。我认为，在线资源主要的最终结果不是缩短精通的时长，而是增加更多学生实现精通的机会。

问：如果请你再次教授《计算机程序设计》（Udacity）这门课程，会做哪些改变呢？

Peter Norvig：我认为这门课程很好，反馈（不管是数量还是质量）大多都是好的。就个人而言，我希望有更多的实例程序和技术。我想修正之前我们犯下的一些错误（主要是因为课程进展太快，没有太多的时间去测试所有的东西）。我希望系统能够更加互动：让学生获得更多的反馈信息，不仅仅是“你的程序不正确”，同时可以让学生看到下一件要做的事情，让他们知道到目前为止已经做了什么。我认为对于学生而言，正则表达式和语言这部分进展速度过快了；另外，我还想添加更多的材料，让学生加快学习速度，同时给他们更多的机会去实践新想法。 

>注：本文主要内容来自 Peter Norvig 最近在 Quora 网站上的回答，链接：<https://www.quora.com/session/Peter-Norvig/1>，感谢刘翔宇、赵屹华、刘帝伟的翻译。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574cf1594898b.jpg" alt="" title="" />