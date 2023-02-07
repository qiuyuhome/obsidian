![[Pasted image 20221230151647.png]]

![[Pasted image 20221230145717.png]]

序号(sequence number)：seq序号，占32位，用来标识从TCP源端向目的端发送的字节流，发起方发送数据时对此进行标记。

确认号（acknowledgement number）：ack序号，占32位，只有ACK标志位为1时，确认序号字段才有效，ack=seq+1。

标志位（Flags）：共6个，即URG、ACK、PSH、RST、SYN、FIN等。具体含义如下：

URG：紧急指针（urgent pointer）有效。

ACK：确认序号有效。（为了与确认号ack区分开，我们用大写表示）

PSH：接收方应该尽快将这个报文交给应用层。

RST：重置连接。

SYN：发起一个新连接。

FIN：释放一个连接。