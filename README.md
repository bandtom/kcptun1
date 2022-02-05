## 关于本分支
详见：[KCP-GO的重传机制以及带宽利用率的提升](https://zhensheng.im/2021/03/10/kcp-go%e7%9a%84%e9%87%8d%e4%bc%a0%e6%9c%ba%e5%88%b6%e4%bb%a5%e5%8f%8a%e5%b8%a6%e5%ae%bd%e5%88%a9%e7%94%a8%e7%8e%87%e7%9a%84%e6%8f%90%e5%8d%87.meow)

所使用的KCP-GO：[https://github.com/yzslab/kcp-go](https://github.com/yzslab/kcp-go)

原README：[点击此处查阅](README.original.md)

## 为什么要使用KCP？
高RTT的网络，TCP的三次握手、慢启动是非常影响体验的机制。

150ms RTT，使用BBR拥塞控制且网络0丢包的情况下，慢启动到100Mbps至少需要2s，500Mbps至少5s，到1Gbps至少10s。

如果没有BBR，情况则更为糟糕：至少8s才能到100Mbps。

而KCP的多路复用、非退让流控正好消除了TCP的这两个缺点。

## 一些原KCPTUN没有的参数解释
+ --iocopybuf

  KCP流与TCP流之间互拷数据的缓冲区大小，默认4096，带宽较大时提高此数值可提高传输速度，特别是发送端
+ --irtobackoff

  超时重传的初始退让，即在“RTO + (RTO >> irtobackoff)”才开始第一次超时重传，启用退让可显著提高带宽利用率，建议置1或2
+ --irtobthresh

  少于多少个数据包时，不应用初始退让，即一个RTO后就立刻重传
+ --noearlyretran

  关闭Early Retransmit，可提高带宽利用率，减少数据包重发

## 使用
### 1. 计算参数
+ 计算BDP
  ```
  BDP = 带宽 * RTT
  ```
  RTT可通过ping测量。

  例如500 Mbps带宽，150ms RTT，那么：
  ```
  BDP = 500 / 8 * 1024 * 1024 * 0.15
  ```
+ 计算窗口大小

  KCP的窗口以数据包数量表示：
  ```
  窗口大小 = BDP / MTU
  ```
  如果上下下不对等，需要分别单独计算
### 2. 调整系统参数
+ 调大发送端qdisc的flow limit（详见：[UDP SndbufErrors & ENOBUFS](https://zhensheng.im/2021/03/12/udp-sndbuferrors-enobufs.meow)）

  一般在服务端系统调整，对客户端来说，不一定要调整，例如客户端以下行为主，上行带宽使用较低，可跳过此步
  ```
  # interface="eth0" # 改成与KCPTUN通讯所通过的网卡名
  # tc qdisc del dev ${interface} root
  # tc qdisc add dev ${interface} root fq limit 20000 flow_limit 256
  ```
+ 把接收端的rmem_max调整为BDP的两倍

  一般在客户端系统调整，对服务端来说，不一定要调整，如果服务端以上行为主，下行带宽使用较低，可跳过此步
  
  编辑/etc/sysctl.conf，在其中加入：
  ```
  net.core.rmem_max=33554432 # 32MB
  ```
  然后运行：
  ```
  # sysctl -p
  ```
  注意不要调整net.core.wmem_max，大带宽下会出现反效果，保持默认值212992即可
### 3. 运行服务端
请自行按需调整参数（特别是wnd，buf、key）：
  ```
  # server_linux_amd64 \
      --target 127.0.0.1:80 \
      --listen 0.0.0.0:80 \
      --mode manual \
      --nodelay 1 \
      --interval 10 \
      --resend 2 \
      --nc 1 \
      --mtu 1350 \
      --nocomp \
      --crypt salsa20 \
      --datashard 0 \
      --parityshard 0 \
      --sndwnd 发送窗口 \
      --rcvwnd 接收窗口 \
      --sockbuf 至少一个BDP \
      --smuxbuf 至少一个BDP \
      --key 密钥 \
      --iocopybuf 65535 \
      --irtobackoff 1 \
      --irtobthresh 256 \
      --noearlyretran
  ```
  ### 4. 运行客户端
请自行按需调整参数（特别是wnd，buf、key）：
```
# client_linux_amd64 \
    --remoteaddr server:80 \
    --localaddr :8088 \
    --mode manual \
    --nodelay 1 \
    --interval 10 \
    --resend 2 \
    --nc 1 \
    --mtu 1350 \
    --nocomp \
    --crypt salsa20 \
    --datashard 0 \
    --parityshard 0 \
    --sndwnd 发送窗口 \
    --rcvwnd 接收窗口 \
    --sockbuf 至少一个BDP \
    --smuxbuf 至少一个BDP \
    --key 密钥 \
    --iocopybuf 8192 \
    --irtobackoff 1 \
    --irtobthresh 256 \
    --noearlyretran
```
### 5. 根据实际使用调优参数
+ 根据丢包情况，决定是否启用FEC
  
  上面给的服务端与客户端的两个示例参数是没有启用FEC的，如果丢包率较高，请调整--datashard与--parityshard启用FEC，至于多少合适，根据丢包率、以及反复调整测试确定
+ 调整发送/接收窗口

  为什么还需要调整？一开始不是计算好了吗？
  
  跑的带宽越大，RTT会越高，所以实际的BDP，会比一开始计算的理论值要大，这个具体值，需自行反复调整测试去确定，记得同时更新rmem_max的值。
  
  注意窗口不要调得太大，如果超出接收端的带宽，会有反效果。
