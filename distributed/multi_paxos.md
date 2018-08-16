**文中的日志项指的其实是为日志选择一个index，可以理解为一个实例**

**value可以看成是一轮basic paxos算法确定的值**
___

# Multi Paxos
Basic Paxos完成一个value的确认，Multi Paxos完成多个值的确认。Mutil paxos最简单的方式就是使用多次Basic完成多个value的确认。
## 选择一个日志项
- 当server收到来自client的请求时
	- 找到第一个没有被确认的日志项，即还没有完成一轮Basic Paxos算法的value；
	- 对该日志项进行basic paxos算法；
	- 判断phase1阶段的返回值；
		- 如果返回值中的acceptedValue != null，对该acceptedValue完成basic paxos余下流程，完成后重新选择一个没有被确认的日志项；
		- 否则，完成该日志项的basic paxos余下的算法流程；
![select log entries](https://github.com/HaHaJeff/note/blob/master/distributed/image/select_log_entrues.png)

**红线表示s3处于offline**

假设需要对jmp进行basic paxos
- S1需要为jmp选择log index，Firstunchosenindex是3
- 对3使用basic paxos算法，如果Prepare阶段返回了acceptedValue，完成该acceptedValue的选择，并将s1,s2的slot3设置为chosen，完成后重新选择firstunchosenindex；
- 否则，重新选择firstunchosenindex，图中需要到slot5才能完成算法流程，此时对于S1而言，firstunchosenindex更新为7，同时对于slot3，slot4，S1 S2则会对acceptedValue进行选择；

**允许并行处理log index，但是保证状态机中的log index处于有序的状态！即投入到状态机的时候需要串行执行，因为需要保证有序性！**

## Improving Efficiency
- 使用Basic Paxos是低效的
	- 当许多proposers并行提交proposal时，冲突和重启会变得非常多(高负载会导致更多的冲突)；
	- 对log index进行选择时，需要两种的RPC通信(prepare，accept)

## 解决方案
1.  选择一个leader
	- 只能有一个server扮演proposer，即一个时刻只有一个proposer提交proposal;
2. 减少Prepare RPC
	- 只对一次log index进行确认(而不是对每一次log index)；
	- 大多数log index的确认只需要进行一种RPC通信；(**为什么是大多数？因为在收到多数派的noMoreAccept应答之前，还是需要进行prepare的**)

## leader election
lamport提出了一种简单的方式：
- 让服务器ID最高的成为leader；
- 每一个server定时向其他服务器发送心跳包；
- 如果一个服务器在两个周期内没有收到来自leader的心跳包，则让该server扮演leader；
- 如果server不是leader
	- 拒绝client的请求(或重定向到leader)；
	- 仅仅扮演acceptor；

## Eliminating prepares
- 为什么需要prepare？
	- 丢弃旧的proposals：
		- 让proposal number对每个日志项都有效，而不是仅仅对一个日志项有效；
	- 发现可能已经被选择的value：
		- 返回当前日志项已经接受的最高proposal；
		- 返回 noMoreAccepted：没有接受过比当前日志项的index大的日志项；
- 如果acceptor对proposer的prepare回应noMoreAccepted，则对于该acceptor可以跳过prepare(直到accept操作被拒绝，说明此时有其他leader进行了提交，且propoasl number大于目前的proposla number)；
- 一旦leader收到了多数派acceptors的noMoreAccepted，则以后将不需要prepare RPC；

## Full Disclosure(需要得到已经被chosen的value)
- 目前的Multi Paxos的信息流是不完整的：
	- 日志项没有也全部备份(只有多数派产生了副本) **Goal：全部备份**
	- 只有proposer知道哪个日志项被选择了 **Goal：所有的服务器都知道被选择的value**
- Solution part 1/4：一直在后台尝试发送Accept RPC，直到得到所有的acceptor回应
	- 完成大多数日志项的备份，因为可能有server宕机，宕机的server没有办法完成备份；
- Solution part 2/4：跟踪已经被选择的日志项
	- 标记已经被选择的日志项(log index)：acceptedProposal[log index] = int.max
	- 每一个服务器包含了一个firstUnchosenIndex：该index表明第一个没有被标记为int.max的日志项
- Solution part 3/4：proposer告诉acceptors已经选择的目录项
	- proposer在accept请求中包含(firstUnchosenIndex)
	- acceptor标记所有的日志项：
		- 如果i < request.firstUnhosenIndex
		- 同时 accptedProposal[i] == request.proposal
	-	结果：大多数被选择的value被所有服务器知道

![full_disclosure](https://github.com/HaHaJeff/note/blob/master/distributed/image/full_disclosure.png)

**为什么要求 i < request.firstUnchosenIndex && acceptedProposal[i] == request.proposal?**
- i < request.firstUnchosenIndex表明proposer中小于firstUnchosenIndex的proposal已经被确定了；
- acceptedProposal[i] == request.proposal表示该acceptor中对应的acceptedProposal[i]为proposer发出的，所以accptedValue[6] == proposer.acceptedValue[6]；

**上图中proposer发出accept请求Accept(proposal = 3.4，index = 8，value = v，firstUnchosenIndex = 7)，该Accept请求表明proposer中的firstUnchosenIndex=7，需要请求的日志项为8；**

**accpetor中，日志项1，2，3，5都已经被确认，则firstUnchosenIndex为4；**

**所以对于acceptor，发现日志项6的acceptedProposal[6] == request.proposal，则可以直接将acceptProposal[6] = int.max。(为什么对日志项4无法进行该操作？因为对于acceptor中日志项4的proposer与现在的proposer不一样，所以其acceptedValue[4] != proposer.AcceptedValue[4])；**

- Solution part 4/4：entries from old leaders：
	- Acceptor 在accept之后返回它的firstUnchosenIndex；
	- Proposer接收到acceptor的回复之后，如果proposer.firstUnchosenIndex > reply.firstUnchosenIndex，则proposer在后台发送一个Success RPC；
-  Success(index, v)：notifies acceptor of chosen entry：
	-  acceptedValue[index] = v
	-  acceptedProspoal[index] = int.max
	-  return firstUnchosenIndex

## Client Protocol
- 向leader发送请求；
	- 如果client不知道哪一个server是leader，则client可以向任何server发送；
	- 如果client不是向leader发送请求，则该server会将请求重定向到leader；

- leader知道请求已经被确认且已经被leader的state machine执行才返回； 
- 如果请求超时(e.g：leader宕机)
	- client重新提交请求；
	- 请求最终被重定向到新的leader中；
	- 新leader完成该请求；

## Client Protocol存在的缺陷
- 如果leader在执行完client的请求之后宕机了，但是还没有给client回复
	- client超时之后会再次发送同样的请求，所以同一条请求可能会执行两次

- 解决方案：client为每一个请求嵌入一个独一无二的id
	- server在日志项中包含该id；
	- state machine记录最新的请求；
	- 在state machine执行请求之前，state machine检查：如果请求已经被执行了：
		- 忽略最新的请求；
		- 返回旧的请求；

- 结果：在client不宕机的时候可以保证每条命令只被执行一次。

## Configuratuin Changes
- 系统参数
	- 每台服务器的ID，address；
	- 确定哪些服务器可以形成多数派；

- 一致性算法必须支持configuration的改变：
	- 替换失效的机器；
	- 改变副本数目；

![configuration_changes](https://github.com/HaHaJeff/note/blob/master/distributed/image/configuration_changes.png)

如上图所示的配置变化
- 旧的配置是前三台机器；
- 新配置添加了两台机器；

**假设对同一日志项而言，在旧的配置上完成v1的选择，在新的配置上完成了v2的确认，此时对于同一日志项产生了两个不同的值，违背了paxos算法！**

- Paxos的解决方案：使用日志项管理配置变化：
	- 配置保存为一个日志项；
	- 对该日志项进行备份(和普通日志项一样)；
	- state machine执行该日志项的时候当且仅当该日志项的前a个日志项全部执行执行完，这样就能保证一致性；

![configuration_change_cont'd](https://github.com/HaHaJeff/note/blob/master/distributed/image/congiguration_changes_cont'd.png)

如上图一样，假设a=3，且在日志项1和日志项3都发生了配置变化
- 对于日志项1而言，state machine执行该配置变化C1必须在日志项4之后，所以对于日志项1，2，3而言，使用的配置都是C0；
- 对于日志项3而言，state machine执行该配置变化C2必须在日日志项6之后，所以对于日志项4，5而言，使用的配置都是C1；

**Paxos的解决方案即为滑动窗口，滑动窗口大小为a**