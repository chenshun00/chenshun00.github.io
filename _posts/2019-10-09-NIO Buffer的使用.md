## NIO Buffer的使用

日常笔记

### Buffer#Capacity,Position 和 Limit

Buffer本质上一个能读写的 `内存块` ,在Java中被包装成一个 `NIO Buffer` 对象。Buffer的3个属性

* capacity
* position
* limit

分别对应buffer的容量，读写的位置,能够读写的最大值.

>写模式下, position代表当前写入的位置,limit代表写入的最大量,一般等于capacity

>读模式下，position代表当前读取的位置，limit代表能够读取的最大量,相当于写模式下position的位置。

借用一幅图表示

![](http://tutorials.jenkov.com/images/java-nio/buffers-modes.png)

**Buffer capacity, position and limit in write and read mode.**

>PS:容量的大小是确定的,在容量满时需要清空才能再次写入 , `Netty` 提供了可扩容的 `Buffer`.

### filp,slice,和duplicate

* filp: 英文翻译为反转
* slice: 切片
* duplicate: 复制

**filp** 用于切换 `读写模式` .  创建**Buffer**时是写入模式，读取时需要切换才能进行读取. 切换时主要变更 Buffer的position和limit.

```java
public class Test {
    public static void main(String[] args) {
        ByteBuffer allocate = ByteBuffer.allocate(16);
        allocate.putInt(1);
        allocate.flip();
        System.out.println(allocate.getInt());

    }
}
```

**slice** 创建一个Buffer , 共享 `父Buffer` 的内容, 在 `slice` 中的修改会直接反馈在 `父Buffer中`, Position,Limit,capacity 相互独立. 

> **主要在不切换父Buffer时使用** , 在 `RocketMq` 中出现该使用方式

```java
public class SliceTest {
    public static void main(String[] args) {
        ByteBuffer allocate = ByteBuffer.allocate(16);
        //切片1
        ByteBuffer slice = allocate.slice();
        slice.position(0);
        slice.putInt(1);
        //切片2
        ByteBuffer two = allocate.slice();
        two.position(4);
        two.putInt(2);
        System.out.println(allocate.getInt());//输出1
        System.out.println(allocate.getInt());//输出2
    }
}
```

创建的ByteBuffer没有直接使用，而是通过共享的slice去进行修改，slice的修改会直接反馈在allocate的内容当中，省去了 `filp` . 避免了忘记 `filp` 的错误.

**duplicate** 复制一个Buffer ，所有的数据和父Buffer保持一致,3个属性值项目独立.

```java
public class DuplicateTest {
    public static void main(String[] args) {
        ByteBuffer allocate = ByteBuffer.allocate(16);
        allocate.putInt(1);
        ByteBuffer duplicate = allocate.duplicate();//复制buffer
        allocate.putInt(2);
        duplicate.flip();
        System.out.println(duplicate.getInt());//输出1
        System.out.println(duplicate.getInt());//输出2
    }
}
```

### 读写示例

通过 `slice` 写入，不需要通过 `filp` ，后读取数据. 限于篇幅贴了部分代码 ,代码地址 [MessageWrite.java](https://gitlab.com/chenshun00/test/blob/master/src/main/java/top/huzhurong/gateway/test/rpc/MessageWrite.java)

```java
public class MessageWrite {
    public static void main(String[] args) throws Exception {
        ByteBuffer map = ByteBuffer.allocate(1024);
        AtomicInteger at = new AtomicInteger(0);
        ByteBuffer byteBuffer = map.slice();
        byteBuffer.position(at.get());
        Message message = new Message();
        message.setBody("我为人人,人人为我".getBytes());
        message.setTopic("chenshun00");
        message.setProperties(new HashMap<String, String>() {{
           this.put("11", "11");
           this.put("22", "22");
           this.put("33", "33");
        }});
        at.set(writeMessage(message, byteBuffer));

        Message read = read(map);
        System.out.println(read.id);
        System.out.println(read.getProperties().toString());
        System.out.println(read.getTopic());
        System.out.println(new String(read.getBody()));
    }

    public static int writeMessage(Message message, ByteBuffer byteBuffer) {
         return 0; 
    }
    public static Message read(ByteBuffer byteBuffer) {
       return null; 
    }
    private static String toStr(Map<String, String> xx) {
        return JSONObject.toJSONString(xx);
    }
    private static Map<String, String> toMap(String json) {
        return JSON.parseObject(json, new TypeReference<HashMap<String, String>>() {
        });
    }

}
```

#### 参考

[Java NIO Tutorial](http://tutorials.jenkov.com/java-nio/index.html)