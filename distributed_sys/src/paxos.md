# Basic Paxos
## Split Votes(分票问题)
- Acceptor只接受第一个到来的value；
- 如果不同的proposer同时产生proposals，可能没有value能够被chosen；
![split_votes](https://github.com/HaHaJeff/note/blob/master/distributed_sys/image/split_votes.png)
** 所以acceptors必须能够accept不同的values **

## Conflicting Choices
- Acceptor接受每一次的value；
- 可能会出现choose多个values；
![conflict_choices](https://github.com/HaHaJeff/note/blob/master/distributed_sys/image/conflict_choices.png)
** 所以一旦一个value已经被选择了，要求以后的proposals将会选择同样的value，于是出现两阶段协议 **

使用两阶段提交协议，s5先广播prepare报文，如果已经有value被选择了，则直接对该value进行一致性请求，否则自定义value

## Conflicting Choices, cont'd
![conflict_choices_cont'd](https://github.com/HaHaJeff/note/blob/master/distributed_sys/image/conflict_choices_cont'd.png)
- s5在s1 accept(red)之后提交提案，**此时并没有value被选择**
- 依旧会出现选择两个不同的value
**所以，现在的问题是如何拒绝掉其中的一个值**

## Proposal Numbers
acceptor无法直接根据value确定其属于哪一次的proposal
- 每一次提案都有一个独一无二的number；
	- higher numbers拥有更高的优先级；
- 一种简单的实现方式
<div align=center>
![proposal_numbers](https://github.com/HaHaJeff/note/blob/master/distributed_sys/image/proposal_numbers.png)
</div>
- 每一个server保存maxRound；
- 产生一个新的proposal number；
	- 增加maxRound；
	- 与server id进行组合；	
- proposers必须将maxRound持久化，这样可以保证在重启或宕机之后不会重用以前的number；