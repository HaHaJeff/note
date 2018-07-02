# Basic Paxos
## Split Votes(分票问题)
- Acceptor只接受第一个到来的value；
- 如果不同的proposer同时产生proposals，可能没有value能够被chosen；
![split_votes](https://github.com/HaHaJeff/note/blob/master/distributed/image/split_votes.png)

**所以acceptors必须能够accept不同的values**

## Conflicting Choices
- Acceptor接受每一次的value；
- 可能会出现choose多个values；
![conflict_choices](https://github.com/HaHaJeff/note/blob/master/distributed/image/conflict_choices.png)

**所以一旦一个value已经被选择了，要求以后的proposals将会选择同样的value，于是出现两阶段协议**

使用两阶段提交协议，s5先广播prepare报文，如果已经有value被选择了，则直接对该value进行一致性请求，否则自定义value

## Conflicting Choices, cont'd
![conflict_choices_cont'd](https://github.com/HaHaJeff/note/blob/master/distributed/image/conflict_choices_cont'd.png)
- s5在s1 accept(red)之后提交提案，**此时并没有value被选择**
- 依旧会出现选择两个不同的value

**所以，现在的问题是如何拒绝掉其中的一个值**

## Proposal Numbers
acceptor无法直接根据value确定其属于哪一次的proposal
- 每一次提案都有一个独一无二的number；
	- higher numbers拥有更高的优先级；
- 一种简单的实现方式

![proposal_number](https://github.com/HaHaJeff/note/blob/master/distributed/image/proposal_number.png)

- 每一个server保存maxRound；
- 产生一个新的proposal number；
	- 增加maxRound；
	- 与server id进行组合；
- proposers必须将maxRound持久化，这样可以保证在重启或宕机之后不会重用以前的number；

## 两阶段协议实现方式：
- phase 1:  广播prepare RPCs
	- 确定是否有value已经被选择；
	- 确认旧的proposals已经完成；
- phase 2: 广播accept RPCs
	- 请求acceptos接受一个指定的value；
### Basic Paxos flow
![basic_paxos](https://github.com/HaHaJeff/note/blob/master/distributed/image/basic_paxos.png)

**Acceptor必须持久化 minProposal(最小的将要被接受的提案), acceptedProposal(已经接受过的提案), acceptedValue(已经接受过的value)**

## Basic Paxos Examples
1. 已经有value被选择：
	- proposer将会在phase1发现它并且使用它；	

![basic_paxos_examples](https://github.com/HaHaJeff/note/blob/master/distributed/image/basic_paxos_examples.png)

2. 没有value被选择，但是已经有acceptors接受过该value，且s5可以发现它；
	- proposer将会在phase1发现它并且使用它；	
	- s1, s2, s3, s4, s5都会成功的完成value 的chose，因为对于value没有发生冲突；

![basic_paxos_examples_cont'd](https://github.com/HaHaJeff/note/blob/master/distributed/image/basic_paxos_examples_cont'd.png)

3.  没有value被选择，但是已经有acceptors接受过该value，且s5不可以发现它；
	- s5发出的提案可以被选择，因为proposal number的存在；
	- s1的proposal将会失败；
	- 
![basic_paxos_examples_cont'd_2](https://github.com/HaHaJeff/note/blob/master/distributed/image/basic_paxos_examples_cont'd_2.png)

## liveness(活锁)

![liveness](https://github.com/HaHaJeff/note/blob/master/distributed/image/liveness.png)

- 解决方案1：出现冲突的时候，在重新开始算法执行流程之前，随机的延迟；**让其他proposes有机会完成value的确认**
- 解决方案2：multi-paxos会使用leader来避免活锁；

## Paxos中值得注意的：
- 只有发起proposal的proposer知道哪一个value被选择了；
- 如果其他proposer想要知道，则必须执行一次paxos算法；

参考文献

[1] [paxos made simple](https://github.com/HaHaJeff/note/blob/master/distributed_sys/doc/Paxos/paxos-simple.pdf)

[2] [Diego Ongaro的ppt以及视频](https://github.com/HaHaJeff/note/blob/master/distributed_sys/doc/Paxos/paxos.pptx)

**Diego Ongaro：raft作者**