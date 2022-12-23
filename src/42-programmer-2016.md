## 风口的物联网技术



文/周建丁

经过多年积累，物联网（IoT）被认为在2016年达到爆发期。根据 Gartner 的最新预测，2016年全球物联网设备的总数为64亿台，同比增长30%。对于物联网开发者来说，在新的一年也迎来了新的机遇和挑战。

### 云计算、人工智能推动物联网
物联网的定义，就是要形成人与物、物与物相联，实现信息化、远程管理控制和智能化的网络，公有云的发展和成熟，为物联网提供了强大的后端，4G/5G 等无线技术也将这些云输送到世界的各个角落。另一方面，随着以深度学习为代表的人工智能技术发展，使得机器在视频、图像、语音、文本理解上都有了质的飞跃，为采用计算机视觉、图像识别、模式识别、行为识别等技术的智能设备和物联网应用的发展带来东风，物联网设备能因此获得空前的感知能力（同时物联网数据库也能为深度学习建模的改进提供支持），无人驾驶（智能交通）就是最典型的代表。

### 风口的嵌入式
对物联网而言，云到端的互联是很重要的，但不意味着一切都可以交给“云”，各种智能传感器的需求就必不可少。在“云”高速发展的同时，“端”也在发生巨大的变化，WiFi SoC、STM32 相关的技术，在2015年就开始成为整个物联网抢占的风口。WiFi 模块的价格已降到10美元以下。随着多家厂商的加入，可以预见2016年将是 WiFi 模块血拼的一年。这个领域的开发者，以及原有的 STM32 开发者，将面临人、物、云连接的任务。同时，深度学习技术在“端”的应用，对“端”芯片的神经网络计算、功耗、可编程性等能力也提出了新的要求。

系统软件方面则相对手机更加多样化。如 Google 既有 Brillo，也提供面向可穿戴开发的 Android Wear。而 Linux 基金会最近宣布了微内核项目 Zephyr，据说能运行在只有10KB RAM 的32位微控制器上，这对开发针对物联网设备的实时操作系统（RTOS）而言无疑是一个福音。

### 应用多样化
服务机器人，无人驾驶，消费级的可穿戴设备，工业级的设备在线监测，乃至虚拟现实（VR）/增强现实（AR）……这一列表还可以更长，在国际国内无数大小会议上，这些内容都是最近的主角。当然，通过这些不同类型的产品，人、物、云的广泛互联，也带来了交互方式的巨大变化，这也是需要开发者放飞想象的地方。

### 数据分析
即使不使用新的人工智能技术，一般的数据分析仍是物联网应用必须考虑的，唯有借助分析才能真正将物联网与工业紧密结合，更不用说很多物联网设备的诞生，初衷就是收集数据用于分析。真正的物联网是数据智能驱动的物联网，由大数据驱动的物联网才是有价值的物联网。当然，分析不仅包括云上的分析，也包括端上的分析。

### 开发平台
语言、平台确实只是工具，但优秀的工具能够提高程序员的工作效率。目前面向物联网的开发平台，包括基于云的物联网开发平台，都有不错的发展。微软发布支持 Windows 10的物联网 PaaS 平台 Azure IoT Suite，亚马逊发布 SDK 支持 C、Java Script 和开源硬件 Arduino的AWS IoT，Intel 推出结合软硬件的物联网平台参考架构，IBM 也成立 Watson IoT 全球总部，开放语音识别、机器学习在内的 API……对于开发者而言，设备的可管理，数据收集的协议，数据传输、存储和分析的能力，可视化，乃至安全等等因素，需要结合自己的实际项目需求来选择。

### 安全新课题
安全是每一类信息系统都需要考虑的问题，匆忙开发物联网应用也有可能落入安全陷阱。网络攻击日趋复杂，物联网的硬件、通信、连接，各个环节都有可能被攻击。大多数物联网设备，尤其是新兴的智能机器人、无人机这样的领域，在设计和部署之初通常都是以最小的安全需求进行测试，可能将导致被不断利用的漏洞。

来自人工智能技术的应用也可能带来新的安全威胁。统计机器学习产生的智能对数据依赖比较高，如果自主决策的人工智能程序设计不完善，虚假的数据和恶意的算法都可能导致不良后果。例如，针对无人驾驶系统，Security Innovation 公司科学家 Jonathan Petit 曾在一篇论文中介绍，使用一种弱激光和脉冲发射器装置可以“欺骗”采用 Ibeo Lux 激光雷达传感器的无人驾驶汽车，使其“看见”根本不存在的“障碍物”。

技术发展很快，物联网开发过程中的坑，谁做谁知道，但不做可能永远不会知道，也无从解决。Gartner 预测到2020年物联网设备安装量将达208亿台，而有人预言2020年将进入万物互联的“机人时代”——所有为人类服务的设备都装上智能传感器，都可以建立云端档案，都将变单一功能为多功能，成为连接人和服务的其中一个节点。就让我们用程序员的智慧把坑填平，推动这个新时代的到来吧！