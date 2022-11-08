# Paxos

## The Consensus algorithm
The safety requirements for consensus are:
- Only a value that has been proposed may be chosen
- Only a single value is chosen, and
- A process never learns that a value has been chosen unless it actually has been.

Roles or agents:
- proposers
- acceptors
- learners

A process可能担任多种角色, 不同角色之间的通信是异步的，非拜占庭问题的.  
- 不同角色有不同执行速度，可能失败停止重启.在某个值被选中之后，可能失败重启，因此需要让记住已经被选中的值.  
- 角色之间的消息的投递时间未知，可能重复，可能丢，但是不会是`corrupted`.  

## Choosing a Value
选择一个值最简单的方法就是只有一个`Acceptor`.但是这唯一的Acceptor挂了就完了.  
因此需要多个`Acceptor Agents`.  
选中的值就是大多数`Acceptor Agents`都接受的值.  
`但是每个Acceptor最多接受1个值.`  

某一种边界条件：如果只有一个人提出了一个提案，也必须得保证这个提案肯定会被选中.  
<font size=10>为了避免在以上情况选不出值</font>，使用以下约束：  

<font size=5>$P1.$ `An Acceptor must accept the first proposal that it receives.`</font>  

引出了一个问题: 如果所有的Acceptor都接受第一个提案的值, 那么可能会出现正好一半的人接受的1个值，另一半的人接受的另一个值，那么结果就是没有任何一个提案会被选中。  
因此一个Acceptor只接受第一个提案是不行的.  
于是我们允许一个Acceptor可以接受多个提案, 只能接受一个值也是不行的,因为如果一个acceptor接受了一个提案,值为v, 如这个acceptor不再接受其他值,如果正好也是一半的acceptor接受了v, 另外一半接受了v2,那么这一半acceptor不会再接受v2,另一半acceptor不会再接受v,那么任何一个值都不会被选中了.  
因此我们需要每个acceptor可以接受`多个提案`, `多个值`.  

给每个提案分配一个唯一递增ID，一个提案包括了这个ID和提案对应的值。  
我们可以允许多个提案被`选中`，但是这些提案的值都得是一样的, 也就是说只有一个值会被选中.  

<font size=10>为了保证只会有一个值会被选中</font>,有了$P2$:  

<font size=5>$P2.$ `If a proposal with value v is chosen, then every higher-numbered proposal that is chosen has value v.`</font>  
也就是如果提案(n,v)已经被选中,那么之后所有ID > n的提案中被选中的提案的值一定是v.   
因为ID是递增的,可以假设(n,v)是所有被选中的提案中id最小的提案,那么由$P2$,其他所有提案(因为其他的提案ID都大于n)的值都是v,因此$P2$保证了只会有一个值会被选中.  

用$P2^a$来满足$P2$:  

<font size=5>$P2^a$. `If a proposal with value v is chosen, then every higher-numbered proposal accepted by any acceptor has value v`.</font>  

意思是:如果(n,v)已经被选中, 那么所有的ID > n的提案中,只有value=v的才会被接受.  
由$P2^a$推导到$P2$很直观.  
$P2^a$是用来限制Acceptor的, 但是Acceptors也不知道哪个值被选中了啊,他们只知道自己接受过哪些值,于是把这一限制转嫁到Proposer.  

于是有了$P2^b$:  

<font size=5>$P2^b$. `If a proposal with value v is chosen, then every higher-numbered proposal issued by an proposer has value v.`  </font>

这一条是对proposer的限制，确实可以满足$P2^a$, 即当某个提案(id1, value1)被选中之后，其他proposer所提的所有`id > id1`的提案都必须满足: `value = value1`.  
如何做到呢？ 
其实也不好做到, 比如proposer1提出的提案(id1, value1), 是否被acceptor接受,是否已经被选中，只有proposer1自己知道, 其他proposer无法知道, 因此无从得知这个值是否已经被多数派接受了.  

为了推出$P2^b$, 我们借助$P2^c$  

假设现在proposal1 (m,v)已经被选中  

<font size=5> $P2^c$. `For any v and n, if a proposal with value v and number n is issued, then there is a set S consisting of a majority of acceptors such that either`</font>  
<font size=5>`(a) no acceptor in S has accepted any proposal numbered less than n, or `</font>  
<font size=5>`(b) v is the value of the highest-numbered proposal among all proposals numbered less than n accepted by the acceptors in S.`</font>

一个proposal2 (n,v2)被提出的条件是存在一个多数派Acceptor集合S要么  
- (a) 每一个Acceptor in S都没有接受过小于n的提案, v2就可以随便. 要么  
- (b) S中有人接受过小于n的提案, 那么v2必须等于这些ID < n的提案中最大ID的提案值.  
<font size=5>注意这里得到的最大ID并不一定就是所有接受过小于n的提案中ID最大的，而是当前proposer收到的回复中ID最大的，因为当前proposer收到的回复中，不一定有ID最大的那个提案.</font>  

$P2^c$让proposer在提出提案的时候有了限制, 限制的目的就是为了满足$P2^b$, 其实就是为了让proposer能够知道当前的进展.   

<font size=4>`首先看(a)`</font>, (a)要求一个proposer在提出一个ID为n的提案之前,存在一个多数派Acceptors集合S, S里的所有Acceptor都没有接受过小于n的提案.  
也就是说一个proposer在提出提案(n,v)之前,先去找是否存在一个多数派集合,里面的Acceptor都没有接受过小于n的提案.  
这条限制是满足$P2^b$的:  
假设条件(a)已经满足, 于是提出了提案(n,v), 因为已经有个多数派表示没有接受过小于n的提案,所以小于n的提案不可能已经被选中, 因为连$P2^b$的`IF`条件都不成立.  
有两个问题:  
- 怎么找呢? 一个一个问么?  
- 即便一个proposer找到了一个多数派集合说当前没有接受过小于n的提案,那么在多数派中的最后一个acceptor告诉这个proposer之后,在proposer提出提案(n,v)之前,Acceptors可能又接受了小于n的提案了, 这个信息时效性如何保证呢?   

后面看这些问题怎么解决的.  

<font size=4>`再看(b)`</font>,一个proposer在提出提案(n,v)之前,虽然这个proposer没有找到一个多数派都没有接受过小于n的提案, 但是收到了多数派集合S的回复，S中有人接受过ID小于n的提案,里面接受的ID < n 并且最大的提案的值是v.  
(b)也是满足$P2^b$的:  

<font size=4>前提条件有:</font>   
- 值为 $v$ 的提案已经被选中, 我们假设这些值为 $v$ 的提案中ID最大的提案是 $(m, v)$ (这是 $P2^b$ 的 `IF` 条件)
- 提案提出受(b)限制,假设在(b)的限制下,提案 $(m+1,v')$ 可以被提出, $v'待定$ 

很显然, $v' = v$ . 因此对于任何 提案$(n, v'), n > m$ ,都有 $v' = v$   
以此类推,任何大于m的提案的值都必须是v  

于是$P2^c$可以推导出$P2^b$.  
要想做到$P2^c$ 的(b), 就得在提出提案(n,v3)之前,知道当前ID < n中ID最大的`已经`或者`将要`被多数派接受的提案的值.  

如何得知是否有一个多数派集合S, 里面已经有人接受过小于n的提案呢?  
如果在多数派集合S中, 已经有人接受过小于n的提案,那么这个多数派中ID < n 且最大的提案的值是多少呢?  
由此引入了`PROMISE`. 

在$P2^c$的限制下,论文中将proposer提出提案的步骤被分成了两步:  

<font size=4>1. PREPARE  </font>  
Proposer首先获取一个ID(n),然后把这个n发给Acceptors,期望acceptors回复:  
(a) a promise never again to `accept` a proposal numbered less than n, and  
(b) The proposal with highest number less than n that it has `accepted`, if any.  

如果没有多数派的回复, 说明不存在一个多数派能够给出这个承诺(比如某些Acceptors已经承诺过不再接受 $ID < n', n' > n$)的提案, 那么当前提案肯定是不能被提出的,因此只考虑收到了多数派回复的场景.
多数派的回复可能两种情况:  
- 承诺不再接受小于n的提案,并且当前没有任何人接受过小n的提案.  
这时proposer没有得到任何值可以用,因此当前proposer可以提任何值.对应 $P2^c$ 中的(a).  
- 承诺不再接受小于n的提案,并且多数派中已经接受的提案中ID最大的且 $id < n$ 的值为v.  
由于当前proposer收到了值v,因此当前proposer只能提值为v的提案. 对应 $P2^c$ 中的(b)    
但是这条规则比 $P2^c$ 中的(b)要严格, 因为即便满足了当前条件, 也不一定存在某个值已经被选中了, 可能只是有一个值为v的提案被接受了, 并且正好被这个proposer发现了. 因此看起来这条规则可以加快选中流程, 会让其他proposer向其他已经被Accept的proposer提交的值学习.  

<font size=4>2. COMMIT  </font>  
在PREPARE之后, proposer再发起一个COMMIT请求,提案内容为 $(n, v)$  

再回头看那两个问题, 论文中使用的方法就是挨个询问, 引入承诺可以解决第二个问题, 只要acceptor回复了,那么它就承诺了不会再接受小于n的提案了.  

PREPARE和COMMIT限制了proposer提提案时 的行为, 对于Acceptor, 只需要它们`遵守承诺`就行了.  
于是将 $P1$ 转化为 $P1^a$  
<font size=5> $P1^a$. `An acceptor can accept a proposal numbered n iff it has not responsed to a prepare request having a number greater than n.`  </font>

显然, $P1^a$ 满足 $P1$  

<font size=5>于是, $P1^a + P2^c$ 满足了选值的唯一性.  </font>

论文中还提到了1个优化:  
是对PREPARE和COMMIT请求的回复, 如果某个Acceptor已经承诺过不再接受 $ID < n$ 的提案, 如果有个 $ID < n$ 的PREPARE请求, 其他这个Acceptor完全可以不回复这个PREPARE请求, 即便他只是承诺不接受COMMIT, 因为即便它回复了PREPARE, 这个Acceptor也不可能接受COMMIT, 那又何必浪费自己的时间呢?不仅如此,其实这个Acceptor也可以帮助一下这个Proposer,叫他也别浪费时间了,你这个提案`可能`大家都不会接受了, 因为已经有更新ID的提案正在进行中.  

## Learning a Chosen Value
好了,我们已经选出一个值,并且保证了唯一性了.  
首先Acceptors知道已经存在一个选中的值了么?
很显然, Acceptors肯定不知道, 他只知道自己接受了谁.  
Proposers知道么?  
知道, 当他发出了Accept  request之后，如果收到了多数派回复，那么他就知道这个值被选中了。  

论文中引入了`Learner`这个角色.  
所有Acceptors在接受一个值之后会通知 `Distinguished Learner`, 告诉它接受了谁, `Distinguished Learner`可以通过这些信息得知谁被选中了. 如果其他Learner需要知道这个消息,那么`Distinguished Learner`可以再挨个通知他们.  

这个方式中需要每个Acceptor给`Distinguished Learner`发送一个消息, 同时需要通知所有的Learners,告诉他们这个消息.  
一共需要 $Num_{Acceptors} + Num_{Learners}$ 个消息.  

有一种情况, 可能一个值被选出来之后,由于网络等问题, 最后一个Acceptor在通知Learner的时候消息丢失了, 那么Learner可以让某一个Proposer尝试提出一个新的提案, 这个新的提案会驱动某个值被选中.  

****

# Multi-Paxos

****
# Raft

