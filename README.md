tcp和udp作为两种不同通信方式，各有优点各有缺点。
UDP具有如下特点：
1.无连接：知道对短的IP和端口号就可以直接传输，不需要建立连接;
2.不可靠：没有确认机制，没有重传机制;如果因为网络故障该段无法发送给对方，UDP协议层也不会给应用层返回任何错误信息;
3.面向数据报：不能够灵活的控制读写数据的次数和数量
UDP的缓冲区：UDP没有真正意义上的发送缓冲区，调用sendto会直接交给内核，由内核将数据传给网络层协议进行后续的传输动作;
UDP具有接收缓冲区，但是这个接收缓冲区不能保证收到的UDP报文的顺序和发送UDP报文的顺序是一致的，如果缓冲区满了，再次到达的UDP数据就会被丢弃
UDP的socket既能读，也能写，使用全双工的机制进行通信。

本次实验中编写了两个C文件，client.c和server.c,对应信息交流中的客户端和服务器端。
首先是client.c，这里使用了socket（）来创建一个接口实例，实例化过程中使用了套接字参数 SOCK_DGRAM,套接字类型，常用的有以下两种SOCK_STREAM对应流式套接字;SOCK_DGRAM是指数据报套接字。然后流式套接字默认TCP协议，数据包套接字默认UDP协议。
之后就可以调用sendto()来进行发送信息，并使用recvfrom()函数接收来自server的信息。
在传输完成之后就可以调用close()方法来关闭接口。
同理，对于server也是这样处理，但是多了一个调用bind()方法来监听端口的过程，之后也是进行发送接收。
集成的话参考lab3就可以了。可以把lab3的全部文件复制过来，然后在main.c中添加上udp客户端和服务器端两个函数即可。

 复制命令： git clone https://github.com/hanzzj/linuxudp.git
 cd linuxudp -> ls -> gcc client.c -o cilent ->gcc -server.c -o server -> ./server ->水平分割终端 ->./client
 集成到menuos之后，就可以使用 udpclient和udpserver来调用这两个模块。

