# Deis 故障排查

本文汇总了用户在安装使用Deis时常见的一些问题。
## 登录到集群
Deis运行在CoreOS之上，所以连接它很简单，使用`ssh`即可。CoreOS默认的用户名是core，使用预分配SSH key即可登录集群。
连接到具有公网ip的任意一个节点：
```
$ ssh core@deis-1.example.com -i ~/.ssh/deis.pub
```

## deis-store 组件启动失败

存储组件是 Deis 中最复杂的组件。正因为如此，有很多方法导致它失败。存储组件代表 Ceph 的服务如下：

 - store-monitor: http://ceph.com/docs/giant/man/8/ceph-mon/
 - store-daemon: http://ceph.com/docs/giant/man/8/ceph-osd/
 - store-gateway: http://ceph.com/docs/giant/radosgw/
 - store-metadata: http://ceph.com/docs/giant/man/8/ceph-mds/
 - store-volume: 挂载了一个 Ceph FS 卷的系统服务，被用于控制器和记录器组件

存储组件的日志输出可以通过使用命令 `deisctl status store-<component>` （比如 `deisctl status store-volume`）查看。另外，Ceph 的健康可以通过进入一个存储容器使用命令 `nse deis-store-monitor` ，然后键入命令 `ceph -s` 查询。集群的健康输出应该像这样：

```
core@deis-1 ~ $ nse deis-store-monitor
root@deis-1:/# ceph -s
    cluster 20038e38-4108-4e79-95d4-291d0eef2949
     health HEALTH_OK
     monmap e3: 3 mons at {deis-1=172.17.8.100:6789/0,deis-2=172.17.8.101:6789/0,deis-3=172.17.8.102:6789/0}, election epoch 16, quorum 0,1,2 deis-1,deis-2,deis-3
     mdsmap e10: 1/1/1 up {0=deis-2=up:active}, 2 up:standby
     osdmap e36: 3 osds: 3 up, 3 in
      pgmap v2096: 1344 pgs, 12 pools, 369 MB data, 448 objects
            24198 MB used, 23659 MB / 49206 MB avail
            1344 active+clean
```

如果你看到 `HEALTH_OK`，这意味着一切工作正常，注意 `monmap e3: 3 mons at...` 也意味着所有三个监控中的容器是启动的并且正常响应了，`mdsmap e10: 1/1/1 up... ` 意味着所有三个元数据（metadata）容器是启动的并且正常响应了，`osdmap e7: 3 osds: 3 up, 3 in ` 意味着所有三个 `daemon` 容器是启动的并且正运行着。

我们也能从 `pgmap` 看到我们拥有 1344 个放置组（placement groups），所有的都是 `active+clean` 的。

关于 Ceph 故障排查的参考资料，请看[故障排查][1]。下面详细列出了指定存储组件的常见问题。
 
### store-monitor

监视器是第一个启动的存储组件，并且是任何其他存储组件正常运转所必需的。如果一个 `deisctl list` 命令表明任何监视器是失败的，那可能是由于主机问题。常见的故障情况包含在主机节点没有足够的存储空间 - 在那种情况下，监视器将失败，错误输出类似：

```
Oct 29 20:04:00 deis-staging-node1 sh[1158]: 2014-10-29 20:04:00.053693 7fd0586a6700  0 mon.deis-staging-node1@0(leader).data_health(6) update_stats avail 1% total 5960684 used 56655
Oct 29 20:04:00 deis-staging-node1 sh[1158]: 2014-10-29 20:04:00.053770 7fd0586a6700 -1 mon.deis-staging-node1@0(leader).data_health(6) reached critical levels of available space on
Oct 29 20:04:00 deis-staging-node1 sh[1158]: 2014-10-29 20:04:00.053772 7fd0586a6700  0 ** Shutdown via Data Health Service **
Oct 29 20:04:00 deis-staging-node1 sh[1158]: 2014-10-29 20:04:00.053821 7fd056ea3700 -1 mon.deis-staging-node1@0(leader) e3 *** Got Signal Interrupt ***
Oct 29 20:04:00 deis-staging-node1 sh[1158]: 2014-10-29 20:04:00.053834 7fd056ea3700  1 mon.deis-staging-node1@0(leader) e3 shutdown
Oct 29 20:04:00 deis-staging-node1 sh[1158]: 2014-10-29 20:04:00.054000 7fd056ea3700  0 quorum service shutdown
Oct 29 20:04:00 deis-staging-node1 sh[1158]: 2014-10-29 20:04:00.054002 7fd056ea3700  0 mon.deis-staging-node1@0(shutdown).health(6) HealthMonitor::service_shutdown 1 services
Oct 29 20:04:00 deis-staging-node1 sh[1158]: 2014-10-29 20:04:00.054065 7fd056ea3700  0 quorum service shutdown
```

当 Deis 部署在裸设备上时，这仅仅是比较有代表性的一个问题，虽然大部分的云提供者有足够大的卷。

### store-daemon

守护进程（daemons） 负责实际存储在文件系统上的数据。集群被配置成允许仅仅一个运行着的守护进程写，但这样集群将运行在降级状态（degraded state），所以尽快恢复所有守护进程运行状态是至关重要的。

守护进程可以使用 `deisctl restart store-daemon` 命令安全的重起，但这会重起所有的守护进程，结果是整个存储集群将不可用直到守护进程恢复。或者，在存在失败守护进程的主机上运行 `sudo systemctl restart deis-store-daemon` 指令将仅仅重起这个守护进程。

### store-gateway

网关（gateway）运行着 Apache 和一个 FastCGI 服务来与集群通信。重起网关 将导致注册组件短暂的停机（并且将阻止数据库备份），但是这些组件会随着网关的恢复而尽快恢复。

### store-metadata

元数据（metadata）服务器对于**卷（volume）** 的正常运转是必需的，在任何时候，仅仅只有一个是主，剩下的是作为热备。当主失败的时候，监视器将提升备份的元数据服务器为主。

### store-volume

没有运行监视器，守护进程和元数据服务器，卷服务将可能被无限的夯住（或者是频繁重起）。如果控制器和记录器恰好运行在一台有失效 store-volume 的主机上，应用日志将被丢失直到卷恢复。

注意由于 CephFS 内核模块， store-volume 要求 CoreOS >= 471.1.0。

## 任何组件启动失败

使用 `deisctl status <component>` 来查看组件状态。你也可以使用 `deisctl journal <component>` 来 tail 一个组件的日志，或者是使用 `deisctl list` 列出所有的组件。

## SSH 客户端初始化失败

deisctl 命令失败：‘Failed initializing SSH client: ssh: handshake failed: ssh: unable to authenticate’。是否记得把你的 SSH key 添加到 ssh-agent？` ssh-add -L` 会列出你用来供应给服务器的 key。如果没有，ssh-add -K /path/to/your/key。

## 所有给定的 peers 都不可达

deisctl 命令失败：‘All the given peers are not reachable (Tried to connect to each peer twice and failed)’。最可能引起该问题的原因是 [new discovery URL](https://discovery.etcd.io/new) 没有生成，并且在集群启动之前没有被更新到 `contrib/coreos/user-data`。每个 Deis 集群必须有一个唯一的 discovery URL，否则 etcd 将重试并且连接老主机失败。尝试销毁集群并且使用一个新的 discovery URL 启动集群。

你可以使用 `discovery-url` 来自动获取一个 new discovery URL。

## 其他问题

运行的东西在这里没有详细介绍？请 [open an issue][2] 或者是进到 Freenode IRC 的 #deis，我们将提供帮助！


  [1]: http://docs.ceph.com/docs/giant/rados/troubleshooting/
  [2]: https://github.com/deis/deis/issues/new
