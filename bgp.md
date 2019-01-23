# BGP

[TOC]

## 基本知识

- AS号范围1-65535, 64512-65535是保留私用的
- BGP的更新是由TCP承载的,是用的端口号是179
- BGP通过Neighbor命令建立起基于TCP的BGP连接以后,就称他们为邻居或对等体
- BGP的更新是触发更新
- BGP是设计在AS之间传递路由的,它的一跳实际就是一个AS
- BGP是无类路由选择协议,距离矢量路由协议,自动汇总默认关闭
- BGP有两个管理距离, IBGP学过来的200, EBGP学过来的20

## BGP的表

1. 邻居表: 查看命令`show ip bgp summary`
2. BGP路由表: 查看命令`show ip bgp`
3. 路由表: 查看命令`show ip route`

## BGP Timer

- Keepalive: 60S
- Hold Time: 3倍的keepalive, 默认180S
- 两端**不一定**要一致,取小的一方
- `timer bgp 0 0`邻居永远不down
- BGP为触发更新, IBGP默认5S, EBGP默认30S

## BGP消息类型

- open
- keepalive
- update
- notification
- router-refresh

## BGP状态机

- idle
- connect
- active
- opensent
- opencomfirm
- established

## BGP邻居关系建立的条件

- TCP连接建立(IGP可达)
- 端口179可达
  - 交换update message
- neighbor是否定义正确
- neighbor认证
- AS号声明正确
- EBGP/IBGP multihop

## BPG邻居的建立

- Neighbor指向相邻AS, `Neighbor 1.1.1.1 remote-as 234`
- EBGP通常用直连接口建立, EBGP默认**TTL=1**
- IBGP通常用Loopback接口建立
- update-source `Neighbor 1.1.1.1 update-source l0`, update-soruce只要一端声明即可
- 如果一端是默认路由,一端是静态路由,两者可以建立BGP邻居,**有明细路由的一端发起连接**, 如果两端都是默认路由, 则不能建立邻居
- EBGP多跳 `Neighbor 1.1.1.1 ebgp-multihop 2`
- EBGP AS边界路由器上通常**对所有邻居**配置`Neighbor 1.1.1.1 next-hop-self`, 以解决IBGP内部下一跳不可达的问题

## IBGP水平分割

- BGP规定不将通过IBGP学到的路由传播给其他IBGP邻居-BGP的水平分割规则

## IBGP和IGP同步(BGP黑洞)

![Image_2019-01-16_09-14-04.png](.\assets\Image_2019-01-16_09-14-04.png)

- 上图中A路由器上1.1.1.1的路由可以传递到F, BGP的控制层没有问题, 路由可以传递
- 但在当F路由器试图ping A路由器是不通的, 因为C和D只运行了OSPF, 并没有运行BGP,并不知道1.1.1.0的数据怎么传递, 至此形成路由黑洞
- **IBGP和IGP同步**: BGP路由器从IBGP邻居学到一条路由后, 是不启用的(不是最优的), 除非它再次从IGP学到相同的路由,才会启用.思科默认是关闭的
- 解决IBGP路由黑洞问题的方法
  - BGP重发布进入OSPF(通常不用)
  - IBGP全互联(且关闭IGP同步)
  - 添加物理链路
  - 路由反射器
  - 联邦

## BGP路由和路由递归

- BGP只传递最优的路由(*>)
- BGP路由递归
  ![Image_2019-01-16_10-13-50.png](.\assets\Image_2019-01-16_10-13-50.png)
  - 去往3.3.3.0的路由下一跳为2.2.2.2, 递归查找2.2.2.2, 关联接口s0/0

## 联邦

### 联邦的特性

- 在联邦内部保留联邦外部Next_hop属性
- 公布给联邦的路由的MED属性在整个联邦范围内予以保留
- 路由的LocalPreference属性在整个联邦范围内予以保留
- 在联邦范围内,将成员AS号压入AS_Path, 但不公布到联邦外,并且使用Type3,4 的AS_Path
- AS_Path中的联邦AS号用于在联邦内部避免环路

### BGP联邦的配置

![Image_2019-01-16_10-24-20.png](.\assets\Image_2019-01-16_10-24-20.png)


- R3配置

    ```
    router bgp 64512
    ! 对外宣告真实的AS号
    bgp confederation identifier 345
    ! 联邦内部的IBGP邻居
    neigbhor 4.4.4.4 remote-as 64512
    neigbhor 4.4.4.4 update-source Loopback1
    neighbor 10.1.13.1 remote-as 100

  ```

- R4配置

    ```
    router bgp 64512
    bgp confederation identifier 345
    ! 联邦内的IBGP邻居
    bgp confederation peer 64512
    neighbor 3.3.3.3 remote-as 64512
    neighbor 3.3.3.3 update-source Loopback0
    ! 联邦内EBGP邻居
    neighbor 5.5.5.5 remote-as 64513
    ! 联邦内EBGP邻居也要配置EBGP-Multihop
    neighbor 5.5.5.5 ebgp-multihop 3
    neighbor 5.5.5.5 update-source Loopback0
  ```

## 路由反射器RR

- RR不修改Next-Hop, AS_Path, Local_Preference和MED,并且增加了Originator和Cluster_list用于防环
- 如果学习自非client,则反射给所有的client和EBGP邻居
- 如果学习自client,则反射给所有非client的IBGP邻居和该client以外的所有client
- 如果学习自EBGP邻居, 则反射给所有的client和非client IBGP邻居
- 总结: 非client传过来的路由不会通过RR再传给另外一个非client
- RR的冗余: 一个簇可以有多个RR, RR也可是别人的client

### RR配置

-RR上面配置

```
router bgp 123
neighbor 1.1.1.1 remote-as 123
neighbor 1.1.1.1 update-source Loopback0
!配置1.1.1.1成为RR的client
neighbor 1.1.1.1 router-reflector-client
```
- 修改cluster-id的命令

```
router bgp 123
bgp cluster-id 22.22.22.22
```

## BGP汇总

- auto-summary(默认关闭)
  - 只汇总通过**重发布**宣告的路由
  - 对network宣告的路由不进行自动汇总
  - 自动汇总后不携带Next-Hop和Metric值
- 手工汇总
  - 任何地方都可以汇总,但是汇总路由器上必须要汇总的明细路由
  - `aggregate-address x.x.x.x 参数`
- Aggregate-address 汇总可带参数
  - summary-only (不带summary-only, 明细和汇总都传递)
  - as-set (防止AS path环路,传递AS信息)
  - suppress-map (通告汇总路由,抑制某些路由)
  - attribute-map (修改汇总路由继承明细路由的部分属性)
  - advertise-map (先汇总,在有条件通告)
- 汇总传递的属性
  - summary-only丢失AS_Path属性
  - summary-only as-set
    - 将收到的所有明细路由的AS好都放在{}中
    - Origin: 继承最差的Orign属性
    - Community: 继承所有明细路由的community
    - MED: 不继承
    - LP: 取明细路由中LP的最大值
    - Next_hop: 汇总路由为0.0.0.0
- summary-only会抑制掉所有明细路由,但是配合unsuppress-map使用可以放行部分明细路由

  ```
  access-list 1 permit 172.16.1.0
  router-map unsupp permit 10
    match ip address 1
  router bgp 300
    neighbor 10.1.35.5 unsuppress-map unsupp
    aggregate-address 172.16.0.0 255.255.0.0 as-set summary-only
  ```

- aggregate-address和attribute-map结合使用例子

  ```
  router bgp 12
  route-map test permit 10
    set origin incomplete
    set metric 101
  aggregate-address 192.168.0.0 255.255.0.0 attribute-map test

  access-list 1 deny 192.168.0.0 0.0.255.255
  neighbor 1.1.23.3 distribute-list 1 out
  // 防止192.168.0.0回传给明细路由器,明细路由器上可以部署相应配置

  ```
- aggregate-address汇总地址as-set advertise-map

  ```
  ip prefix-list 1 permit 172.16.1.0/24
  ip prefix-list 1 permit 172.16.2.0/24
  route-map test permit 10
    match ip address prefix-list 1
  router bgp 300
    aggregate-address 172.16.0.0 255.255.0.0 as-set summary-only advertise-map test
  ```

### set community no-advertise和aggregate结合使用的例子
![Image_2019-01-17_11-00-57.png](.\assets\Image_2019-01-17_11-00-57.png)
```
R3宣告了192.168.1.0, 2.0同时设置1.0的community属性no-adv
access-list 1 permit 192.168.1.0
route-mpa test permit 10
  match ip address 1
  set community no-advertise
route-map test permit 20
  set community none
router bgp 3
  ! send-community开启传送community功能
  neighbor 1.1.23.2 send-community
  ! out方向
  neighbor 1.1.23.2 route-map test out

```
- 默认情况下R2可以学习到1.0,2.0, R1只能学习到2.0
- 当R2设置`aggregate-address 192.168.0.0 255.255.0.0`之后, R1能学习到汇总路由和2.0
- 当R2设置`aggregate-address 192.168.0.0 255.255.0.0 as-set`之后, R1能学习到2.0, 汇总路由被抑制掉了, 原因是1.0携带了no-adv的community, ***汇总路由加了as-set之后, 会继承它的community***,这个时候R2需要添加如下设置

  ```
  access-list 11 deny 192.168.1.0
  access0list 11 permit any
  route-map adv permit 10
  match ip address 11
  router bgp 12
    aggregate-address 192.168.0.0 255.255.0.0 as-set advertise-map adv
  // 即可忽略1.0的community
  //注意:构建汇总路由时,不考虑advertise-map中拒绝的路由,而单纯通过adv-map是无法过滤明细路由的
  ```
- 正确使用: exist-map/non-exist-map和advertise-map一起使用
  - `neighbor x.x.x.x advertise-map ADV exist-map EXI`
  - `neighbor x.x.x.x advertise-map ADV non-exist-map EXI`


### Route-map和perfixlist permit和Deny的组合

- 下面的组合假设和aggregate-address x.x.x.x suppress-map xxx as-set
- `aggregate-address 11.0.0.0 255.0.0.0 suppress-map test`

 ```
  ip prefix-list 1 permit 11.11.11.0/24
  route-map test permit 10
    match ip address prefix 1
  // 干掉11.0, 放行除了11.0以外的所有明细

  ip prefix-list 1 permit 11.11.11.0/24
  route-map test deny 10
    match ip address prefix 1
  // 等同于route-map不匹配(permit)任何条目,因此所有明细都放行

  ip prefix-list 1 deny 11.11.11.0/24
  route-map test permit 10
    match ip address prefix 1
  // 效果同上,prefix没有permit任何

  router-map test permit 10
  // 这样写就是匹配全部,和suppress-map联系起来就是明细全都不放行

  route-map test deny 10
  // 明细全放行
 ```

### Exist-map, Inject-map和BGP Deaggregate(BGP拆分)

- `bgp inject-map map1 exist-map map2`
- Exist-map使用的route-map最少具有以下两个match语句
  - `match ip address prefix-list`
  - `match ip route-source` 匹配发生汇总路由的邻居IP
- Inject-map使用的route-map中
  - `set ip address prefix-list`
  - `show ip bgp injected-path`用来显示
- 例子
![Image_2019-01-17_11-27-41.png](.\assets\Image_2019-01-17_11-27-41.png)
- R2上配置如下
  ```
  ip prefix-list huizong permit 172.16.0.0/16  //用来匹配汇总路由
  ip prefix-list mingxi permit 172.16.1.0/24   // 用来定义准备注入的条件前缀
  ip prefix-list xiayitiao permit 10.1.24.4/32 // 用来匹配传递给我汇总路由的BGP邻居,这里是R4的IP
  route-map RP_mingxi permit 10
    set community 100:200 no-export
    set ip address prefix-list mingxi
  route-map RP_huizong permit 10
    match ip address prefix-list huizong
    match ip source xiayitiao
  router bgp 300
    bgp inject-map RP_mingxi exist-map RP_huizong copy-attributes
    neighbor 10.1.23.2 remote-as 200

  ```
## 不同协议间的重分布

- 任何协议重分布进RIP, 必须加Metric
- 任何协议重分布EIGRP要加5个K
- 任何协议重分布进OSPF不需要加
- 任何协议重分布进BGP不需要加
- 重分布删除时有可能要删除两次 


## BGP Dampenig

- 当路由出现摆动（不断失效、恢复),就给它分配一个惩罚值，摆动越多，惩罚值越大并且不停地积累。 而同时惩罚值又以一定的速率降低，每一个半衰期结束，惩罚值变成原来的一半（如果路由不再翻动的话） 如果惩罚值超出了预先设置的门限“抑制界限：惩罚值达到这个门限，路由被抑制，即不发布，直到N个半衰期 以后，惩罚值降低到另一个门限：重新使用门限（解除抑制的门限）时，才解除对路由的抑制。 
- 惩罚值: 每次摆动增加1000
- 抑制界限: 2000
- 重新使用界限:750
- 半衰期:15分钟
- 最大抑制时间:60分钟（半衰期的4倍）
- 配置

  ```
  bgp dampening   //开启
  bgp dampening 半衰期 重新使用界限 拟制界限 最大拟制时间
  ```

- `show ip bgp` 如果路由标记D,则表示该路由被抑制,如果标记H,则表示路由有翻动
- `show ip bgp flap`
- `show ip bgp dampening` 查看哪些路由被拟制

## BGP路径属性

- 公认属性(well-know)
  - 公认必遵(well-know mandatory)
    - BGP必须都能识别,且在更新消息必须包含
    - Origin, AS-Path, Next Hop|
  - 公认自决(well-know discretinary)
    - BGP必须都能识别,更新消息可包含可不包含
    - Local-Preference, ATOMIC_Aggregate
- 可选属性(option)
  - 可选传递(optional transitive)
    - 可以不支持该属性,但即使不支持,也应当接收包含该属性的路由并传递给其他邻居
    - Community, Aggregator
  - 可选非传递(optional non-transitive)
    - 可以不支持该属性,不识别的BGP进程忽略包含这个属性的更新消息,并且不传递给其他BGP邻居
    - MED,Originate_ID,Cluster_list, Weight

### Origin

- i: IGP
- ?: incomplete
- e: EGP (已经淘汰)
- **优选原则**: i>e>?

### AS-Path

- 防环
- AS-path越短越好
- AS_Path的四种类型
  - AS_set: 无序的AS列表
  - AS_Sequence: 有序列表
  - AS_confed_sequence: 联邦内AS列表
  - AS_confed_set: 去往联邦内特定目的地无序AS列表
- 控制方式`set as-path prepend 666`

  ```
  ip prefix-list 1 permit 1.1.1.0/24
  route-map test
    match ip address prefix-list 1
    set as-path prepend 666
  router b 123
    neighbor 10.1.13.3 route-map test out
  ```
- ***注意***: AS-path只能在BGP路由器对其EBGP邻居执行, AS-PATH只有在AS边界才发生变化
- AS-PATH的几个命令
  - `neighbor 路由来源邻居 allowas-in 几个我的AS号`
  - `bgp bestpath as-path ingnore` 在其决策过程中忽略AS-path属性

### Next-hop

BGP next-hop规则如下
1. 如果BGP路由传递自EBGP Peer
  - 这条BGP路由的Next-hop就是通告者的接口IP, 直连接口或者loopback接口地址
2. 如果路由传递自IBGP邻居, 并描述的是AS外的目的地
  - next-hop始终指向下一个AS(即EBGP Peer的接口IP)
  - 在IBGP AS内保持不变
3. 如果路由传递在IBGP邻居,并由AS内BGP路由器引入
  - 如果是通过Aggregate-address命令注入, 那么next-hop等于执行汇总路由器的更新源IP
  - 如果是通过network或者重发布注入, 那么在注入前该前缀的IGP下一跳将成为BGP的next-hop
  - 如果本地的BGP宣告着成了下一跳地址, 那么在本地BGP RIB中的下一跳字段就是0.0.0.0
4. 通过next-hop-self可以修改next-hop属性
  - 通常在EBG的边界路由器上对身后的IBGP邻居使用

### Local-Preference

- 只能在IBGP邻居直接传递,不会传递给其他EBGP邻居
- local-pref***影响AS内的选路***
- 缺省100 (**选大的LP值**)
- 当一个AS收到去往同一目的地,但经过两个AS的路由, 则根据两条路由的local-pref值
- 在路由协议下配置`bgp default local-preference 500` 修改默认的LP值
- 或者可以通过route-map `set local-preference 101`来修改某条具体路由的local-preference值

### MED

- 非传递,用于***影响AS之间路由***
- 默认情况下, 只比较来自同一邻居AS的BGP路由的MED值,就是说如果同一个目的地的两条路由来自不同的AS，则不进行MED值的比较。换句话说，默认对于多条路径，只有在AS_SEQUENCE中的第一个AS相同的情况下，才比较MED；任何打头的AS_CONFED_SEQUENCE都将被忽略。MED只是在直接相连的自治系统间影响业务量，而**不会跨AS传递**
- MED设置方式
  - IGP引入BGP时通过Route-map进行设置
  - 对BGP Peer应用in/out方向的route-map
  - network或redistribute方式将IGP路由引入BGP时, MED将继承IGP路由的Metric(直连和静态路由metric为0)
  - aggregate-address方式引入的路由, MED为空
- MED值的传递
  - MED值会在IBGP之间传递
  - 本地始发(network或redistribute)的,则携带MED值发送给EBGP peer(如果为空, 则置0)
  - 如果是层其他BGP peer学过来, 则MED不会被传出本AS
- MED值继承IGP的Metric
  - network/redistribute本地从IGP路由协议学习到的路由进BGP，MED值继承IGP协议中的metric
  - network/redistribute本地直连接口的网段进BGP，MED值为0；network/redistribute本地静态路由进BGP，MED值为0
- 其他配置命令
  - `bgp always-compare-med`
  - `bgp bestpath med missing-as-worst`
  - `set metric-type internal`
  - `bgp bestpath med confed`选路时,路由器只比较所有带有AS_CONFD_SEQ属性的路由条目
  - `bgp deterministic-med`如果这条命令和`bgp always-compare-med`同时使用下面这种情况会2,3同AS内先比较,2的MED小所以优选, 2再和1比较,由于打开了`always-compare-med`所以比较MED,2的MED小,所以2为最佳路径
  ![Image_2019-01-18_09-42-20.png](.\assets\Image_2019-01-18_09-42-20.png)
- `default-metric x` 修改MED值的缺省值

### Community

- 可选传递属性,用于简化路由策略的执行
- 是一组4个8位组的数值,RFC1997规定前2B表示AS豪, 后2B表示基于管理项目的设置的标示符, 格式AA:NN, `ip bgpcoummunity new-format`改为RFC新格式
- 需要配置`neighbor x.x.x.x send-community`
- 在route-map中set community

  ```
route-map test
set community ?
  <1-4294967295>  community number
  aa:nn           community number in aa:nn format
  gshut           Graceful Shutdown (well-known community)
  internet        Internet (well-known community) //默认所有路由都属于该团体
  local-AS        Do not send outside local AS (well-known community) //只能在本AS内传递(如果定义了联邦,则只能在本联邦成员AS内传递)
  no-advertise    Do not advertise to any peer (well-known community)  //不通告给任何BGP邻居
  no-export       Do not export to next AS (well-known community)  //不通告给任何EBGP邻居
  none            No community attribute

  ```
- 例子

  ```
  ip prefix 11 permit 11.11.11.0/24
  route-map test permit 10
    match ip address prefix-list 11
    set community 100:11
  router bgp 100
    network 11.11.11.0 mask 255.255.255.0
    neighbor 10.1.12.2 remote-as 200
    neighbor 10.1.12.2 send-Community
    neigbhor 10.1.12.2 route-map test out

  ```
- 使用的时候使用`ip community-list`匹配community值

  ```
  ip community-list 11 permit 100:11
  route-map test permit 10
    match community 11
    set community no-export additive
  route bg 200
    neighbor 10.1.23.3 remote-as 300
    neighbor 10.1.23.3 send-community
    neighbor 10.1.23.3 route-map test out

  ```
- community的逻辑关系
  - `ip community 11 permit 100:11 no-advertise`同时包含100:11和no-advertise
  - 下面是或的关系 
    ```
    ip community-list 11 permit 100:11
    ip community-list 11 permit no-exprot
   ```
- 删除community

  ```
  ip community-list 1 permit no-exprot
  route-map test permit 10
    set community-list 1 delet
  ```
- cost community
  - 使用`set extcommunity cost x`在route-map去设置
  - 使用`neighbor send-community`的时候要加上extend关键字
    ```
    ip prefix 100 permit 100.100.100.0/24
    route-map COST1 permit 10
      match ip address prefix-list 100
      set extcommunity cost 1 10
    route-map COST2 permit 10
      set extcommunity cost 1 5
    router bgp 123
        neighbor 1.1.1.1 route-map COST1 in
        neighbor 3.3.3.3 route-map COST2 in
    ```
  -  按照下面这样配置,COST的比较会超越weight,成为最优选比较原则
    ```
    route-map COST permit 10
      match ip address prefix-list 100
      set extcommunity cost pre-bestpath 1 10
    ```
### Atomic_Aggregate和Aggregate

- Atomic_Aggregate公认自决属性, 可选可传递
- 当上游路由器都带相同的明细路由, 而汇总路由器汇总时不带AS-set 关键字时, 下游路由器接收到的汇总路由是带atomic-aggregate属性的, 会明确指出该汇总路由是由那台路由器产生的. 
- 而当汇总路由器汇总时带上AS-set关键字是, 汇总路由就继承了明细的AS-Path属性, 这时汇总路由就不带Atomic_Aggregate的属性了

### Originator_ID 和 Cluster_list

- AS_path属性在AS内部是不会发生变化的(只有在离开本AS才会发生变化)
- 这两个属性是反射器使用的可选非传递属性
- Originator_ID指的是本地AS中路由器发起方的IBGP routerID
- Cluster_list是一串路由传递所进过的路由反射簇的ID

### Weight

- 思科私有, 0-65535
- 如果是学过来的, 默认是0, 本地network,重发布直连,静态路由, 本地汇总产生的路由weight是32768

## BGP配置

- `router bgp as`
- `network xxx mask yyy route-map zzz`, network宣告的是路由, 子网掩码必须和路由表中的相匹配
- `neighbor x.x.x remote-as yy`
- `neighbor x.x.x ebgp multi-hop 2`
- `neighbor x.x.x update source l0`

### BGP重发布

1. BGP重发布到IGP
  - 默认不重发布IBGP路由,只重发布EBGP路由, 需要使用`bgp redistribute-internal`来让IBGP路由顺利被重发布
  - ***MPLS VPN***的PE没有这个限制
2. OSPF重发布进BGP

```
router bgp x
  redistribute ospf 1
```
- 默认情况,只会讲OSPF中的O, O IA的路由重发布进BGP
- `redistribute ospf 1 match external 1 external 2`只重发布OSPF外部路由E1,E2进BGP
- `redistribute ospf 1 match internal 1 external 2`只重发布OSPF内部路由,外部路由E1进BGP
- `redistribute ospf 1 match nssa-external 1 nssa-external 2`只重发布OSPF NSSA路由进BGP

3. 不同协议的重分布时:
  - 任何协议重分布进RIP, 必须加Metric
  - 任何协议重分布EIGRP要加5个K
  - 任何协议重分布进OSPF不需要加
  - 任何协议重分布进BGP不需要加
  - 总结: ***分布进入距离矢量协议需要手工添加metric值***

### BGP PeerGroup配置

```
router bgp 1234
  neighbor IBGP_peers peer-group
  neighbor IBGP_peers remote-as 1234
  neighbor IBGP_peers update-source l0
  neighbor IBGP_peers password cisco
  neighbor IBGP_peers send-community

  neighbor 2.2.2.2 peer-group IBPG_peers
  neighbor 3.3.3.3 peer-group IBPG_peers
```

### BGP查看和验证

1. `show ip bgp`
2. **`show ip bgp summary`**
3. `show ip bgp rib-failure`
4. **`show ip bgp x.x.x.x`**
5. `show tcp brief`
6. `show ip bgp neighbor x.x.x.x received-routes`
7. `show ip bgp neighbors x.x.x.x routes`
8. `show ip bgp neighbors x.x.x.x advertised-routes`

### BGP邻居身份验证

- `neighbor xxx password xxx`
- MD5身份验证, *并不是加密*

### BGP邻居管理

- `neighbor maximum-prefix xx`
- `neighbor maximum-prefix xx restart 2 (mins)`
- `neighbor maximum-prefix 300 90% warning-only (logging to log)`
- BGP连接重置
  - `clear ip bgp *`
  - `clear ip bgp {neighbor-address}`
  - `clear ip bgp * soft`
  - `clear ip bgp soft out/in`
  - `clear ip bgp {neighbor-address} soft in/out`
- 存储路由更新信息的配置方法, 需要在接收方`router bgp x`下配置
  - `neighbor x.x.x.x soft-reconfiguration inbound`

## BGP policy accounting
- 用于计费

```
! 对30.30.30.0/24的路由设置community
ip prefix-list 30 permit 30.30.30.0/24
route-map setComm permit 10
  match ip address prefix-list 30
  set community 300:30

ip community-list 30 permit 300:30
route-map PA permit 10
  match community 30
  set traffic-index 30

router bgp 12
  table-map PA
  !对来自10.1.23.3设置community
  neighbor 10.1.23.3 route-map setComm in

!接口设置accounting
interface fa0/0
  ip address 10.1.12.2 255.255.255.0
  bgp-policy accounting input    
```

- 查看命令`show cef interface s0/0 policy-statistics input`

## BGP选路

| 简写 | 描述                                  | 比较方式               |
| :-   | :-                                    | :-                     |
| W    | highest weight                        | 大
| L    | highest local pref                    | 大
| L    | local originate route                 | 本地network或aggregate
| A    | shortest AS path                      | 短
| O    | origin type                           | (IGP>EGP>? incomplete)
| M    | smallest MED                          | 小
| N    | neighbor type                         | prefer Ebgp over IBGP
| I    | lowest IGP metric to the BGP next hop | 最小的IGP metric
| M    | Multipath                             | maximum-path 4
| O    | oldest path (oldest EBGP Neighbor)    | 最老的EBGP邻居
| R    | Lowest Router ID                      | 最小的router_id
| C    | shortest Cluster ID                   | RR中最短的路径
| N    | lowest neighbor address               | 最小的邻居地址
- WLLA, OMNI,MOR,CN

- BGP multipath配置`maximum-paths [ibgp/ebgp/eibgp] n`
- 可以进入BGP选路的**先决条件**
  - BGP路由最优(*>)
    - 下一跳可达
    - IGP和BGP路由同步, 如果开启路由同步功能
    - 入方向的BGP没有过滤
    - 路由没有被dampend

- BGP路由信息数据库RIB
  - ADJ-RIBs-IN 来自对等体的,未经处理的消息
  - Loc-RIB BGP路由器对ADJ-RIBs-IN中的路由使用它的本地路由策略而选择的路由
  - ADJ-RIBs-OUT 公布给对等体的路由

- BGP非等价负载均衡

  ```
router bgp 200
  bgp dmzlink-bw
  neighbor 10.1.12.1 dmzlink-bw
  neighbor 10.1.23.3 dmzlink-bw
  ```

### Weight
- 只能在in方向使用
- 几种配置方法
  -  `neighbor 1.1.1.1 weight 2000`
  - 方法2
    ```
    ip as-path access-list 1 permit _1$ //匹配AS(100,1)及(200,1)
    ip as-path access-list 1 permit _2$ //匹配AS(100,2)及(200,2)
    router b 300
      neighbor R3 filter-list 1 weight 4000
      neighbor R3 filter-list 2 weight 5000

    ```
  - 方法3

    ```
    !匹配路由77.1.1.0
    access-list 1 permit 77.1.1.0
    Route-map W
    match ip address 1
    set weight 10
    Route-map W permit 999
    exit
    !要大号放行

    router bg 100
    neighbor 22.1.1.1 route-map W in

    !或者方法2, 后面没有方向,针对R7发生所有路由都修改Weight值
    router bg 100
    neighbor 22.1.1.1 weight 10

    ```

## 正则表达式

- 组成
  - 常规字符
  - 控制字符
    - 原子字符(位于常规字符之前或之后,用来限制或扩充常规字符的控制字符或占位字符)
    - 乘法字符(跟在原子字符或常规字符之后,用来描述它前面字符的重复方式)
    - 范围字符(限定一个范围)
- 原子字符

| 符号 | 描述                                              |
| -    | -                                                 |
| .    | 匹配任何单个的字符包括空格                        |
| ^    | 一个字符串的开始
| \$   | 一个字符串的结束
| _    | 下划线,匹配任意的一个分割符如^,\$,空格,tab,逗号{}
| \|   | 管道符,逻辑或
| \    | 转义符,用来将紧跟其后的控制字符转变为普通字符
