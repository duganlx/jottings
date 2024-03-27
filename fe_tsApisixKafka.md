# typescript 通过 apisix 实现与 kafka 交互

## 准备

kafka addr: `192.168.1.132:9092, 192.168.1.132:9093, 192.168.1.131:9092`

Offset Explorer 2.3.3

> Offset Explorer 连接 Kafka 配置
>
> - Cluster name: fat
> - Kafka Cluster Version: 2.8
> - Zookeeper Host: 192.168.1.132
> - Zookeeper Port: 2181
> - chroot path: /

APISIX Dashboard 3.0.1, fat: http://192.168.12.29:9100/

APISIX node {etcd_version: 3.4.0, version: 3.2.0}

PubSub Kafka: https://apisix.apache.org/zh/docs/apisix/pubsub/kafka/

communication protocol: https://github.com/apache/apisix/blob/master/apisix/include/apisix/model/pubsub.proto

protoc: https://github.com/protocolbuffers/protobuf

protoc-gen-ts: https://github.com/thesayyn/protoc-gen-ts

## 工序

下载 pubsub.proto 协议文件, 使用命令 `protoc --ts_out=. pubsub.proto` 生成 ts sdk. 在 APISIX Dashboard 中配置 _上游_ 和 _路由_, 配置如下

```text
// 上游
{
  "nodes": [
    {
      "host": "192.168.1.131",
      "port": 9092,
      "weight": 1
    },
    {
      "host": "192.168.1.132",
      "port": 9092,
      "weight": 1
    },
    {
      "host": "192.168.1.132",
      "port": 9093,
      "weight": 1
    }
  ],
  "timeout": {
    "connect": 6,
    "send": 60,
    "read": 60
  },
  "type": "roundrobin",
  "scheme": "kafka",
  "pass_host": "pass",
  "name": "kafka test",
  "desc": "test Kafka pull msg",
  "keepalive_pool": {
    "idle_timeout": 60,
    "requests": 1000,
    "size": 320
  }
}

// 路由
{
  "uri": "/kafka/websocket",
  "name": "ws-kafka",
  "methods": [
    "GET",
    "POST",
    "PUT",
    "DELETE",
    "PATCH",
    "HEAD",
    "OPTIONS",
    "CONNECT",
    "TRACE",
    "PURGE"
  ],
  "upstream_id": "479987480301404864",
  "enable_websocket": true,
  "status": 1
}
```

pubsub.proto 定义了四种请求命令 `CmdPing, CmdEmpty, CmdKafkaListOffset, CmdKafkaFetch`, 先用前两个来弄个 hello world. 代码如下所示. 其中的请求中的`sequence`参数是用于关联响应.

```typescript
const serverIp = "ws://eqw-test.eam.com/kafka/websocket";
// ws.readyState: 0(CONNECTING), 1(OPEN), 2(CLOSING), 3(CLOSED)
const ws = new WebSocket(serverIp);

ws.onopen = (e) => {
  console.log("ws state is open...");

  // ping request
  const pingseq = 100;
  console.log("send ping req via ws, request seq:", pingseq);
  const pingreq = new PubSubReq({
    sequence: pingseq,
    cmd_ping: new CmdPing({ state: new Uint8Array([72, 65, 100, 10, 90]) }),
  });
  ws.send(pingreq.serialize());

  // empty request
  const emptyseq = 101;
  console.log("send empty req via ws, request seq:", emptyseq);
  const emptyreq = new PubSubReq({
    sequence: emptyseq,
    cmd_empty: new CmdEmpty(),
  });
  ws.send(emptyreq.serialize());
};
ws.onmessage = (e) => {
  const msgBlob: Blob = e.data;

  msgBlob.arrayBuffer().then((res) => {
    const resp: PubSubResp = PubSubResp.deserialize(new Uint8Array(res));
    const respobj = resp.toObject();

    if (respobj.pong_resp !== undefined) {
      console.log(
        "receive pong response\n",
        `corresponding request seq: ${respobj.sequence}\n`,
        `ping request content: ${respobj.pong_resp.state}\n`,
        respobj
      );
    } else if (respobj.error_resp !== undefined) {
      console.log(
        "receive empty response\n",
        `corresponding request seq: ${respobj.sequence}\n`,
        respobj
      );
    } else {
    }
  });
};
ws.onerror = (e) => {
  console.log("ws error", e);
};
ws.onclose = (e) => {
  console.log("ws close", e);
};
```

根据官网的描述，目前只提供 `ListOffset` 和 `Fetch` 两个 API 去拉取 Kafka 消息, 暂不支持消费者组特性, 也不能管理偏移量.

ListOffset 只是获取 `offset`, 它需要三个参数 `topic, partition, timestamp`, 其中 timestamp 可以取三种值 `-1, -2, unix timestamp`, 具体含义如下. 根据测试发现, 使用 `unix timestamp` 场景下, 当存在多个相同 unix timestamp 的记录时, 会返回第一条记录的 offset.

- `-1`: 目前 partition 中最后一条消息的 offset
- `-2`: 目前 partition 中第一条消息的 offset
- `unix timestamp`: 在指定时间戳之后的第一条消息的 offset

Fetch 需要三个参数 `topic, partition, offset`, 测试发现当指定一个 _合理_ 的 offset 时, 它会返回一个 KafkaMessage 数组(`KafkaMessage{offset: int64, timestamp: int64, key: bytes, value: bytes}`), 这个数组的第一条消息的 offset 一定是小于 指定的 `offset`, 但具体小多少并没有总结出规律, 而最后一条一定是目前 topic 的 parition 中最后一条记录. 经过初步测试发现, 这个 _合理_ 的 `offset` 是 $offset ≈ offset_{latestmsg} - 200$ .

```typescript
// == In ws onopen function ==
// kafka list offset request
const kloseq = 100;
const kloreq = new PubSubReq({
  sequence: kloseq,
  cmd_kafka_list_offset: new CmdKafkaListOffset({
    topic: "rms_order",
    partition: 0,
    timestamp: 1695353632271,
  }),
});
ws.send(kloreq.serialize());

// kafka fetch request
const kfseq = 101;
const kfreq = new PubSubReq({
  sequence: kfseq,
  cmd_kafka_fetch: new CmdKafkaFetch({
    topic: "rms_order",
    partition: 0,
    offset: 54764,
  }),
});
ws.send(kfreq.serialize());

// == In ws onmessage function ==
const msgBlob: Blob = e.data;

msgBlob.arrayBuffer().then((res) => {
  const resp: PubSubResp = PubSubResp.deserialize(new Uint8Array(res));
  const respobj = resp.toObject();

  console.log(respobj);
});
```

所以, 使用这一套方案最好的使用方式就是在建立 websocket 连接时, 先发送一个 `ListOffset({timestamp=-1})` 的请求获取指定 topic 的指定 partition 下最新的一条消息的 _offset_, 接着以该 _offset_ 为起点开始发送 `Fetch({offset})` 去取得消息数组, 将 _offset_ 设置为数组最后一条记录的 offset 并在下一次 `Fetch()` 时作为参数传递. 需要注意的是:

- ⚠️ 需要控制好发送`Fetch`的频率, 如果在某一时刻进入该 topic 的消息非常多, 可能会出现 websocket 关闭的异常情况
- ❌ 如果想要获取从指定的某个 offset 开始的消息, 这一套方案并不是适用！！

最后, 将该套方案封装成一个工具类`ApisixpullkafkaWebSocket`如下.

```typescript
import {
  CmdKafkaFetch,
  CmdKafkaListOffset,
  PubSubReq,
  PubSubResp,
} from "./pubsub";

export interface StructuredKafkaMessage {
  offset: number;
  timestamp: number;
  value: string;
}

export interface apisixpullkafkaWsParam {
  url: string;
  topic: string;
  partition: number;
  notify: (msgs: StructuredKafkaMessage[]) => void;
}

export default class ApisixpullkafkaWebSocket {
  // ws state: 0(CONNECTING), 1(OPEN), 2(CLOSING), 3(CLOSED)
  socket!: WebSocket;
  url: string;
  offset: number;
  topic: string;
  partition: number;
  notify!: (msgs: StructuredKafkaMessage[]) => void;

  constructor(param: apisixpullkafkaWsParam) {
    this.url = param.url;
    this.offset = -1;
    this.notify = param.notify;
    this.topic = param.topic;
    this.partition = param.partition;

    const wsurl = `ws://${this.url}`;
    const websocket2 = new WebSocket(wsurl);
    this.socket = websocket2;
    this.socket.onopen = this.onopen;
    this.socket.onmessage = this.onmessage;
    this.socket.onclose = this.onclose;
    this.socket.onerror = this.onerror;
  }

  onopen = () => {
    console.log("[first] send kafka_list_offset req, timestamp is ", -1);
    this.sendKafkaListOffsetReq(-1);
  };

  unitoNotify = (msgs: StructuredKafkaMessage[], preoffset: number) => {
    const newMsgs = msgs.filter((msg) => {
      return msg.offset > preoffset;
    });

    if (this.notify === undefined || newMsgs.length === 0) {
      return;
    }

    console.log(
      "notify new message, offset is",
      newMsgs.map((msg) => msg.offset)
    );
    this.notify(newMsgs);
  };

  onmessage = (ev: MessageEvent) => {
    const msgBlob: Blob = ev.data;

    msgBlob.arrayBuffer().then((res) => {
      const resp: PubSubResp = PubSubResp.deserialize(new Uint8Array(res));
      const respobj = resp.toObject();

      console.log("receive new message", respobj);

      if (respobj.kafka_list_offset_resp !== undefined) {
        // first
        this.offset = respobj.kafka_list_offset_resp.offset || -1;
        this.offset = this.offset - 10;
        console.log("kafka_list_offset_resp, offset is", this.offset);
        if (this.offset !== -1) {
          console.log("[first] send kafka_fetch req, offset is", this.offset);
          this.sendKafkaFetchReq(this.offset);
        }
      } else if (respobj.kafka_fetch_resp !== undefined) {
        // subsequent
        const msgs = respobj.kafka_fetch_resp.messages || [];
        console.log("kafka_fetch_resp, msgs is", msgs);
        const remsgs = this.uint8tostr(msgs);
        console.log("after uint8tostr, msg is", remsgs);

        if (remsgs.length > 0) {
          this.unitoNotify(remsgs, this.offset);
          const latestOffset = remsgs[remsgs.length - 1].offset || -1;
          this.offset = latestOffset;
        }

        // 5s 轮询
        setTimeout(() => {
          console.log(
            "[subsequent] send kafka_fetch req, offset is",
            this.offset
          );
          this.sendKafkaFetchReq(this.offset);
        }, 5000);
      } else {
        console.log(respobj);
      }
    });
  };

  // -1: latest msg; -2: first msg; unix timestamp
  sendKafkaListOffsetReq = (timestamp: number) => {
    const kloseq = 101;
    const kloreq = new PubSubReq({
      sequence: kloseq,
      cmd_kafka_list_offset: new CmdKafkaListOffset({
        topic: this.topic,
        partition: this.partition,
        timestamp: timestamp,
      }),
    });
    this.socket.send(kloreq.serialize());
  };

  sendKafkaFetchReq = (offset: number) => {
    const kfseq = 102;
    const kfreq = new PubSubReq({
      sequence: kfseq,
      cmd_kafka_fetch: new CmdKafkaFetch({
        topic: this.topic,
        partition: this.partition,
        offset: offset,
      }),
    });
    this.socket.send(kfreq.serialize());
  };

  // unit8array -> string
  uint8tostr = (msgs: any[]) => {
    const remsgs: StructuredKafkaMessage[] = [];

    for (let i = 0; i < msgs.length; i++) {
      const msg = msgs[i];
      const offset = msg.offset;
      const timestamp = msg.timestamp;
      const decoder = new TextDecoder();
      const value = decoder.decode(msg.value);

      const item: StructuredKafkaMessage = {
        offset: offset,
        timestamp: timestamp,
        value: value,
      };

      remsgs.push(item);
    }

    return remsgs;
  };

  onerror = (e: Event) => {
    console.log("websocket connect error:", e);
  };

  onclose = (e: Event) => {
    console.log("websocket connect close:", e);
  };
}
```

该工具类使用方式以及运行结果如下所示. 每次有新数据过来时, 就会触发调用 `notify()` 方法, 并将新数据作为参数传入, 只需要接受即可.

```typescript
const [ldata, setLData] = useState<any[]>([]);
const [series, setSeries] = useState<any[]>([]);

useEffect(() => {
  const wsCfg = {
    url: "eqw-test.eam.com/kafka/websocket",
    notify: (msgs: any[]) => setLData([...msgs]),
    topic: "rms_order",
    partition: 0,
  };
  new ApisixpullkafkaWebSocket(wsCfg);
}, []);
```

## 附录

### communication protocol

```js
syntax = "proto3";

option java_package = "org.apache.apisix.api.pubsub";
option java_outer_classname = "PubSubProto";
option java_multiple_files = true;
option go_package = "github.com/apache/apisix/api/pubsub;pubsub";

/**
 * Ping command, used to keep the websocket connection alive
 *
 * The state field is used to pass some non-specific information,
 * which will be returned in the pong response as is.
 */
message CmdPing {
    bytes state = 1;
}

/**
 * An empty command, a placeholder for testing purposes only
 */
message CmdEmpty {}

/**
 * Get the offset of the specified topic partition from Apache Kafka.
 */
message CmdKafkaListOffset {
    string topic = 1;
    int32 partition = 2;
    int64 timestamp = 3;
}

/**
 * Fetch messages of the specified topic partition from Apache Kafka.
 */
message CmdKafkaFetch {
    string topic = 1;
    int32 partition = 2;
    int64 offset = 3;
}

/**
 * Client request definition for pubsub scenarios
 *
 * The sequence field is used to associate requests and responses.
 * Apache APISIX will set a consistent sequence for the associated
 * requests and responses, and the client can explicitly know the
 * response corresponding to any of the requests.
 *
 * The req field is the command data sent by the client, and its
 * type will be chosen from any of the lists in the definition.
 *
 * Field numbers 1 to 30 in the definition are used to define basic
 * information and future extensions, and numbers after 30 are used
 * to define commands.
 */
message PubSubReq {
    int64 sequence = 1;
    oneof req {
        CmdEmpty           cmd_empty             = 31;
        CmdPing            cmd_ping              = 32;
        CmdKafkaFetch      cmd_kafka_fetch       = 33;
        CmdKafkaListOffset cmd_kafka_list_offset = 34;
    };
}

/**
 * The response body of the service when an error occurs,
 * containing the error code and the error message.
 */
message ErrorResp {
    int32 code = 1;
    string message = 2;
}

/**
 * Pong response, the state field will pass through the
 * value in the Ping command field.
 */
message PongResp {
    bytes state = 1;
}

/**
 * The definition of a message in Kafka with the current message
 * offset, production timestamp, Key, and message content.
 */
message KafkaMessage {
    int64 offset = 1;
    int64 timestamp = 2;
    bytes key = 3;
    bytes value = 4;
}

/**
 * The response of Fetch messages from Apache Kafka.
 */
message KafkaFetchResp {
    repeated KafkaMessage messages = 1;
}

/**
 * The response of list offset from Apache Kafka.
 */
message KafkaListOffsetResp {
    int64 offset = 1;
}

/**
 * Server response definition for pubsub scenarios
 *
 * The sequence field will be the same as the value in the
 * request, which is used to associate the associated request
 * and response.
 *
 * The resp field is the response data sent by the server, and
 * its type will be chosen from any of the lists in the definition.
 */
message PubSubResp {
    int64 sequence = 1;
    oneof resp {
        ErrorResp           error_resp             = 31;
        PongResp            pong_resp              = 32;
        KafkaFetchResp      kafka_fetch_resp       = 33;
        KafkaListOffsetResp kafka_list_offset_resp = 34;
    };
}
```

### kafka-logger

一个用来推送消息到 kafka 的插件

- kafka-logger: https://apisix.apache.org/zh/docs/apisix/plugins/kafka-logger/
- APISIX 集成 Kafka 实现高效实时日志监控: https://apisix.apache.org/zh/blog/2022/01/17/apisix-kafka-integration/
