# NetHide Secure and Practical Network Topology Obfuscation
---
## 来源
1. 参考[网页link](https://www.usenix.org/conference/usenixsecurity18/presentation/meier)

## 翻译
### 摘要
- 背景： 简单路径追踪（trace）的工具如 traceroute 允许恶意用户远程推测网络拓扑，并以此来发动高级的拒绝服务（DoS）攻击，如链路泛洪攻击(LFA)。然而，尽管存在风险，大多数网络运营商仍然允许路径跟踪，因为它是必不可少的网络调试工具。
- **NetHide**:
	- 一种网络拓扑混淆框架，能减少LFA，同时保留路径追踪工具的实用性。
	- 关键思想： 把网络混淆视为一个**多目标优化问题**，即灵活权衡安全性（编码为硬约束）和可用性（编码为软约束）
	- NetHide可以通过仅考虑候选解决方案的子集而不降低混淆质量来大规模地模糊拓扑。实际上，NetHide通过直接在数据平面中拦截和修改路径跟踪探测包来模糊拓扑。利用最新一代的可编程网络设备，展示了这一过程可以在无线状态下以线速进行
	- 全面实现了NetHide，并在实际拓扑上进行了评估。结果表明，NetHide能够混淆大型拓扑（> 150个节点），同时保留了近乎完美的调试功能。特别地，尽管混淆，运营商仍然可以精确追溯> 90％的链路故障。

### 1. 引言
- 僵尸网络驱动的 DDos 攻击构成当今 Internet 主要威胁之一，攻击按照攻击目标不同可以分为两类：
	1. 基于量的攻击(volume-based attack)
		- 以终端主机和服务为目标
		- 最简单，通过发送大量数据到选择的目标，如针对 Dyn 的 DNS 服务的 1.2Tbps 的 DDoS 攻击和 2018 年 2 月针对  github 的 1.35Tbps 的 DDoS 攻击
		- 当前这些攻击可以通过大型 CDN 基设施转移入流量，如 CloudFlare 的设施可以减缓每秒 Tb 级别的这类攻击
	2. 链路泛洪攻击（LFA）
		- 以网络基础设施为目标
		- 更复杂，通过僵尸网络在一组僵尸机器或公共服务之间产生低速流，使得所有这些流量穿过一组给定的网络链路或节点，降低甚至阻止所有使用它们服务的连接性
		- 难以检测：
			1. 流量相对较小，10 Gbps 或 40 Gbps 的攻击足以杀死所有 Internet 链接；
			2. 攻击流量和正常流量无法区分，代表例子就是针对在欧洲和亚洲的 IXP 链接的 Spamhaus 攻击
		- LFA 攻击需要攻击者知道目标网络的拓扑和转发行为，否则会降低攻击效率，模拟表明在不知道拓扑的情况下，拥塞任意链路需要5倍以上的流量，而拥塞特定链路更难。然而攻击者通过路径追踪工具可以轻易获取拓扑知识，如 traceroute。实际上，过去的研究表明通过使用足够多的观测点（vantage point），traceroute 可以准确发现整个拓扑，如使用大规模测量平台RIPE Atlas
- 已有工作
	- 已经存在的 LFA 对策包括主被动的方式：
		1. 被动措施动态调整流量转发方式或者以网络协作的方式来检测恶意流量
			- 缺点：缺乏部署激励；流量动态调整与流量工程的目标冲突
		2. 主动措施以混淆网络拓扑的方式来防止攻击者发现潜在的目标
			- 优点：独立保护每个网络，而不影响正常的流量转发
			- 缺点：大大降低路径追踪工具的可用性，如 traceroute；混淆程度低，可以被暴力攻击轻易破坏
- 问题描述
	- 鉴于现有技术的局限性，是否有可能混淆网络拓扑，来减轻 LFA 同时保留路径追踪工具的可用性
- 关键挑战
	1. 必须针对任何可能的攻击者位置对拓扑进行混淆：攻击者可以位于任何地方，并且他们的追踪流量通常与合法用户请求无法区分；
	2. 混淆的逻辑不应是可逆的，同时可以拓展到大的拓扑；
	3. 混淆逻辑需要能够以线速（line-rate）拦截和修改追踪流量。 为了保持网络运营商的故障排除能力，追踪流量仍应流经正确的物理链路，例如，物理拓扑中的链路故障在混淆的拓扑中可见
		- 说明：交换机中 line-rate 用于端口限速，主要用于出端口上；traffic-limit 用于流限速，主要用于入端口上
### 2. 模型
- 网络模型
	- 考虑 layer 3（IP）层网络由单一权威管理，如一个 ISP 或 一个公司
	- 假定路由确定，意味一对节点之间的流量设定为沿着单一路径，虽然不符合网络可能的负载均衡的机制，但由于路径是持久的更易学习，攻击变得更容易
	- 假定一些路由器是可编程的:
		1. 匹配任意 IP TTL 值
		2. 根据原始目的地址和 TTL 改变源目的地址
		3. 当对修改的包回复到达时，恢复源目的地址
	- 实现：使用 P4 语言，在现有路由器硬件之上实现也可以
- 攻击模型
	- 假定攻击者控制一组主机如一个僵尸网络，目标是执行 LFA 如 Coremelt 或 [Crossfire](www.freebuf.com/articles/network/67107.html)
	- 假定遵循Kerckhoff 原则攻击者知道一切网络中的部署保护机制除了秘密输入和随机决策
### 3. NetHide
- 以一个简单的例子说明 NetHide 如何计算一个安全并可用（可调试的）的混淆拓扑
	1. 输入	
		- 4 个输入
			1. 物理网络拓扑图
			2. 特定转发行为
			3. 每条链路能力
			4. 一组攻击流
		- 关键：混淆拓扑数量随着网路规模成倍增长，简单枚举不可行，NetHide 仅考虑候选方案的一个子集，并表明这是有效的 
	2. 预选择一组安全的候选拓扑
		- 随机计算一组混淆拓扑，随机选择作为秘密增加逆向混淆算法难度
		- 混淆从两个维度：修改拓扑图；修改转发行为
	3. 选择一个可用的混淆拓扑
		- 评估虚拟拓扑可用性（usability）从两个指标：
			1. 准确性（accuracy）：原始拓扑和混淆拓扑在使用 traceroute 发现路径下的逻辑相似性
			2. 实用性（utility）：物理拓扑和虚拟拓扑中追踪数据包实际采用的路径的物理相似性
	4. 部署混淆拓扑
		1. NetHide 利用可编程网络设备直接在数据层面拦截和修改追踪数据包，而不影响网络性能。特别地，在网络边缘，将追踪数据包发送给物理网络中假装目的地前，拦截和修改这些数据包
### 4. 生成安全的拓扑
- 优化问题
	- 安全约束
	- 目标函数
- 准确性（accuracy）指标和实用性（utility）指标的计算公式
- 可拓展性
	- **关键**：为每个节点预计算一组转发树（forwarding tree），然后优化结合它们计算虚拟拓扑（V）
		- 具体地，转发的计算：对 V 用 Dijkstra 算法计算每个节点的转发树，重复知道每个节点获取一定数量的转发树，每轮迭代链路权重 w(e)~Uniform(1, 10)
		- 复杂度：每个节点预计算一定数量的转发树，ILP solver 只需找到 O(|N'|)的优化结合，而非O(|N'|^2)条链接和O(|N'|^|N'|)个转发树
- 安全
	- 考虑 2 种不同攻击策略
		1. 从虚拟拓扑重构物理拓扑
		2. 基于观察到的虚拟拓扑选择一种攻击：随机；瓶颈+随机；瓶颈+紧密度
### 5. NetHide 拓扑部署
- 挑战
	- 在虚拟拓扑中反应物理事件
		- 确保 NetHide 高实用性（utility）的关键想法是发送追踪数据包穿过实际的物理网络而非在网络边缘或被中心控制器应答
	- 基于时间的设备指纹
		- RTT 可以潜在地被用来识别路径中的混淆部分；数据包的转发在硬件上完成不产生明显的时延，而应答一个超时（TTL=0）的 IP 包包括路由控制层面会造成明显的时延；实验表明路由器应答超时数据包花费的时间不仅差异大，同时取决于设备本身，使得基于它处理时间的分布识别一个设备是可能的
		- NetHide 通过确保被认为由节点 n 应答的数据包实际上被 n 应答，使得 RTT 测量是实际的
	- 数据包以线速处理
		- 避免减弱网络性能，NetHide 需要以线速（line rate）解析和修改网络数据包，特别地，需要处理 TTL　字段，源目的　IP 地址，重新计算校验和
- NetHide 和 P4
	- P4 是特定域（domain-specific）的编程语言，允许编程网络的数据层面，设计为协议无关和目标无关（protocol- and target-independent），P4 程序可以被编译为不同的目标，如路由器或交换机，在不同硬件上执行，如CPU,FPGA,ASIC。软件目标提供开发和测试 P4 的环境，而硬件目标可以以**线速**执行 P4 程序
	- NetHide 实现采用 P4_14，利用 P4 的定制头部格式来以**线速**重写追踪包，在设备中**不用保存状态**（每个包，流或主机）
- 架构
	- 一个 controller：将 V 转化为可编程网络设备的配置
	- 包处理软件
- 包处理软件
	- 识别潜在追踪包：一旦接受新的数据包，NetHide 设备检查是否是对 NetHide 修改包的回复，如果不是，检查该包的虚拟路径是否和物理路径不同并修改。NetHide 不区分 traceroute 和 其他网络流量，只依赖于报道 TTL 值和源目的地址。
	- 编码虚拟拓扑
		- TTL 修改：数据包 TTL 值足够高能穿过出口路由器，NetHide 无需修改地址，TTL 值需要格局虚拟与物理路径长度差异来增减
		- 地址修改：数据包 TTL 值较低以至于在到达目的地前超时，NetHide 需要保证数据包在相对于 V 的正确节点超时。首先修改目的地址使得发送给根据 V 应该应答的节点，此外，设置源地址为处理改包的 NetHide 设备地址，使得 NetHide 设备收到回复，基于此 NetHide 需要**存储数据包的源目的地址**，转发回复给发送者。
	- 以线速重写追踪数据包：NetHide 有时需要在生产流量(production traffic，不影响延迟或时延并且已经在路由器中实现)中修改 TTL，同时需要发送追踪数据包到不同路由器（对观测的 RTT 有影响，但针对 TTL 在到达目的地超时的追踪数据包）
	- 无状态重写数据包：由于在设备中缓存很快会超过限制内存，NetHide 采用更好的策略：替代在设备中保存状态信息，把它**编码进数据包**里，更准确地说，将数据包中添加额外的头部，包括原始的源目的地址，TTL值和签名（哈希值包含额外头部和设备特定秘密值），称为 **meta header**，放在 layer 3 payload 之上
	- 防止数据包注入：数据包到达时检查是否包含 meta header同时签名是否合法，在发送到输出接口时存储源目的地址同时移除meta header
- NetHide 控制器（controller）
	- 配置拓扑：基于 P4 设备，配置入口作为 match+action 表（被包处理程序问询）的入口。NetHide 配置入口格式(des,TTL)-> (virtual des IP, hops to virtual des)，地址采用前缀匹配
	- **分布式修改包**：每个流选择一个可编程设备来处理所有流的数据包，该设备必须位于第一个欺骗节点（如虚拟路径的第一个节点）之前，为了负载均衡，随机选择合格的节点（不影响混淆），为了更多冗余性，每个流划分多个设备
	- **实时修改拓扑**：由于包处理软件和配置表入口的分离
- **部分部署**
	- NetHide 只需要部署在部分设备上足以保护网络，关键因素是对于每个流最多只需要一个节点来修改数据包
- 应对拓扑改变
	- NetHide 通过物理拓扑 P 发送追踪包使得在根据 V 的正确节点超时。P中的改变以2种方式影响 NetHide
		1. 链路添加到 P 或路由行为改变:某些流不再通过用于混淆它们的设备，这可以通过配置多个设备解决。由于 V 还是安全的，没必要立即根据 P 来做出改变，基于 P' 重新计算新的 V' 并部署
		2. 链路从 P 中移除：V 中链路失败不会影响 V 的安全，链路永久移除，重新计算和部署
### 6. 评估
- 实验表明 NetHide：
	- 混淆拓扑同时保持高准确性和实用性
	- 1h 内计算混淆拓扑，即使考虑大的拓扑；计算离线完成，不会影响网络性能
	- 可以抵御时间攻击
	- 部分部署有效
	- 减轻实际攻击
	- 对 debug 工具影响较小
- 指标和方法
	- 指标：平均流密度降低系数（average flow density reduction factor）
	- 数据集：三个公共可用的网络拓扑，小型的 Abilene，中型的 Switch，大型的 US Carrier
	- 参数：P 中所有节点可以作为恶意流量的出入节点；所有链接能力相同；NetHide 只添加虚拟链接而不加节点；每个节点考虑 100 个转发树；ILP solver 设定 maximum relative gap 为 2%；每个配置至少运行 NetHide 5 次取平均
- 保护 vs 准确性和实用性
	- 观察发现：大的拓扑一般以小拓扑结果好，基于这样的事实，更大的拓扑上的小修改比小拓扑在正确性上影响更小，并仍然提供高混淆
- 准确性 vs 实用性
	- 实验表明准确性权重的改变对结果影响较小，这证实了有着高准确性的拓扑一般实用性也较高，路径相似（高准确性），数据包也经相同链路路由（高实用性）
- 搜索空间减少和运行时间
	- 实验表明：只考虑一定数量转发树不会显著降低准确性和实用性，但大大减少了运行时间
- 路径长度
	- 比较 P 和 V 中路径长度差异（大的差异会导致不实际的 RTT 并且泄露关于混淆的信息），虚拟路径比实际路径要短，但差异较小，这支撑了时间信息的泄露主要是最后节点的处理时间，而非传输时延
- 部分部署
	- 在随机位置部署 40% 的 NetHide 设备可保护 60% 的需要混淆的流
- 安全
	- 攻击策略：随机；瓶颈+随机；瓶颈+紧密度（比瓶颈+随机更强，需要更多混淆）
- 案例研究：链路故障检测
	- 大多数链路失败精准反应到虚拟拓扑中
### 7. 常见问题
- 拓扑可以通过分析时间信息去混淆嘛？
- 拓扑可以通过分析链路失败去混淆嘛？
- NetHide 是否与链路访问控制或VLAN兼容？
- NetHide 是否支持负载平衡？
- NetHide 计算的解决方案与最佳值有多接近？
- NetHide 可以与其他指标一起用于计算流量密度？
### 8. 相关工作
- 被动（Reactive）方法
	- CoDef
	- SPIFFY
	- Liaskos 等的系统
	- Nyx
- 主动（proactive）方法
	- HoneyNet
	- Trassare 等的拓扑混淆方法
	- Linkbait
### 9. 结论
- 提出了一种新的，可用于混淆网络拓扑的方法。核心思想是将混淆任务表述为一个多目标优化问题，其中安全性要求被编码为硬约束，可用性被编码为使用准确性和实用性概念的软约束。
- 构建了一个名为NetHide的系统，该系统依靠 ILP solver 和有效的启发式方法来计算兼容的混淆拓扑，并在可编程网络设备上以线速捕获和修改跟踪流量。我们对现实拓扑和模拟攻击的评估表明，NetHide可以对大型拓扑进行模糊处理，同时对可用性影响很小，包括部分部署

## 问题
1. 拓扑混淆与隧道技术比较？
2. V 中虚拟节点如何部署？
3. 平均流密度降低系数比较的是 P 和 V 中相同链路，还是 V 中链路对应的所有 P 中链路？

























