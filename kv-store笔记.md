KVStore初始化
---
- 通过python函数init_optimizer初始化
  - 调用kvstore的C++函数create:
  ```C++
  //src/kvstore/kvstore.cc:17
  KVStore* KVStore::Create(const char *type_name) {
    std::string tname = type_name;
    std::transform(tname.begin(), tname.end(), tname.begin(), ::tolower);
    KVStore* kv = nullptr;
    bool use_device_comm = false;
    auto has = [tname](const std::string& pattern) {
      return tname.find(pattern) != std::string::npos;
    };  
    if (has("device")) {
      use_device_comm = true;
    }

    if (has("dist")) {
//if MXNET_USE_DIST_KVSTORE
      kv = new kvstore::KVStoreDist(use_device_comm);
      if (!has("_async") && kv->IsWorkerNode() && kv->get_rank()  == 0) {
      // configure the server to be the sync mode
        kv->SendCommandToServers(kvstore::kSyncMode, "");
      }
//else
      LOG(FATAL) << "compile with USE_DIST_KVSTORE=1 to use " << tname;
      return nullptr;
//endif  // MXNET_USE_DIST_KVSTORE
    } else {
      kv =  new kvstore::KVStoreLocal(use_device_comm);
    }
    kv->type_ = tname;
    return kv;
  }
  ```

  - 建立local或者dist模型KVStore
    - 单机：KVStoreLocal
    - 分布式：KVStoreDist
      - 初始化KVStoreDist类
        ```C++
        explicit KVStoreDist(bool use_device_comm)
        : KVStoreLocal(use_device_comm), ps_worker_(nullptr), server_(nullptr) {
          if (IsWorkerNode()) {
            ps_worker_ = new ps::KVWorker<real_t>(0);
            ps::StartAsync("mxnet\0");
            if (!ps::Postoffice::Get()->is_recovery()) {
              ps::Postoffice::Get()->Barrier(
                ps::kWorkerGroup + ps::kServerGroup + ps::kScheduler);
            }
          }
          bigarray_bound_ = dmlc::GetEnv("MXNET_KVSTORE_BIGARRAY_BOUND", 1000 * 1000);
        }
        ```
        -  PostOffice start
          ```C++
          inline void StartAsync(const char* argv0 = nullptr) {
            Postoffice::Get()->Start(argv0, false);
          }
          void Postoffice::Start(const char* argv0, const bool do_barrier) {  
            //......
            van->Start()
          }
          ```
      - 如果是同步模式，创建KVStoreDist时会给Server发个命令
          ```C++
          kv->SendCommandToServers(kvstore::kSyncMode, "");
          // 具体函数
          void KVStoreDist::SendCommandToServers(int cmd_id,
                                  const std::string& cmd_body) override {
            CHECK_NOTNULL(ps_worker_);
            ps_worker_->Wait(ps_worker_->Request(cmd_id,cmd_body,                                                       ps::kServerGroup));
          }
          ```

kv-dist Server运行
---
- 当本机是server节点时，通过python函数启动KVStoreDistServer
  ```python
  def _init_kvstore_server_module():
    """Start server/scheduler"""
    is_worker = ctypes.c_int()
    check_call(_LIB.MXKVStoreIsWorkerNode(ctypes.byref(is_worker)))
    if is_worker.value == 0:
        kvstore = create('dist')#初始化KVStoreDist
        server = KVStoreServer(kvstore)
        server.run()#给server发命令，打包发送优化算法的函数
        sys.exit()
  ```
- 初始化KVStoreDistServer
    ```C++
      KVStoreDistServer() {
        using namespace std::placeholders;
        ps_server_ = new ps::KVServer<float>(0);
        static_cast<ps::SimpleApp*>(ps_server_)->set_request_handle(
            std::bind(&KVStoreDistServer::CommandHandle, this, _1, _2));
        ps_server_->set_request_handle(
        std::bind(&KVStoreDistServer::DataHandle, this, _1, _2, _3));
        sync_mode_ = false;
      }
    ```
    > ps::KVServer<float>* ps_server_;

    > set_request_handle(CommandHandle)//SimpleApp

    > set_request_handle(DataHandle)//KVServer

- 给server发命令，初始化control
  ```C++
  void KVStoreDist::RunServer(const Controller& controller) override {
    CHECK(!IsWorkerNode());
    if (IsServerNode()) {
      server_ = new KVStoreDistServer();
      server_->set_controller(controller);
    }

    ps::StartAsync("mxnet_server\0");
    if (!ps::Postoffice::Get()->is_recovery()) {
      ps::Postoffice::Get()->Barrier(
              ps::kWorkerGroup + ps::kServerGroup + ps::kScheduler);
    }
    if (server_) server_->Run();
    ps::Finalize();
    if (server_) {
      delete server_;
    }
    server_ = nullptr;
  }
  ```

- KVServer
  > Process{request_handle_}
  > Customer bind Process//(const RecvHandle& recv_handle)
    ```C
    Process() {
       response_handle_
       request_handle_
    }
    ```

- SimpleApp
  > Process{request_handle_}
  > Customer bind Process//(const RecvHandle& recv_handle)

- Customer
  > 处理信息：Receiving函数不断调用recv_handle_函数处理
    ```C++
    Customer::Customer(int id, const Customer::RecvHandle& recv_handle)
            : id_(id), recv_handle_(recv_handle) {
      Postoffice::Get()->AddCustomer(this);
      recv_thread_ = std::unique_ptr<std::thread>(new std::thread(&Customer::Receiving, this));
    }
    void Customer::Receiving() {
      while (true) {
      Message recv;
      recv_queue_.WaitAndPop(&recv);
      if (!recv.meta.control.empty() &&
        recv.meta.control.cmd == Control::TERMINATE) {
        break;
      }
      recv_handle_(recv);
      if (!recv.meta.request) {
        std::lock_guard<std::mutex> lk(tracker_mu_);
        tracker_[recv.meta.timestamp].second++;
        tracker_cond_.notify_all();
      }
    }
  }
  ```
  > 接收信息：通过Van类不断接收信息，然后放到Customer类中
  ```C++
  void Van::Receiving() {
    while(1) {
      RecvMsg(&msg);
      obj->Accept(msg);
    }
  }
  ```
  ```C++
  void Customer::Accept(const Message& recved) { recv_queue_.Push(recved); }
  ```

  - Van类
    > 通过Start函数开始，然后开辟一个线程（Receiving)

    > 实际继承类ZMQVan，它的接口如下
      - Bind
      - Stop
      - Start
      - Connect
      - SendMsg
      - RecvMsg

更新操作
---
- kvlocal
  - push操作：CommCPU进行reduce聚合，然后再更新
  - pull操作：广播
- kv-dist
