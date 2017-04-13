## dep-engine 代码解析笔记
下面主要分析的是ThreadedEngine引擎
### 将操作和数据封装的过程如下：
1.  调用`PushAsync`函数，作用：将数据（const_vars,mutable_vars)和操作封装成`ThreadedOpr`结构体
```cpp
void ThreadedEngine::PushAsync(AsyncFn fn, Context exec_ctx,
                             std::vector<VarHandle> const& const_vars,
                             std::vector<VarHandle> const& mutable_vars,
                             FnProperty prop, int priority)
```

2. 调用`Push`函数
  + 将`ThreadedOpr`、context、依赖的变量数、优先级等数据进一步封装成`OprBlock`数据结构

  + 添加读写依赖
  ```cpp
  // Add read dependencies.
  for (auto&& i : threaded_opr->const_vars) {
      i->AppendReadDependency(opr_block);
  }
  // Add write dependencies.
  for (auto&& i : threaded_opr->mutable_vars) {
      i->AppendWriteDependency(opr_block);
  }
  ```
  + 当这个操作的变量没有任何等待，那直接可以
  ```cpp
  if (opr_block->decr_wait() == 0) {
    this->PushToExecute(opr_block, true);
  }
  ```

3. `PushToExecute`函数
    + 异步模式
    ```cpp
    if (opr_block->opr->prop == FnProperty::kAsync && pusher_thread) {
        if (ctx.dev_mask() == gpu::kDevMask) {
    #if MXNET_USE_CUDA
        MSHADOW_CATCH_ERROR(mshadow::SetDevice<gpu>(ctx.dev_id));
    #endif
    }
    RunContext run_ctx;
    run_ctx.stream = nullptr;
    this->ExecuteOprBlock(run_ctx, opr_block);
    ```

4. `ExecuteOprBlock`函数
    +   构建回调函数
    +   非关闭模式时，执行AsyncFn fn；关闭模式时，直接执行回调函数callback
```cpp
		/*! \brief Asynchronous operation to pass to engine. */
		typedef std::function<void(RunContext, CallbackOnComplete)> AsyncFn;
```
