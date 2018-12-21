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
集成的话参考lab3就可以了。可以把lab3的全部文件复制过来，然后在main.c中添加上udp客户端和服务器端两个函数即可。这里要注意Linux下fork子进程的判断条件，再fork一次产生子进程对应的pid并不是一直不变的，在调用了别的函数操作之后，即使没有修改pid的代码，pid也已经由0变为了1。还有，在使用bind方法帮点端口时不要急于绑定IP，要在调用时传参才不会出问题。（玄学）

 复制命令： git clone https://github.com/hanzzj/linuxudp.git
 cd linuxudp -> ls -> gcc client.c -o cilent ->gcc -server.c -o server -> ./server ->水平分割终端 ->./client
 集成到menuos之后，就可以使用 udpclient和udpserver来调用这两个模块。
 
 
 关于UDP sendto发送数据过程：
 据网上资料得知，linux下UDP发送数据包并不是有数据就发送，而是使用了cork机制，即UDP相关的数据经常会存储在一个名为cork的变量中，等到了数据量足够时再发送，这样可以减轻工作量。
 UDP发送数据包的过程首先是在udp_sendmsg调用udp_send_skb,ip_append_data方法，查询可知该函数是IP层提供的UDP和RAW Socket的发包函数，这个函数在TCP中也用于发送ACK和RST报文，ip_send_reply最终也会调用此函数。
 将数据拷贝到适合的skb( Struct sk_buffer，数据缓冲区)中，可能有两种情况: 放入skb的线性数据存储区(skb->data)中或者放入skb_shared_info的分片(frag)中。
 具体思路流程如下：
 首先在UDP.C文件1070行中调用ip_append_data方法来传输数据，使用了err来判断处理错误。
 err = ip_append_data(sk, fl4, getfrag, msg->msg_iov, ulen,
		     sizeof(struct udphdr), &ipc, &rt,
		     corkreq ? msg->msg_flags|MSG_MORE : msg->msg_flags);
  
 而ip_append_data方法的定义是在ip_output.c文件中定义的，头部分具体定义内容如下：
 int ip_append_data(struct sock *sk, struct flowi4 *fl4,
		   int getfrag(void *from, char *to, int offset, int len,int odd, struct sk_buff *skb),
		   void *from, int length, int transhdrlen,
		   struct ipcm_cookie *ipc, struct rtable **rtp,
	    unsigned int flags)
     
 函数的参数含义：*sk 是指向之前创建的socket，包括IP,端口号等信息，     
              flowi4是匹配源，当作key，去路由缓存或者路由表查具体路由信息，结果放在fibresult 。
              下一个是函数getfrag的返回值来做参数。这个函数指针的具体操作是将数据复制到缓冲区，即传递给*skb对应内存区域。
               *from :指向来源数据 
              int length : 数据长度
              int transhdrlen : 传输层首部长度，同时也是标志是否为第一个fragment的标志
              struct ipcm_cookie *ipc 和struct rtable **rpt 主要是对路由信息的操作，无关紧要。 
              unsigned int flags : 处理标志，主要是MSG_PROBE和MSG_MORE，对MTU进行探测，确定传输过程要不要分组，后续是否还有数据发送。
              
函数内部具体流程如下：
if (flags&MSG_PROBE）return 0; //首先判断是否进行了MTU探测，若没有，就直接返回，因为不能确定传输数据包大小
if (skb_queue_empty(&sk->sk_write_queue)) {  //判断输出队列是否为空
	err = ip_setup_cork(sk, &inet->cork.base, ipc, rtp);//如果传输控制块(sock)的的输出队列为空，则需要设置一些临时信息，如果不为空，那就可以使用上次发送时的相关信息，也就是用来填充的字节
if (err)
		return err;
	} else {
		transhdrlen = 0;
	}
return __ip_append_data(sk, fl4, &sk->sk_write_queue, &inet->cork.base,
				sk_page_frag(sk), getfrag,
				from, length, transhdrlen, flags);
    
 等到准备工作完成以后，再调用__ip_append_data进行处理。处理过程大致如下：
 mtu = cork->fragsize; /*获取链路层首部的长度*/.     hh_len = LL_RESERVED_SPACE(rt->dst.dev);  /*获取IP首部(包括IP选项)的长度*/     fragheaderlen = sizeof(struct iphdr) + (opt ? opt->optlen : 0);    /*IP数据包中数据的最大长度，通过mtu计算，并进行8字节对齐，目的是提升计算效率*/     maxfraglen = ((mtu - fragheaderlen) & ~7) + fragheaderlen;    /*输出的报文长度不能超过IP数据报能容纳的最大长度(64K)*/
 这里调用了系统信息来确定发包时构造的信息。
 保存缓冲区的步骤：skb = skb_peek_tail(queue); 
 这里skb有两种情况，如果队列为空， 则skb = NULL，否则就是尾部skb的指针。后面会在进行处理。
 对于UDP报文，若数据长度大于MTU，并且需要进行分片，则需要通过分片机制进行处理。接下来使用了一个分支判断，代码如下：
 if (((length > mtu) || (skb && skb_is_gso(skb))) &&	    (sk->sk_protocol == IPPROTO_UDP) &&	    (rt->dst.dev->features & NETIF_F_UFO) && !rt->dst.header_len)
 {      err = ip_ufo_append_data(sk, queue, getfrag, from, length,				
        hh_len, fragheaderlen, transhdrlen,					 maxfraglen, flags);		
	if (err)			
	  goto error;		
	return 0;	}
分支的判断条件主要有：数据长度是否大于mtu?skb是否为空？skb_is_gso用来获取当前系统分片长度和分段长度。协议是否为UDP？目的dst信息是否完整，等等，若有不成功的，则返回错误 goto err。
再往下走，就是if (!skb)	goto alloc_new_skb;的判断分支，它主要是判断skb缓冲区是否存在，不存在的调用分配函数进行分配。之后就可以根据MTU（最大传输单元）和skb缓冲区中数据长度skb->len。比较情况进行处理。这里使用了一个循环来对数据进行处理：
while (length > 0) {         /* Check if the remaining data fits into current packet. */       
计算当前skb中还能放多少数据，通过mtu-skb的数据长度计算         
copy = mtu - skb->len;84.        .         * 如果skb的剩余空间不足以存放完这次需要放入的数据长度length，则将当前skb填满即可，剩余数据留下一个skb发送86.          * 其中maxfraglen是8字节对齐后的mtu         
if (copy < length)             copy = maxfraglen - skb->len        /*如果当前skb中的数据本身就已经大于MTU了，那就需要新分配skb来容纳新数据了          skb_prev = skb;100.                     
由于原有的skb数据区空间不足，而需要分配新skb，计算不足的大小，这次需要新分配的数据区大小length加上原来skb中不足的大小，为新skb需要分配的数据区大小                     datalen = length + fraggap;                  /*如果新skb需要分配的数据区大小超过了mtu，那这次还是只能分配mtu的大小，剩余数据需要通过循环分配新skb来容纳*/114.                     if (datalen > mtu - fragheaderlen)115.                         datalen = maxfraglen - fragheaderlen;                    /*数据报分片大小需要加上IP首部的长度*/117.                     fraglen = datalen + fragheaderlen;118.                     /*如果设置了MSG_MORE标记，表明需要等待新数据，一直到超过mtu为止，则设置"分配空间大小"为mtu*/                     if ((flags & MSG_MORE) &&120.                      !(rt->dst.dev->features&NETIF_F_SG))121.                         alloclen = mtu;122.                     /*否则需要分配的空间大小为数据报分片大小*/123.                     else124.                         alloclen = fraglen;125.                     /*再加上IPsec相关的头部长度*/126.                     alloclen += exthdrlen;127.          128.                     /* The last fragment gets additional space at tail.129.                      * Note, with MSG_MORE we overallocate on fragments,130.                      * because we have no idea what fragment will be131.                      * the last.132.                      */133.                     if (datalen == length + fraggap)134.                         alloclen += rt->dst.trailer_len;135.                     /*136.                      * 根据是否存在传输层首部，确定分配skb的方法:137.                      * 如果存在，则说明该分片为分片组中的第一个分片，那就需要考虑更多的情况，比如:发送是否超时、是否发生未处理的致命错误、138.                      * 发送通道是否已经关闭等；当不存在传输层首部时，说明不是第一个分片，则不需考虑这些情况。139.                      */140.                     if (transhdrlen) {141.                         /*分配skb，并进行相关处理*/142.                         skb = sock_alloc_send_skb(sk,143.                                 alloclen + hh_len + 15,144.                                 (flags & MSG_DONTWAIT), &err);145.                     } else {146.                         skb = NULL;147.                         if (atomic_read(&sk->sk_wmem_alloc) <=148.                          2 * sk->sk_sndbuf)149.                          /*分配skb*/150.                             skb = sock_wmalloc(sk,151.                                      alloclen + hh_len + 15, 1,152.                                      sk->sk_allocation);153.                         if (unlikely(skb == NULL))154.                             err = -ENOBUFS;155.                         else156.                             /* only the initial fragment is157.                              time stamped */158.                             cork->tx_flags = 0;159.                     }160.                     if (skb == NULL)161.                         goto error;162.          163.                     /*164.                      *    Fill in the control structures165.                      */166.                     /*初始化skb中的相关成员*/167.                     skb->ip_summed = csummode;168.                     skb->csum = 0;169.                     skb_reserve(skb, hh_len);170.                     skb_shinfo(skb)->tx_flags = cork->tx_flags;171.          172.                     /*173.                      *    Find where to start putting bytes.174.                      */175.                     /*在skb中预留存放二层首部、三层首部和数据的空间*/176.                     data = skb_put(skb, fraglen + exthdrlen);177.                     /*设置IP头指针位置*/178.                     skb_set_network_header(skb, exthdrlen);179.                     /*计算传输层头部长度*/180.                     skb->transport_header = (skb->network_header +181.                                  fragheaderlen);182.                     /*计算数据存入的位置*/183.                     data += fragheaderlen + exthdrlen;184.                     /*185.                      * 如果上一个skb的数据大于mtu(8字节对齐)，那么需要将上一个skb中超出的数据和传输层首部186.                      * 复制到当前的skb中，并重新计算校验和。187.                      */188.                     if (fraggap) {189.                         skb->csum = skb_copy_and_csum_bits(190.                             skb_prev, maxfraglen,191.                             data + transhdrlen, fraggap, 0);192.                         /*上一个skb的校验和也需要重新计算*/193.                         skb_prev->csum = csum_sub(skb_prev->csum,194.                                      skb->csum);195.                         /*拷贝新数据后，再次移动data指针，更新数据写入的位置*/196.                         data += fraggap;197.                         /*已8字节对齐的MTU大小截取上一个skb，多余的数据已经拷贝到新的skb中了，需要截掉*/198.                         pskb_trim_unique(skb_prev, maxfraglen);199.                     }200.                     /*拷贝新数据到数据区*/201.                     copy = datalen - transhdrlen - fraggap;202.                     if (copy > 0 && getfrag(from, data + transhdrlen, offset, copy, fraggap, skb) < 0) {203.                         err = -EFAULT;204.                         kfree_skb(skb);205.                         goto error;206.                     }207.                     /*更新下次拷贝相关的数据*/208.                     offset += copy;209.                     length -= datalen - fraggap;210.                     /*由于传输层首部已经拷贝过了，所以相关变量置0*/211.                     transhdrlen = 0;212.                     exthdrlen = 0;213.                     csummode = CHECKSUM_NONE;214.          215.                     /*216.                      * Put the packet on the pending queue.217.                      */218.                     /*复制完数据的skb添加到sock输出队列的末尾*/ 219.                     __skb_queue_tail(queue, skb);220.                     continue;221.                 }222.                 /*如果当前skb中的剩余数据区大小(相对于mtu)，那说明这次要发送的数据可以直接放入当前的skb中，直接拷贝即可。*/223.                 if (copy > length)224.                     copy = length;225.                 /*如果硬件不支持SG(分散聚集特性，使用skb中的非线性区(shared_info))*/226.                 if (!(rt->dst.dev->features&NETIF_F_SG)) {227.                     unsigned int off;228.          229.                     off = skb->len;230.                     /*将数据拷贝到skb中的线性区*/        231.                     if (getfrag(from, skb_put(skb, copy),232.                             offset, copy, off, skb) < 0) {233.                         __skb_trim(skb, off);234.                         err = -EFAULT;235.                         goto error;236.                     }237.                 } else {/*如果硬件支持SG，则将数据拷贝到skb的非线性区(shared_info)中*/238.                     int i = skb_shinfo(skb)->nr_frags;239.          240.                     err = -ENOMEM;241.                     /*在分片列表(frags)中使用原有分片(返回相应分片的指针)或分配新页来存放数据*/242.                     if (!sk_page_frag_refill(sk, pfrag))243.                         goto error;244.                     /*245.                      * 如果传输控制块(sock)中的缓存页pfrag，不是当前skb->shared_info中的最后一个分片(分散聚集IO页面)所在的页面，则直接使用该页面，246.                      * 将其添加 到分片列表(分散聚集IO页面数组)中，否则说明传输控制块(sock)中的缓存页pfrag就是分散聚集IO页面的最后一个页面，247.                      * 则直接向其中拷贝数据即可。248.                      */249.                     if (!skb_can_coalesce(skb, i, pfrag->page,250.                              pfrag->offset)) {251.                         err = -EMSGSIZE;252.                         if (i == MAX_SKB_FRAGS)253.                             goto error;254.          255.                         __skb_fill_page_desc(skb, i, pfrag->page,256.                                  pfrag->offset, 0);257.                         skb_shinfo(skb)->nr_frags = ++i;258.                         get_page(pfrag->page);259.                     }260.                     copy = min_t(int, copy, pfrag->size - pfrag->offset);261.                     /*拷贝数据至skb中非线性区分片(分散聚集IO页面)中*/262.                     if (getfrag(from,263.                          page_address(pfrag->page) + pfrag->offset,264.                          offset, copy, skb->len, skb) < 0)265.                         goto error_efault;266.                     /*移动相应数据指针*/267.                     pfrag->offset += copy;268.                     /*增加分片大小*/269.                     skb_frag_size_add(&skb_shinfo(skb)->frags[i - 1], copy);270.                     /*增加skb数据相关大小*/271.                     skb->len += copy;272.                     skb->data_len += copy;273.                     skb->truesize += copy;274.                     /*增加sock发送缓存区已分配数据大小*/275.                     atomic_add(copy, &sk->sk_wmem_alloc);276.                 }277.                 offset += copy;278.                 /*length减去已经拷贝的大小，如果拷完了，则结束循环，否则继续拷贝*/279.                 length -= copy;280.             }281.          282.             return 0;283.          284.         error_efault:285.             err = -EFAULT;286.         error:287.             cork->length -= length;288.             IP_INC_STATS(sock_net(sk), IPSTATS_MIB_OUTDISCARDS);289.             return err;290.         }






              
              
              
