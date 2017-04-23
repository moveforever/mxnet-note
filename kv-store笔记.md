KVStore初始化
---
- python: init_optimizer
- kvstore create:
  ```
  src/kvstore/kvstore.cc:17
  KVStore* KVStore::Create(const char *type_name)
  ```
  > 建立local或者dist模型KVStore
    - DIST kvstore
      - 初始化KVStoreDist类
        ```
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
        >  PostOffice start
          ```
          void Postoffice::Start(const char* argv0, const bool do_barrier){van->Start()}
          ```
      - 如果是异步模式，则
        ```
        kv->SendCommandToServers(kvstore::kSyncMode, "");
        ```

kv-dist 内部运行
---
- python:`_init_kvstore_server_module()`
- 发送`void RunServer(const Controller& controller)`
- 初始化KVStoreDistServer
  ```
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

- KVServer
  > Process{request_handle_}
  > Customer bind Process//(const RecvHandle& recv_handle)
    ```
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
    ```
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
  ```
  void Van::Receiving() {
    while(1) {
      RecvMsg(&msg);
      obj->Accept(msg);
    }
  }
  ```
  ```
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
