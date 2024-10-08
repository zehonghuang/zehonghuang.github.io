+++
title = '【JRaft源码分析01】启用以及选举过程'
date = 2021-03-04T10:18:15+08:00
draft = false
categories = [
    "CAP理论",
    "RAFT算法",
    "分布式一致性",
    "java"
]
+++


最近潜心cap理论和raft算法，选用了蚂蚁金服的sofa-jraft，深入研究具体的实现。该框架参考自百度的BRAFT，可以说是非常优秀的分布式通用框架，很值得学习。

Raft算法的理论就不再多说了，感性认识的话可以看这个[动画](http://thesecretlivesofdata.com/raft/)，非常好懂。

## 1、启动入口

示例在github的[jraft-example](https://github.com/zehonghuang/sofa-jraft/tree/master/jraft-example)

``` java
RaftGroupService raftGroupService = new RaftGroupService(groupId, serverId, nodeOptions, rpcServer);
//依次实例化NodeManager、NodeImpl、RpcServer
Node node = raftGroupService.start();

public synchronized Node start(final boolean startRpcServer) {
    NodeManager.getInstance().addAddress(this.serverId.getEndpoint());
    this.node = RaftServiceFactory.createAndInitRaftNode(this.groupId, this.serverId, this.nodeOptions);
    if (startRpcServer) {
        this.rpcServer.startup();
    }
}
```
## 2、包罗万象的Node

分布式系统关键单体就是节点Node，它包括raft分布式算法中需要的所有行为，不限于选举、投票、日志、复制、接收rpc请求等，梦开始的地方。

![Node结构图](https://raw.githubusercontent.com/zehonghuang/github_blog_bak/master/source/image/Node%E7%BB%93%E6%9E%84%E5%9B%BE.png)
<!--more-->

jraft也是遵循这个思路，所以`NodeImpl`包含了众多核心逻辑，先看一下初始化了什么内容。
``` java
public boolean init(final NodeOptions opts) {
    this.electionTimeoutCounter = 0;
    //定时器，后面heartbeat、block会有用到
    this.timerManager = new TimerManager();
    if (!this.timerManager.init(this.options.getTimerPoolSize())) {
        return false;
    }
    // Init timers
    final String suffix = getNodeId().toString();

    //如上图所说，下面依次是投票计时器、选举计时器、下线计时器、快照计时器
    this.voteTimer = new RepeatedTimer("JRaft-VoteTimer-" + suffix, this.options.getElectionTimeoutMs()) {
        @Override
        protected void onTrigger() {
            handleVoteTimeout();
        }
        //在一定范围内的随机超时时间
        //adjustTimeout -> randomTimeout()
    };
    this.electionTimer = new RepeatedTimer("JRaft-ElectionTimer-" + suffix, this.options.getElectionTimeoutMs()) {
        //onTrigger -> handleElectionTimeout()
        //adjustTimeout -> randomTimeout()
    };
    this.stepDownTimer = new RepeatedTimer("JRaft-StepDownTimer-" + suffix,
        this.options.getElectionTimeoutMs() >> 1) {
        //onTrigger -> handleStepDownTimeout()
    };
    this.snapshotTimer = new RepeatedTimer("JRaft-SnapshotTimer-" + suffix,
        this.options.getSnapshotIntervalSecs() * 1000) {
        //onTrigger -> handleSnapshotTimeout()
        //adjustTimeout -> 赋值逻辑不一样，说到快照再提
    };

    this.configManager = new ConfigurationManager();
    //单线程处理来自apply()的Task，核心处理方法在LogEntryAndClosureHandler -> onEvent -> executeApplyingTasks()
    this.applyDisruptor = DisruptorBuilder.<LogEntryAndClosure> newInstance() 
        .setRingBufferSize(this.raftOptions.getDisruptorBufferSize()) 
        .setEventFactory(new LogEntryAndClosureFactory()) 
        .setThreadFactory(new NamedThreadFactory("JRaft-NodeImpl-Disruptor-", true)) 
        .setProducerType(ProducerType.MULTI) 
        .setWaitStrategy(new BlockingWaitStrategy()) 
        .build();
    this.applyDisruptor.handleEventsWith(new LogEntryAndClosureHandler());
    this.applyDisruptor.setDefaultExceptionHandler(new LogExceptionHandler<Object>(getClass().getSimpleName()));
    this.applyQueue = this.applyDisruptor.start(); // applyQueue用于接收储存来自apply()的Task
    if (this.metrics.getMetricRegistry() != null) {
        this.metrics.getMetricRegistry().register("jraft-node-impl-disruptor",
            new DisruptorMetricSet(this.applyQueue));
    }
    //有限状态机
    this.fsmCaller = new FSMCallerImpl();
    if (!initLogStorage()) {
        return false; //实例化LogManagerImpl，提供了rocksdb作为日志持久化
    }
    if (!initMetaStorage()) {
        return false; //元数据，存储term、当前vote peerid，不能持久化
    }
    if (!initFSMCaller(new LogId(0, 0))) {
        return false;
    }
    //选票箱不是选举用的，在于写入且复制日志后，执行状态机
    this.ballotBox = new BallotBox();
    final BallotBoxOptions ballotBoxOpts = new BallotBoxOptions();
    ballotBoxOpts.setWaiter(this.fsmCaller);
    ballotBoxOpts.setClosureQueue(this.closureQueue);
    if (!this.ballotBox.init(ballotBoxOpts)) {
        return false;
    }
    if (!initSnapshotStorage()) {
        return false; //实例化SnapshotExecutorImpl
    }

    final Status st = this.logManager.checkConsistency();//检查快照index是否落在first & last log index之内
    if (!st.isOk()) {
        return false;
    }
    
    //管理refactor的
    this.replicatorGroup = new ReplicatorGroupImpl();
    this.rpcService = new BoltRaftClientService(this.replicatorGroup);
    final ReplicatorGroupOptions rgOpts = new ReplicatorGroupOptions();
    //rgOpts.set...一堆设置

    if (!this.rpcService.init(this.options)) {
        return false;
    }
    this.replicatorGroup.init(new NodeId(this.groupId, this.serverId), rgOpts);
    //只读服务，包括readindex也是在这里被调用
    this.readOnlyService = new ReadOnlyServiceImpl();
    final ReadOnlyServiceOptions rosOpts = new ReadOnlyServiceOptions();
    rosOpts.setFsmCaller(this.fsmCaller);
    rosOpts.setNode(this);
    rosOpts.setRaftOptions(this.raftOptions);
    if (!this.readOnlyService.init(rosOpts)) {
        return false;
    }

    // set state to follower
    this.state = State.STATE_FOLLOWER;

    if (this.snapshotExecutor != null && this.options.getSnapshotIntervalSecs() > 0) {
        this.snapshotTimer.start();
    }
    if (!this.conf.isEmpty()) {
        stepDown(this.currTerm, false, new Status()); // 这里是用于节点下线的方法，这里是为了启动选举倒计时
    }
    if (!NodeManager.getInstance().add(this)) {
        return false;
    }
    // Now the raft node is started , have to acquire the writeLock to avoid race
    //... 还有单点自举的代码，忽略
    return true;
}
```
从初始化代码来看，几乎核心逻辑都交给定时器，以及Disruptor来处理，所以单纯靠debug跟踪是不可能的，需要先了解raft算法逻辑以及每个定时器的作用来人肉跳转。

## 3、选举出Leader

节点启动完成后，会被立即启动ElectionTimer，执行该方法。
``` java 
private void handleElectionTimeout() {
    boolean doUnlock = true;
    this.writeLock.lock();
    try {
        if (this.state != State.STATE_FOLLOWER) {
            return;
        }
        if (isCurrentLeaderValid()) {
            return; // 心跳包会不断更新lastLeaderTimestamp，如果间隔超过electionTimeoutMs，Follower会发起第一轮选举
        }
        resetLeaderId(PeerId.emptyPeer(), new Status(RaftError.ERAFTTIMEDOUT, "Lost connection from leader %s.",
            this.leaderId));
        // 检查优先级是否允许该节点进行选举
        if (!allowLaunchElection()) {
            return;
        }
        doUnlock = false;
        preVote();  //开始第一轮投票
    } finally {
        if (doUnlock) {
            this.writeLock.unlock();
        }
    }
}
```
### 3.1、Pre-Vote是否有必要

一般来说，raft的选举仅且只有一轮投票选出集群leader，但会存在缺陷，就是网络分区后，少数派节点虽然都不会得到足够的票数成为分区Leader，但任选编号term却会不断增加，在网络分区恢复，少数派节点会因为term较大，而迫使多数派Leader下线，我们叫他做`捣蛋鬼`。为了避免`捣蛋鬼`谋权篡位，我们引入`Pre-Vote`，只有得到绝大多数选票，才有被提拔为候选人的资格。

在Diego Ongaro的博士论文[《CONSENSUS: BRIDGING THEORY AND PRACTICE》](https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf)中，有一节较为全面的解释。
![9.6 Preventing disruptions when a server rejoins the cluster](https://raw.githubusercontent.com/zehonghuang/github_blog_bak/master/source/image/WX20200329-223149%402x.png)

JRaft也有实现了`Pre-Vote`，下面来看看

``` java
private void preVote() {
    long oldTerm; // oldTerm = this.currTerm;
    //这里有一段快照 & serverId的校验，then writeLock.unlock();

    final LogId lastLogId = this.logManager.getLastLogId(true);

    boolean doUnlock = true;
    this.writeLock.lock();
    try {
        // pre_vote need defense ABA after unlock&writeLock
        if (oldTerm != this.currTerm) {
            return;
        }
        //这一句在计算quorum
        this.prevVoteCtx.init(this.conf.getConf(), this.conf.isStable() ? null : this.conf.getOldConf());
        for (final PeerId peer : this.conf.listPeers()) {
            if (peer.equals(this.serverId)) {
                continue;
            }
            if (!this.rpcService.connect(peer.getEndpoint())) {
                continue;
            }
            final OnPreVoteRpcDone done = new OnPreVoteRpcDone(peer, this.currTerm);
            done.request = RequestVoteRequest.newBuilder() //
                .setPreVote(true) // it's a pre-vote request.
                .setGroupId(this.groupId) //
                .setServerId(this.serverId.toString()) //
                .setPeerId(peer.toString()) //
                .setTerm(this.currTerm + 1) // next term 这里不用currTerm++，为了不让本地节点term暴增
                .setLastLogIndex(lastLogId.getIndex()) //
                .setLastLogTerm(lastLogId.getTerm()) //
                .build();
            this.rpcService.preVote(peer.getEndpoint(), done.request, done); //我们来看处理handlePreVoteRequest()
        }
        this.prevVoteCtx.grant(this.serverId);//为自己投一票
        if (this.prevVoteCtx.isGranted()) {
            doUnlock = false;
            electSelf(); //假如获取到大于等于quorum票数，则进入下一轮投票
        }
    } finally {
        if (doUnlock) {
            this.writeLock.unlock();
        }
    }
}
```
发出第一轮投票请求后，其他节点随即处理请求。
``` java
public Message handlePreVoteRequest(final RequestVoteRequest request) {
    boolean doUnlock = true;
    this.writeLock.lock();
    try {
        
        boolean granted = false;
        // noinspection ConstantConditions
        do {
            //Leader可用，在租期内发送过心跳包
            if (this.leaderId != null && !this.leaderId.isEmpty() && isCurrentLeaderValid()) {
                break;
            }
            //注意preVote发送的是currTerm+1，所以要么小于，要么刚好等于
            //如果小于则没资格成为候选人
            if (request.getTerm() < this.currTerm) {
                // A follower replicator may not be started when this node become leader, so we must check it.
                checkReplicator(candidateId);
                break;
            } else if (request.getTerm() == this.currTerm + 1) {
                checkReplicator(candidateId);
            }
            doUnlock = false;
            this.writeLock.unlock();
            //若刚好等于，就看看谁的LogIndex更大了
            final LogId lastLogId = this.logManager.getLastLogId(true);

            doUnlock = true;
            this.writeLock.lock();
            //通常来说，能正常工作的多数派LastLogId更大
            final LogId requestLastLogId = new LogId(request.getLastLogIndex(), request.getLastLogTerm());
            granted = requestLastLogId.compareTo(lastLogId) >= 0;

        } while (false);
        //yep， 发送结果
        return RequestVoteResponse.newBuilder() //
            .setTerm(this.currTerm) //
            .setGranted(granted) //
            .build();
    } finally {
        if (doUnlock) {
            this.writeLock.unlock();
        }
    }
}
```

注意到个非常有意思的地方，我们都知道，选举中每个选民手里只有一张票，要么自己要么候选人。但在`Pre-Vote`选民可以投至少一张票（不考虑Leaner），只要满足条件即可获取选票，最后只看选票是否过半得出候选人名单。

然后处理第一轮选票结果。

``` java
public void handlePreVoteResponse(final PeerId peerId, final long term, final RequestVoteResponse response) {
    boolean doUnlock = true;
    this.writeLock.lock();
    try {
        if (this.state != State.STATE_FOLLOWER) {
            return; //可能后面的选票来晚了，节点要么出错，要么晋升为候选人或者当选为Leader
        }
        if (term != this.currTerm) {
            return; //此时可能已经屈服为FOLLOWER
        }
        //无论任何情况，只要发现任选编号大于自己，都会被强制下线
        //这也是为什么需要Pre-Vote的原因，如果没有简直是牛逼坏了
        if (response.getTerm() > this.currTerm) {
            stepDown(response.getTerm(), false, new Status(RaftError.EHIGHERTERMRESPONSE,
                "Raft node receives higher term pre_vote_response."));
            return;
        }
        // 第一轮选票顺利进行，该节点达标则立即进入第二轮
        if (response.getGranted()) {
            this.prevVoteCtx.grant(peerId);
            if (this.prevVoteCtx.isGranted()) {
                doUnlock = false;
                electSelf();
            }
        }
    } finally {
        if (doUnlock) {
            this.writeLock.unlock();
        }
    }
}
```
### 3.2、正式选举Leader

有了第一轮投票，随即进行第二轮，不进行timeout等待。
``` java
//先投自己一票，然后发起投票请求
private void electSelf() {
    long oldTerm;
    try {
        if (this.state == State.STATE_FOLLOWER) {
            this.electionTimer.stop();
        }
        resetLeaderId(PeerId.emptyPeer(), new Status(RaftError.ERAFTTIMEDOUT,
            "A follower's leader_id is reset to NULL as it begins to request_vote."));
        this.state = State.STATE_CANDIDATE; // 角色转换为候选人
        this.currTerm++;    //这里递增了
        //这是NodeImpl全局变量PeerId voteId，由于存储该节点把选票投给谁
        //候选人投自己一票
        this.votedId = this.serverId.copy();
        //启动投票时限，如果stepDownWhenVoteTimedout=true，会重新第一轮投票，否则直接重新发起第二轮
        this.voteTimer.start();
        //初始化quorum
        this.voteCtx.init(this.conf.getConf(), this.conf.isStable() ? null : this.conf.getOldConf());
        oldTerm = this.currTerm;
    } finally {
        this.writeLock.unlock();
    }
    //刷盘后的lastLogIndexId
    final LogId lastLogId = this.logManager.getLastLogId(true);

    this.writeLock.lock();
    try {
        // vote need defense ABA after unlock&writeLock
        if (oldTerm != this.currTerm) {
            return;
        }
        for (final PeerId peer : this.conf.listPeers()) {
            if (peer.equals(this.serverId)) {
                continue;
            }
            if (!this.rpcService.connect(peer.getEndpoint())) {
                continue;
            }
            final OnRequestVoteRpcDone done = new OnRequestVoteRpcDone(peer, this.currTerm, this);
            done.request = RequestVoteRequest.newBuilder() //
                .setPreVote(false) // It's not a pre-vote request.
                .setGroupId(this.groupId) //
                .setServerId(this.serverId.toString()) //
                .setPeerId(peer.toString()) //
                .setTerm(this.currTerm) //
                .setLastLogIndex(lastLogId.getIndex()) //
                .setLastLogTerm(lastLogId.getTerm()) //
                .build();
            this.rpcService.requestVote(peer.getEndpoint(), done.request, done);
        }

        this.metaStorage.setTermAndVotedFor(this.currTerm, this.serverId);//记录你把票投给谁了
        this.voteCtx.grant(this.serverId);//给自己一票，如果只有你自己一人，直接成为Leader
        if (this.voteCtx.isGranted()) {
            becomeLeader();
        }
    } finally {
        this.writeLock.unlock();
    }
}
```

`Candidate`和`Follower`都会收到除自己外其他候选人的投票请求，对于F来说就是谁先来且符合条件我就投给谁，仅有一票，所以用`this.votedId`标记投出去的选票变得尤为重要。
``` java
public Message handleRequestVoteRequest(final RequestVoteRequest request) {
    boolean doUnlock = true;
    this.writeLock.lock();
    try {
        do {
            // check term
            if (request.getTerm() >= this.currTerm) {
                // 这还用说么，如果是candidate或leader直接降级。更正任期term，以及改变状态为follower
                if (request.getTerm() > this.currTerm) {
                    stepDown(request.getTerm(), false, new Status(RaftError.EHIGHERTERMRESPONSE,
                        "Raft node receives higher term RequestVoteRequest."));
                }
            } else {
                break; //这里是处理晚到过期的投票
            }
            doUnlock = false;
            this.writeLock.unlock();

            final LogId lastLogId = this.logManager.getLastLogId(true);

            doUnlock = true;
            this.writeLock.lock();
            // vote need ABA check after unlock&writeLock
            if (request.getTerm() != this.currTerm) {
                break; //this.currTerm被上面stepDown()更正过，如果在写锁unlock时又被更正，则说明本次投票请求失效
            }
            //下面是投出宝贵的一票
            final boolean logIsOk = new LogId(request.getLastLogIndex(), request.getLastLogTerm())
                .compareTo(lastLogId) >= 0;
            //查看该节点是否已经投出选票
            if (logIsOk && (this.votedId == null || this.votedId.isEmpty())) {
                stepDown(request.getTerm(), false, new Status(RaftError.EVOTEFORCANDIDATE,
                    "Raft node votes for some candidate, step down to restart election_timer."));
                this.votedId = candidateId.copy();
                this.metaStorage.setVotedFor(candidateId);
            }
        } while (false);

        return RequestVoteResponse.newBuilder() //
            .setTerm(this.currTerm) //
            .setGranted(request.getTerm() == this.currTerm && candidateId.equals(this.votedId)) //
            .build();
    } finally {
        if (doUnlock) {
            this.writeLock.unlock();
        }
    }
}
```

港真，我在`votedId`绕了一两个钟，就为了想明白request.term和currTerm在各种情况下的表现，最后来到handleRequestVoteResponse真的开心，选举快结束了。蔡英文当了总统，手动微笑。
``` java
public void handleRequestVoteResponse(final PeerId peerId, final long term, final RequestVoteResponse response) {
    this.writeLock.lock();
    try {
        if (this.state != State.STATE_CANDIDATE) {
            return;
        }
        if (term != this.currTerm) {
            return;
        }
        if (response.getTerm() > this.currTerm) {
            stepDown(response.getTerm(), false, new Status(RaftError.EHIGHERTERMRESPONSE,
                "Raft node receives higher term request_vote_response."));
            return;
        }
        //上面不用看了，都因为term不够，不能转正
        // check granted quorum?
        if (response.getGranted()) {
            this.voteCtx.grant(peerId);
            if (this.voteCtx.isGranted()) {
                becomeLeader(); //成为leader
            }
        }
    } finally {
        this.writeLock.unlock();
    }
}
```
这一段就没什么好说得了，发现一个新角色就是Leaner(学习者)以后再说。
``` java
private void becomeLeader() {
    // cancel candidate vote timer
    stopVoteTimer();
    this.state = State.STATE_LEADER;
    this.leaderId = this.serverId.copy();
    this.replicatorGroup.resetTerm(this.currTerm);
    // Start follower's replicators
    for (final PeerId peer : this.conf.listPeers()) {
        if (peer.equals(this.serverId)) {
            continue;
        }
        if (!this.replicatorGroup.addReplicator(peer)) {
            //
        }
    }
    for (final PeerId peer : this.conf.listLearners()) {
        if (!this.replicatorGroup.addReplicator(peer, ReplicatorType.Learner)) {
        }
    }
    // init commit manager
    this.ballotBox.resetPendingIndex(this.logManager.getLastLogIndex() + 1);

    if (this.confCtx.isBusy()) {
        throw new IllegalStateException();
    }
    this.confCtx.flush(this.conf.getConf(), this.conf.getOldConf());//进行Leader的第一次刷盘
    this.stepDownTimer.start(); //定时刷新leaderLeaseTimeoutMs，在读一致性那里会详细说明
}
```

### 3.3、下线方法stepDown到底做了些什么？

上面很多地方出现`stepDown`，到底是何方神圣，来康一康吧！

``` java
private void stepDown(final long term, final boolean wakeupCandidate, final Status status) {
    LOG.debug("Node {} stepDown, term={}, newTerm={}, wakeupCandidate={}.", getNodeId(), this.currTerm, term,
        wakeupCandidate);
    //这里就不多做解释了
    if (this.state == State.STATE_CANDIDATE) {
        stopVoteTimer();
    } else if (this.state.compareTo(State.STATE_TRANSFERRING) <= 0) {
        stopStepDownTimer();
        this.ballotBox.clearPendingTasks();
        // signal fsm leader stop immediately
        if (this.state == State.STATE_LEADER) {
            onLeaderStop(status);
        }
    }
    resetLeaderId(PeerId.emptyPeer(), status);

    // 重新降级为FOLLOWER
    this.state = State.STATE_FOLLOWER;
    this.confCtx.reset();
    updateLastLeaderTimestamp(Utils.monotonicMs());
    if (this.snapshotExecutor != null) {
        this.snapshotExecutor.interruptDownloadingSnapshots(term);//停止进行中的快照
    }

    // 更正currTerm
    if (term > this.currTerm) {
        this.currTerm = term;
        this.votedId = PeerId.emptyPeer();
        this.metaStorage.setTermAndVotedFor(term, this.votedId);
    }

    if (wakeupCandidate) {//唤醒一个最优质的，日志完整度和优先级最高的节点，直接进入第二轮选举
        this.wakingCandidate = this.replicatorGroup.stopAllAndFindTheNextCandidate(this.conf);
        if (this.wakingCandidate != null) {
            Replicator.sendTimeoutNowAndStop(this.wakingCandidate, this.options.getElectionTimeoutMs());
        }
    } else {
        this.replicatorGroup.stopAll();//否则所有复制器停止工作
    }
    if (this.stopTransferArg != null) {
        if (this.transferTimer != null) {
            this.transferTimer.cancel(true);
        }
        // There is at most one StopTransferTimer at the same term, it's safe to
        // mark stopTransferArg to NULL
        this.stopTransferArg = null;
    }
    // Learner node will not trigger the election timer.
    if (!isLearner()) {
        this.electionTimer.start();
    } else {}//LOG
}
```

整个选举算法的过程就是这样，理顺后，逻辑简单明了，实现起来并不困难，基本围绕者term和lastLogId。stepDown确实很重要，让一个节点回到出厂设置，这个方法代码可能会随着框架自定义内容增加而增加，自己实现simple版至少不用wakeupCandidate和transferTimer。