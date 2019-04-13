# TCP 粘包

数据传输时，我们经常会使用 TCP 协议传输数据。当我们使用 TCP 长连接传输不同类型的数据时，会产生粘包、拆包的问题。

![img](https://img-blog.csdn.net/20160122215438685?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

## 粘包

粘包可能在服务端产生也可能在客户端产生，提交数据给 TCP 发送时，TCP 并不立刻发送此段数据，而是等待一小段时间，看看在等待期间是否还有要发送的数据，若有则会一次把这两段数据发送出去，造成粘包；另一端在接收到数据后，放到接收缓冲区中，如果消息没有被及时从缓冲区取走，下次取数据的时候可能就会出现一次取出多个数据包的情况，造成粘包现象。

## 拆包

使用 TCP 传送数据时，有可能数据过大，使得发送方缓冲区无法一次发送，造成另一端只收到的数据不完整。还有一种情况是进行 MSS (最大报文长度) 大小的 TCP 分段，当 TCP 报文长度 - TCP 头部长度 > MSS 的时候将发生拆包。

## 如何处理粘包、拆包

通常会有以下一些常用的方法：

1. 使用带消息头的协议、消息头存储消息开始标识及消息长度信息，服务端获取消息头的时候解析出消息长度，然后向后读取该长度的内容。
2. 设置定长消息，服务端每次读取既定长度的内容作为一条完整消息，当消息不够长时，空位补上固定字符。
3. 设置消息边界，服务端从网络流中按消息边界分离出消息内容，一般使用‘\n ’ (但是如果在正文中出现了'\n'，则会出现错误的消息分离)。
4. 更为复杂的协议。

## TCP 粘包拆包的代码实践

下面演示了使用规定消息头，消息体的方式来解决 TCP 的粘包，拆包问题。

client 代码：客户端代码主要逻辑是组装要发送的消息，然后发送到服务端。

```java
import java.io.*;
import java.net.InetSocketAddress;
import java.net.Socket;

/**
 * @author wuhf
 * @Date 2018/7/16 15:45
 **/
public class TestSocketClient {
    public static void main(String args[]) throws IOException {
        Socket clientSocket = new Socket();
        clientSocket.connect(new InetSocketAddress(8089));
        new SendThread(clientSocket).start();

    }

    static class SendThread extends Thread {
        Socket socket;
        PrintWriter printWriter = null;

        public SendThread(Socket socket) {
            this.socket = socket;
            try {
                printWriter = new PrintWriter(new OutputStreamWriter(socket.getOutputStream()));
            } catch (IOException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
        }

        @Override
        public void run() {
            String reqMessage = "HelloWorld ！ from clientsocket this is test half packages!";
            for (int i = 0; i < 100; i++) {
                sendPacket(reqMessage);
            }
            if (socket != null) {
                try {
                    socket.close();
                } catch (IOException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }
            }

        }

        public void sendPacket(String message) {
            try {
                OutputStream writer = socket.getOutputStream();
                writer.write(message.getBytes());
                writer.flush();
            } catch (IOException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
        }
    }
}
```

server 端代码：server 端代码的主要逻辑是接收客户端发送过来的消息，重新组装出消息，并打印出来。

```java
import java.io.*;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.net.Socket;

/**
 * @author wuhf
 * @Date 2018/7/16 15:50
 **/
public class TestSocketServer {
    public static void main(String args[]) {
        ServerSocket serverSocket;
        try {
            serverSocket = new ServerSocket();
            serverSocket.bind(new InetSocketAddress(8089));
            while (true) {
                Socket socket = serverSocket.accept();
                new ReceiveThread(socket).start();

            }
        } catch (IOException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
    }

    static class ReceiveThread extends Thread {
        public static final int PACKET_HEAD_LENGTH = 2;//包头长度
        private Socket socket;
        private volatile byte[] bytes = new byte[0];

        public ReceiveThread(Socket socket) {
            this.socket = socket;
        }

        public byte[] mergebyte(byte[] a, byte[] b, int begin, int end) {
            byte[] add = new byte[a.length + end - begin];
            int i = 0;
            for (i = 0; i < a.length; i++) {
                add[i] = a[i];
            }
            for (int k = begin; k < end; k++, i++) {
                add[i] = b[k];
            }
            return add;
        }

        @Override
        public void run() {
            int count = 0;
            while (true) {
                try {
                    InputStream reader = socket.getInputStream();
                    if (bytes.length < PACKET_HEAD_LENGTH) {
                        byte[] head = new byte[PACKET_HEAD_LENGTH - bytes.length];
                        int couter = reader.read(head);
                        if (couter < 0) {
                            continue;
                        }
                        bytes = mergebyte(bytes, head, 0, couter);
                        if (couter < PACKET_HEAD_LENGTH) {
                            continue;
                        }
                    }
                    // 下面这个值请注意，一定要取 2 长度的字节子数组作为报文长度，你懂得
                    byte[] temp = new byte[0];
                    temp = mergebyte(temp, bytes, 0, PACKET_HEAD_LENGTH);
                    String templength = new String(temp);
                    int bodylength = Integer.parseInt(templength);//包体长度
                    if (bytes.length - PACKET_HEAD_LENGTH < bodylength) {//不够一个包
                        byte[] body = new byte[bodylength + PACKET_HEAD_LENGTH - bytes.length];//剩下应该读的字节(凑一个包)
                        int couter = reader.read(body);
                        if (couter < 0) {
                            continue;
                        }
                        bytes = mergebyte(bytes, body, 0, couter);
                        if (couter < body.length) {
                            continue;
                        }
                    }
                    byte[] body = new byte[0];
                    body = mergebyte(body, bytes, PACKET_HEAD_LENGTH, bytes.length);
                    count++;
                    System.out.println("server receive body:  " + count + new String(body));
                    bytes = new byte[0];
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
    }

}
```

## 参考资料

[tcp粘包和拆包、断包](https://blog.csdn.net/chen199199/article/details/50564015)

[TCP 粘包问题浅析及其解决方案](https://www.v2ex.com/t/478610?p=2)