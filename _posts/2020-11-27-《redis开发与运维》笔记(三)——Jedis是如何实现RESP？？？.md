---
layout: post
title: "《redis开发与运维》笔记(三)——Jedis是如何实现RESP？？？"
date: 2020-11-27 
description: "Jedis是如何实现RESP？？？"
tag: Redis
--- 

#### 简化版RESP
针对Jedis源码[简化版RESP](https://gitee.com/jaychen_cn/jaychen_redis_client)

#### 什么RESP？

至于什么是RESP请参考上一篇文章[《redis开发与运维-笔记(二)-(RESP)客户端通信协议》](http://101.200.221.53:4000/2020/11/redis%E5%BC%80%E5%8F%91%E4%B8%8E%E8%BF%90%E7%BB%B4-%E7%AC%94%E8%AE%B0(%E4%BA%8C)-(RESP)%E5%AE%A2%E6%88%B7%E7%AB%AF%E9%80%9A%E4%BF%A1%E5%8D%8F%E8%AE%AE/)
	
#### Jedis实现RESP源码分析

在源码分析前，我们先看看如何对Jedis简单的操作

```
  public static void main(String[] args) {
  
        final Jedis jedis = new Jedis("localhost", 6379);<1>
		
        jedis.set("hello", "world");<2>
		
        jedis.get("hello");<3>
    }
```
上面这代码比较简单，分为三个步骤

<1>：连接redis

<2>：添加key

<3>：获取key

<br >

#### Jedis类结构

![image][imageOne]

Jedis继承了BinaryJedis，并实现了一系列的Commands接口。Commands主要是支持对Redis的命令操作。

看完类结构后，从入口来看看Jedis实列化做了什么
```
final Jedis jedis = new Jedis("localhost", 6379);
```

```
Jedis类

 public Jedis(final String host, final int port) {
    super(host, port);
  }
```
我们继续看父类的构建器

```
BinaryJedis

public BinaryJedis(final String host, final int port) {
    client = new Client(host, port);
  }

```

**重点！！！** **重点！！！** **重点！！！**

#### client类结构

<br >

![image2]

<br >

我们发现父类**BinaryJedis**构造器创建了一个**client**实列。并将**host，port**参数传递进去。

进入到Client类中，发现client类继承了BinaryClient，并实现了Commands接口。我们仔细看Commands接口发现Commands接口提供的全是对redis命令操作。**BinaryClient**继承了**Connection**。而Connection主要是跟Redis进行通讯。此时发现client才是真正对redis进行交互的**客户端**。


```
重点分析一下Connection，Connection主要是跟Redis进行通讯。

  // 主机地址 
  private String host = Protocol.DEFAULT_HOST;
  
  // 端口号
  private int port = Protocol.DEFAULT_PORT;
  
  // socket jedis与redis是通过socket通信
  private Socket socket;
  
  // 输出流
  private RedisOutputStream outputStream;
  
  // 输入流
  private RedisInputStream inputStream;
  
  // 默认超时时间
  private int connectionTimeout = Protocol.DEFAULT_TIMEOUT;
  
  接下来看看一个比较重要的方法
  
  public void connect() {
    // 检测是否已经连接redis服务器，如果未连接则创建一个新的socket连接
    if (!isConnected()) {
      try {
        socket = new Socket();
        socket.setReuseAddress(true);
        socket.setKeepAlive(true);
        socket.setTcpNoDelay(true);
        socket.setSoLinger(true, 0);

        // 连接
        socket.connect(new InetSocketAddress(host, port), connectionTimeout);
        socket.setSoTimeout(soTimeout);

        // 证书相关
        if (ssl) {
          if (null == sslSocketFactory) {
            sslSocketFactory = (SSLSocketFactory)SSLSocketFactory.getDefault();
          }
          socket = sslSocketFactory.createSocket(socket, host, port, true);
          if (null != sslParameters) {
            ((SSLSocket) socket).setSSLParameters(sslParameters);
          }
          if ((null != hostnameVerifier) &&
              (!hostnameVerifier.verify(host, ((SSLSocket) socket).getSession()))) {
            String message = String.format(
                "The connection to '%s' failed ssl/tls hostname verification.", host);
            throw new JedisConnectionException(message);
          }
        }
        // 创建socket输出流/输入流。跟redis服务器进行数据传输
        outputStream = new RedisOutputStream(socket.getOutputStream());
        inputStream = new RedisInputStream(socket.getInputStream());
      } catch (IOException ex) {
        broken = true;
        throw new JedisConnectionException("Failed connecting to host " 
            + host + ":" + port, ex);
      }
    }
}

// 检测是否已经打开socket连接
public boolean isConnected() {
    return socket != null && socket.isBound() && !socket.isClosed() && socket.isConnected()
        && !socket.isInputShutdown() && !socket.isOutputShutdown();
  }  


/**
* 向redis服务器发送数据。部分代码省略掉
* cmd：操作的命令 如：set 命令
* args：命令后面的参数。如：set 【hello world】
*/
public void sendCommand(final ProtocolCommand cmd, final byte[]... args) {

    // 进行连接
    connect();
    // 发送命令
    Protocol.sendCommand(outputStream, cmd, args);
    
  }
```

##### sendCommand

```
public static void sendCommand(final RedisOutputStream os, final ProtocolCommand command,
      final byte[]... args) {
    sendCommand(os, command.getRaw(), args);
  }

  // 这里主要是构建OutputStream
  private static void sendCommand(final RedisOutputStream os, final byte[] command,
      final byte[]... args) {
    try {
     
      // 针对操作命令RESP
      os.write(ASTERISK_BYTE);
      os.writeIntCrLf(args.length + 1);
      os.write(DOLLAR_BYTE);
      os.writeIntCrLf(command.length);
      os.write(command);
      os.writeCrLf();

     // 遍历key与value(RESP)
      for (final byte[] arg : args) {
        os.write(DOLLAR_BYTE);
        os.writeIntCrLf(arg.length);
        os.write(arg);
        os.writeCrLf();
      }
    } catch (IOException e) {
      throw new JedisConnectionException(e);
    }
  }

```

回过头来看看前面说到的列子
```
  public static void main(String[] args) {
  
        final Jedis jedis = new Jedis("localhost", 6379);<1>
		
        jedis.set("hello", "world");<2>
		
        jedis.get("hello");<3>
    }
```
进入set
```
public String set(final String key, final String value) {
    // <1>
    client.set(key, value);
    // <2>
    return client.getStatusCodeReply();
  }

<1>：利用client进行set命令操作
<2>：结果返回解析


// Client

 @Override
  public void set(final String key, final String value) {
    // <1>
    set(SafeEncoder.encode(key), SafeEncoder.encode(value));
  }

 public static byte[] encode(final String str) {
    try {
      if (str == null) {
        throw new JedisDataException("value sent to redis cannot be null");
      }
      return str.getBytes(Protocol.CHARSET);
    } catch (UnsupportedEncodingException e) {
      throw new JedisException(e);
    }
  }

  <1>：SafeEncoder.encode()序列化key与value

```

进入BinaryClient set方法
```
 public void set(final byte[] key, final byte[] value) {
    // 发送
    sendCommand(SET, key, value);
  }
  
   public void sendCommand(final ProtocolCommand cmd, final byte[]... args) {
      // 连接 
      connect();
      // 发送
      Protocol.sendCommand(outputStream, cmd, args);
  }
  
  public static void sendCommand(final RedisOutputStream os, final ProtocolCommand command,
      final byte[]... args) {
    // 发送  
    sendCommand(os, command.getRaw(), args);
  }

  // 构建OutputStream内容
  private static void sendCommand(final RedisOutputStream os, final byte[] command,
      final byte[]... args) {
    try {
      // 针对操作命令RESP
      os.write(ASTERISK_BYTE);
      os.writeIntCrLf(args.length + 1);
      os.write(DOLLAR_BYTE);
      os.writeIntCrLf(command.length);
      os.write(command);
      os.writeCrLf();

     // 遍历key与value(RESP)
      for (final byte[] arg : args) {
        os.write(DOLLAR_BYTE);
        os.writeIntCrLf(arg.length);
        os.write(arg);
        os.writeCrLf();
      }
    } catch (IOException e) {
      throw new JedisConnectionException(e);
    }
  }
  
<1>：发送命令，进入之前分析的Connection.sendCommand。
     注意这里的SET。点进去是一个枚举类Command，发现里面的值是对应着操作redis的命令
     这里是SET，也就是对应着服务器上SET命令。
```
到这里发送消息就完毕了。接下来看看是如何解析服务端返回的消息

```
  public static void main(String[] args) {
  
        final Jedis jedis = new Jedis("localhost", 6379);<1>
		
        jedis.set("hello", "world");<2>
		
        jedis.get("hello");<3>
    }
```
进入jedis.get("hello");

```
 public String get(final String key) {
    // get命令。这里就不展开说了，比较简单，跟set命令差不多。
    client.get(key);
    // <1>
    return client.getBulkReply();
  }

<1>：从名字可以看出来是获取回复
```

进入getBulkReply

```
public String getBulkReply() {

    // <1>
    final byte[] result = getBinaryBulkReply();
    if (null != result) {
    // <2>
      return SafeEncoder.encode(result);
    } else {
      return null;
    }
  }
  
  <1>：获取回复的字节数组
  <2>：反序列化
  
```

getBinaryBulkReply
```

 public byte[] getBinaryBulkReply() {
    // <1>
    flush();
    // <2>
    return (byte[]) readProtocolWithCheckingBroken();
  }
  
  protected void flush() {
    try {
      outputStream.flush();
    } catch (IOException ex) {
      broken = true;
      throw new JedisConnectionException(ex);
    }
  }
  
  // readProtocolWithCheckingBroken
  
   protected Object readProtocolWithCheckingBroken() {
      // 读取
      return Protocol.read(inputStream);
  }
  
  public static Object read(final RedisInputStream is) {
    return process(is);
  }
  
  
  
  // 字符串回复
  public static final byte DOLLAR_BYTE = '$';
  
  // 多条字符串回复
  public static final byte ASTERISK_BYTE = '*';
  
  // 状态回复
  public static final byte PLUS_BYTE = '+';
  
  // 错误回复
  public static final byte MINUS_BYTE = '-';
  
  // 整数回复
  public static final byte COLON_BYTE = ':';
  
  // 针对服务器不同的回复类型进行相应的处理。有兴趣的可以继续跟一下看看解析过程
  
  private static Object process(final RedisInputStream is) {
    final byte b = is.readByte();
    switch(b) {
    case PLUS_BYTE:
      return processStatusCodeReply(is);
    case DOLLAR_BYTE:
      return processBulkReply(is);
    case ASTERISK_BYTE:
      return processMultiBulkReply(is);
    case COLON_BYTE:
      return processInteger(is);
    case MINUS_BYTE:
      processError(is);
      return null;
    default:
      throw new JedisConnectionException("Unknown reply: " + (char) b);
    }
  }
  
<1>：对流进行flush  
<2>：对服务器返回的流进行读取

```
至此简单的**set get**分析完毕。

这里总结一下Jedis底层其实是利用Socket跟服务器进行通信。在发送set命令时，将命令序列化成RESP协议。通过Socket发送给服务器。通过服务器不同的返回类型进行响应的处理。




























[image2]:data:image/png;base64,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



[imageOne]:data:image/png;base64,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
