#      发现一个魔改版的kcptun，实测确实要快一些 #866



### **[PHCSJC]()** commented [on 5 Mar](#issue-1160342588) • edited 

原作者在这：https://github.com/yzslab/kcptun curl测试，电信宽带下要快10%左右，联通宽带要快25%左右 youtube在15%左右组合是ss+kcptun+udp2raw 参数是： 

./server_linux_amd64 -l 127.0.0.1:222-t 127.0.0.1:333 --mtu 1200 --sndwnd 2200 --smuxver 1 --nocomp --crypt none --key aaa -mode manual -nodelay 1 -interval 11 -resend 2 -nc 1 -datashard 20 -parityshard 10 --iocopybuf 65535 --irtobackoff 1 --irtobthresh 256 --noearlyretran


./client_linux_amd64 -r 127.0.0.1:222-l 127.0.0.1:333--mtu 1200 --rcvwnd 2200 --smuxver 1 --nocomp --crypt none --key aaa -mode manual -nodelay 1 -interval 11 -resend 2 -nc 1 -datashard 20 -parityshard 10 --iocopybuf 8192 --irtobackoff 1 --irtobthresh 256 --noearlyretran

供大佬参考，也许还有优化的空间







### **[PHCSJC]()** commented [on 6 Mar](#issuecomment-1059939621)

总结一下就是：在丢包率低的环境下实际速度可以接近理论带宽，在丢包率高的环境下速度比原版快10%几这样





### **[keeno1982]()** commented [on 15 Mar](#issuecomment-1067738211)

我也来试试，udp2raw的参数方便提供一下吗







### **[PHCSJC]()** commented [on 15 Mar](#issuecomment-1067740418)

我也来试试，udp2raw的参数方便提供一下吗  
--cipher-mode xor --auth-mode simple --seq-mode 3 --fix-gro





### **[keeno1982]()** commented [on 15 Mar](#issuecomment-1067744088)

谢罗，我这边遇到个问题，如果只使用kcptun，速度可以到5M，但是套了udp2raw，速度最高只能到2M多。你那边有没有这个问题。





### **[yzslab]()** commented [on 9 Apr](#issuecomment-1093845546)

难怪最近多了几个STAR，原来是有人发到这来了





### **[keeno1982]()** commented [on 12 Apr](#issuecomment-1096109281)

[@yzslab](https://github.com/yzslab) 用了你修改的kcptun确实速度有所提升，你那边没有开放issue，我只好到这里问了。计算窗口公式为BDP / MTU，如果我服务端的带宽上下行为1G，客户端带宽下行是100m，上行是20m。按公式算下来，服务端的窗口是应该设为1000/8*1024*1024*0.18/1350=17476，还是按照客户端的下行？100/8*1024*1024*0.18/1350=1747。这个窗口数值如果取整是用2000还是用1500？





### **[keeno1982]()** commented [on 12 Apr](#issuecomment-1096117347)

[@yzslab](https://github.com/yzslab) 如果ping值是180ms，服务器带宽1000M，本地带宽100M，上行20M的，5%的丢包率，不考虑省流量，最大能跑满本地带宽的配置应该是怎样的呢





### **[yzslab]()** commented [on 12 Apr](#issuecomment-1096127811)

[@keeno1982](https://github.com/keeno1982)两边的窗口设置可以不一样，服务端按服务器的1G带宽设置窗口，客户端按客户端100M下行设置接收窗口，20M上行设置发送窗口。 服务端窗口比客户端大没事（当然不超过服务器的上行带宽），因为服务端会根据客户端告知的接收窗口发包。 算出来的值只是理论值，一般来说可能需要往大设，因为发包多了，路上排队的数据包会增加，KCP要处理的数据包也越多，从而导致延迟增加。不过如果引起拥塞了，导致丢包率上升，那就应该往小设。具体怎样设置，需要你自己调整，反复测量，算出来的值只是参考用。





### **[keeno1982]()** commented [on 12 Apr](#issuecomment-1096724727)

[@yzslab](https://github.com/yzslab) 3Q，看来畅通时和拥堵时都需要准备不同的配置，大概明白了



