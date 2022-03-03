---
title: 实现简单的RPC(Remote procedure call)
excerpt: RPC在稍微大一点的系统中一般都会用到，让我们使用JAVA语言，实现简单的RPC吧！！！
date: 2022-02-06 14:29:18
category:
- Java
tags:
- RPC
---

### 整体架构

![整体架构.png](整体架构.png)

### 代码实现

#### 服务接口

```java
[EchoService] []
public interface EchoService {
    String echo(String ping);
}
```

#### 服务实现

```java
[EchoServiceImpl] []
public class EchoServiceImpl implements EchoService {
    public String echo(String ping) {
        return ping != null ? ping + " --> I am Ok." : "I am Ok.";
    }
}
```

#### 服务发布者(Exporter)

服务发布者
主要职责如下:

1. 作为服务端,监听客户端TCP连接,接收到客户端连接后,将其封装成Task,由线程池执行
2. 将客户端发送的码流发序列化成对象,反射调用服务实现者,获取执行结果
3. 将执行结果对象反序列化,通过ocket发送给客户端.
4. 远程服务调用完成之后,释放Socket等连接资源,防止句柄泄露



```java
[RpcExporter] []
public class RpcExporter {
    static Executor executor = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());

    public static void exporter(String hostname, int port) throws Exception {
        ServerSocket server = new ServerSocket();
        server.bind(new InetSocketAddress(hostname, port));

        try {
            while (true) {
                executor.execute(new ExporterTask(server.accept()));
            }
        } finally {
            server.close();
        }
    }

    private static class ExporterTask implements Runnable {

        Socket client = null;

        public ExporterTask(Socket client) {
            this.client = client;
        }

        public void run() {
            ObjectInputStream input = null;
            ObjectOutputStream output = null;
            try {
                input = new ObjectInputStream(client.getInputStream());
                String interfaceName = input.readUTF();
                Class<?> service = Class.forName(interfaceName);
                String methodName = input.readUTF();
                Class<?>[] paramterType = (Class<?>[]) input.readObject();
                Object[] arguments = (Object[]) input.readObject();
                Method method = service.getMethod(methodName, paramterType);
                Object result = method.invoke(service.newInstance(), arguments);
                output = new ObjectOutputStream(client.getOutputStream());
                output.writeObject(result);

            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                if (output != null) {
                    try {
                        output.close();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
                if (input != null) {
                    try {
                        input.close();
                    } catch (Exception e) {

                    }
                }
                if (client != null) {
                    try {
                        client.close();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }
}
```

#### 本地代理(Importer)

本地代理
主要职责如下:

1. 将本地的接口调用转换成JDK动态代理,在动态代理中实现接口的远程调用

2. 创建Socket客户端,根据指定地址连接远程服务提供者

3. 将远程服务端调用所需的接口类,方法名,参数列表等编码后发送给服务提供者

4. 同步阻塞等待服务端返回应答,获取应答之后返回



```java
[RpcImporter] []
public class RpcImporter<S> {
    public S importer(final Class<?> serviceClass, final InetSocketAddress addr) {
        return (S) Proxy.newProxyInstance(serviceClass.getClassLoader(), new Class<?>[]{serviceClass.getInterfaces()[0]}, new InvocationHandler() {
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                Socket socket = null;
                ObjectOutputStream output = null;
                ObjectInputStream input = null;
                try {
                    socket = new Socket();
                    socket.connect(addr);
                    output = new ObjectOutputStream(socket.getOutputStream());
                    output.writeUTF(serviceClass.getName());
                    output.writeUTF(method.getName());
                    output.writeObject(method.getParameterTypes());
                    output.writeObject(args);
                    input = new ObjectInputStream(socket.getInputStream());
                    return input.readObject();
                } finally {
                    if (socket != null) {
                        socket.close();
                    }
                    if (output != null) {
                        output.close();
                    }
                    if (input != null) {
                        input.close();
                    }
                }
            }
        });
    }
}
```

#### 测试类

```java
[RpcTest] []
public class RpcTest {
    public static void main(String[] args) {
        //发布服务
        new Thread(new Runnable() {
            public void run() {
                try {
                    RpcExporter.exporter("localhost", 8080);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();

        RpcImporter<EchoService> importer = new RpcImporter<EchoService>();
        EchoService echoService = importer.importer(EchoServiceImpl.class, new InetSocketAddress("localhost", 8080));
        System.out.println(echoService.echo("Are you Ok?"));

    }
}
```

