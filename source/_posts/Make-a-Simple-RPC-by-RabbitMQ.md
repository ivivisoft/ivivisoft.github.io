---
title: RabbitMQ 实现简单的RPC
date: 2022-02-04 16:02:43
category:
- RabbitMQ
tags:
- RPC
---

### Remote procedure call (RPC)

RPC简单的说就是，远程调用一个函数方法，并得到响应的结果。

关于RPC方法要注意的点：

1. 明确什么函数需要写在本地，什么函数需要写在Remote
2. 编写好响应的文档，明确调用关系
3. 处理错误情况。什么时候客户端应该重新调用当服务端挂了。
4. 消费RPC服务时，最好不要阻塞的方式等待结果，最好是异步的。

### 准备

1. 在本地启动rabbit－server（3.6.1）

### 整体架构

![整体架构.png](整体架构.png)

我们的RPC是这样工作：

1. 当客户端启动后，他将创建一个匿名的回调队列，用来存放服务的返回结果。
2. 作为一个RPC请求，这个客户端发送的信息包括两个属性，**replyTo**:它指向回调队列。
   **correlationId**：为每一个请求分配一个唯一的值，以备后面获取服务返回的值使用。
3. 请求发送到rpc_queue队列上
4. RPC worker（Server）监控rpc_queue的这个队列，消费上面的信息，将结果发送到replyTo指定的队列上。
5. 客户端监控回调队列，通过correlationId取出属于自己的结果。

### 代码实现

#### maven GAV

```xml
<dependency>
   <groupId>com.rabbitmq</groupId>
   <artifactId>amqp-client</artifactId>
   <version>3.6.1</version>
 </dependency>
```

#### 客户端（client）

```java
[RPCClient] []
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.QueueingConsumer;
import com.rabbitmq.client.AMQP.BasicProperties;
import java.util.UUID;

public class RPCClient {

    private Connection connection;
    private Channel channel;
    private String requestQueueName = "rpc_queue";
    private String replyQueueName;
    private QueueingConsumer consumer;

    public RPCClient() throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        connection = factory.newConnection();
        channel = connection.createChannel();
        //获取自动命名并且退出就自动销毁的队列,作为回调队列
        replyQueueName = channel.queueDeclare().getQueue();
        //在channel使用队列消费
        consumer = new QueueingConsumer(channel);
        //注册消费这个队列
        channel.basicConsume(replyQueueName, true, consumer);
    }

    public String call(String message) throws Exception {
        String response = null;
        String corrId = UUID.randomUUID().toString();
        //设置correlationId和回调队列的名字
        BasicProperties props = new BasicProperties
                .Builder()
                .correlationId(corrId)
                .replyTo(replyQueueName)
                .build();
        //发送消息到rpc请求队列,带上props属性
        channel.basicPublish("", requestQueueName, props, message.getBytes("UTF-8"));
        //监控回调队列的结果变化
        while (true) {
            QueueingConsumer.Delivery delivery = consumer.nextDelivery();
            //比较correlationId以确定是否为自己调用的结果
            if (delivery.getProperties().getCorrelationId().equals(corrId)) {
                response = new String(delivery.getBody(),"UTF-8");
                break;
            }
        }

        return response;
    }

    public void close() throws Exception {
        connection.close();
    }

    public static void main(String[] argv) {
        RPCClient fibonacciRpc = null;
        String response = null;
        try {
            fibonacciRpc = new RPCClient();
            System.out.println(" [x] Requesting fib(30)");
            //执行调用
            response = fibonacciRpc.call("30");
            System.out.println(" [.] Got '" + response + "'");
        }
        catch  (Exception e) {
            e.printStackTrace();
        }
        finally {
            if (fibonacciRpc!= null) {
                try {
                    fibonacciRpc.close();
                }
                catch (Exception ignore) {}
            }
        }
    }
}
```

#### RPC服务端（RPCServer）

```java
[RPCServer] []
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.QueueingConsumer;
import com.rabbitmq.client.AMQP.BasicProperties;

public class RPCServer {
    
    //监控的rpc请求队列名字
    private static final String RPC_QUEUE_NAME = "rpc_queue";
    
    //服务
    private static int fib(int n) {
        if (n ==0) return 0;
        if (n == 1) return 1;
        return fib(n-1) + fib(n-2);
    }

    public static void main(String[] argv) {
        Connection connection = null;
        Channel channel = null;
        try {
            ConnectionFactory factory = new ConnectionFactory();
            factory.setHost("localhost");

            connection = factory.newConnection();
            channel = connection.createChannel();

            channel.queueDeclare(RPC_QUEUE_NAME, false, false, false, null);
            //保障在发送确认消息之前,broker不再发送消息过来
            channel.basicQos(1);
            QueueingConsumer consumer = new QueueingConsumer(channel);
            channel.basicConsume(RPC_QUEUE_NAME, false, consumer);
            System.out.println(" [x] Awaiting RPC requests");
            //监控rpc请求消息队列
            while (true) {
                String response = null;
                //获取队列消息载体
                QueueingConsumer.Delivery delivery = consumer.nextDelivery();
                //获取消息带过来的参数
                BasicProperties props = delivery.getProperties();
                //新组织发放回调队列的参数,correlationId以备客户端可以获取属于自己的返回结果
                BasicProperties replyProps = new BasicProperties
                        .Builder()
                        .correlationId(props.getCorrelationId())
                        .build();

                try {
                    //获取队列消息
                    String message = new String(delivery.getBody(),"UTF-8");
                    int n = Integer.parseInt(message);

                    System.out.println(" [.] fib(" + message + ")");
                    //调用服务函数
                    response = "" + fib(n);
                }
                catch (Exception e){
                    System.out.println(" [.] " + e.toString());
                    response = "";
                }
                finally {
                    //发送结果信息到客户端的回调队列
                    channel.basicPublish( "", props.getReplyTo(), replyProps, response.getBytes("UTF-8"));
                    //发送确认消息
                    channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
                }
            }
        }
        catch  (Exception e) {
            e.printStackTrace();
        }
        finally {
            if (connection != null) {
                try {
                    connection.close();
                }
                catch (Exception ignore) {}
            }
        }
    }
}
```

### 总结

#### 优点：

1. 如果RPC服务太慢，你可以再启动一个服务端，会自动负载均衡
2. 客户端只需要发送和接收一个信息，不需要同步调用服务。这样一个RPC服务请求只需要一次网络回路

#### 遗留问题：

1. 如果没有RPC服务在运行，客户端应该怎样重新请求
2. 客户端是否应该有超时的RPC
3. 服务端执行过程有一个异常是否应该抛给客户端
4. 服务端信息校验等保护自己避免攻击的方法
