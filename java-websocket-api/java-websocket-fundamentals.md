---
description: >-
  Quality is more important than quantity. One home run is better than two
  doubles.
---

# Java WebSocket的基本原理

在Java EE中，WebSocket已经有了完整的实现，在我们部署Java EE服务的时候就会自动载入我们设定好的WebSocket端点服务，并不需要我们额外进行项目级部署，仅仅需要我们实现WS的业务层即可。

跟许多API一样，WebSocket有编程式和注解式两种实现，编程式更能体现实现原理，而注解式具有更好的低耦合性，更适合高效开发。该笔记将以注解式实现为主，配合以编程式实现讲解原理。

先来看一下书中的简单示例：

```java
import javax.websocket.OnMessage;
import javax.websocket.server.ServerEndpoint;

@ServerEndpoint("/echo")
public class EchoServer {
    
    @OnMessage
    public String echo(String incomingMessage) {
        return "I got this (" + incomingMessage + ")"
                + " so I am sending it back !";
    }

}
```

服务端只需要这段代码就可以完成消息获取和回传。注解@ServerEndpoint表示该类是一个服务端的WebSocket端点，定义的字符串即为连接该端点的URI地址。被@OnMessage注解标注的方法是用来处理收到客户端传来的消息时调用的方法。其返回值会被当做服务端向客户端的响应直接回传给客户端。

此处我们只需要在网页中用js实现ws连接：

```markup
<!DOCTYPE html>
<html>
    <head>
        <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
        <title>Web Socket JavaScript Echo Client</title>
        <script language="javascript" type="text/javascript">
            var echo_websocket;
            
            function init() {
                output = document.getElementById("output");
            }

            function send_echo() {
                var wsUri = "ws://localhost:8080/echoserver/echo";
                writeToScreen("Connecting to " + wsUri);
                echo_websocket = new WebSocket(wsUri);
                echo_websocket.onopen = function (evt) {
                    writeToScreen("Connected !");
                    doSend(textID.value);
                };
                echo_websocket.onmessage = function (evt) {
                    writeToScreen("Received message: " + evt.data);
                    echo_websocket.close();
                };
                echo_websocket.onerror = function (evt) {
                    writeToScreen('<span style="color: red;">ERROR:</span> '
                     + evt.data);
                    echo_websocket.close();
                }; 
            }

            function doSend(message) {
                echo_websocket.send(message);
                writeToScreen("Sent message: " + message);
            }

            function writeToScreen(message) {
                var pre = document.createElement("p");
                pre.style.wordWrap = "break-word";
                pre.innerHTML = message;
                output.appendChild(pre);
            }
            
            window.addEventListener("load", init, false);
            
        </script>
    </head>
    <body>
        <h1>Echo Server</h1>
        
        <div style="text-align: left;">
            <form action="">
                <input onclick="send_echo()" value="Press to send" type="button">
                <input id="textID" name="message" value="Hello Web Sockets" type="text"><br>
            </form>
        </div>
        <div id="output"></div>
    </body>
</html>
```

这样就已经实现了客户端和服务端的最简单WebSocket实现。

在客户端，HTML5标准已经将WebSocket吸纳，并且可以直接通过Web API中WebSocket类直接用JavaScript控制WebSocket的连接和响应。我们更关心的是服务端是如何实现。我们先从WebSocket协议说起。

## WebSocket协议

WebSocket协议定义了客户端和服务端之间长时间存活的专用TCP连接，因此比传统Web请求/响应模型更进一步。WebSocket协议定义了WebSocket连接上往返传输的数据的各个块的格式。一旦连接建立，传输的元数据帧就会描述双端的行为和用途。

协议中包含两种主要类型的帧：控制帧（Control Frame）和数据帧（Data Frame），控制帧用于执行协议的一些内部功能逻辑，例如关闭帧（close frame）就是一种传递关闭信号的帧。另一对检测连接健康性的控制帧是Ping帧和Pong帧。协议规定任何一端收到Ping帧必须相应Pong帧，发送端收到Pong帧即可确认当前连接状态，而相应时间也可作为判定连接的稳定性。

数据帧分为两种基本类型：文本型和二进制型，显然文本型主体就是字符串，用于文本消息交换。而二进制就可以发送任何数据了，比如图片。不管是哪种类型，WebSocket都提供了两种发送方式，一种是一次消息携带完整的消息，一种是分批次携带完整消息的一部分，这种数据帧称为部分帧（partial frame）。当传送大量数据时，比如文件，部分帧就变得非常有用了。

基于WebSocket协议的模型，Java API的实现也完全遵循标准。我们先从WebSocket的端点实现说起。

## 端点（Endpoint）

端点配置了一个WebSocket连接的终端，相应的也规定了终端的基本行为和响应方式。作为连接我们当然需要两个端点，客户端可以通过JavaScript来实现端点配置，我们将来另外讨论，我们只关注Java的API实现。

从上文的简单示例中我们了解，我们可以通过一个类加上`@ServerEndpoint`注解就可以配置出一个服务端点。WebSocket共有四种行为相应，分别对应方法注解`@OnOpen`、`@OnMessage`、`@OnError`、`@OnClose`。也就是一个完整的端点业务代码是这样的：

```java
@ServerEndPoint("/server")
public class ServerWebSocketEndPoint{
    @OnOpen //连接建立时调用该方法，一般执行一些初始化
    public void whenOpen(){
    ...
    }
    @OnMessage //收到客户端数据帧时调用的方法
    public void whenGetMessage(){
    ...
    }
    @OnError //连接出现错误时调用的方法，用于异常处理
    public void whenError(){
    ...
    }
    @OnClose //收到关闭帧时调用的方法
    public void whenClose(){
    ...
    }
}
```

注解式一目了然，一个端点四个事件，分别实现即可。Java EE已经为注解扫描做了封装，并不需要做额外的配置，也无需在`web.xml`中添加配置，类似于注解式的Servlet可以直接被注册。

从使用和开发的角度来说注解是非常好的选择。我们只通过编程式的代码来了解一下Java WebSocket的实现原理，并不推荐使用编程式完成WebSocket开发。

首先注解式的端点类其实会被封装成一个抽象类`javax.websocket.Endpoint`的继承，我们来看一个编程式端点的示例：

```java
public class JavaServerEndPoint extends Endpoint {

    /**
     * onOpen方法定义方法连接时，即为跟一个客户端打开连接时候执行的初始化操作，为
     * EndPoint的抽象方法，必须实现。一般需要通过session的addMessageHandler方法添
     * 加处理消息的处理器。
     * @param session 表示客户端的传入对象，通过给session添加消息处理器完成异步操作
     * @param endpointConfig 该对象用来获取端点的配置信息，包括用户属性和编解码器
     */
    @Override
    public void onOpen(Session session, EndpointConfig endpointConfig) {
        System.out.println("Java Chat");
        /*
         * 通过session的addMessageHandler方法添加消息处理器，MessageHandler为接口
         * 但接口其实内部有两个子接口，一般都实现其中的一个即可。Whole接口表示一次接收
         * 的消息是一条，而Partial表示需要多片消息衔接成为一条。
         * 处理消息的参数为泛型，可以自己定义任意的处理类型，常见的有处理文字的String
         * 和处理文件的byte[]，而分片消息则可以使用StringBuffer和ByteBuffer这种具有
         * 缓存机制的处理对象
         *
         * ！实现接口的匿名类不能写作Lambda表达式，否则会产生空指针异常
         */
        session.addMessageHandler(new MessageHandler.Whole<String>() {
            @Override
            public void onMessage(String s) {
                /*
                 * 若向客户端发送消息则机制有两种，同步的BasicRemote和异步的AsyncRemote
                 * 同步会产生阻塞，即若无法确认客户端收到信息会一直阻塞方法，若前后代码
                 * 不存在必然逻辑推荐使用异步处理
                 */
                session.getAsyncRemote().sendText(s);
            }
        });
        /*
         * WebSocket中有一组特别的消息叫做Ping和Pong，一般用来测试两端的连接有效性和网络
         * 延迟。使用ByteBuffer作为参数，方法规定传送的消息不可超过125字节，否则将抛出
         * IllegalArgumentException。
         */
        try {
            session.getAsyncRemote().sendPing(ByteBuffer.wrap("1234".getBytes()));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void onClose(Session session, CloseReason closeReason) {
        super.onClose(session, closeReason);
    }

    @Override
    public void onError(Session session, Throwable thr) {
        super.onError(session, thr);
    }
}
```

`Endpoint`类只有一个抽象方法必须实现就是`onOpen`方法。也就是说API希望开发者提供其具体实现而不是留空，这个方法对应`@OnOpen`注解的方法。该方法有两个参数`Session`和`EndpointConfig` ，这两个默认传入的对象我们将在后续讨论，本节只讨论四个基本事件的实现。

`Endpoint`类还有两个可以重写的方法就是`onClose`和`onError`，分别对应`@OnClose`和`@OnError`两个注解，这都非常好理解。那么非常奇怪的就是，`@OnMessage`注解对应的方法去哪了了呢？答案就在`Session`对象中。`Session`与`HttpSession`对象一样对应一个客户端连入，编程式端点是通过给这个对象添加消息处理器（MessageHandler）来完成对收到客户端消息的处理。上文代码中我们可以看到，MessageHandler需要一个接口的实现。在`MessageHandler`接口中有两个子接口，分别是`MessageHandler.Whole和MessageHandler.Partial`分别对应协议中的完整数据帧和部分数据帧。根据需要我们实现其中的一个就可以完成对消息的处理。其中的`OnMessage`方法就对应了`@OnMessage`注解的方法。一般我们会在`onOpen`方法中添加消息处理器，否则连接成功也无法处理收到的传入消息，也许这就是API单独将`onOpen`方法抽象的原因。从`addMessageHandler`方法我们可以猜到，一个端点是可以注册多个消息处理器的，但Java WebSocket为了实现能将入站消息分配到正确的消息处理器上，有一个严格的限制：每种消息类型（文本消息，二进制消息和Pong消息）最多只能有一个消息处理器。这一点在注解式端点也是一样的，只能为每种消息类型提供一个`@OnMessage`注解方法。若出现了额外的消息处理器，则会抛出`java.lang.IllegalStateException`异常。

{% hint style="info" %}
这里有两个注意的点：

一、经过测试发现，在实现MessageHandler的子接口时不能使用Lambda表达式，即使加上了类型强转声明依然会出现异常，所以还是使用内部匿名类的方式来实现接口。其原因是WebSocket API中接口并没有添加@FunctionalInterface特性。

二、注解式端点中并不需要在方法中添加编程式端点中方法的全部参数，只需根据需要调用即可，因此注解式端点开发还是更为直观和便捷。
{% endhint %}

编程式端点与注解式有一个非常明显的区别就是注解式只需要一个`@ServerEndpoint`注解即可注册对应的WebSocket连接地址，像注解`@Servlet`一样便捷。编程式端点我们通过`Endpoint`类实现了事件的处理，但并没有地方可以定义该端点的连接地址。因此我们还需要额外实现一个接口`javax.websocket.server.ServerApplicationConfig`为端点注册。在当前的Java EE版本中只需要一个`@ServerEndpoint`注解就会帮我们完成这个功能了。

我们来关注一下这个接口：

```java
public interface ServerApplicationConfig {
    Set<ServerEndpointConfig> getEndpointConfigs(Set<Class<? extends Endpoint>> endpointClasses);

    Set<Class<?>> getAnnotatedEndpointClasses(Set<Class<?>> scanned);
}
```

`getAnnotatedEndpointClasses`方法传入的是所有注解式端点的类，是由Java EE自动扫描的，而我们已经可以注解来完成所有配置的装配，因此这个方法我们无需手动实现任何配置，只需要返回传入`Set`即可，当然我们可以在这个方法中对集合进行增删，这就根据业务需求了。

`getEndpointConfigs`方法传入的是所有实现`Endpoint`类的子类，而我们需要对这些端点类进行装配，生成对应的`ServerEndpointConfig`接口实现（实际我们需要实现的是一个`final`类`DefaultServerEndpointConfig`的对象），并返回出去，这样才能被Java EE实现为对应的WebSocket连接。

`DefaultServerEndpointConfig`中包含许多对象属性，有幸的是API中`ServerEndpointConfig`接口有一个内部`Builder`类方便我们快速便捷的生成`ServerEndpointConfig`对象：

```java
    @Override
    public Set<ServerEndpointConfig> getEndpointConfigs(Set<Class<? extends Endpoint>> set) {
        Set<ServerEndpointConfig> configs = new HashSet<>();
        configs.add(ServerEndpointConfig.Builder
                .create(JavaServerEndPoint.class, "/javaChat")
//                .subprotocols(subprotocals) 设定连接的协议，传入的是一个List<String>
//                .encoders(encoders) 设定自定义编码器，传入的是一个List<Encoder>
//                .decoders(decoders) 设定自定义解码器，传入的是一个List<Encoder>
//                .extensions(extensions) 设定连接的扩展，传入的是一个List<Extension>
//                .configurator(configurator) 设定连接的配置器，传入的是一个ServerEndpointConfig.Configurator对象
                .build());
        return configs;
    }
```

通过`Builder`类我们可以从容的选择需要配置的部分，这里凸显了Builder创建者设计模式的强大。

虽然该方法传入了`Endpoint`类的实现类集合，但是与其通过`getName`方法一个个判断类名来为每个类定义各自的配置，不如直接用新的`Set`来装填，因为我们自己实现的类是啥再熟悉不过了。当然传入的`Set`并不是完全无用，若是想要其他jar包中的端点类，就可以通过这种形式来载入到当前Java EE中。不过还是推荐开发使用注解式，更加轻量级，具有很好的低耦合性。

在`Builder`类中，我们看到WebSocket有着其他属性可以配置，我们会在下面的章节分别讲解其作用。

