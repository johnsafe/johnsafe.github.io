## 先进的银行反欺诈架构设计

文 / 杨税令

在互联网金融领域井喷式发展的同时， 安全问题也日益严重， 本文作者以银行为例， 全面解读了反欺诈的架构设计。

先举两个真实案例，第一个就发生在我身上，我存在某互联网金融公司的四万多理财资金在一个周五晚上十点的一个小时内全部被盗，账户被别人在异地使用新手机登录并修改了登录密码、支付密码、更换了我绑定的银行卡、并额外绑定了三张别人的银行卡，这期间我无法重置支付密码、无法解绑银行卡、无法冻结账户、打客服提示已下班，束手无策，只有绝望，这个过程中发生了多少敏感操作，而我的手机没有收到一条变更确认的短信和变更成功后的通知，只有最后收到一条我的账户被提现到某某卡的通知，从这个过程就可以看出这家公司居然没有用户身份真伪识别的机制，更别说交易真实性识别了，完全就是拿着用户的钱在网上裸奔，谁能在旁边叫出钱的名字钱就给谁，作为一家金融公司实在是让人震惊。第二个案例是发生在银行间市场，有个人通过向 A 银行购买十万理财产品的方式获取了 A 银行的理财产品说明书、协议书、税务登记证、营业执照、组织机构代码证、客户权益须知等文件，然后冒充 A 银行工作人员利用 A 银行的贵宾室，向 B 银行高息兜售，连续多天在 A 银行的表演和略施小计骗过了 B 银行的审核人员，从而卖出了一份40亿的理财资金，但是这笔交易被 B 银行的反欺诈侦测列入了风险监控列表，经过人工审核确认后堵截了这起诈骗事件（详细过程可查看银监会安徽监管局发的2016第55号文件）。对比 B 银行该案例中表现出来的反欺诈侦测能力，某互联网金融公司的做法就是在作死，互联网金融公司安全能力的提升迫在眉睫也任重道远。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574e6b58af9f5.jpg" alt="图1 报警回执" title="图1 报警回执" />

图1 报警回执

互联网金融公司想要提升自己的安全能力，最好的学习榜样就是银行，现在我们就开始探讨银行的反欺诈是如何设计的。一般概念上的欺诈分内部欺诈和外部欺诈，外部中主要有三类欺诈：当事人欺诈、第三方欺诈以及洗钱欺诈，内部欺诈主要有未经授权的行为与盗窃。对于欺诈的防控分事前防控、事中防控与事后防控，并在以下层面进行防控：
1. 外部渠道层：重点侦测交易发生前的客户接入、会话可疑行为；交易发生中的交易对手是否在可疑欺诈名单。
2. 内部渠道层：重点侦测业务违规与可疑操作。
3. 产品服务层：重点侦测产品服务内的欺诈交易，跨产品的欺诈交易。
4. 数据集成层：重点侦测跨产品、渠道的组合/复杂欺诈交易。

这些不同的层侦测逻辑不一样，其侧重防控的欺诈行为也不一样，渠道层可能侦测以下行为：

1. 异地更换网银盾后首次进行大额转账，这可能意味着客户的信息已泄露，这种交易需要挂起，并打电话与客户进行核实。
2. 客户通过手机或网银渠道向黑名单收款账户转账，被阻断交易后，当天该账户又向其它账户进行大额转账，这可能是客户账户被盗或被电信诈骗分子利用社会工程学的手段实施了诈骗，这种交易需要挂起，并需要打电话给客户进行核实。
3. 异地升级网银盾后首次进行大额转账，这可能是客户身份被盗用，身份证、登录密码等已泄露，这种交易需要挂起，并需要打电话给客户进行核实。
4. 新开通的网银客户进行大额转账，这可能是客户被电信诈骗分子利用社会工程学的手段实施了诈骗，这种交易需要挂起，并需要打电话给客户进行核实。

用户登录所使用的设备指纹（MAC 地址、IP、主板序列号、硬盘序列号）、登录时间、设备所在地，与其常用的对应信息不一致，这可能是客户账户已被盗用，这种情况需要进行人工核实。

产品层可能侦测以下行为：
1. 进入黑名单商户的交易，对于已支付未确认付款的交易需要实施冻结，防止资金流入该商户。
2. 根据客户的投诉确认商户是否存在虚假交易，如果是也需要实施冻结。
3. 如果同卡同天当笔交易为上一笔的倍数，这可能是客户账户被盗用，这种交易需要挂起，并人工进行核实。
4. 如果同卡同商户同金额，这可能是商户正在配合客户正在套现，这种交易需要人工核实。
5. 如果同卡同商户五分钟内交易超限，这可能是在进行虚假交易，这种交易需要人工核实。
6. 如果对公客户的交易额不在其合理的范围内（通过其注册资本、代发代付的累计额等评估的范围），这种交易可能需要拒绝并人工进行调查。
7. 如果使用伪卡进行交易，此后该商户发生的交易可能都需要阻断或告警。

客户层可能侦测以下行为：
1. 特定年龄段客户以往习惯在非柜面进行小额交易，突然第一笔发生大额转账，这可能是账户被盗，需要进行人工调查。
2. 客户账户多日连续多笔密码验证错误，尝试成功后就进行转账操作，这可能是账户被盗，其发起的交易可能需要被阻断，该客户使用的其他产品可能均需要挂起，并进行人工核实处理。
3. 同一个客户的一个或多个产品短时间内在不同地区/国家使用，这可能是客户的卡被复制存在伪卡，这种交易需要人工核实处理。
4. 在一定时间内，同一个客户在特定高风险国家发生多笔或进行大额交易，这可能是伪卡，这种交易需要人工核实处理。

通过对客户和员工不同纬度外部欺诈、内部欺诈风险及黑名单信息的分类评估，需要实现对客户欺诈风险的联合防控，它们之间的风险关系梳理如图2。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574e6b6a0b74a.jpg" alt="图2 风险关系" title="图2 风险关系" />

图2 风险关系

如果在防控的前、中、后三个阶段都对各个产品的多个纬度进行统一欺诈防控与处理，那么我们需要基于它们整体建立一套防控体系，也就是企业级反欺诈体系，通过整理并抽象总结前面提出的侦测行为，需要实现的目标梳理如下应该具有统一的：
1. 数据集市。
2. 数据采集、加工过程。
3. 侦测策略定义过程。
4. 基于流程引擎的侦测问题流转管理。
5. 基于流程引擎的案件管理，记录、跟踪、评估、回顾相关的处理过程。
6. 基于规则引擎的实时、准实时、批量风险侦测。
7. 信息外送处理。

通过这些目标，我们将需要具备的功能梳理如下：
1. 反欺诈业务处理：告警管理、案件调查、交易控制、侦测处理。
2. 反欺诈运营管理：运营管控、流程管理、策略管理。
3. 反欺诈数据报表：数据整合、数据报告。
4. 反欺诈模型研究：规划研究、变量加工、贴源数据。
5. 反欺诈行为分析：行为分析、关联分析、评级计算、批量处理。

基于前面的要求，我们来梳理一下与反欺诈有关的上下文关系，如图3。
<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574e6b7516002.jpg" alt="图3 上下文关系" title="图3 上下文关系" />

图3 上下文关系

图中蓝色线是交易访问关系，橙色线是批量数据访问关系，通过这些关系，我们再来细化它们在应用架构中的位置，如图4所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574e6b8c6bcab.jpg" alt="图4 应用架构图" title="图4 应用架构图" />

图4 应用架构图

再把它们在数据架构中的位置也梳理出来，如图5。
<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574e6b9d516a1.jpg" alt="图5 数据架构图" title="图5 数据架构图" />

图5 数据架构图

渠道层的处理流程梳理如图6。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574e6bafcf5c3.jpg" alt="图6 渠道层处理流程" title="图6 渠道层处理流程" />

图6 渠道层处理流程

产品层的处理流程梳理如图7。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574e6bc589bc8.jpg" alt="图7 产品层处理流程" title="图7 产品层处理流程" />

图7 产品层处理流程

客户层的处理流程梳理如图8。
<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574e6bd7e1abd.jpg" alt="图8 客户层处理流程" title="图8 客户层处理流程" />

图8 客户层处理流程

在这些处理流程中，对于需要加强认证的行为，需要将该次交易列入风险监控列表中，经事后人工确认确实存在欺诈行为的，将此类行为列入风险行为模型中，完成欺诈侦测随着欺诈行为的变异而不断进化。

好了，到这里反欺诈的主体部分就算设计完成了，这是在企业级架构中逻辑各层已解耦的前提下进行的设计，分阶段分层各司其职分而治之，通过建立行为模型灵活应对用户的各种行为，适应现在与未来，对于那些新出现的欺诈手段，主动学习并生成欺诈行为模型，将可有效杜绝现在与未来可能发生的欺诈。

通过反欺诈设计的过程，我们可以总结几招识别一家互联网金融公司是否具备反欺诈能力的小技巧：
1. 将帐户在其它手机上登陆，测试渠道层反欺诈能力；
2. 将帐户在异地登陆，测试渠道层反欺诈能力；
3. 修改登陆密码，测试产品层反欺诈能力；
4. 修改支付密码，测试产品层反欺诈能力：
5. 修改身份信息，测试客户层反欺诈能力；
6. 绑定新的银行卡，测试产品层反欺诈能力；
7. 用新卡提现，测试交易反欺诈能力；
8. 用他人手机提现，测试交易反欺诈能力；
9. 异地全额提现，测试交易反欺诈能力。

进行以上任意一步操作，如果收到短信提醒，说明有帐户异常行为识别机制；如果有收到短信验证码，说明有帐户行为控制机制；如果收到电话确认，说明有用户身份真伪识别。如果只有短信提醒，请谨慎使用，如果都没有，请立刻马上提现并卸载。