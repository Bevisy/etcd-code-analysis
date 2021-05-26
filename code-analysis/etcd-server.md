# etcd-server

## server start
```shell
main.main
|-etcdmain.Main()
  |-etcdmain.startEtcdOrProxyV2()
    |-cfg := newConfig()
    |-stopped, errc, err = startEtcd(&cfg.ec)
      |-embed.StartEtcd()
        |-err = inCfg.Validate()
        |-e.Server, err = etcdserver.NewServer(srvcfg) // 初始化server，包括 backend 初始化（bbolt）
        |-e.Server.Start()
          |-EtcdServer.start() // prepares and starts server in a new goroutine
            |-go s.run() // new goroutine
              |-sn, err := s.r.raftStorage.Snapshot()
                |-
              |-sched := schedule.NewFIFOScheduler()
              |-rh := &raftReadyHandler{} // 提供 raftNode handler
              |-s.r.start(rh) // start prepares and starts raftNode in a new goroutine
              |-[for]

```
