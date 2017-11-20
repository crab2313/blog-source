---
layout: post
title: "TCP/IP Sockets in Java:1 TCP/UDP Socket"
date: 2016-04-07T23:35:04+08:00
---

# 地址
为了建立两台位于互联网中主机的连接，需要IP地址。Java中用`InetAddress`表示一个IP地址。然而，只有IP地址还不够，操作系统支持运行大量的进程，两台主机间的通信实质上变为了两个进程见的通信。为此TCP与UDP协议中都有端口号这一概念，应用程序在进行网络通信的时候会向操作系统请求占用某个端口号。所以一个TCP连接可以由连接双方的地址及端口号这个四元组唯一标识。

然而，IP地址对于用户并不是很友好，通过DNS我们可以将一个主机名与多个IP地址联系起来。我们只需要记住主机名，就可以通过DNS解析到对应的IP地址。这样做的好处是主机名比IP地址更加用户友好，同时，IP地址的变化可以被DNS掩盖。

`InetAddress`用常见的构造方式：

``` java
static InetAddress[] getAllByName(String host);
static InetAddress getByName(String host);
static InetAddress getLocalHost();
```

其中`getAllByName`是得到所有`host`对应的IP地址，`getByName`是得到一个。而`getLocalHost`是得到本机的IP地址。

地址的字符串表示：

``` java
String toString();
String getHostAddress();
String getHostName();
String getCanonicalHostName();
```

``` java

import java.net.*;

public class InetAddressExample {
    public static void main(String[] args) {
        try {
            InetAddress address = InetAddress.getLocalHost();
            System.out.println("Local Host:");
            System.out.println("\t" + address.getHostName());
            System.out.println("\t" + address.getHostAddress());
        } catch (UnknownHostException e) {
            System.out.println("Unable to determine the host's address");
        }

        for (int i = 0; i < args.length; i++) {
            try {
                InetAddress[] addressList = InetAddress.getAllByName(args[i]);
                System.out.println(args[i] + ":");
                System.out.println("\t" + addressList[0].getHostName());
                for (int j = 0; j < addressList.length; j++) {
                    System.out.println("\t" + addressList[j].getHostAddress());
                }
            } catch (UnknownHostException e) {
                System.out.println("Unable to find address for " + args[i]);
            }
        }
    }
}

```

# TCP套接字(TCP Socket)

java提供两种TCP套接字，`Socket`与`ServerSocket`。一个`Socket`实例表示一个TCP连接的一端。而`ServerSocket`会监听特定端口，并在得到连接的时候建立`Socket`来处理连接。

一个典型的TCP客户端可以通过以下三步建立：

1.  以目标地址及端口为参数构建`Socket`对像实例
2.  从该`Socket`实例得到`InputStream`和`OutputStream`，进行通信
3.  使用`close`方法关闭连接

``` java
import java.net.*;
import java.io.*;

public class TCPEchoClient {
    public static void main(String[] args) throws Exception {
        if ((args.length < 2) || (args.length > 3))
            throw new IllegalArgumentException(
                    "Parameters: <Server> <Word> [<Port>]");

        String server = args[0];
        byte[] byteBuffer = args[1].getBytes();

        int servPort = (args.length == 3) ? Integer.parseInt(args[2]) : 7;

        Socket socket = new Socket(server, servPort);
        System.out.println("Connected to server... sending echo string");
        
        InputStream in = socket.getInputStream();
        OutputStream out = socket.getOutputStream();
        out.write(byteBuffer);

        int totalBytesRcvd = 0;
        int bytesRcvd;

        while (totalBytesRcvd < byteBuffer.length) {
            if ((bytesRcvd = in.read(byteBuffer, totalBytesRcvd, 
                            byteBuffer.length - totalBytesRcvd)) == -1)
                throw new SocketException("Connection closed prematurely");
            totalBytesRcvd += bytesRcvd;
            
            System.out.println("Received " + new String(byteBuffer));
            socket.close();
        }
    }
}
```


一个典型的TCP服务端可以这样建立：

1.  以监听端口号为参数构建`ServerSocket`对象
2.  使用`accept`方法得到TCP连接对应的`Socket`，处理连接后`close`
3.  重复第二步

``` java
import java.net.*;
import java.io.*;

public class TCPEchoServer {
    private static final int BUFSIZE = 32;

    public static void main(String[] args) throws IOException {
        if (args.length != 1) 
            throw new IllegalArgumentException("Paramter(s) <Port>");

        int servPort = Integer.parseInt(args[0]);

        ServerSocket servSock = new ServerSocket(servPort);

        int recvMsgSize;

        byte[] byteBuffer = new byte[BUFSIZE];

        while (true) {
            Socket s = servSock.accept();
            System.out.println("Handling client at " + 
                    s.getInetAddress().getHostAddress() + " on port " + 
                    s.getPort());

            InputStream in = s.getInputStream();
            OutputStream out = s.getOutputStream();

            while ((recvMsgSize = in.read(byteBuffer)) != -1)
                out.write(byteBuffer, 0, recvMsgSize);

            s.close();
        }
    }
}
```

# UDP套接字(UDP Socket)
java中可以使用`DatagramPacket`和`DatagramSocket`进行UDP通信。

典型的UDP通信：

1.  构建`DatagramSocket`对象，由于UDP的特性，我们可以省略本地地址和端口
2.  我们可以使用`send`和`receive`方法收发`DatagramPacket`对象
3.  通信结束时使用`close`关闭连接

``` java
import java.net.*;
import java.io.*;

public class UDPEchoClientTimeout {
    private static final int TIMEOUT = 3000;
    private static final int MAXTRIES = 5;

    public static void main(String[] args) throws IOException {
        if ((args.length < 2) || (args.length > 3))
            throw new IllegalArgumentException(
                    "Parameters: <Server> <Word> [<Port>]");

        InetAddress serverAddress = InetAddress.getByName(args[0]);

        byte[] bytesToSend = args[1].getBytes();

        int servPort = (args.length == 3) ? Integer.parseInt(args[2]) : 7;

        DatagramSocket socket = new DatagramSocket();

        socket.setSoTimeout(TIMEOUT);

        DatagramPacket sendPacket = new DatagramPacket(bytesToSend, 
                bytesToSend.length, serverAddress, servPort);

        DatagramPacket receivePacket = 
            new DatagramPacket(new byte[bytesToSend.length],bytesToSend.length);

        int tries = 0;
        boolean receivedResponse = false;
        do {
            socket.send(sendPacket);
            try {
                socket.receive(receivePacket);

                if (!receivePacket.getAddress().equals(serverAddress))
                    throw new IOException(
                            "Received packet from unknown source");
                receivedResponse = true;
            } catch (InterruptedIOException e) {
                tries += 1;
                System.out.println("Timed out, " + (MAXTRIES - tries) + 
                        " more tires...");
            }
        } while ((!receivedResponse) && (tries < MAXTRIES));

        if (receivedResponse) 
            System.out.println("Received: " + 
                    new String(receivePacket.getData()));
        else
            System.out.println("No response -- giving up.");

        socket.close();
    }
}


```


``` java
import java.net.*;
import java.io.*;

public class UDPEchoServer {
    private static final int ECHOMAX = 255;

    public static void main(String[] args) throws IOException {
        if (args.length != 1) 
            throw new IllegalArgumentException("Parameters: <Port>");

        int servPort = Integer.parseInt(args[0]);

        DatagramSocket socket = new DatagramSocket(servPort);
        DatagramPacket packet = new DatagramPacket(new byte[ECHOMAX], ECHOMAX);

        while (true) {
            socket.receive(packet);
            System.out.println("Handling client at " +
                    packet.getAddress().getHostAddress() + 
                    " on port " + packet.getPort());
            socket.send(packet);
            packet.setLength(ECHOMAX);
        }
    }
}

```
