# Etcd错误使用案例

<show-structure depth="3"/>

## 1. 只连接1个Etcd实例

**架构**：Kubernetes集群的Master节点的APIServer只访问本地Etcd。

<procedure>
<img src="kubernetes-bad-arch.svg"  thumbnail="true"/>
</procedure>

**现象**：
1. 1个Master节点I/O性能下降，Etcd报大量请求超过10s
2. 该节点上的kube-apiserver、kube-controller-manager还在工作
3. 连接到该节点的Node被标记为NotReady，连接到其它节点的Node正常

**问题**

Etcd中所有写请求都是Leader完成的，Follower收到写请求也会转发给Leader，为什么Follower节点的IO异常会导致上述现象？

答：通过下面分析可知，**在IO异常时，Follower无法将写请求转发给Leader**，所以Node上报的状态更新请求会**丢失**，kube-controller-manager判断指定时间内未上报心跳，将Node标记为NotReady。

下面通过代码解析和实验佐证这个结论。

### 1.1 集群正常运行

```Bash
kangxiaoning@localhost:~/workspace/etcd$ bin/etcdctl --write-out=table --endpoints=localhost:12379,localhost:22379,localhost:32379 endpoint status
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|    ENDPOINT     |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| localhost:12379 | 8211f1d0f64f3269 |  3.5.17 |   20 kB |     false |      false |        10 |         42 |                 42 |        |
| localhost:22379 | 91bc3c398fb3c146 |  3.5.17 |   20 kB |      true |      false |        10 |         42 |                 42 |        |
| localhost:32379 | fd422379fda50e48 |  3.5.17 |   20 kB |     false |      false |        10 |         42 |                 42 |        |
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
kangxiaoning@localhost:~/workspace/etcd$
```

### 1.2 单个节点IO异常

通过在`func (w *WAL) Save(st raftpb.HardState, ents []raftpb.Entry)`函数中打断点模拟I/O异常，有两种情况。
- 有数据需要持久化时会停在这个函数，可以观察到集群只有**2**个节点可用。
- 只有心跳通信，没有数据需要持久化，可以观察到集群还是**3**个节点可用。

```Bash
kangxiaoning@localhost:~/workspace/etcd$ bin/etcdctl --write-out=table --endpoints=localhost:12379,localhost:22379,localhost:32379 endpoint status
{"level":"warn","ts":"2024-12-27T17:46:54.313+0800","logger":"etcd-client","caller":"v3@v3.5.6/retry_interceptor.go:62","msg":"retrying of unary invoker failed","target":"etcd-endpoints://0x40002a4000/localhost:12379","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = context deadline exceeded"}
Failed to get the status of endpoint localhost:12379 (context deadline exceeded)
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|    ENDPOINT     |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| localhost:22379 | 91bc3c398fb3c146 |   3.5.6 |   25 kB |      true |      false |         7 |         85 |                 85 |        |
| localhost:32379 | fd422379fda50e48 |   3.5.6 |   25 kB |     false |      false |         7 |         85 |                 85 |        |
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
kangxiaoning@localhost:~/workspace/etcd$
```

客户端发起一个Put请求，经过Propose进入raft模块，raft处理后会通过Ready这个channel发送到应用层，应用层在`func (r *raftNode) start(rh *raftReadyHandler)`函数中接收Ready数据并继续处理。

```Go
// start prepares and starts raftNode in a new goroutine. It is no longer safe
// to modify the fields after it has been started.
func (r *raftNode) start(rh *raftReadyHandler) {
	internalTimeout := time.Second

	go func() {
		defer r.onStop()
		islead := false

		for {
			select {
			case <-r.ticker.C:
				r.tick()
			case rd := <-r.Ready():
				if rd.SoftState != nil {
					newLeader := rd.SoftState.Lead != raft.None && rh.getLead() != rd.SoftState.Lead
					if newLeader {
						leaderChanges.Inc()
					}

					if rd.SoftState.Lead == raft.None {
						hasLeader.Set(0)
					} else {
						hasLeader.Set(1)
					}

					rh.updateLead(rd.SoftState.Lead)
					islead = rd.RaftState == raft.StateLeader
					if islead {
						isLeader.Set(1)
					} else {
						isLeader.Set(0)
					}
					rh.updateLeadership(newLeader)
					r.td.Reset()
				}

				if len(rd.ReadStates) != 0 {
					select {
					case r.readStateC <- rd.ReadStates[len(rd.ReadStates)-1]:
					case <-time.After(internalTimeout):
						r.lg.Warn("timed out sending read state", zap.Duration("timeout", internalTimeout))
					case <-r.stopped:
						return
					}
				}

				notifyc := make(chan struct{}, 1)
				ap := apply{
					entries:  rd.CommittedEntries,
					snapshot: rd.Snapshot,
					notifyc:  notifyc,
				}

				updateCommittedIndex(&ap, rh)

				waitWALSync := shouldWaitWALSync(rd)
				if waitWALSync {
					// gofail: var raftBeforeSaveWaitWalSync struct{}
					if err := r.storage.Save(rd.HardState, rd.Entries); err != nil {
						r.lg.Fatal("failed to save Raft hard state and entries", zap.Error(err))
					}
				}

				select {
				case r.applyc <- ap:
				case <-r.stopped:
					return
				}

				// the leader can write to its disk in parallel with replicating to the followers and them
				// writing to their disks.
				// For more details, check raft thesis 10.2.1
				if islead {
					// gofail: var raftBeforeLeaderSend struct{}
					r.transport.Send(r.processMessages(rd.Messages))
				}

				// Must save the snapshot file and WAL snapshot entry before saving any other entries or hardstate to
				// ensure that recovery after a snapshot restore is possible.
				if !raft.IsEmptySnap(rd.Snapshot) {
					// gofail: var raftBeforeSaveSnap struct{}
					if err := r.storage.SaveSnap(rd.Snapshot); err != nil {
						r.lg.Fatal("failed to save Raft snapshot", zap.Error(err))
					}
					// gofail: var raftAfterSaveSnap struct{}
				}

				if !waitWALSync {
					// gofail: var raftBeforeSave struct{}
					if err := r.storage.Save(rd.HardState, rd.Entries); err != nil {
						r.lg.Fatal("failed to save Raft hard state and entries", zap.Error(err))
					}
				}
				if !raft.IsEmptyHardState(rd.HardState) {
					proposalsCommitted.Set(float64(rd.HardState.Commit))
				}
				// gofail: var raftAfterSave struct{}

				if !raft.IsEmptySnap(rd.Snapshot) {
					// Force WAL to fsync its hard state before Release() releases
					// old data from the WAL. Otherwise could get an error like:
					// panic: tocommit(107) is out of range [lastIndex(84)]. Was the raft log corrupted, truncated, or lost?
					// See https://github.com/etcd-io/etcd/issues/10219 for more details.
					if err := r.storage.Sync(); err != nil {
						r.lg.Fatal("failed to sync Raft snapshot", zap.Error(err))
					}

					// etcdserver now claim the snapshot has been persisted onto the disk
					notifyc <- struct{}{}

					// gofail: var raftBeforeApplySnap struct{}
					r.raftStorage.ApplySnapshot(rd.Snapshot)
					r.lg.Info("applied incoming Raft snapshot", zap.Uint64("snapshot-index", rd.Snapshot.Metadata.Index))
					// gofail: var raftAfterApplySnap struct{}

					if err := r.storage.Release(rd.Snapshot); err != nil {
						r.lg.Fatal("failed to release Raft wal", zap.Error(err))
					}
					// gofail: var raftAfterWALRelease struct{}
				}

				r.raftStorage.Append(rd.Entries)

				if !islead {
					// finish processing incoming messages before we signal raftdone chan
					msgs := r.processMessages(rd.Messages)

					// now unblocks 'applyAll' that waits on Raft log disk writes before triggering snapshots
					notifyc <- struct{}{}

					// Candidate or follower needs to wait for all pending configuration
					// changes to be applied before sending messages.
					// Otherwise we might incorrectly count votes (e.g. votes from removed members).
					// Also slow machine's follower raft-layer could proceed to become the leader
					// on its own single-node cluster, before apply-layer applies the config change.
					// We simply wait for ALL pending entries to be applied for now.
					// We might improve this later on if it causes unnecessary long blocking issues.
					waitApply := false
					for _, ent := range rd.CommittedEntries {
						if ent.Type == raftpb.EntryConfChange {
							waitApply = true
							break
						}
					}
					if waitApply {
						// blocks until 'applyAll' calls 'applyWait.Trigger'
						// to be in sync with scheduled config-change job
						// (assume notifyc has cap of 1)
						select {
						case notifyc <- struct{}{}:
						case <-r.stopped:
							return
						}
					}

					// gofail: var raftBeforeFollowerSend struct{}
					r.transport.Send(msgs)
				} else {
					// leader already processed 'MsgSnap' and signaled
					notifyc <- struct{}{}
				}

				r.Advance()
			case <-r.stopped:
				return
			}
		}
	}()
}
```
{collapsible="true" collapsed-title="s.r.start(rh)" default-state="collapsed"}

正常流程会经过如下步骤：
1. 接收到Ready数据，MessageType为**MsgProp**，包含了本次`Put`相关的数据
<procedure>
<img src="etcd-put-message-1.png"  thumbnail="true"/>
</procedure>
2. 如果本次的Ready中有数据需要持久化，则在`start()`函数中调用`r.storage.Save(rd.HardState, rd.Entries)`执行WAL持久化操作（注：并不是本次Put的数据）
3. 因为该节点是Follower角色，因此调用`r.transport.Send(msgs)`将这个Message发Leader处理（条件断点：ms[i].Entries != nil) 
<procedure>
<img src="etcd-put-message-2.png"  thumbnail="true"/>
</procedure>
4. 再次接收到Ready数据，调用`r.storage.Save(rd.HardState, rd.Entries)`进行WAL持久化
<procedure>
<img src="etcd-put-message-3.png"  thumbnail="true"/>
</procedure>

<procedure>
<img src="etcd-put-message-4.png"  thumbnail="true"/>
</procedure>
5. 执行其它后续操作，最终Apply到状态机，完成Etcd数据写入，这里省略相关过程

根据`func (r *raftNode) start(rh *raftReadyHandler)`代码逻辑，需要先执行`r.storage.Save(rd.HardState, rd.Entries)`，然后才能执行`r.transport.Send(msgs)`。所以**在I/O异常的情况下，`start()`函数会卡在`r.storage.Save(rd.HardState, rd.Entries)`这一步，无法执行到`r.transport.Send(msgs)`，也就是请求无法转发给Leader，最终会丢失掉这些消息**。

后面是具体的实验过程。

### 1.3 向异常节点写入数据

已打断点，向异常节点写入数据会hang住，最终超时失败，数据丢失。

```Bash
kangxiaoning@localhost:~/workspace/etcd$ bin/etcdctl --endpoints=localhost:12379 put "hello" "kangxiaoning"
{"level":"warn","ts":"2025-01-03T17:52:40.595073+0800","logger":"etcd-client","caller":"v3/retry_interceptor.go:63","msg":"retrying of unary invoker failed","target":"etcd-endpoints://0x40000e6000/localhost:12379","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = context deadline exceeded"}
Error: context deadline exceeded
kangxiaoning@localhost:~/workspace/etcd$ bin/etcdctl --write-out=table --endpoints=localhost:12379,localhost:22379,localhost:32379 endpoint status
{"level":"warn","ts":"2025-01-03T17:55:04.810896+0800","logger":"etcd-client","caller":"v3/retry_interceptor.go:63","msg":"retrying of unary invoker failed","target":"etcd-endpoints://0x4000536000/localhost:12379","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = context deadline exceeded"}
Failed to get the status of endpoint localhost:12379 (context deadline exceeded)
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|    ENDPOINT     |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| localhost:22379 | 91bc3c398fb3c146 |  3.5.17 |   20 kB |      true |      false |        10 |         42 |                 42 |        |
| localhost:32379 | fd422379fda50e48 |  3.5.17 |   20 kB |     false |      false |        10 |         42 |                 42 |        |
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
kangxiaoning@localhost:~/workspace/etcd$ bin/etcdctl --endpoints=localhost:12379,localhost:22379,localhost:32379 get "hello"
hello
world
kangxiaoning@localhost:~/workspace/etcd$
```

**注**：怀疑真正导致数据丢失的原因是该Follower节点已经不属于集群。线上环境看到如下报错。

```Bash
ignored out-of-date read index response; local node read indexes queueing up and waiting to be in sync with leader
```

报错来自如下函数，

```Go
func (s *EtcdServer) linearizableReadLoop() {
	var rs raft.ReadState

	for {
		ctxToSend := make([]byte, 8)
		id1 := s.reqIDGen.Next()
		binary.BigEndian.PutUint64(ctxToSend, id1)
		leaderChangedNotifier := s.leaderChangedNotify()
		select {
		case <-leaderChangedNotifier:
			continue
		case <-s.readwaitc:
		case <-s.stopping:
			return
		}

		nextnr := newNotifier()

		s.readMu.Lock()
		nr := s.readNotifier
		s.readNotifier = nextnr
		s.readMu.Unlock()

		lg := s.getLogger()
		cctx, cancel := context.WithTimeout(context.Background(), s.Cfg.ReqTimeout())
		if err := s.r.ReadIndex(cctx, ctxToSend); err != nil {
			cancel()
			if err == raft.ErrStopped {
				return
			}
			if lg != nil {
				lg.Warn("failed to get read index from Raft", zap.Error(err))
			} else {
				plog.Errorf("failed to get read index from raft: %v", err)
			}
			readIndexFailed.Inc()
			nr.notify(err)
			continue
		}
		cancel()

		var (
			timeout bool
			done    bool
		)
		for !timeout && !done {
			select {
			case rs = <-s.r.readStateC:
				done = bytes.Equal(rs.RequestCtx, ctxToSend)
				if !done {
					// a previous request might time out. now we should ignore the response of it and
					// continue waiting for the response of the current requests.
					id2 := uint64(0)
					if len(rs.RequestCtx) == 8 {
						id2 = binary.BigEndian.Uint64(rs.RequestCtx)
					}
					if lg != nil {
						lg.Warn(
							"ignored out-of-date read index response; local node read indexes queueing up and waiting to be in sync with leader",
							zap.Uint64("sent-request-id", id1),
							zap.Uint64("received-request-id", id2),
						)
					} else {
						plog.Warningf("ignored out-of-date read index response; local node read indexes queueing up and waiting to be in sync with leader (request ID want %d, got %d)", id1, id2)
					}
					slowReadIndex.Inc()
				}
			case <-leaderChangedNotifier:
				timeout = true
				readIndexFailed.Inc()
				// return a retryable error.
				nr.notify(ErrLeaderChanged)
			case <-time.After(s.Cfg.ReqTimeout()):
				if lg != nil {
					lg.Warn("timed out waiting for read index response (local node might have slow network)", zap.Duration("timeout", s.Cfg.ReqTimeout()))
				} else {
					plog.Warningf("timed out waiting for read index response (local node might have slow network)")
				}
				nr.notify(ErrTimeout)
				timeout = true
				slowReadIndex.Inc()
			case <-s.stopping:
				return
			}
		}
		if !done {
			continue
		}

		if ai := s.getAppliedIndex(); ai < rs.Index {
			select {
			case <-s.applyWait.Wait(rs.Index):
			case <-s.stopping:
				return
			}
		}
		// unblock all l-reads requested at indices before rs.Index
		nr.notify(nil)
	}
}
```
{collapsible="true" collapsed-title="linearizableReadLoop()" default-state="collapsed"}

### 1.4 向集群写入数据

已打断点，在只有2个节点存活的情况下，endpoints指定所有实例，再向集群写入数据成功，所以正确的使用方法是Client指定集群所有节点。

```Bash
kangxiaoning@localhost:~/workspace/etcd$ bin/etcdctl --write-out=table --endpoints=localhost:12379,localhost:22379,localhost:32379 endpoint status
{"level":"warn","ts":"2025-01-03T18:08:52.608232+0800","logger":"etcd-client","caller":"v3/retry_interceptor.go:63","msg":"retrying of unary invoker failed","target":"etcd-endpoints://0x40003e4000/localhost:12379","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = context deadline exceeded"}
Failed to get the status of endpoint localhost:12379 (context deadline exceeded)
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|    ENDPOINT     |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| localhost:22379 | 91bc3c398fb3c146 |  3.5.17 |   20 kB |      true |      false |        10 |         46 |                 46 |        |
| localhost:32379 | fd422379fda50e48 |  3.5.17 |   20 kB |     false |      false |        10 |         46 |                 46 |        |
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
kangxiaoning@localhost:~/workspace/etcd$ bin/etcdctl --endpoints=localhost:12379,localhost:22379,localhost:32379 get "hello"
hello
world
kangxiaoning@localhost:~/workspace/etcd$ bin/etcdctl --endpoints=localhost:12379,localhost:22379,localhost:32379 put "hello" "kangxiaoning"
OK
kangxiaoning@localhost:~/workspace/etcd$ bin/etcdctl --endpoints=localhost:12379,localhost:22379,localhost:32379 get "hello"
hello
kangxiaoning
kangxiaoning@localhost:~/workspace/etcd$
```

**结论**：正确的使用方法是连接Etcde的Client指定集群所有节点执行操作，比如APIServer的`--etcd-servers`参数应该指定所有Etcd实例。