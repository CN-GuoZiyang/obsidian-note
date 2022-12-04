# 前言

Raft 是用于管理日志复制的共识算法。共识算法适用于包含多台机器的集群中，以保证在其中有些机器挂掉时仍然能正常对外提供服务。由此，共识算法在构建可靠的大规模软件系统中十分重要。

关于 Raft 算法主要的论文就是《In Search of an Understandable Consensus Algorithm (Extended Version)》，可以在[这里](https://raft.github.io/raft.pdf)阅读。论文也不长，18 页。文中大量出现了将 Raft 和 Paxos 的对比（开头就是一顿诉苦），凸显了 Raft 最大的优势：more understandable。

本文是在论文阅读中的笔记。

# 前情提要

共识算法的提出主要是用于复制状态机（Replicated state machines）模型。复制状态机通常用复制日志实现，每个日志中都包含一系列指令。集群中的每台机器都按相同的顺序执行这一系列指令，最终就会到达相同的状态。注意这个最终，表面这是最终一致性而非强一致性的。

共识算法保证复制日志在集群中是连续的，共识模块和其他机器上的共识模块通信来保证每台机器最终都按照相同顺序执行了相同的指令，即使其中某些机器挂掉了。这样保证了所有机器像单台机器一样共同对外提供服务。

共识算法仅适用于非拜占庭场景，即节点不会蓄意伪造信息。

# 算法描述

Raft 共识算法可以分成三个相对独立的子模块：
- Leader 选举：如果一个现有的 Leader 挂掉了，必须选举出一个新的 Leader
- 日志复制：Leader 必须从客户端接收日志，并在集群中复制，保证其他机器上的日志和自身的一致
- 安全性保障：如果任何机器接受了一个特定指令，那么其他机器就不可能再接收拥有和该指令相同的日志序号（log index）的指令

## Raft 基础

Raft 集群中的机器在任何时候都只有三种状态：
- Leader：负责处理所有的客户端请求
- Follower：不处理请求，被动接收 Leader 和 Candidate 的请求并应答
- Candidate：用于选举出一个 Leader

正常执行的情况下，集群只有一个 Leader，其他所有机器都是 Follower。

Raft 将时间划分为多个任期，每个任期开始时都会在集群中举行选举，一个或多个 Candidate 会尝试成为 Leader。如果某个 Candidate 赢得了选举，那么它就会变成 Leader，其余机器变为 Follower。

任期是一个单调递增的整数，每台机器中都会保存当前任期，机器之间通信时也会带上当前任期。如果某台机器发现自己的当前任期小于其他机器，就会更新自己的当前任期。如果 Candidate 或者 Leader 发现自己的当前任期小于其他任期（出现新任期的 Leader），就会立刻转变成为 Follower。

Raft 机器之间的基本通信只需要两种类型的 RPC。拉票请求（RequestVote RPCs，这个翻译怎么怪怪的）在选举中由 Candidate 发起；追加请求（AppendEntries RPCs）由 Leader 发起，用于复制日志和维持心跳。

## Leader 选举

机器启动时的状态是 Follower，并且会在收到合适的 RPC 的情况下一直保持 Follower 状态。Leader 会定时发送不包含命令的追加请求作为心跳来维持 Server 身份。一旦某个 Follower 一段时间内没有收到请求，就会发起一场选举。

发起选举的 Follower 会转变为 Candidate，并将当前任期 +1，给自己投票后并行地向集群中的其他机器发送拉票请求。Candidate 会维持该状态直到一下三种情况发生：
1. 如果某个 Candidate 获得了集群中大多数机器的票，那么它就赢得了选举。每个机器在某个任期中最多只会给一个 Candidate 投票，按照先来先得的原则，这样保证了每个任期中最多只有一个 Candidate 赢得选举。赢得选举的 Candidate 会变成 Leader，并向其他机器发送心跳来维持 Leader 地位。
2. Candidate 可能会收到其他机器的追加请求：如果该请求中的任期大于等于 Candidate 的当前任期，Candidate 就会转变为 Follower；否则会拒绝该请求并维持 Candidate 状态。
3. 如果许多 Follower 同时转变为 Candidate，那么可能没有一个 Candidate 可以获得大多数票。这种情况下 Candidate 会超时，将当前任期 +1 并发起一场新的选举。
 
为了防止场景 3 无限循环下去，Candidate 的超时时间一般设置为某个固定区间的随机时间以防止同时超时。

## 日志复制

每个客户端请求都包含一条需要日志复制状态机执行的指令，Leader 收到请求后会将该指令添加到 log 中，并行向集群中的机器发起追加请求来复制指令。当指令完成复制后，Leader 会提交（Commit）该命令到状态机并回复客户端执行成功。

每个追加请求中除了包含需要复制到指令，还包含 Leader 收到指令时的当前任期，同时还包含了一个整数序号来标明其在日志中的位置。

当接收该命令的 Leader 成功将命令复制到大多数机器上时，这条命令就会被 Leader 提交。Raft 会保证被提交的命令是持久化的，且最终会被所有可用的状态机执行。这个提交会同时提交所有前序的指令，即使是由其他的 Leader 创建的。Leader 会持续记录即将提交的日志序号并在所有追加请求中包含该序号。当 Follower 得知某个日志条目已经被提交了，它也会将该条目提交到它的状态机中。

同时 Raft 还需要保证以下两条规则：
- 如果两个日志条目拥有相同的任期和日志序号，那么它们一定保存了相同的指令
- 如果两个日志条目拥有相同的任期和日志序号，那么这两条日志的所有前序日志都是相同的

第一条的保证比较简单，主要看第二条的保证。

当发送追加请求时，Leader 还会将新日志条目前面那一条条目的任期和日志序号包含在请求中。如果 Follower 在其日志序列中找不到该条日志，就会拒绝该请求，这就是连续性检查。由此，只要请求成功返回，Leader 就会知道 Follower 的日志和自己的是一致的。

当连续性检查失败时，Leader 会强制要求 Follower 的日志序列和自身保持一致。具体：Leader 需要找到找到和 Follower 相同的最后一条日志，删除 Follower 日志序列中那一条日志中的后面所有日志，并将自己的后续日志发给 Follower。Leader 需要给每个 Follower 维护一个 nextIndex，保存着下一个将要给该 Follower 发送的日志序号。当 Leader 刚启动时，会将所有 Follower 的 nextIndex 都初始化为其 log 序列最后一个日志序号 +1。当某个 Follower 没有通过，Leader 会将该 Follower 的 nextIndex -1 并重发追加请求。 

> 这里说的有点模糊，我估计应该是每个追加请求发送的是从 nextIndex 到后一个日志条目，否则在连续性检查通过后，仅仅是将 Follower 的日志序列裁剪到了 Leader 和 Follower 一致处，一致处后续的日志并没有同步到 Follower

## 安全性保障

以上描述的机制并不能完全保证安全性，例如可能存在这种可能性：某台机器突然不可达，在此期间 Leader 提交了多条日志，该机器可能会被选举为 Leader，导致这些日志被覆写。本节增加了一个选举时的限制来完善 Raft 算法。该限制保证了任一任期的 Leader 一定包含了前一任期的全部提交了的日志。

首先，拉票请求会包含 Candidate 的日志信息，如果其他机器发现自己的日志比 Candidate 的日志更新（最后一个条目的任期更大、日志序号更大），那么该机器就会拒绝投票请求。

另外，一旦某条当前任期的日志条目被大多数机器接收，Leader 就会将其提交。如果 Leader 在提交条目时就挂了，下一任 Leader 会继续尝试完成这个条目的复制。但是 Leader 不能立刻确定某个前一任期的条目已经被提交，即使它已经存储在大多数机器上。由此可能会导致论文 Figure 8 中的问题。

由此，Raft 并不通过检查副本的个数来尝试提交前一个任期的日志条目。只会检查副本个数来提交当前任期的日志条目，当某个当前任期的条目被提交，那么该条目前面的所有条目都被隐式地提交了。

## 集群节点变更

在集群节点变更的过程中，如果想要不让整个集群下线而实现集群配置变更，集群存在出现两个独立的“主体”的可能，即可能会选举出两个 Leader。

为了保证节点变更的安全性，Raft 采用了两阶段方法。首先集群切换到一个联合共识（joint consensus）状态，在联合共识被提交后，系统再切换到新配置。联合共识状态下，集群仍然可以正常地对外提供服务。在联合共识状态下：
- 日志会被复制到新旧配置中的所有机器上
- 两个配置上的每台机器都允许成为 Leader
- 选举和追加需要在新旧两个配置上都分别获得大多数同意才能生效

集群配置在日志中使用特殊的条目存储和传输。流程如下：
1. Leader 收到了一个请求，将集群配置从 C_old 切换为 C_new
2. Leader 将 C_old 和 C_new 作为联合共识 C_old,new 配置存储到一条日志条目中
3. 将该条日志条目追加到新旧配置的所有机器上
4. 当某台机器接收到这样的日志条目添加其日志上时（无需提交），后续所有的操作都以该配置为标准进行
5. 当 C_old,new 被多数机器接受后，Leader 将其提交。这时， 任何 C_old 或者 C_new 配置的机器就都不可能被选为 Leader
6. Leader 创建一条 C_new 的日志条目，并追加到所有机器上并提交

集群节点变更仍然存在以下三个问题：
1. 新机器加入集群时没有存储任何日志，需要花一段时间才能追上旧机器，这可能会在短时间内造成集群的可用性降低。Raft 在将节点加入集群中时，引入了一个新阶段，在这个阶段，新机器可以正常收到追加请求，但是不会被认为是可投票的节点，不必考虑这些新节点即可达成共识。当这些新节点的日志赶上了旧机器的进度，再进行上述处理
2. 集群的 Leader 可能不是新配置的一部分。在这种情况下，当 Leader 提交 C_new 时，它自身就应当被撤下。这导致了有一段时间，Leader 管理着一个不包含它自己的集群，它复制日志但是不把自己视作主体。
3. 那些被撤下的服务器可能会破坏集群，由于这些机器接收不到心跳，可能会举行选举。随后发送带有新任期的拉票请求，导致集群的 Leader 恢复成 Follower。虽然这种情况下这种选举不会成功，新 Leader 仍然会在新集群中产生，但是被撤下的机器仍然会选举超时，导致整体可用性较差。

为了防止问题 3 发生，Raft 增加了一条限制：如果服务器在从当前 Leader 处收到心跳的超时时间内收到拉票请求，就不会更新任期和投票。这使得一个 Leader 只要可以维持当前集群的心跳，就不会被更大的任期选票踢下位。

## 日志压缩



# 算法实现

论文在 Figure 2 中给出了一个非常详细的实现（~~that's why raft is awesome!~~），不包含节点变更和日志压缩

## Server 状态

```go
type ServerState struct {
	/***** 所有 Server 都包含的持久状态 *****/
	// CurrentTerm 机器遇到的最大的任期，启动时初始化为 0，单调递增
	CurrentTerm int64;
	// VotedFor 当前任期内投票的 Candidate ID，未投票则为 nil
	VotedFor    *int64;
	// Logs 日志条目，每个条目都包含了一条状态机指令和 Leader 接收该条目时的任期，index 从 1 开始
	Logs        []*Log;

	/***** 所有 Server 都包含的可变状态 *****/
	// CommitIndex 已知的最大的即将提交的日志序号，启动时初始化为 0，单调递增
	CommitIndex int64;
	// LastApplied 最大的已提交的日志序号，启动时初始化为 0，单调递增
	LastApplied int64;

	/******* Leader 包含的可变状态，选举后初始化 *******/
	// NextIndex 每台机器下一个要发送的日志条目的序号，初始化为 Leader 最后一个日志序号 +1
	NextIndex  []int64;
	// MatchIndex 每台机器已知复制的最高的日志条目，初始化为 0，单调递增
	MatchIndex []int64;
}
```

## 追加请求


```go
type AppendEntriesRequest struct {
	// Term Leader 的任期
	Term         int64
	// LeaderID Follower 可以将客户端请求重定向到 Leader
	LeaderID     int64
	// PrevLogIndex 新日志条目前一个日志条目的日志序号
	PrevLogIndex int64
	// PrevLogTerm 前一个日志条目的任期
	PrevLogTerm  int64
	// Entries 需要保存的日志条目，心跳包为空
	Entries      []*Log
	// LeaderCommit Leader 的 CommitIndex
	LeaderCommit int64
}

type AppendEntriesResponse struct {
	// Term Follower 当前任期
	Term    int64
	// Success Follower 包含 PrevLogIndex 和 PrevLogTerm 的日志条目为 true
	Success bool
}
```

接收者的实现：
1. 如果 Term 小于 CurrentTerm，返回 false
2. 如果日志中不包含 PrevLogIndex 和 PrevLogTerm 对应的日志条目，返回 false
3. 如果某个现有的日志条目和新条目包含了相同的日志序号但是有不同的任期，删除该条和其后所有的日志
4. 添加所有在日志中不存在的日志条目
5. 如果 LeaderCommit 大于 CommitIndex，则将 CommitIndex 设置为 LeaderCommit 或者新日志中最后一个序号之间的较小值

## 拉票请求


```go
type RequestVoteRequest struct {
	// Term Candidate 的任期
	Term         int64
	// CandidateId 拉票的 Candidate 的 ID
	CandidateId  int64
	// LastLogIndex Candidate 最后一条日志序列的序号
	LastLogIndex int64
	// LastLogTerm Candidate 最后一条日志序列的任期
	LastLogTerm  int64
}

type RequestVoteResponse struct {
	// Term 当前任期
	Term        int64
	// VoteGranted true 则拉票成功
	VoteGranted bool
}
```

接收者实现：
1. 如果 Term 小于 CurrentTerm，返回 false
2. 如果 VotedFor 是 nil 或者 CandidateId，且 Candidate 的日志和接收者一样新或者更新，则返回 true

## Server 规则

对于所有机器：
- 如果 CommitIndex 大于 LastApplied，LastApplied +1，并将日志 log\[LastApplied\] 提交到状态机
- 如果 RPC 请求或者相应中的 Term 大于 CurrentTerm，则将 CurrentTerm 设置为 Term，并转变为 Follower

对于 Follower：
- 相应来自 Candidate 和 Leader 的 RPC
- 如果一直到选举超时也没有收到来自当前 Leader 的追加请求或者给某个 Candidate 投票（注意不是收到拉票请求，而是投票），则转变为 Candidate

对于 Candidate：
- 一旦转变成 Candidate，即开始选举：
	- 将当前任期 +1
	- 给自己投票
	- 重制选举超时计时器
	- 向所有其他机器发送拉票请求
- 如果收到了大多数机器的票，则转变成 Leader
- 如果收到来自新 Leader 的追加请求，转变为 Follower
- 如果选举超时，开始新一轮选举

对于 Leader：
- 一旦转变成 Leader，向其他所有机器发送空的追加请求；在空闲时重复发送空追加请求来防止选举超时
- 如果收到来自客户端的指令：添加日志条目到日志中，在条目提交到状态机后返回相应
- 如果最后一条日志序号大于某个客户端的 NextIndex：向客户端发送包含 NextIndex 及其后所有日志条目的追加请求
	- 如果成功，更新 Follower 的 NextIndex 和 MatchIndex
	- 如果因为日志连续性而失败，NextIndex -- 并重拾
- 如果存在一个 N 大于 CommitIndex，大多数 MatchIndex 大于等于 N，且第 N 条日志的任期等于当前任期，则设置 CommitIndex 为 N