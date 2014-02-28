leveldb write merge实现（DBImpl::Write）
================================================================================

<pre><code>Status DBImpl::Write(const WriteOptions& options, WriteBatch* my_batch) {
// -----A begin-------  
  Writer w(&mutex_);  
  w.batch = my_batch;  
  w.sync = options.sync;  
  w.done = false;  
// -----A end  --------  


// -----B begin-------  
  MutexLock l(&mutex_);  
  writers_.push_back(&w);  
  while (!w.done && &w != writers_.front()) {  
    w.cv.Wait();  
  }  
  if (w.done) {  
    return w.status;  
  }  
// -----B end  -------  

  // May temporarily unlock and wait.  
  Status status = MakeRoomForWrite(my_batch == NULL);  
  uint64_t last_sequence = versions_->LastSequence();  
  Writer* last_writer = &w;  
  if (status.ok() && my_batch != NULL) {  // NULL batch is for compactions  
    WriteBatch* updates = BuildBatchGroup(&last_writer);  
    WriteBatchInternal::SetSequence(updates, last_sequence + 1);  
    last_sequence += WriteBatchInternal::Count(updates);  

    // Add to log and apply to memtable.  We can release the lock  
    // during this phase since &w is currently responsible for logging  
    // and protects against concurrent loggers and concurrent writes  
    // into mem_.  
    {
// -----C begin-------  
      mutex_.Unlock();  
// -----C end  -------  
      status = log_->AddRecord(WriteBatchInternal::Contents(updates));  
      if (status.ok() && options.sync) {  
        status = logfile_->Sync();  
      }  
      if (status.ok()) {  
        status = WriteBatchInternal::InsertInto(updates, mem_);  
      }  
// -----D begin-------  
      mutex_.Lock();  
// -----D end  -------  
    }  
    if (updates == tmp_batch_) tmp_batch_->Clear();  

    versions_->SetLastSequence(last_sequence);  
  }  

// -----E begin-------  
  while (true) {  
    Writer* ready = writers_.front();  
    writers_.pop_front();  
    if (ready != &w) {  
      ready->status = status;  
      ready->done = true;  
      ready->cv.Signal();  
    }  
    if (ready == last_writer) break;  
  }  
// -----E end -------  


// -----F begin-------  
  // Notify new head of write queue  
  if (!writers_.empty()) {  
    writers_.front()->cv.Signal();  
  }  
// -----F end-------  

  return status;  
}</code></pre>


如上，A段代码定义一个Writer w, w的成员包括了batch信息，同时初始化了一个条件变量成员(port::CondVar)  

假设同时有w1, w2, w3, w4, w5, w6 并发请求写入  

B段代码让竞争到mutex资源的w1获取了锁。添加到writers队列里去，此时队列只有一个w1, 从而其顺利的进行BuildBatchGroup。当运行到c段代码时，mutex互斥锁释放，这时(w2, w3, w4, w5, w6)会竞争锁，由于B段代码中不满足队首条件，均等待并释放锁了。从而队列可能会如(w3, w5, w2, w4).  

当w1完成log写入和memtable写入，进入d段代码，则mutex又锁住，这时B段代码中队列因为获取不到锁则队列不会修改。  

进入E段代码后，w1被pop出来，由于reader==w, 并且ready==last_writer,所以直接到F段代码，唤醒了此时处于队首的w3.  

w3唤醒时，发现自己是队首，可以顺利的进行进入BuildBatchGroup，在该函数中，遍历了目前所有的队列元素，形成一个update的batch，即将w3, w5, w2, w4合并为一个batch. 并将last_writer置为此时处于队尾的最后一个元素w4，c段代码运行后，因为释放了锁资源，队列可能随着DBImpl::Write的调用而更改，如队列状况可能为(w3, w5, w2, w4, w6, w9, w8).  

C段和D段间的代码将w3, w5, w2, w4整个的batch写入log和memtable.  
    



