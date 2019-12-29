本文是6.824 lab2的实验笔记。



# Lab 2: Raft

>  https://pdos.csail.mit.edu/6.824/labs/lab-raft.html

## 简介

Lab2的目的是实现Raft。这部分内容将作为后续两个实验的基础：后续的两个实验都是基于Raft来实现一致性和容错的。

Lab2分为A，B，C三个部分。

Lab2A的目的是实现leader选举以及心跳包机制，Lab2B则是在2A基础上完善日志新增和日志复制相关的功能。最后一部分Lab2C则是需要实现状态持久化。通过整个测试需要正确应对网络分区、RPC延迟/乱序、节点重启等情况。

## 参考资料

在实现lab2的过程中，除了Raft-extened论文和实验说明之外，我发现以下的参考资料非常有用：

1. [Raft lecture](https://www.youtube.com/watch?v=YbZ3zDzDnrw)

   Raft论文作者Diego Ongaro发布的Raft讲座。似乎就是Raft论文中Understandability一节所提到的video lecture，用于比较Raft和Paxos的可理解性。Paxos的部分也可以找到。时长大约1个小时，讲得非常清晰，不过其中不包含持久化和快照的部分。

2. [Raft Locking Advice](https://pdos.csail.mit.edu/6.824/labs/raft-locking.txt)

   实验说明中提到的关于锁的使用的建议。如果你无法在-race选项下通过测试，最好读一读这个 。

3. [Studentes's Guide to Raft](https://thesquareplanet.com/blog/students-guide-to-raft/)

   6.824助教写的一份实验指导。提到了许多常见问题。另外详细描述了论文中一笔带过的日志冲突时如何回溯nextIndex的优化方案。

此外还有其他一些有用的博客，笔记等。


## 实现

---

![raft-figure 2](/assets/raft-figure 2.png)

论文中的图2是一份十分详细的实现指南，它指明了raft服务器之间的各种行为。

---

### Raft结构体

图2中提到的服务器应当保存的状态有：

**所有服务器（应当持久化的状态）**

- currentTerm 当前任期
- votedFor 当前任期投票对象
- log[] 日志记录

> 由于lab中检测日志记录从索引1开始，所以初始化slice时可以在其索引0处放入空日志项。或者最好的是将其作为 logs 类型并在其上定义get等方法来访问而不是使用log[i]来访问，这样可以很方便的修改以适应后续实验中快照和日志压缩的需要。

**所有服务器上的变量**

- commitIndex 已提交日志索引
- lastApplied 已应用日志索引

**leader服务器上的变量**

- nextIndex[] 发送AppendEntries RPC时应当包括的下一条日志索引
- matchIndex[] 记录follower实际匹配的日志索引

> 在无差错的情况下，nextIndex[i]应当等于matchIndex[i]+1 。实际上nextIndex是leader发送RPC之前对follower日志匹配状况的**猜测**，在成为leader时初始化为log长度减1。而matchIndex则是通过成功完成的AppendEntriesRPC来确定的日志匹配程度的**实际状况**，初始化为0。



此外还需要一些用于程序的变量：

- state 状态，为follower/candidate/leader之一
- timer 计时器，用于重置选举周期。
- applyCh lab2中用于测试raft应用日志项的channel

- 以及其他可能需要的用于锁定/同步/通知的变量



### RPC handler

基本的Raft实现只有两种RPC，分别为RequestVoteRPC和AppendEntriesRPC。两者的参数，结果以及处理规则都在图2中有着详细的描述。只需要**严格按照图2中的描述实现**即可。



#### RequestVote RPC

candidate向所有其它服务器请求投票的RPC。

**参数**

- Term	发送者任期
- CandidateId    发送者Id
- LastLogIndex    发送者最后一条日志的索引
- LastLogTerm    发送者最后一条日志的任期

**结果**

- Term 任期
- VoteGranted 是否同意本次投票

**接收者实现**

1. 如果参数中的任期小于接收者当前任期，拒绝投票
2. 如果接收者的votedFor为空或者等于候选人id，且候选人的日志**至少和自己一样新**，那么同意投票。（其他所有情况下都拒绝投票）

> 什么是比较新的日志？
>
> 1.(最后一条)有着更新的任期
>
> 2.任期一致时，日志总长度更长的日志比较新。

此外，还需要实现下方RulesForServers中提到的：

1. 在发送或者收到RPC回复时，检查任期。如果大于当前任期，转变自身状态为follower并更新任期。
2. 对于follower，在它**同意**投票请求时重置选举超时时间。

**所以具体的实现应该是：**

​	首先检查任期，如果任期比自己的新，更新自己的任期并继续（更新任期时要将votedFor设为空，因为每个server每个任期有一次投票权）；如果任期比自己的旧，那么直接回复false并包含自己的任期数。

 	如果执行上述部分后没有返回，那么现在任期应该与请求参数中的任期一致。检查自己这个任期是否投过票，或者给该请求的候选人投过票。如果没投过票或者投票给了该candidate（已经投票给该candidate的话可能是由于上次RPC的回复丢失了，所以candidate重发了请求），那么继续检查日志是否至少一样新。

​	继续检查参数中的最后一条日志任期/索引是否和自己的一样新或者更新：

	- 如果是，同意投票并更新自己的选举超时时间（重置选举计时器，给candidate完成选举留出时间）。
	- 如果不是，回复false。（如果之后一直到选举超时仍没有有效投票或者来自自己所认可的leader的AppendEntriesRPC那么自身应该转变为candidate）。

**实现代码如下**

```go
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
	// Your code here (2A, 2B).
	rf.Lock()
	defer rf.Unlock()
	rf.updateTerm(args.Term)
	reply.Term = rf.currentTerm

	if args.Term < rf.currentTerm {
		reply.VoteGranted = false
		return
	}
	if rf.votedFor == -1 || rf.votedFor == args.CandidateId {
		index, term := rf.getLastLogInfo()
		// candidate's log is at least as up-to-date as receiver's log
		// bigger Term || same Term ,more logs
		if term < args.LastLogTerm ||
			(term == args.LastLogTerm && index <= args.LastLogIndex) {
			rf.votedFor = args.CandidateId
			reply.VoteGranted = true
			rf.resetTimer()
			rf.persist()
			return
		}
	}
	reply.VoteGranted = false
	return
}
```



#### AppendEntries RPC

**参数：**

- Term 发送者任期
- LeaderId 可以让follower将客户端请求重定向至leader
- PrevLogIndex 将要Append的日志项(entries[])的前一条记录索引
- PrevLogTerm 将要Append的日志项(entries[])的前一条记录任期
- Entries[] 要添加的日志项，如果用于心跳可以为空。
- LeaderCommit leader已提交的日志项索引

**结果：**

- Term 接收者的当前任期
- Success 是否成功（如果follower有匹配PrevLogIndex/PrevLogTerm的记录则成功）

- ConflictIndex 冲突日志索引
- ConflictTerm 冲突日志任期

> ConflictIndex/ConflictTerm用于在AppendEntries失败（follower不包含PrevLogIndex/PrevLogTerm匹配的日志即失败）时，leader快速向前回溯。而不是一次RPC向前移动一条并重试。论文5.3节末尾处提到了可以通过ConflictIndex和ConflictTerm来优化，而[Studentes's Guide to Raft  #The accelerated log backtracking optimization...](https://thesquareplanet.com/blog/students-guide-to-raft/#an-aside-on-optimizations)则详细描述了一种方案。

**接收者实现：**

1. 如果参数中任期<当前任期，回复false和自己的任期
2. 如果自己的日志中没有与PrevLogIndex/PrevLogTerm匹配的项（不包含该索引或者索引处任期不一致），回复false。
3. 如果在接收者的日志中存在与Entries[]中冲突的项，那么删除这条已经存在的记录和它之后的记录。
4. 将Entries[]中接收者还没有的日志项添加到自身日志中。
5. 如果LeaderCommit> CommitIndex，将自身的CommitIndex设置为 min(LeaderCommit, indexOfLastNewEntry)

> 如果存在与Entries[]中冲突的项才删除记录必须被严格遵守，不能将其简单地实现为PrevLog匹配时删除PrevLogLogIndex之后的条目并添加Entries[]到日志末尾：如果有RPC乱序，这种实现会导致follower存在删除已经提交的记录的可能性。
>
> （例如：leader新增日志项5（指索引）的时候发送了一次RPC，被延迟了;它在新增日志项6之后又发送了一次RPC并最终得到了成功的回复。如果只有3台服务器那么此时leader会认为日志6已提交（2 of 3）；而上述错误的实现则会让follower在收到延时的RPC时删除日志6，如果此时leader恰好挂掉了而另一台服务器也没有该记录的话就会出现已提交日志6丢失，不再满足线性一致性。

其他规则（图中RulesForServers部分）：

1. 在发送或者收到RPC回复时，检查任期。如果大于当前任期，转变自身状态为follower并更新任期。
2. 作为candidate时，如果收到了来自新leader的AppendEntriesRPC，转变为follower。（上面的一条是currentTerm<term的情况，所以这一条指的是上一条中不包含的currentTerm=term的情况，等于的情况如果不转为follower对一致性倒是没有影响，但是会有一些不必要的行为）

日志回溯优化：

1. 如果follower在prevLogIndex处没有日志项（log[]过短），它应回复false以及ConflictIndex = len(log) 和ConflictTerm=None。
2. 如果follower在PrevLogIndex处有日志项但是该日志项的任期与PrevLogTerm不匹配，回复false以及ConflictTerm = log[PrevLogIndex].Term。而ConflictIndex应当设置为它的日志中该任期的第一条记录的索引。

**代码：**

```go
func (rf *Raft) AppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply) {
	rf.Lock()
	defer rf.Unlock()
	//1.Reply false if Term < currentTerm
	if args.Term < rf.currentTerm {
		reply.Term = rf.currentTerm
		reply.Success = false
		return
	}

	// reset timer when receiving AppendEntries from current leader
	rf.resetTimer()

	// update currentTerm
	rf.updateTerm(args.Term)
	if rf.state == Candidate && args.Term == rf.currentTerm {
		rf.state = Follower
	}

	// 2.Reply false if log doesnt contain an entry at PrevLogIndex
	// whose Term matches prevLogTerm

	// conflictIndex & conflictTerm
	logEntry := rf.getLogEntry(args.PrevLogIndex)
	if logEntry == nil{
		reply.Term = rf.currentTerm
		reply.Success = false
		reply.ConflictIndex=len(rf.log)
		reply.ConflictTerm=0
		return
	}
	if logEntry.Term != args.PrevLogTerm {
		reply.Term = rf.currentTerm
		reply.Success = false
		reply.ConflictTerm = logEntry.Term
		reply.ConflictIndex = rf.firstIndexOfTerm(logEntry.Term)
		return
	}

	// 3.If an existing entry conflicts with new one...
	// 4.Append any new entries not already in the log
	rf.addEntries(args.PrevLogIndex, args.Entries)

	// 5.If leaderCommit > commitIndex ,set commitIndex = min(leaderCommit,index of lastNewEntry)
	if args.LeaderCommit > rf.commitIndex {
		lastNewEntry := args.PrevLogIndex + len(args.Entries)
		rf.commitIndex = min(args.LeaderCommit, lastNewEntry)
		rf.cond.Broadcast()
	}
	rf.persist()
	reply.Term = rf.currentTerm
	reply.Success = true
	return
}

```



### 选举超时

依照Rules For Servers中的描述，如果选举超时：

- Follower应该转变为Candidate
- 一旦转变为Candidate，应当立即
  - 将任期+1
  - 投票给自己
  - 重置选举计时器
  - 发送RequestVoteRPC

- Candidate选举超时时应当开启一次新的选举

所以为了监测选举超时状况，需要一个计时器以及一个goroutine。

我使用time.Timer作为计时器。选定一个基础周期T如T=300ms，重置时将其重新设置为T到2T之间的一个随机值，以避免出现选票瓜分的状况。

**代码**

```go
func (rf *Raft) electionWatcher() {
	for range rf.timer.C {
		if rf.killed() {
			return
		}
		rf.Lock()
		rf.cond.Broadcast()  // ×
		switch rf.state {
		case Leader:
			rf.resetTimer()	// ×
		case Follower, Candidate:
			rf.startElection()
		}
		rf.Unlock()
	}
}
```

> 注：此处代码中对Leader的resetTimer()和rf.cond.Broadcast()两行代码源于我对于应用日志实现的错误：我使用一个goroutine来进行apply动作，并且使用rf.cond.Wait()来让其等待信号。大概像这样。然后在所有更新commitIndex的地方调用rf.cond.BroadCast()。
>
> for{
>
> ​	rf.mu.Lock()
>
> ​	rf.cond.Wait()
>
> ​	rf.applyMsg()
>
> ​	rf.mu.Unlock()
>
> }
>
> 如果如果去掉上述两行，仅靠更新commitIndex处的Broadcast通知，会出现在wait之前就Broadcast的情况。这样的话在TestConcurrentStarts2B中会有很低的概率失败（<10%）。
>
> 这处错误是我在写这篇实验笔记的时候发现的，因为我发现此处的leader重置Timer和Broadcast是多余的操作，但我将其删除之后会有小概率在测试中失败。之后我删掉了cond，去掉了此前用于监测apply状况的goroutine，并将所有之前调用其Broadcast的地方换成了rf.applyMsg()  。



### 开始选举

当一个服务器开始一次选举，那么：

- 使当前状态为candidate
- 将当前任期+1,并将这个任期的票投给自己
- 重置选举计时器
- 向所有的其它服务器发送RequestVote RPC请求投票
- 在选举计时器超时前等待投票结果
- 如果收到的选票过半，那么转变为leader状态
- 如果在超时之前没有收到过半的选票，则重新开始选举（重复上述过程）
- 如果在上述过程中收到了任期比自己新的任何RPC或者RPC回复或者收到了当前任期的leader的心跳，终止选举并转变为follower。

所以一个开始选举的实现应该包括发送RequestVoteRPC和接收并处理对应回复两部分。

**代码：**

```go
func (rf *Raft) startElection() {
    rf.state=Candidate
	rf.currentTerm++ //safe <- under Lock
	rf.votedFor = rf.me
	rf.persist()
	rf.resetTimer()
	replyCh := make(chan *RequestVoteReply, len(rf.peers))

	rf.broadcastRequestVotes(replyCh)
	rf.handleReqeustVoteReply(replyCh)

}
```

#### 广播RequestVote

为了防止阻塞以及满足并发发送RequestVote的需要。应当在一个单独的goroutine中进行每一次发送（即调用sendRequestVote()方法）。

发送的参数应当为图中描述的Term，CandidateId，LastLogIndex以及LastLogTerm。获取这些数据时需要加锁避免data race，但是执行发送时不能加锁。

RPC调用失败（超时，返回值不ok）时应当重试。[Raft lecture](https://www.youtube.com/watch?v=YbZ3zDzDnrw)对此的说法是"over and over again"。不过对本lab而言，是否重试对测试结果都没有影响。

**代码：**

```go
func (rf *Raft) broadcastRequestVotes(replyCh chan *RequestVoteReply) {
	for i := 0; i < len(rf.peers); i++ {
		if i == rf.me {
			continue
		}
		args := new(RequestVoteArgs)
		args.Term = rf.currentTerm
		args.CandidateId = rf.me
		args.LastLogIndex, args.LastLogTerm = rf.getLastLogInfo()
		go func(id int) {
			for i := 0; i < 1; i++ {
				reply := new(RequestVoteReply)
				ok := rf.sendRequestVote(id, args, reply)
				if ok {
					replyCh <- reply
					return
				}
			}
		}(i)
	}
}
```

#### 处理RequestVote回复

使用一个单独的goroutine来处理所有RequestVote的回复。

- 用votes变量来统计收到的同意票数量，当同意票过半时，转变自身状态为leader。
- 如果在此过程中收到了来自任期更新的回复，则应当结束选举，更新任期并转变为follower。
- 如果此过程中任期/自身状态与发送请求时发生了改变，应当终止这一过程。
- 使用select来实现超时退出而不要一直等待replyCh的回复



### 选举成功及Leader行为

如果选举成功应当转变为leader

- 服务器应当从candidate状态转变为leader状态
- 初始化nextIndex[] 和matchIndex[] 

- 向所有其他服务器发送初始化的空AppendEntriesRPC以维持本次选举结果；并在一定的空余时间之后不停的重复发送，以阻止跟随者超时。

其它Leader行为

- 如果收到来自客户端的请求，将其应用到本地日志，并在其应用到状态机后回应。（本lab中Start()为立即回应，而通过applyCh来确认应用状况）

- 如果自身的最后一条日志索引>=nextIndex[i]，使用该nextIndex值向i发送AppendentriesRPC
  - 如果成功，更新对应的nextIndex和matchIndex
  - 如果失败，减小nextIndex[i]并重试
- 如果存在一个满足`N > commitIndex`的 N，并且大多数的`matchIndex[i] ≥ N`成立（即N为matchIndex[]的中位数），并且`log[N].term == currentTerm`成立，那么令 commitIndex 等于这个 N。



#### **选举成功立即转变为leader并发送心跳**

```go
func (rf *Raft) beLeader(term int) {
	rf.Lock()
	defer rf.Unlock()
	if rf.currentTerm != term {
		return
	}
	rf.state = Leader
	rf.initNextIndex()
	go rf.heartBeats()
}

func (rf *Raft) heartBeats() {
	for !rf.killed() {
		rf.Lock()
		rf.heartBeatsTimer.Reset(HeartBeatCycle)
		isLeader := rf.state == Leader
		rf.Unlock()
		if !isLeader {
			return
		}
		rf.broadcastAppendEntries()
		<-rf.heartBeatsTimer.C
	}
}
```

#### 发送AppendEntries并处理回复

发送AppendEntries和处理回复时，需要注意[Raft Locking Advice](https://pdos.csail.mit.edu/6.824/labs/raft-locking.txt)中提到的一些情况。

**参数：**

- Term 发送者任期
- LeaderId 可以让follower将客户端请求重定向至leader
- PrevLogIndex 将要Append的日志项(entries[])的前一条记录索引
  - =nextIndex[i]
- PrevLogTerm 将要Append的日志项(entries[])的前一条记录任期
  - =nextIndex[i].Term
- Entries[] 要添加的日志项，如果用于心跳可以为空。
  - 包含自身日志中PrevLogIndex之后的项目
- LeaderCommit leader已提交的日志项索引

**处理回复**

- 首先如果回复表明自身任期过旧，应当转变为follower

- 此外，为应对乱序延迟等情况，忽略任期不一致的情况。

- 如果reply.Success==false，应当减少nextIndex[i]并重试（根据nextIndex[i]重新生成参数）

  最简单的方案是将nextIndex每次减少1,但这样会使测试中一些unreliable的情况因超时而无法通过。

  优化方案是使用ConflictIndex/ConflictTerm：

  ​	简单方案是使得nextInex[i]=ConflictIndex而忽略ConflictTerm

  ​	充分利用的方案是在将nextIndex[i]移动到log中任期等于conflictTerm的最后一项处，如果log中没有ConflictTerm的项，则令nextInex[i] = ConflictIndex

- 如果reply.Success==true，表明本次更新成功了。将对应的matchIndex[i]更新。应当使用发送参数来更新。而且需要注意处理回复乱序时的情况（让其只增不减）。

  ​    matchIndex=max(PrevLogIndex+len(Entries[]),matchIndex)

- 如果更新了matchIndex，检查自己的commitIndex是否也可以更新。即检查多数matchIndex>N的N处任期是否等于当前任期。大于当前任期则更新commitIndex。

- 如果更新了commitIndex，应根据lastApplied将日志中已提交未应用的项目应用到状态机

#### 处理客户端请求

lab中使用rf.Start(command interface{}) (int, int, bool) 来处理客户端命令。该函数应当立即响应并回复自身任期，新增命令在日志中的索引和是否是leader。

非leader（只包含服务器认为自己不是leader的情况）应当返回自己不是leader，对传入的命令不作处理。而leader（认为自己是leader）应当将命令添加到自己的日志中，然后立即向其他服务器发送AppendEntries请求。

### 所有服务器上的行为

- **应用日志项到状态机**

  如果`commitIndex > lastApplied`，那么就 lastApplied 加一，并把`log[lastApplied]`应用到状态机中。

  可以在所有改变commitIndex的地方触发该检查。

- **如果RPC请求或回复的任期>自身任期，更新任期并转变自身状态为follower**

  在所有处理RPC请求或者回复的地方都需要检查任期。

### 持久化

根据论文中的描述，只有3个状态需要持久化：

- CurrentTerm
- VotedFor
- log[]

所以只需要在所有改变了上述状态的地方，调用rf.persist()进行持久化即可。

### Lab2B与Lab2C

Lab2C中最主要的部分是需要实现持久化来处理节点重启的情况。

但是2C的测试相比于2B的测试，增加了一些unreliable的情况。如果在之前的实现中充分考虑到了网络延迟乱序丢失等情况，那么从lab2B到lab2C只需要完成rf.persist(),rf.readPersist()两个函数并在需要持久化的地方调用rf.persist()即可通过测试。如果没有考虑不可靠网络的情况，则需要完善这部分才能通过测试。

#### 需要注意的一些点：

- matchIndex[i]除了初始化为0之外，其余时候都是增量更新的。它是对日志复制状况的确认。更新完之后再将nextIndex[i]设置为matchIndex[i]+1。

### Kill

虽然注释中说you're not required to do anything about this，但实际上不实现的话会有很大的影响。

## 总结

 ![raft-test](/assets/raft-test.png)

测试通过。

---

整个Lab2是对Raft的一个基本实现，像这样的分布式系统细节中的坑非常多。所以一定要严格按照论文中的描述来实现，否则会遇到许多问题。比如在处理AppendEntries请求时遇到什么情况才截断自身日志，什么时候重置选举超时计时器，这些都要严格按图2中的说明实现。此外，对并发和锁的情况处理不当也会造成问题。比如在加锁的代码块中发送和接收chan数据可能造成死锁等。

整个lab2我调试了很久，重写了2次才通过测试，整个过程跟[Studentes's Guide to Raft](https://thesquareplanet.com/blog/students-guide-to-raft/)提到的一样：

> Inevitably, the first iteration of your Raft implementation will be buggy. So will the second. And third. And fourth. In general, each one will be less buggy than the previous one, and, from experience, most of your bugs will be a result of not faithfully following Figure 2.

最开始的实现充满了bug，每次迭代都会让bug更少，但是确实很多bug是因为没有严格遵循图2中的规则造成的。

