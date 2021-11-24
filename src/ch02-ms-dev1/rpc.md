# Spring Boot集成gRPC框架

gRPC是谷歌开源的高性能、开源、通用RPC框架。由于gRPC基于HTTP2协议，所以其对移动端非常友好。

本节将介绍Spring Boot集成gRPC的服务端、客户端。

### 安装protoc及gRPC

gRPC默认使用[Protocol Buffers]([Protocol Buffers &nbsp;|&nbsp; Google Developers](https://developers.google.com/protocol-buffers))做为序列化协议，我们首先安装protoc编译器：

在这里下载最新版本的[protoc](https://github.com/protocolbuffers/protobuf/releases/tag/v3.17.3)编译器，请根据你的操作系统选择对应版本，这里我选用MacOSX的。

```bash
wget https://github.com/protocolbuffers/protobuf/releases/download/v3.17.3/protoc-3.17.3-osx-x86_64.zip
unzip protoc-3.17.3-osx-x86_64.zip
```

解压缩后，将其加入PATH路径下：

```bash
export PATH=$PATH:$YOUR_PROTOC_PATH
```

试一下是能否执行：

```bash
protoc --version
libprotoc 3.17.3
```

除此之外，我们还需要一个gRPC的Java插件，才能生成gRPC的桩代码，你可以在[这里]([Maven Central Repository Search](https://search.maven.org/search?q=a:protoc-gen-grpc-java))找到最新版本。这里我们依然选择OSX的64位版本：

```bash
wget https://search.maven.org/remotecontent?filepath=io/grpc/protoc-gen-grpc-java/1.40.1/protoc-gen-grpc-java-1.40.1-osx-x86_64.exe
```

下载后，将其加入PATH路径中。尝试定位一下：

```bash
which protoc-gen-grpc-java 
Your_Path/protoc-gen-grpc-java
```

至此，protoc和grpc的安装准备工作已经就绪。

## Client侧集成

首先是集成依赖，我们放在client子项目的builld.gradle中：

```groovy
implementation 'com.google.protobuf:protobuf-java:3.17.3'
implementation "io.grpc:grpc-stub:1.39.0"
implementation "io.grpc:grpc-protobuf:1.39.0"
implementation 'io.grpc:grpc-netty-shaded:1.39.0'
```

由于版本依赖较多，我建议使用platform统一管理，可以参考[前文](./spring-boot.md)。

接着，我们编写protoc文件，HomsDemo.proto：

```protobuf
syntax = "proto3";
option java_package = "com.coder4.homs.demo";
option java_outer_classname = "HomsDemoProto";
;

message AddRequest {
    int32 val1 = 1;
    int32 val2 = 2;
}

message AddResponse {
    int32 val = 1;
}

message AddSingleRequest {
    int32 val = 1;
}

service HomsDemo {
    rpc Add(AddRequest) returns (AddResponse);
    rpc Add2(stream AddSingleRequest) returns (AddResponse);
}
```

我们添加了两个RPC方法：

- Add是正常的调用

- Add2是单向Stream调用

接着，我们需要编译，生成桩文件：

```bash
#!/bin/sh

DIR=`cd \`dirname ${BASH_SOURCE[0]}\`/.. && pwd`

protoc HomsDemo.proto --java_out=${DIR}/homs-demo-client/src/main/java --proto_path=${DIR}/homs-demo-client/src/main/java/com/coder4/homs/demo/
protoc HomsDemo.proto --plugin=protoc-gen-grpc-java=`which protoc-gen-grpc-java` --grpc-java_out=${DIR}/homs-demo-client/src/main/java --proto_path=${DIR}/homs-demo-client/src/main/java/com/coder4/homs/demo/
```

这里分为两个步骤：

- 第一次protoc编译，生成protoc的桩文件

- 第二次protoc编译，使用了protoc-gen-grpc-java的插件，生成gRPC的服务端和客户端文件

编译成功后，路径如下：

```bash
homs-demo-client

├── build.gradle
└── src
    └── main
        └── java
            └── com
                └── coder4
                    └── homs
                        └── demo
                            ├── HomsDemo.proto
                            ├── HomsDemoGrpc.java
                            └── HomsDemoProto.java
```

如上所示：HomsDemoProto是protoc的桩文件，HomsDemoGrpc是gRPC服务的桩文件。

下面我们来编写客户端代码，HomsDemoClient.java：

```java
package com.coder4.homs.demo.client;

import com.coder4.homs.demo.HomsDemoGrpc;

import com.coder4.homs.demo.HomsDemoProto.AddRequest;
import com.coder4.homs.demo.HomsDemoProto.AddResponse;
import com.coder4.homs.demo.HomsDemoProto.AddSingleRequest;
import io.grpc.Channel;
import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import io.grpc.StatusRuntimeException;
import io.grpc.stub.StreamObserver;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.Arrays;
import java.util.Collection;
import java.util.Optional;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;


/**
 * @author coder4
 */
public class HomsDemoClient {

    private Logger LOG = LoggerFactory.getLogger(HomsDemoClient.class);

    private final HomsDemoGrpc.HomsDemoBlockingStub blockingStub;

    private final HomsDemoGrpc.HomsDemoStub stub;

    /**
     * Construct client for accessing HelloWorld server using the existing channel.
     */
    public HomsDemoClient(Channel channel) {
        blockingStub = HomsDemoGrpc.newBlockingStub(channel);
        stub = HomsDemoGrpc.newStub(channel);
    }

    public Optional<Integer> add(int val1, int val2) {
        AddRequest request = AddRequest.newBuilder().setVal1(val1).setVal2(val2).build();
        AddResponse response;
        try {
            response = blockingStub.add(request);
            return Optional.ofNullable(response.getVal());
        } catch (StatusRuntimeException e) {
            LOG.error("RPC failed: {0}", e.getStatus());
            return Optional.empty();
        }
    }

    public Optional<Integer> add2(Collection<Integer> vals) {

        try {

            CountDownLatch cdl = new CountDownLatch(1);

            AtomicLong respVal = new AtomicLong();

            StreamObserver<AddSingleRequest> requestStreamObserver =
                    stub.add2(new StreamObserver<AddResponse>() {
                        @Override
                        public void onNext(AddResponse value) {
                            respVal.set(value.getVal());
                        }

                        @Override
                        public void onError(Throwable t) {
                            cdl.countDown();
                        }

                        @Override
                        public void onCompleted() {
                            cdl.countDown();
                        }
                    });

            for (int val : vals) {
                requestStreamObserver.onNext(AddSingleRequest.newBuilder().setVal(val).build());
            }
            requestStreamObserver.onCompleted();

            try {
                cdl.await(1, TimeUnit.SECONDS);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            return Optional.ofNullable(respVal.intValue());
        } catch (StatusRuntimeException e) {
            LOG.error("RPC failed: {0}", e.getStatus());
            return Optional.empty();
        }
    }
}
```

代码如上所示：Add还是相对简单的，但是使用了Stream的Add2就比较复杂了。

在上述代码中，需要传入Channel做为连接句柄，在假设知道IP和端口的情况下，可以如下构造：

```java
String target = "127.0.0.1:5000";
        ManagedChannel channel = null;
        try {
            channel = ManagedChannelBuilder
                    .forTarget(target)
                    .usePlaintext()
                    .build();
        } catch (Exception e) {
            LOG.error("open channel excepiton", e);
            return;
        }


        HomsDemoClient client = new HomsDemoClient(channel);
```

在微服务架构下，实例众多，获取每个IP显得不太实际，我们会在后续章节介绍集成服务发现的Channel构造方案。

## Server侧集成

老套路，首先是依赖集成：

```groovy
implementation 'com.google.protobuf:protobuf-java:3.17.3'
implementation "io.grpc:grpc-stub:1.39.0"
implementation "io.grpc:grpc-protobuf:1.39.0"
implementation 'io.grpc:grpc-netty-shaded:1.39.0'
```

与上述客户端的集成完全一致。

接下来我们实现RPC的服务逻辑：

```java
/**
 * @(#)HomsDemoImpl.java, 8月 12, 2021.
 * <p>
 * Copyright 2021 coder4.com. All rights reserved.
 * CODER4.COM PROPRIETARY/CONFIDENTIAL. Use is subject to license terms.
 */
package com.coder4.homs.demo.server.grpc;

import com.coder4.homs.demo.HomsDemoGrpc.HomsDemoImplBase;
import com.coder4.homs.demo.HomsDemoProto.AddRequest;
import com.coder4.homs.demo.HomsDemoProto.AddResponse;
import com.coder4.homs.demo.HomsDemoProto.AddSingleRequest;
import io.grpc.stub.StreamObserver;

/**
 * @author coder4
 */
public final class HomsDemoGrpcImpl extends HomsDemoImplBase {

    @Override
    public void add(AddRequest request, StreamObserver<AddResponse> responseObserver) {
        responseObserver.onNext(AddResponse.newBuilder()
                .setVal(request.getVal1() + request.getVal2())
                .build());
        responseObserver.onCompleted();
    }

    @Override
    public StreamObserver<AddSingleRequest> add2(StreamObserver<AddResponse> responseObserver) {
        return new StreamObserver<AddSingleRequest>() {

            int sum = 0;

            @Override
            public void onNext(AddSingleRequest value) {
                sum += value.getVal();
            }

            @Override
            public void onError(Throwable t) {

            }

            @Override
            public void onCompleted() {
                responseObserver.onNext(AddResponse.newBuilder()
                        .setVal(sum)
                        .build());
                sum = 0;
                responseObserver.onCompleted();
            }
        };
    }
}
```

这里要特别说明，因为gRPC都是异步回调的方式，所以其RPC在实现上有点反直觉：

- 通过responseObserver.onNext返回调用结果

- 通过responseObserver.onCompleted结束调用

而add2方法，由于采用了Client-Streaming，所以实现会更加复杂一些。

实际上，gRPC支持[4种调用模式]([Generated-code reference | Java | gRPC](https://grpc.io/docs/languages/java/generated-code/))：

- Unary: 客户端单输入，服务端单输出

- Client-Streaming: 客户端多输入，服务端单输出

- Server-Streaming: 客户端单输入，服务端多输出

- Bidirectional-Streaming: 客户端多输入，服务端多输出

由于篇幅所限，本文种只实现了前2种，推荐你手动实现另外的两种模式。
