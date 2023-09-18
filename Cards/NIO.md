<a name="nXV3v"></a>
# 一、三大组件简介
Channel与Buffer
Java NIO系统的**核心**在于：**通道(Channel)和缓冲区(Buffer)**。通道表示打开到 IO 设备(例如：文件、套接字)的连接。若需要使用 NIO 系统，需要获取用于**连接 IO 设备的通道**以及用于**容纳数据的缓冲区**。然后操作缓冲区，对数据进行处理<br />简而言之，**通道负责传输，缓冲区负责存储**
**常见的Channel有以下四种**，其中FileChannel主要用于文件传输，其余三种用于网络通信

- FileChannel
- DatagramChannel
- SocketChannel
- ServerSocketChannel

**Buffer有以下几种**，其中使用较多的是ByteBuffer

- **ByteBuffer**
   - MappedByteBuffer
   - DirectByteBuffer
   - HeapByteBuffer
- ShortBuffer
- IntBuffer
- LongBuffer
- FloatBuffer
- DoubleBuffer
- CharBuffer

[![](https://cdn.nlark.com/yuque/0/2023/png/22271853/1683682707986-864ffebf-221f-4056-920c-363650cf0e21.png#averageHue=%23313130&clientId=ua1ecfc51-6760-4&from=paste&id=u5b3a4ba1&originHeight=414&originWidth=562&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=uebb6b86b-9477-4b81-bf2d-fbec1785f2d&title=)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20210412135510.png)
<a name="bFFHS"></a>
## 1、Selector
在使用Selector之前，处理socket连接还有以下两种方法<br />**使用多线程技术**<br />为每个连接分别开辟一个线程，分别去处理对应的socke连接<br />[![](https://cdn.nlark.com/yuque/0/2023/png/22271853/1683682707966-909816e4-8cf8-4e8b-82f1-982edca0f06a.png#averageHue=%23fcfcda&clientId=ua1ecfc51-6760-4&from=paste&id=ud8875236&originHeight=249&originWidth=584&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=ue41df3a2-5ec0-45d0-9e92-9e17e386f69&title=)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20210418181918.png)<br />这种方法存在以下几个问题

- 内存占用高
- 每个线程都需要占用一定的内存，当连接较多时，会开辟大量线程，导致占用大量内存
- 线程上下文切换成本高
- 只适合连接数少的场景
- 连接数过多，会导致创建很多线程，从而出现问题

**使用线程池技术**<br />使用线程池，让线程池中的线程去处理连接<br />[![](https://cdn.nlark.com/yuque/0/2023/png/22271853/1683682707962-273afae8-af6b-443e-ae6b-9bbf6fd57cc9.png#averageHue=%23fcfcda&clientId=ua1ecfc51-6760-4&from=paste&id=uf4689016&originHeight=246&originWidth=771&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=u12a6a2fe-22eb-45ed-ba55-b8e0fb460ea&title=)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20210418181933.png)<br />这种方法存在以下几个问题

- 阻塞模式下，线程仅能处理一个连接
   - 线程池中的线程获取任务（task）后，**只有当其执行完任务之后（断开连接后），才会去获取并执行下一个任务**
   - 若socke连接一直未断开，则其对应的线程无法处理其他socke连接
- 仅适合**短连接**场景
   - 短连接即建立连接发送请求并响应后就立即断开，使得线程池中的线程可以快速处理其他连接

**使用选择器**<br />**selector 的作用就是配合一个线程来管理多个 channel（fileChannel因为是阻塞式的，所以无法使用selector）**，获取这些 channel 上发生的**事件**，这些 channel 工作在**非阻塞模式**下，当一个channel中没有执行任务时，可以去执行其他channel中的任务。**适合连接数多，但流量较少的场景**<br />[![](https://cdn.nlark.com/yuque/0/2023/png/22271853/1683682707952-14d86951-6264-4286-b970-841e278b7f0f.png#averageHue=%23fdfddb&clientId=ua1ecfc51-6760-4&from=paste&id=u4f6ec5ba&originHeight=356&originWidth=592&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=uac26ab0f-fcc8-4acc-9241-6bd4031b035&title=)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20210418181947.png)<br />若事件未就绪，调用 selector 的 select() 方法会阻塞线程，直到 channel 发生了就绪事件。这些事件就绪后，select 方法就会返回这些事件交给 thread 来处理
<a name="GY8Zs"></a>
## 2、ByteBuffer
<a name="w7EDn"></a>
### 使用案例
<a name="s8A5e"></a>
#### 使用方式

- 向 buffer 写入数据，例如调用 channel.read(buffer)
- 调用 flip() 切换至**读模式**
   - **flip会使得buffer中的limit变为position，position变为0**
- 从 buffer 读取数据，例如调用 buffer.get()
- 调用 clear() 或者compact()切换至**写模式**
   - 调用clear()方法时**position=0，limit变为capacity**
   - 调用compact()方法时，**会将缓冲区中的未读数据压缩到缓冲区前面**
- 重复以上步骤

**使用ByteBuffer读取文件中的内容**
```java
public class TestByteBuffer {
    public static void main(String[] args) {
        // 获得FileChannel
        try (FileChannel channel = new FileInputStream("stu.txt").getChannel()) {
            // 获得缓冲区
            ByteBuffer buffer = ByteBuffer.allocate(10);
            int hasNext = 0;
            StringBuilder builder = new StringBuilder();
            while((hasNext = channel.read(buffer)) > 0) {
                // 切换模式 limit=position, position=0
                buffer.flip();
                // 当buffer中还有数据时，获取其中的数据
                while(buffer.hasRemaining()) {
                    builder.append((char)buffer.get());
                }
                // 切换模式 position=0, limit=capacity
                buffer.clear();
            }
            System.out.println(builder.toString());
        } catch (IOException e) {
        }
    }
}
```
打印结果<br />0123456789abcdefCopy
<a name="OpLw3"></a>
### 核心属性
字节缓冲区的父类Buffer中有几个核心属性，如下
```java
// Invariants: mark <= position <= limit <= capacity
private int mark = -1;
private int position = 0;
private int limit;
private int capacity;
```

- **capacity**：缓冲区的容量。通过构造函数赋予，一旦设置，无法更改
- **limit**：缓冲区的界限。位于limit 后的数据不可读写。缓冲区的限制不能为负，并且**不能大于其容量**
- **position**：**下一个**读写位置的索引（类似PC）。缓冲区的位置不能为负，并且**不能大于limit**
- **mark**：记录当前position的值。**position被改变后，可以通过调用reset() 方法恢复到mark的位置。**

以上四个属性必须满足以下要求<br />**mark <= position <= limit <= capacity**
<a name="Ik9YZ"></a>
### 核心方法
<a name="eLPSD"></a>
#### put()方法

- put()方法可以将一个数据放入到缓冲区中。
- 进行该操作后，postition的值会+1，指向下一个可以放入的位置。capacity = limit ，为缓冲区容量的值。

[![](https://cdn.nlark.com/yuque/0/2023/png/22271853/1683682707956-1f7d2885-b20a-4728-9149-058023d5b939.png#averageHue=%23f7f3f3&clientId=ua1ecfc51-6760-4&from=paste&id=u5894c6d5&originHeight=258&originWidth=858&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=u4a22f3a0-5458-47fd-b05f-eb6d4098606&title=)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201109145709.png)
<a name="SgmsP"></a>
#### flip()方法

- flip()方法会**切换对缓冲区的操作模式**，由写->读 / 读->写
- 进行该操作后
   - 如果是写模式->读模式，position = 0 ， limit 指向最后一个元素的下一个位置，capacity不变
   - 如果是读->写，则恢复为put()方法中的值

[![](https://cdn.nlark.com/yuque/0/2023/png/22271853/1683682708442-27c1c6f4-c139-412b-a694-9cd4ed10827b.png#averageHue=%23f6f1f1&clientId=ua1ecfc51-6760-4&from=paste&id=u5a3fbe68&originHeight=221&originWidth=883&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=u3c161855-1911-4091-bc10-2b805046e9e&title=)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201109145753.png)
<a name="So82e"></a>
#### get()方法

- get()方法会读取缓冲区中的一个值
- 进行该操作后，position会+1，如果超过了limit则会抛出异常
- **注意：get(i)方法不会改变position的值**

[![](https://cdn.nlark.com/yuque/0/2023/png/22271853/1683682708459-05c8620d-6588-414c-9e2e-c9eb9a5a40a8.png#averageHue=%23f5f0f0&clientId=ua1ecfc51-6760-4&from=paste&id=u37e2e12b&originHeight=197&originWidth=853&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=uf1c8fa58-62fc-4a02-bef7-ca480a5013a&title=)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201109145822.png)
<a name="JdX4w"></a>
#### rewind()方法

- 该方法**只能在读模式下使用**
- rewind()方法后，会恢复position、limit和capacity的值，变为进行get()前的值

[![](https://cdn.nlark.com/yuque/0/2023/png/22271853/1683682708521-e2bcabd6-a8ed-49e9-8ed0-de2b7d5b4d44.png#averageHue=%23f6f2f2&clientId=ua1ecfc51-6760-4&from=paste&id=uae0da6ff&originHeight=239&originWidth=871&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=u6b15bb59-bab8-415c-9d98-d9434ca2494&title=)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201109145852.png)
<a name="c10ye"></a>
#### clean()方法

- clean()方法会将缓冲区中的各个属性恢复为最初的状态，position = 0, limit = capacity
- **此时缓冲区的数据依然存在**，处于“被遗忘”状态，下次进行写操作时会覆盖这些数据

[![](https://cdn.nlark.com/yuque/0/2023/png/22271853/1683682708548-7804d19e-a584-484e-9a07-a2e9f92f6556.png#averageHue=%23f8eeee&clientId=ua1ecfc51-6760-4&from=paste&height=280&id=u5328a354&originHeight=280&originWidth=889&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=u962d61e3-422c-4492-9c9f-d289cf57623&title=&width=889)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201109145905.png)
<a name="ijrr9"></a>
#### mark()和reset()方法

- mark()方法会将postion的值保存到mark属性中
- reset()方法会将position的值改为mark中保存的值
<a name="E9LF2"></a>
#### compact()方法
**此方法为ByteBuffer的方法，而不是Buffer的方法**

- compact会把未读完的数据向前压缩，然后切换到写模式
- 数据前移后，原位置的值并未清零，写时会**覆盖**之前的值

[![](https://cdn.nlark.com/yuque/0/2023/png/22271853/1683682708626-9ee983e5-b703-42b4-b505-3e102c3a31d5.png#averageHue=%23f7f3f3&clientId=ua1ecfc51-6760-4&from=paste&id=u70bda894&originHeight=295&originWidth=1225&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=uaa3a5091-4468-4049-b670-46a79c24cb1&title=)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20210412155726.png)
<a name="FURb4"></a>
#### clear() VS compact()
clear只是对position、limit、mark进行重置，而compact在对position进行设置，以及limit、mark进行重置的同时，还涉及到数据在内存中拷贝（会调用arraycopy）。**所以compact比clear更耗性能。**但compact能保存你未读取的数据，将新数据追加到为读取的数据之后；而clear则不行，若你调用了clear，则未读取的数据就无法再读取到了<br />**所以需要根据情况来判断使用哪种方法进行模式切换**
<a name="psx3I"></a>
### 方法调用及演示
<a name="atW3N"></a>
#### ByteBuffer调试工具类
需要先导入netty依赖
```xml
<dependency>
  <groupId>io.netty</groupId>
  <artifactId>netty-all</artifactId>
  <version>4.1.51.Final</version>
</dependency>Copy
```
```java
import java.nio.ByteBuffer;

import io.netty.util.internal.MathUtil;
import io.netty.util.internal.StringUtil;
import io.netty.util.internal.MathUtil.*;


/**
* @author Panwen Chen
* @date 2021/4/12 15:59
*/
public class ByteBufferUtil {
    private static final char[] BYTE2CHAR = new char[256];
    private static final char[] HEXDUMP_TABLE = new char[256 * 4];
    private static final String[] HEXPADDING = new String[16];
    private static final String[] HEXDUMP_ROWPREFIXES = new String[65536 >>> 4];
    private static final String[] BYTE2HEX = new String[256];
    private static final String[] BYTEPADDING = new String[16];

    static {
        final char[] DIGITS = "0123456789abcdef".toCharArray();
        for (int i = 0; i < 256; i++) {
            HEXDUMP_TABLE[i << 1] = DIGITS[i >>> 4 & 0x0F];
            HEXDUMP_TABLE[(i << 1) + 1] = DIGITS[i & 0x0F];
        }

        int i;

        // Generate the lookup table for hex dump paddings
        for (i = 0; i < HEXPADDING.length; i++) {
            int padding = HEXPADDING.length - i;
            StringBuilder buf = new StringBuilder(padding * 3);
            for (int j = 0; j < padding; j++) {
                buf.append("   ");
            }
            HEXPADDING[i] = buf.toString();
        }

        // Generate the lookup table for the start-offset header in each row (up to 64KiB).
        for (i = 0; i < HEXDUMP_ROWPREFIXES.length; i++) {
            StringBuilder buf = new StringBuilder(12);
            buf.append(StringUtil.NEWLINE);
            buf.append(Long.toHexString(i << 4 & 0xFFFFFFFFL | 0x100000000L));
            buf.setCharAt(buf.length() - 9, '|');
            buf.append('|');
            HEXDUMP_ROWPREFIXES[i] = buf.toString();
        }

        // Generate the lookup table for byte-to-hex-dump conversion
        for (i = 0; i < BYTE2HEX.length; i++) {
            BYTE2HEX[i] = ' ' + StringUtil.byteToHexStringPadded(i);
        }

        // Generate the lookup table for byte dump paddings
        for (i = 0; i < BYTEPADDING.length; i++) {
            int padding = BYTEPADDING.length - i;
            StringBuilder buf = new StringBuilder(padding);
            for (int j = 0; j < padding; j++) {
                buf.append(' ');
            }
            BYTEPADDING[i] = buf.toString();
        }

        // Generate the lookup table for byte-to-char conversion
        for (i = 0; i < BYTE2CHAR.length; i++) {
            if (i <= 0x1f || i >= 0x7f) {
                BYTE2CHAR[i] = '.';
            } else {
                BYTE2CHAR[i] = (char) i;
            }
        }
    }

    /**
* 打印所有内容
* @param buffer
*/
    public static void debugAll(ByteBuffer buffer) {
        int oldlimit = buffer.limit();
        buffer.limit(buffer.capacity());
        StringBuilder origin = new StringBuilder(256);
        appendPrettyHexDump(origin, buffer, 0, buffer.capacity());
        System.out.println("+--------+-------------------- all ------------------------+----------------+");
                           System.out.printf("position: [%d], limit: [%d]\n", buffer.position(), oldlimit);
                           System.out.println(origin);
                           buffer.limit(oldlimit);
                           }

                           /**
                           * 打印可读取内容
                           * @param buffer
                           */
                           public static void debugRead(ByteBuffer buffer) {
                           StringBuilder builder = new StringBuilder(256);
                           appendPrettyHexDump(builder, buffer, buffer.position(), buffer.limit() - buffer.position());
                           System.out.println("+--------+-------------------- read -----------------------+----------------+");
                           System.out.printf("position: [%d], limit: [%d]\n", buffer.position(), buffer.limit());
                           System.out.println(builder);
                           }

                           private static void appendPrettyHexDump(StringBuilder dump, ByteBuffer buf, int offset, int length) {
                           if (MathUtil.isOutOfBounds(offset, length, buf.capacity())) {
                           throw new IndexOutOfBoundsException(
                           "expected: " + "0 <= offset(" + offset + ") <= offset + length(" + length
                           + ") <= " + "buf.capacity(" + buf.capacity() + ')');
                           }
                           if (length == 0) {
                           return;
                           }
                           dump.append(
                           "         +-------------------------------------------------+" +
                           StringUtil.NEWLINE + "         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |" +
                           StringUtil.NEWLINE + "+--------+-------------------------------------------------+----------------+");

                           final int startIndex = offset;
                           final int fullRows = length >>> 4;
                           final int remainder = length & 0xF;

                           // Dump the rows which have 16 bytes.
                           for (int row = 0; row < fullRows; row++) {
                           int rowStartIndex = (row << 4) + startIndex;

                           // Per-row prefix.
                           appendHexDumpRowPrefix(dump, row, rowStartIndex);

                           // Hex dump
                           int rowEndIndex = rowStartIndex + 16;
                           for (int j = rowStartIndex; j < rowEndIndex; j++) {
                           dump.append(BYTE2HEX[getUnsignedByte(buf, j)]);
                           }
                           dump.append(" |");

                           // ASCII dump
                           for (int j = rowStartIndex; j < rowEndIndex; j++) {
                           dump.append(BYTE2CHAR[getUnsignedByte(buf, j)]);
                           }
                           dump.append('|');
                           }

                           // Dump the last row which has less than 16 bytes.
                           if (remainder != 0) {
                           int rowStartIndex = (fullRows << 4) + startIndex;
                           appendHexDumpRowPrefix(dump, fullRows, rowStartIndex);

                           // Hex dump
                           int rowEndIndex = rowStartIndex + remainder;
                           for (int j = rowStartIndex; j < rowEndIndex; j++) {
                           dump.append(BYTE2HEX[getUnsignedByte(buf, j)]);
                           }
                           dump.append(HEXPADDING[remainder]);
                           dump.append(" |");

                           // Ascii dump
                           for (int j = rowStartIndex; j < rowEndIndex; j++) {
                           dump.append(BYTE2CHAR[getUnsignedByte(buf, j)]);
                           }
                           dump.append(BYTEPADDING[remainder]);
                           dump.append('|');
                           }

                           dump.append(StringUtil.NEWLINE +
                           "+--------+-------------------------------------------------+----------------+");
                           }

                           private static void appendHexDumpRowPrefix(StringBuilder dump, int row, int rowStartIndex) {
                           if (row < HEXDUMP_ROWPREFIXES.length) {
                           dump.append(HEXDUMP_ROWPREFIXES[row]);
                           } else {
                           dump.append(StringUtil.NEWLINE);
                           dump.append(Long.toHexString(rowStartIndex & 0xFFFFFFFFL | 0x100000000L));
                           dump.setCharAt(dump.length() - 9, '|');
                           dump.append('|');
                           }
                           }

                           public static short getUnsignedByte(ByteBuffer buffer, int index) {
                           return (short) (buffer.get(index) & 0xFF);
                           }
                           }
```
<a name="fiW5d"></a>
#### 调用ByteBuffer的方法
```java
public class TestByteBuffer {
    public static void main(String[] args) {
        ByteBuffer buffer = ByteBuffer.allocate(10);
        // 向buffer中写入1个字节的数据
        buffer.put((byte)97);
        // 使用工具类，查看buffer状态
        ByteBufferUtil.debugAll(buffer);

        // 向buffer中写入4个字节的数据
        buffer.put(new byte[]{98, 99, 100, 101});
        ByteBufferUtil.debugAll(buffer);

        // 获取数据
        buffer.flip();
        ByteBufferUtil.debugAll(buffer);
        System.out.println(buffer.get());
        System.out.println(buffer.get());
        ByteBufferUtil.debugAll(buffer);

        // 使用compact切换模式
        buffer.compact();
        ByteBufferUtil.debugAll(buffer);

        // 再次写入
        buffer.put((byte)102);
        buffer.put((byte)103);
        ByteBufferUtil.debugAll(buffer);
    }
}
```
运行结果
```java
// 向缓冲区写入了一个字节的数据，此时postition为1
+--------+-------------------- all ------------------------+----------------+
    position: [1], limit: [10]
    +-------------------------------------------------+
    |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
    +--------+-------------------------------------------------+----------------+
    |00000000| 61 00 00 00 00 00 00 00 00 00                   |a.........      |
    +--------+-------------------------------------------------+----------------+

    // 向缓冲区写入四个字节的数据，此时position为5
    +--------+-------------------- all ------------------------+----------------+
    position: [5], limit: [10]
    +-------------------------------------------------+
    |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
    +--------+-------------------------------------------------+----------------+
    |00000000| 61 62 63 64 65 00 00 00 00 00                   |abcde.....      |
    +--------+-------------------------------------------------+----------------+

    // 调用flip切换模式，此时position为0，表示从第0个数据开始读取
    +--------+-------------------- all ------------------------+----------------+
    position: [0], limit: [5]
    +-------------------------------------------------+
    |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
    +--------+-------------------------------------------------+----------------+
    |00000000| 61 62 63 64 65 00 00 00 00 00                   |abcde.....      |
    +--------+-------------------------------------------------+----------------+
    // 读取两个字节的数据             
    97
    98

        // position变为2             
        +--------+-------------------- all ------------------------+----------------+
        position: [2], limit: [5]
        +-------------------------------------------------+
        |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
        +--------+-------------------------------------------------+----------------+
        |00000000| 61 62 63 64 65 00 00 00 00 00                   |abcde.....      |
        +--------+-------------------------------------------------+----------------+

        // 调用compact切换模式，此时position及其后面的数据被压缩到ByteBuffer前面去了
        // 此时position为3，会覆盖之前的数据             
        +--------+-------------------- all ------------------------+----------------+
        position: [3], limit: [10]
        +-------------------------------------------------+
        |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
        +--------+-------------------------------------------------+----------------+
        |00000000| 63 64 65 64 65 00 00 00 00 00                   |cdede.....      |
        +--------+-------------------------------------------------+----------------+

        // 再次写入两个字节的数据，之前的 0x64 0x65 被覆盖         
        +--------+-------------------- all ------------------------+----------------+
        position: [5], limit: [10]
        +-------------------------------------------------+
        |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
        +--------+-------------------------------------------------+----------------+
        |00000000| 63 64 65 66 67 00 00 00 00 00                   |cdefg.....      |
        +--------+-------------------------------------------------+----------------+
```
<a name="fOf1L"></a>
### **字符串与ByteBuffer的相互转换**
<a name="X78tA"></a>
#### 方法一
**编码**：字符串调用getByte方法获得byte数组，将byte数组放入ByteBuffer中<br />**解码**：**先调用ByteBuffer的flip方法，然后通过StandardCharsets的decoder方法解码**
```java
public class Translate {
    public static void main(String[] args) {
        // 准备两个字符串
        String str1 = "hello";
        String str2 = "";


        ByteBuffer buffer1 = ByteBuffer.allocate(16);
        // 通过字符串的getByte方法获得字节数组，放入缓冲区中
        buffer1.put(str1.getBytes());
        ByteBufferUtil.debugAll(buffer1);

        // 将缓冲区中的数据转化为字符串
        // 切换模式
        buffer1.flip();

        // 通过StandardCharsets解码，获得CharBuffer，再通过toString获得字符串
        str2 = StandardCharsets.UTF_8.decode(buffer1).toString();
        System.out.println(str2);
        ByteBufferUtil.debugAll(buffer1);
    }
}
```
运行结果
```java
+--------+-------------------- all ------------------------+----------------+
    position: [5], limit: [16]
    +-------------------------------------------------+
    |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
    +--------+-------------------------------------------------+----------------+
    |00000000| 68 65 6c 6c 6f 00 00 00 00 00 00 00 00 00 00 00 |hello...........|
    +--------+-------------------------------------------------+----------------+
    hello
    +--------+-------------------- all ------------------------+----------------+
    position: [5], limit: [5]
    +-------------------------------------------------+
    |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
    +--------+-------------------------------------------------+----------------+
    |00000000| 68 65 6c 6c 6f 00 00 00 00 00 00 00 00 00 00 00 |hello...........|
    +--------+-------------------------------------------------+----------------+
```
<a name="nScy3"></a>
#### 方法二
**编码**：通过StandardCharsets的encode方法获得ByteBuffer，此时获得的ByteBuffer为读模式，无需通过flip切换模式<br />**解码**：通过StandardCharsets的decoder方法解码
```java
public class Translate {
    public static void main(String[] args) {
        // 准备两个字符串
        String str1 = "hello";
        String str2 = "";

        // 通过StandardCharsets的encode方法获得ByteBuffer
        // 此时获得的ByteBuffer为读模式，无需通过flip切换模式
        ByteBuffer buffer1 = StandardCharsets.UTF_8.encode(str1);
        ByteBufferUtil.debugAll(buffer1);

        // 将缓冲区中的数据转化为字符串
        // 通过StandardCharsets解码，获得CharBuffer，再通过toString获得字符串
        str2 = StandardCharsets.UTF_8.decode(buffer1).toString();
        System.out.println(str2);
        ByteBufferUtil.debugAll(buffer1);
    }
}
```
运行结果
```java
+--------+-------------------- all ------------------------+----------------+
    position: [0], limit: [5]
    +-------------------------------------------------+
    |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
    +--------+-------------------------------------------------+----------------+
    |00000000| 68 65 6c 6c 6f                                  |hello           |
    +--------+-------------------------------------------------+----------------+
    hello
    +--------+-------------------- all ------------------------+----------------+
    position: [5], limit: [5]
    +-------------------------------------------------+
    |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
    +--------+-------------------------------------------------+----------------+
    |00000000| 68 65 6c 6c 6f                                  |hello           |
    +--------+-------------------------------------------------+----------------+
```
<a name="rBkvE"></a>
#### **方法三**
**编码**：字符串调用getByte()方法获得字节数组，将字节数组传给**ByteBuffer的wrap()方法**，通过该方法获得ByteBuffer。**同样无需调用flip方法切换为读模式**<br />**解码**：通过StandardCharsets的decoder方法解码
```java
public class Translate {
    public static void main(String[] args) {
        // 准备两个字符串
        String str1 = "hello";
        String str2 = "";

        // 通过StandardCharsets的encode方法获得ByteBuffer
        // 此时获得的ByteBuffer为读模式，无需通过flip切换模式
        ByteBuffer buffer1 = ByteBuffer.wrap(str1.getBytes());
        ByteBufferUtil.debugAll(buffer1);

        // 将缓冲区中的数据转化为字符串
        // 通过StandardCharsets解码，获得CharBuffer，再通过toString获得字符串
        str2 = StandardCharsets.UTF_8.decode(buffer1).toString();
        System.out.println(str2);
        ByteBufferUtil.debugAll(buffer1);
    }
}
```
运行结果
```java
+--------+-------------------- all ------------------------+----------------+
    position: [0], limit: [5]
    +-------------------------------------------------+
    |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
    +--------+-------------------------------------------------+----------------+
    |00000000| 68 65 6c 6c 6f                                  |hello           |
    +--------+-------------------------------------------------+----------------+
    hello
    +--------+-------------------- all ------------------------+----------------+
    position: [5], limit: [5]
    +-------------------------------------------------+
    |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
    +--------+-------------------------------------------------+----------------+
    |00000000| 68 65 6c 6c 6f                                  |hello           |
    +--------+-------------------------------------------------+----------------+
```
<a name="Cg2wK"></a>
### 粘包与半包
<a name="KTldW"></a>
#### 现象
网络上有多条数据发送给服务端，数据之间使用 \n 进行分隔<br />但由于某种原因这些数据在接收时，被进行了重新组合，例如原始数据有3条为

- Hello,world\n
- I’m Nyima\n
- How are you?\n

变成了下面的两个 byteBuffer (粘包，半包)

- Hello,world\nI’m Nyima\nHo
- w are you?\n
<a name="v6fTE"></a>
#### 出现原因
**粘包**<br />发送方在发送数据时，并不是一条一条地发送数据，而是**将数据整合在一起**，当数据达到一定的数量后再一起发送。这就会导致多条信息被放在一个缓冲区中被一起发送出去<br />**半包**<br />接收方的缓冲区的大小是有限的，当接收方的缓冲区满了以后，就需要**将信息截断**，等缓冲区空了以后再继续放入数据。这就会发生一段完整的数据最后被截断的现象
<a name="Cqjm8"></a>
#### 解决办法

- 通过get(index)方法遍历ByteBuffer，遇到分隔符时进行处理。**注意**：get(index)不会改变position的值
   - 记录该段数据长度，以便于申请对应大小的缓冲区
   - 将缓冲区的数据通过get()方法写入到target中
- 调用**compact方法**切换模式，因为缓冲区中可能还有未读的数据
```java
public class ByteBufferDemo {
    public static void main(String[] args) {
        ByteBuffer buffer = ByteBuffer.allocate(32);
        // 模拟粘包+半包
        buffer.put("Hello,world\nI'm Nyima\nHo".getBytes());
        // 调用split函数处理
        split(buffer);
        buffer.put("w are you?\n".getBytes());
        split(buffer);
    }

    private static void split(ByteBuffer buffer) {
        // 切换为读模式
        buffer.flip();
        for(int i = 0; i < buffer.limit(); i++) {

            // 遍历寻找分隔符
            // get(i)不会移动position
            if (buffer.get(i) == '\n') {
                // 缓冲区长度
                int length = i+1-buffer.position();
                ByteBuffer target = ByteBuffer.allocate(length);
                // 将前面的内容写入target缓冲区
                for(int j = 0; j < length; j++) {
                    // 将buffer中的数据写入target中
                    target.put(buffer.get());
                }
                // 打印查看结果
                ByteBufferUtil.debugAll(target);
            }
        }
        // 切换为写模式，但是缓冲区可能未读完，这里需要使用compact
        buffer.compact();
    }
}
```
运行结果
```java
+--------+-------------------- all ------------------------+----------------+
    position: [12], limit: [12]
    +-------------------------------------------------+
    |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
    +--------+-------------------------------------------------+----------------+
    |00000000| 48 65 6c 6c 6f 2c 77 6f 72 6c 64 0a             |Hello,world.    |
    +--------+-------------------------------------------------+----------------+
    +--------+-------------------- all ------------------------+----------------+
    position: [10], limit: [10]
    +-------------------------------------------------+
    |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
    +--------+-------------------------------------------------+----------------+
    |00000000| 49 27 6d 20 4e 79 69 6d 61 0a                   |I'm Nyima.      |
    +--------+-------------------------------------------------+----------------+
    +--------+-------------------- all ------------------------+----------------+
    position: [13], limit: [13]
    +-------------------------------------------------+
    |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
    +--------+-------------------------------------------------+----------------+
    |00000000| 48 6f 77 20 61 72 65 20 79 6f 75 3f 0a          |How are you?.   |
    +--------+-------------------------------------------------+----------------+
```
<a name="i6AnL"></a>
# 二、文件编程
<a name="aLFSc"></a>
## 1、FileChannel
<a name="C0MPg"></a>
### 工作模式
FileChannel**只能在阻塞模式下工作**，所以无法搭配Selector
<a name="Ujsh8"></a>
### 获取
不能直接打开 FileChannel，**必须**通过 FileInputStream、FileOutputStream 或者 RandomAccessFile 来获取 FileChannel，它们都有 getChannel 方法

- 通过 FileInputStream 获取的 channel **只能读**
- 通过 FileOutputStream 获取的 channel **只能写**
- 通过 RandomAccessFile 是否能读写**根据构造 RandomAccessFile 时的读写模式决定**
<a name="soSkN"></a>
### 读取
通过 FileInputStream 获取channel，通过read方法将数据写入到ByteBuffer中<br />read方法的返回值表示读到了多少字节，若读到了文件末尾则返回-1<br />int readBytes = channel.read(buffer);Copy<br />**可根据返回值判断是否读取完毕**
```java
while(channel.read(buffer) > 0) {
    // 进行对应操作
    ...
}
```
<a name="KKvi6"></a>
### 写入
因为channel也是有大小的，所以 write 方法并不能保证一次将 buffer 中的内容全部写入 channel。必须**需要按照以下规则进行写入**
```java
// 通过hasRemaining()方法查看缓冲区中是否还有数据未写入到通道中
while(buffer.hasRemaining()) {
    channel.write(buffer);
}
```
<a name="Ir063"></a>
### 关闭
通道需要close，一般情况通过try-with-resource进行关闭，**最好使用以下方法获取strea以及channel，避免某些原因使得资源未被关闭**
```java
public class TestChannel {
    public static void main(String[] args) throws IOException {
        try (FileInputStream fis = new FileInputStream("stu.txt");
             FileOutputStream fos = new FileOutputStream("student.txt");
             FileChannel inputChannel = fis.getChannel();
             FileChannel outputChannel = fos.getChannel()) {

            // 执行对应操作
            ...

        }
    }
}
```
<a name="wP99y"></a>
### 位置
**position**<br />channel也拥有一个保存读取数据位置的属性，即position<br />long pos = channel.position();Copy<br />可以通过position(int pos)设置channel中position的值
```java
long newPos = ...;
channel.position(newPos);
```
设置当前位置时，如果设置为文件的末尾

- 这时读取会返回 -1
- 这时写入，会追加内容，但要注意如果 position 超过了文件末尾，再写入时在新内容和原末尾之间会有空洞（00）
<a name="Snapm"></a>
### 强制写入
操作系统出于性能的考虑，会将数据缓存，不是立刻写入磁盘，而是等到缓存满了以后将所有数据一次性的写入磁盘。可以调用 **force(true)** 方法将文件内容和元数据（文件的权限等信息）立刻写入磁盘
<a name="RHJiJ"></a>
## 2、两个Channel传输数据
<a name="Fv0uc"></a>
### transferTo方法
使用transferTo方法可以快速、高效地将一个channel中的数据传输到另一个channel中，但**一次只能传输2G的内容**<br />transferTo底层使用了零拷贝技术
```java
public class TestChannel {
    public static void main(String[] args){
        try (FileInputStream fis = new FileInputStream("stu.txt");
             FileOutputStream fos = new FileOutputStream("student.txt");
             FileChannel inputChannel = fis.getChannel();
             FileChannel outputChannel = fos.getChannel()) {
            // 参数：inputChannel的起始位置，传输数据的大小，目的channel
            // 返回值为传输的数据的字节数
            // transferTo一次只能传输2G的数据
            inputChannel.transferTo(0, inputChannel.size(), outputChannel);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
当传输的文件**大于2G**时，需要使用以下方法进行多次传输
```java
public class TestChannel {
    public static void main(String[] args){
        try (FileInputStream fis = new FileInputStream("stu.txt");
             FileOutputStream fos = new FileOutputStream("student.txt");
             FileChannel inputChannel = fis.getChannel();
             FileChannel outputChannel = fos.getChannel()) {
            long size = inputChannel.size();
            long capacity = inputChannel.size();
            // 分多次传输
            while (capacity > 0) {
                // transferTo返回值为传输了的字节数
                capacity -= inputChannel.transferTo(size-capacity, capacity, outputChannel);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
<a name="dd7O8"></a>
## 3、Path与Paths

- Path 用来表示文件路径
- Paths 是工具类，用来获取 Path 实例
```java
Path source = Paths.get("1.txt"); // 相对路径 不带盘符 使用 user.dir 环境变量来定位 1.txt

Path source = Paths.get("d:\\1.txt"); // 绝对路径 代表了  d:\1.txt 反斜杠需要转义

Path source = Paths.get("d:/1.txt"); // 绝对路径 同样代表了  d:\1.txt

Path projects = Paths.get("d:\\data", "projects"); // 代表了  d:\data\projects
```

- . 代表了当前路径
- .. 代表了上一级路径

例如目录结构如下
```java
d:
|- data
    |- projects
    |- a
    |- b
```
代码
```java
Path path = Paths.get("d:\\data\\projects\\a\\..\\b");
System.out.println(path);
System.out.println(path.normalize()); // 正常化路径 会去除 . 以及 ..
```
输出结果为
```java
d:\data\projects\a\..\b
d:\data\projects\b
```
<a name="cEmIb"></a>
## 4、Files
<a name="kRreG"></a>
### 查找
检查文件是否存在
```java
Path path = Paths.get("helloword/data.txt");
System.out.println(Files.exists(path));
```
<a name="gr0Me"></a>
### 创建
创建**一级目录**
```java
Path path = Paths.get("helloword/d1");
Files.createDirectory(path);
```

- 如果目录已存在，会抛异常 FileAlreadyExistsException
- 不能一次创建多级目录，否则会抛异常 NoSuchFileException

创建**多级目录用**
```java
Path path = Paths.get("helloword/d1/d2");
Files.createDirectories(path);
```
<a name="hHFpL"></a>
### 拷贝及移动
**拷贝文件**
```java
Path source = Paths.get("helloword/data.txt");
Path target = Paths.get("helloword/target.txt");

Files.copy(source, target);
```

- 如果文件已存在，会抛异常 FileAlreadyExistsException

如果希望用 source **覆盖**掉 target，需要用 StandardCopyOption 来控制<br />Files.copy(source, target, StandardCopyOption.REPLACE_EXISTING);Copy<br />移动文件
```java
Path source = Paths.get("helloword/data.txt");
Path target = Paths.get("helloword/data.txt");

Files.move(source, target, StandardCopyOption.ATOMIC_MOVE);
```

- **StandardCopyOption.ATOMIC_MOVE 保证文件移动的原子性**
<a name="jUvBu"></a>
### 删除
删除文件
```java
Path target = Paths.get("helloword/target.txt");

Files.delete(target);
```

- 如果文件不存在，会抛异常 NoSuchFileException

删除目录
```java
Path target = Paths.get("helloword/d1");

Files.delete(target);
```

- 如果**目录还有内容**，会抛异常 DirectoryNotEmptyException
<a name="YJGIo"></a>
### 遍历
可以**使用Files工具类中的walkFileTree(Path, FileVisitor)方法**，其中需要传入两个参数

- Path：文件起始路径
- FileVisitor：文件访问器，**使用访问者模式**
   - 接口的实现类**SimpleFileVisitor**有四个方法
      - preVisitDirectory：访问目录前的操作
      - visitFile：访问文件的操作
      - visitFileFailed：访问文件失败时的操作
      - postVisitDirectory：访问目录后的操作
```java
public class TestWalkFileTree {
    public static void main(String[] args) throws IOException {
        Path path = Paths.get("F:\\JDK 8");
        // 文件目录数目
        AtomicInteger dirCount = new AtomicInteger();
        // 文件数目
        AtomicInteger fileCount = new AtomicInteger();
        Files.walkFileTree(path, new SimpleFileVisitor<Path>(){
            @Override
            public FileVisitResult preVisitDirectory(Path dir, BasicFileAttributes attrs) throws IOException {
                System.out.println("===>"+dir);
                // 增加文件目录数
                dirCount.incrementAndGet();
                return super.preVisitDirectory(dir, attrs);
            }

            @Override
            public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) throws IOException {
                System.out.println(file);
                // 增加文件数
                fileCount.incrementAndGet();
                return super.visitFile(file, attrs);
            }
        });
        // 打印数目
        System.out.println("文件目录数:"+dirCount.get());
        System.out.println("文件数:"+fileCount.get());
    }
}
```
运行结果如下
```java
...
===>F:\JDK 8\lib\security\policy\unlimited
F:\JDK 8\lib\security\policy\unlimited\local_policy.jar
F:\JDK 8\lib\security\policy\unlimited\US_export_policy.jar
F:\JDK 8\lib\security\trusted.libraries
F:\JDK 8\lib\sound.properties
F:\JDK 8\lib\tzdb.dat
F:\JDK 8\lib\tzmappings
F:\JDK 8\LICENSE
F:\JDK 8\README.txt
F:\JDK 8\release
F:\JDK 8\THIRDPARTYLICENSEREADME-JAVAFX.txt
F:\JDK 8\THIRDPARTYLICENSEREADME.txt
F:\JDK 8\Welcome.html
文件目录数:23
文件数:279
```
<a name="c9Bmk"></a>
# 三、网络编程
<a name="ZPsYW"></a>
## 1、阻塞

- 阻塞模式下，相关方法都会导致线程暂停
   - ServerSocketChannel.accept 会在**没有连接建立时**让线程暂停
   - SocketChannel.read 会在**通道中没有数据可读时**让线程暂停
   - 阻塞的表现其实就是线程暂停了，暂停期间不会占用 cpu，但线程相当于闲置
- 单线程下，阻塞方法之间相互影响，几乎不能正常工作，需要多线程支持
- 但多线程下，有新的问题，体现在以下方面
   - 32 位 jvm 一个线程 320k，64 位 jvm 一个线程 1024k，如果连接数过多，必然导致 OOM，并且线程太多，反而会因为频繁上下文切换导致性能降低
   - 可以采用线程池技术来减少线程数和线程上下文切换，但治标不治本，如果有很多连接建立，但长时间 inactive，会阻塞线程池中所有线程，因此不适合长连接，只适合短连接

**服务端代码**
```java
public class Server {
    public static void main(String[] args) {
        // 创建缓冲区
        ByteBuffer buffer = ByteBuffer.allocate(16);
        // 获得服务器通道
        try(ServerSocketChannel server = ServerSocketChannel.open()) {
            // 为服务器通道绑定端口
            server.bind(new InetSocketAddress(8080));
            // 用户存放连接的集合
            ArrayList<SocketChannel> channels = new ArrayList<>();
            // 循环接收连接
            while (true) {
                System.out.println("before connecting...");
                // 没有连接时，会阻塞线程
                SocketChannel socketChannel = server.accept();
                System.out.println("after connecting...");
                channels.add(socketChannel);
                // 循环遍历集合中的连接
                for(SocketChannel channel : channels) {
                    System.out.println("before reading");
                    // 处理通道中的数据
                    // 当通道中没有数据可读时，会阻塞线程
                    channel.read(buffer);
                    buffer.flip();
                    ByteBufferUtil.debugRead(buffer);
                    buffer.clear();
                    System.out.println("after reading");
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
客户端代码
```java
public class Client {
    public static void main(String[] args) {
        try (SocketChannel socketChannel = SocketChannel.open()) {
            // 建立连接
            socketChannel.connect(new InetSocketAddress("localhost", 8080));
            System.out.println("waiting...");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
运行结果

- 客户端-服务器建立连接前：服务器端因accept阻塞

[![](https://cdn.nlark.com/yuque/0/2023/png/22271853/1683682708938-3b9d83dc-b52a-4bdb-b4c6-5d770bd2b098.png#averageHue=%238b7a66&clientId=ua1ecfc51-6760-4&from=paste&id=ucfb37912&originHeight=126&originWidth=410&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=uf9f0945b-68c4-4904-9a0e-9f68e64ee98&title=)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20210413213318.png)

- 客户端-服务器建立连接后，客户端发送消息前：服务器端因通道为空被阻塞

[![](https://cdn.nlark.com/yuque/0/2023/png/22271853/1683682708944-e2abdd9b-5d27-4a31-bb24-91bf0d5f384c.png#averageHue=%23323131&clientId=ua1ecfc51-6760-4&from=paste&id=uc4a551cd&originHeight=188&originWidth=424&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=u3080a102-eec4-4d66-9928-518fe396e62&title=)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20210413213446.png)

- 客户端发送数据后，服务器处理通道中的数据。再次进入循环时，再次被accept阻塞

[![](https://cdn.nlark.com/yuque/0/2023/png/22271853/1683682708973-953981fd-e385-4b1f-a6d1-595facac1b48.png#averageHue=%232d2c2c&clientId=ua1ecfc51-6760-4&from=paste&id=ue8377337&originHeight=368&originWidth=969&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=u5c3c3669-2926-4750-a303-3a9eda07e7e&title=)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20210413214109.png)

- 之前的客户端再次发送消息**，服务器端因为被accept阻塞**，无法处理之前客户端发送到通道中的信息

[![](https://cdn.nlark.com/yuque/0/2023/png/22271853/1683682709132-f9cd8b3e-0bf5-4831-b7a0-35fa5256053c.png#averageHue=%237d9676&clientId=ua1ecfc51-6760-4&from=paste&id=u583926d4&originHeight=184&originWidth=1215&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=u61e52d38-e22d-4f65-aab8-2f5f64ef392&title=)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20210413214505.png)
<a name="R1D4a"></a>
## 2、非阻塞

- 可以通过ServerSocketChannel的configureBlocking(**false**)方法将**获得连接设置为非阻塞的**。此时若没有连接，accept会返回null
- 可以通过SocketChannel的configureBlocking(**false**)方法将从通道中**读取数据设置为非阻塞的**。若此时通道中没有数据可读，read会返回-1

服务器代码如下
```java
public class Server {
    public static void main(String[] args) {
        // 创建缓冲区
        ByteBuffer buffer = ByteBuffer.allocate(16);
        // 获得服务器通道
        try(ServerSocketChannel server = ServerSocketChannel.open()) {
            // 为服务器通道绑定端口
            server.bind(new InetSocketAddress(8080));
            // 用户存放连接的集合
            ArrayList<SocketChannel> channels = new ArrayList<>();
            // 循环接收连接
            while (true) {
                // 设置为非阻塞模式，没有连接时返回null，不会阻塞线程
                server.configureBlocking(false);
                SocketChannel socketChannel = server.accept();
                // 通道不为空时才将连接放入到集合中
                if (socketChannel != null) {
                    System.out.println("after connecting...");
                    channels.add(socketChannel);
                }
                // 循环遍历集合中的连接
                for(SocketChannel channel : channels) {
                    // 处理通道中的数据
                    // 设置为非阻塞模式，若通道中没有数据，会返回0，不会阻塞线程
                    channel.configureBlocking(false);
                    int read = channel.read(buffer);
                    if(read > 0) {
                        buffer.flip();
                        ByteBufferUtil.debugRead(buffer);
                        buffer.clear();
                        System.out.println("after reading");
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
这样写存在一个问题，因为设置为了非阻塞，会一直执行while(true)中的代码，CPU一直处于忙碌状态，会使得性能变低，所以实际情况中不使用这种方法处理请求
<a name="EpZsC"></a>
## 3、Selector
<a name="YrOfi"></a>
### 多路复用
单线程可以配合 Selector 完成对多个 Channel 可读写事件的监控，这称之为多路复用

- **多路复用仅针对网络 IO**，普通文件 IO **无法**利用多路复用
- 如果不用 Selector 的非阻塞模式，线程大部分时间都在做无用功，而 Selector 能够保证
   - 有可连接事件时才去连接
   - 有可读事件才去读取
   - 有可写事件才去写入
      - 限于网络传输能力，Channel 未必时时可写，一旦 Channel 可写，会触发 Selector 的可写事件
<a name="TDqAO"></a>
### 创建
```java
Selector selector = Selector.open();
```
<a name="ztgbB"></a>
### 绑定 Channel 事件
也称之为注册事件，绑定的事件 selector 才会关心
```java
channel.configureBlocking(false);
SelectionKey key = channel.register(selector, 绑定事件);
```

- channel 必须工作在非阻塞模式
- FileChannel 没有非阻塞模式，因此不能配合 selector 一起使用
- 绑定的事件类型可以有
   - connect - 客户端连接成功时触发
   - accept - 服务器端成功接受连接时触发
   - read - 数据可读入时触发，有因为接收能力弱，数据暂不能读入的情况
   - write - 数据可写出时触发，有因为发送能力弱，数据暂不能写出的情况
<a name="VdeKl"></a>
### 监听 Channel 事件
可以通过下面三种方法来监听是否有事件发生，方法的返回值代表有多少 channel 发生了事件<br />方法1，阻塞直到绑定事件发生<br />`int count = selector.select();`<br />方法2，阻塞直到绑定事件发生，或是超时（时间单位为 ms）<br />`int count = selector.select(long timeout);`<br />方法3，不会阻塞，也就是不管有没有事件，立刻返回，自己根据返回值检查是否有事件<br />`int count = selector.selectNow();`
<a name="on8Fw"></a>
### 💡 select 何时不阻塞

- 事件发生时
   - 客户端发起连接请求，会触发 accept 事件
   - 客户端发送数据过来，客户端正常、异常关闭时，都会触发 read 事件，另外如果发送的数据大于 buffer 缓冲区，会触发多次读取事件
   - channel 可写，会触发 write 事件
   - 在 linux 下 nio bug 发生时
- 调用 selector.wakeup()
- 调用 selector.close()
- selector 所在线程 interrupt
<a name="c8xcu"></a>
## 4、使用及Accpet事件
要使用Selector实现多路复用，服务端代码如下改进
```java
public class SelectServer {
    public static void main(String[] args) {
        ByteBuffer buffer = ByteBuffer.allocate(16);
        // 获得服务器通道
        try(ServerSocketChannel server = ServerSocketChannel.open()) {
            server.bind(new InetSocketAddress(8080));
            // 创建选择器
            Selector selector = Selector.open();

            // 通道必须设置为非阻塞模式
            server.configureBlocking(false);
            // 将通道注册到选择器中，并设置感兴趣的事件
            server.register(selector, SelectionKey.OP_ACCEPT);
            while (true) {
                // 若没有事件就绪，线程会被阻塞，反之不会被阻塞。从而避免了CPU空转
                // 返回值为就绪的事件个数
                int ready = selector.select();
                System.out.println("selector ready counts : " + ready);

                // 获取所有事件
                Set<SelectionKey> selectionKeys = selector.selectedKeys();

                // 使用迭代器遍历事件
                Iterator<SelectionKey> iterator = selectionKeys.iterator();
                while (iterator.hasNext()) {
                    SelectionKey key = iterator.next();

                    // 判断key的类型
                    if(key.isAcceptable()) {
                        // 获得key对应的channel
                        ServerSocketChannel channel = (ServerSocketChannel) key.channel();
                        System.out.println("before accepting...");

                        // 获取连接并处理，而且是必须处理，否则需要取消
                        SocketChannel socketChannel = channel.accept();
                        System.out.println("after accepting...");

                        // 处理完毕后移除
                        iterator.remove();
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
**步骤解析**

- 获得选择器Selector

Selector selector = Selector.open();Copy

- 将**通道设置为非阻塞模式**，并注册到选择器中，并设置感兴趣的事件
   - channel 必须工作在非阻塞模式
   - FileChannel 没有非阻塞模式，因此不能配合 selector 一起使用
   - 绑定的**事件类型**可以有
      - connect - 客户端连接成功时触发
      - accept - 服务器端成功接受连接时触发
      - read - 数据可读入时触发，有因为接收能力弱，数据暂不能读入的情况
      - write - 数据可写出时触发，有因为发送能力弱，数据暂不能写出的情况
```java
// 通道必须设置为非阻塞模式
server.configureBlocking(false);
// 将通道注册到选择器中，并设置感兴趣的实践
server.register(selector, SelectionKey.OP_ACCEPT);
```

- 通过Selector监听事件，并获得就绪的通道个数，若没有通道就绪，线程会被阻塞
   - 阻塞直到绑定事件发生int count = selector.select();Copy
   - 阻塞直到绑定事件发生，**或是超时**（时间单位为 ms）int count = selector.select(long timeout);Copy
   - **不会阻塞**，也就是不管有没有事件，立刻返回，自己根据返回值检查是否有事件int count = selector.selectNow();Copy
- 获取就绪事件并**得到对应的通道**，然后进行处理
```java
// 获取所有事件
Set<SelectionKey> selectionKeys = selector.selectedKeys();

// 使用迭代器遍历事件
Iterator<SelectionKey> iterator = selectionKeys.iterator();

while (iterator.hasNext()) {
    SelectionKey key = iterator.next();

    // 判断key的类型，此处为Accept类型
    if(key.isAcceptable()) {
        // 获得key对应的channel
        ServerSocketChannel channel = (ServerSocketChannel) key.channel();

        // 获取连接并处理，而且是必须处理，否则需要取消
        SocketChannel socketChannel = channel.accept();

        // 处理完毕后移除
        iterator.remove();
    }
}
```
**事件发生后能否不处理**<br />事件发生后，**要么处理，要么取消（cancel）**，不能什么都不做，**否则下次该事件仍会触发**，这是因为 nio 底层使用的是水平触发
<a name="shpUG"></a>
## 5、Read事件

- 在Accept事件中，若有客户端与服务器端建立了连接，**需要将其对应的SocketChannel设置为非阻塞，并注册到选择其中**
- 添加Read事件，触发后进行读取操作
```java
public class SelectServer {
    public static void main(String[] args) {
        ByteBuffer buffer = ByteBuffer.allocate(16);
        // 获得服务器通道
        try(ServerSocketChannel server = ServerSocketChannel.open()) {
            server.bind(new InetSocketAddress(8080));
            // 创建选择器
            Selector selector = Selector.open();
            // 通道必须设置为非阻塞模式
            server.configureBlocking(false);
            // 将通道注册到选择器中，并设置感兴趣的实践
            server.register(selector, SelectionKey.OP_ACCEPT);
            // 为serverKey设置感兴趣的事件
            while (true) {
                // 若没有事件就绪，线程会被阻塞，反之不会被阻塞。从而避免了CPU空转
                // 返回值为就绪的事件个数
                int ready = selector.select();
                System.out.println("selector ready counts : " + ready);
                // 获取所有事件
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                // 使用迭代器遍历事件
                Iterator<SelectionKey> iterator = selectionKeys.iterator();
                while (iterator.hasNext()) {
                    SelectionKey key = iterator.next();
                    // 判断key的类型
                    if(key.isAcceptable()) {
                        // 获得key对应的channel
                        ServerSocketChannel channel = (ServerSocketChannel) key.channel();
                        System.out.println("before accepting...");
                        // 获取连接
                        SocketChannel socketChannel = channel.accept();
                        System.out.println("after accepting...");
                        // 设置为非阻塞模式，同时将连接的通道也注册到选择其中
                        socketChannel.configureBlocking(false);
                        socketChannel.register(selector, SelectionKey.OP_READ);
                        // 处理完毕后移除
                        iterator.remove();
                    } else if (key.isReadable()) {
                        SocketChannel channel = (SocketChannel) key.channel();
                        System.out.println("before reading...");
                        channel.read(buffer);
                        System.out.println("after reading...");
                        buffer.flip();
                        ByteBufferUtil.debugRead(buffer);
                        buffer.clear();
                        // 处理完毕后移除
                        iterator.remove();
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
**删除事件**<br />**当处理完一个事件后，一定要调用迭代器的remove方法移除对应事件，否则会出现错误**。原因如下<br />以我们上面的 **Read事件** 的代码为例

- 当调用了 server.register(selector, SelectionKey.OP_ACCEPT)后，Selector中维护了一个集合，**用于存放SelectionKey以及其对应的通道**
```java
// WindowsSelectorImpl 中的 SelectionKeyImpl数组
private SelectionKeyImpl[] channelArray = new SelectionKeyImpl[8];
```
[![](https://cdn.nlark.com/yuque/0/2023/png/22271853/1683682709181-e9518a98-3adf-46c3-9cdd-964e6c151aa3.png#averageHue=%23cadbed&clientId=ua1ecfc51-6760-4&from=paste&id=uc9fd1418&originHeight=226&originWidth=290&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=ua6a9ac19-cb34-427c-8259-c6dee68cf2e&title=)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20210414192429.png)

- 当**选择器中的通道对应的事件发生后**，selecionKey会被放到另一个集合中，但是**selecionKey不会自动移除**，所以需要我们在处理完一个事件后，通过迭代器手动移除其中的selecionKey。否则会导致已被处理过的事件再次被处理，就会引发错误[![](https://cdn.nlark.com/yuque/0/2023/png/22271853/1683682709384-c450666b-5715-4977-afa1-c8c3b58cbfa5.png#averageHue=%23cdddc5&clientId=ua1ecfc51-6760-4&from=paste&id=u394b7ab8&originHeight=240&originWidth=802&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=u46b610b2-4324-4c8f-831b-cc127fe1c5d&title=)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20210414193143.png)
<a name="p4gj1"></a>
### 断开处理
当客户端与服务器之间的连接**断开时，会给服务器端发送一个读事件**，对异常断开和正常断开需要加以不同的方式进行处理

- **正常断开**
   - 正常断开时，服务器端的channel.read(buffer)方法的返回值为-1，**所以当结束到返回值为-1时，需要调用key的cancel方法取消此事件，并在取消后移除该事件**
```java
int read = channel.read(buffer);
// 断开连接时，客户端会向服务器发送一个写事件，此时read的返回值为-1
if(read == -1) {
    // 取消该事件的处理
    key.cancel();
    channel.close();
} else {
    ...
}
// 取消或者处理，都需要移除key
iterator.remove();
```

- 异常断开
   - 异常断开时，会抛出IOException异常， 在try-catch的**catch块中捕获异常并调用key的cancel方法即可**
<a name="bKo9A"></a>
### 消息边界
**不处理消息边界存在的问题**<br />将缓冲区的大小设置为4个字节，发送2个汉字（你好），通过decode解码并打印时，会出现乱码
```java
ByteBuffer buffer = ByteBuffer.allocate(4);
// 解码并打印
System.out.println(StandardCharsets.UTF_8.decode(buffer));
```
```
你�
��
```
这是因为UTF-8字符集下，1个汉字占用3个字节，此时缓冲区大小为4个字节，**一次读时间无法处理完通道中的所有数据，所以一共会触发两次读事件**。这就导致 你好 的 好 字被拆分为了前半部分和后半部分发送，解码时就会出现问题<br />**处理消息边界**<br />传输的文本可能有以下三种情况

- 文本大于缓冲区大小
   - 此时需要将缓冲区进行扩容
- 发生半包现象
- 发生粘包现象

[![](https://cdn.nlark.com/yuque/0/2023/png/22271853/1683682709577-43eeaf1e-1a27-4772-bd0a-a616a0264974.png#averageHue=%23ececcb&clientId=ua1ecfc51-6760-4&from=paste&id=uee592612&originHeight=262&originWidth=358&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=u523335cd-f058-4fb7-a8f4-932eda123c9&title=)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20210415103442.png)<br />解决思路大致有以下三种

- **固定消息长度**，数据包大小一样，服务器按预定长度读取，当发送的数据较少时，需要将数据进行填充，直到长度与消息规定长度一致。缺点是浪费带宽
- 另一种思路是按分隔符拆分，缺点是效率低，需要一个一个字符地去匹配分隔符
- **TLV 格式，即 Type 类型、Length 长度、Value 数据**（也就是在消息开头**用一些空间存放后面数据的长度**），如HTTP请求头中的Content-Type与**Content-Length**。类型和长度已知的情况下，就可以方便获取消息大小，分配合适的 buffer，缺点是 buffer 需要提前分配，如果内容过大，则影响 server 吞吐量
   - Http 1.1 是 TLV 格式
   - Http 2.0 是 LTV 格式[![](https://cdn.nlark.com/yuque/0/2023/png/22271853/1683682709571-9db13f38-01f1-47bc-b96f-ffd2949f8d7c.png#averageHue=%23e4e3de&clientId=ua1ecfc51-6760-4&from=paste&id=u8399bd7f&originHeight=408&originWidth=771&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=u5c45c6aa-70a0-412e-8185-022da7423dc&title=)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20210415103926.png)

下文的消息边界处理方式为**第二种：按分隔符拆分**<br />**附件与扩容**<br />Channel的register方法还有**第三个参数**：附件，可以向其中放入一个Object类型的对象，该对象会与登记的Channel以及其对应的SelectionKey绑定，可以从SelectionKey获取到对应通道的附件<br />public final SelectionKey register(Selector sel, int ops, Object att)Copy<br />可通过SelectionKey的**attachment()方法获得附件**<br />ByteBuffer buffer = (ByteBuffer) key.attachment();Copy<br />我们需要在Accept事件发生后，将通道注册到Selector中时，**对每个通道添加一个ByteBuffer附件**，让每个通道发生读事件时都使用自己的通道，避免与其他通道发生冲突而导致问题
```
// 设置为非阻塞模式，同时将连接的通道也注册到选择其中，同时设置附件
socketChannel.configureBlocking(false);
ByteBuffer buffer = ByteBuffer.allocate(16);
// 添加通道对应的Buffer附件
socketChannel.register(selector, SelectionKey.OP_READ, buffer);
```
当Channel中的数据大于缓冲区时，需要对缓冲区进行**扩容**操作。此代码中的扩容的判定方法：**Channel调用compact方法后，的position与limit相等，说明缓冲区中的数据并未被读取（容量太小），此时创建新的缓冲区，其大小扩大为两倍。同时还要将旧缓冲区中的数据拷贝到新的缓冲区中，同时调用SelectionKey的attach方法将新的缓冲区作为新的附件放入SelectionKey中**
```java
// 如果缓冲区太小，就进行扩容
if (buffer.position() == buffer.limit()) {
    ByteBuffer newBuffer = ByteBuffer.allocate(buffer.capacity()*2);
    // 将旧buffer中的内容放入新的buffer中
    ewBuffer.put(buffer);
    // 将新buffer作为附件放到key中
    key.attach(newBuffer);
}
```
**改造后的服务器代码如下**
```java
public class SelectServer {
    public static void main(String[] args) {
        // 获得服务器通道
        try(ServerSocketChannel server = ServerSocketChannel.open()) {
            server.bind(new InetSocketAddress(8080));
            // 创建选择器
            Selector selector = Selector.open();
            // 通道必须设置为非阻塞模式
            server.configureBlocking(false);
            // 将通道注册到选择器中，并设置感兴趣的事件
            server.register(selector, SelectionKey.OP_ACCEPT);
            // 为serverKey设置感兴趣的事件
            while (true) {
                // 若没有事件就绪，线程会被阻塞，反之不会被阻塞。从而避免了CPU空转
                // 返回值为就绪的事件个数
                int ready = selector.select();
                System.out.println("selector ready counts : " + ready);
                // 获取所有事件
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                // 使用迭代器遍历事件
                Iterator<SelectionKey> iterator = selectionKeys.iterator();
                while (iterator.hasNext()) {
                    SelectionKey key = iterator.next();
                    // 判断key的类型
                    if(key.isAcceptable()) {
                        // 获得key对应的channel
                        ServerSocketChannel channel = (ServerSocketChannel) key.channel();
                        System.out.println("before accepting...");
                        // 获取连接
                        SocketChannel socketChannel = channel.accept();
                        System.out.println("after accepting...");
                        // 设置为非阻塞模式，同时将连接的通道也注册到选择其中，同时设置附件
                        socketChannel.configureBlocking(false);
                        ByteBuffer buffer = ByteBuffer.allocate(16);
                        socketChannel.register(selector, SelectionKey.OP_READ, buffer);
                        // 处理完毕后移除
                        iterator.remove();
                    } else if (key.isReadable()) {
                        SocketChannel channel = (SocketChannel) key.channel();
                        System.out.println("before reading...");
                        // 通过key获得附件（buffer）
                        ByteBuffer buffer = (ByteBuffer) key.attachment();
                        int read = channel.read(buffer);
                        if(read == -1) {
                            key.cancel();
                            channel.close();
                        } else {
                            // 通过分隔符来分隔buffer中的数据
                            split(buffer);
                            // 如果缓冲区太小，就进行扩容
                            if (buffer.position() == buffer.limit()) {
                                ByteBuffer newBuffer = ByteBuffer.allocate(buffer.capacity()*2);
                                // 将旧buffer中的内容放入新的buffer中
                                buffer.flip();
                                newBuffer.put(buffer);
                                // 将新buffer放到key中作为附件
                                key.attach(newBuffer);
                            }
                        }
                        System.out.println("after reading...");
                        // 处理完毕后移除
                        iterator.remove();
                    }
                    }
                    }
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                    }

                        private static void split(ByteBuffer buffer) {
                        buffer.flip();
                        for(int i = 0; i < buffer.limit(); i++) {
                        // 遍历寻找分隔符
                        // get(i)不会移动position
                        if (buffer.get(i) == '\n') {
                        // 缓冲区长度
                        int length = i+1-buffer.position();
                        ByteBuffer target = ByteBuffer.allocate(length);
                        // 将前面的内容写入target缓冲区
                        for(int j = 0; j < length; j++) {
                        // 将buffer中的数据写入target中
                        target.put(buffer.get());
                    }
                        // 打印结果
                        ByteBufferUtil.debugAll(target);
                    }
                    }
                        // 切换为写模式，但是缓冲区可能未读完，这里需要使用compact
                        buffer.compact();
                    }
                    }
```
<a name="pEkSB"></a>
### ByteBuffer的大小分配

- 每个 channel 都需要记录可能被切分的消息，因为 **ByteBuffer 不能被多个 channel 共同使用**，因此需要为每个 channel 维护一个独立的 ByteBuffer
- ByteBuffer 不能太大，比如一个 ByteBuffer 1Mb 的话，要支持百万连接就要 1Tb 内存，因此需要设计大小可变的 ByteBuffer
- 分配思路可以参考
   - 一种思路是首先分配一个较小的 buffer，例如 4k，如果发现数据不够，再分配 8k 的 buffer，将 4k buffer 内容拷贝至 8k buffer，优点是消息连续容易处理，缺点是数据拷贝耗费性能
      - 参考实现 [http://tutorials.jenkov.com/java-performance/resizable-array.html](http://tutorials.jenkov.com/java-performance/resizable-array.html)
   - 另一种思路是用多个数组组成 buffer，一个数组不够，把多出来的内容写入新的数组，与前面的区别是消息存储不连续解析复杂，优点是避免了拷贝引起的性能损耗
<a name="oTxMd"></a>
## 6、Write事件
服务器通过Buffer向通道中写入数据时，**可能因为通道容量小于Buffer中的数据大小，导致无法一次性将Buffer中的数据全部写入到Channel中，这时便需要分多次写入**，具体步骤如下

- 执行一次写操作，向将buffer中的内容写入到SocketChannel中，然后判断Buffer中是否还有数据
- 若Buffer中还有数据，则**需要将SockerChannel注册到Seletor中，并关注写事件，同时将未写完的Buffer作为附件一起放入到SelectionKey中**
```java
int write = socket.write(buffer);
// 通道中可能无法放入缓冲区中的所有数据
if (buffer.hasRemaining()) {
    // 注册到Selector中，关注可写事件，并将buffer添加到key的附件中
    socket.configureBlocking(false);
    socket.register(selector, SelectionKey.OP_WRITE, buffer);
}
```

- 添加写事件的相关操作key.isWritable()，对Buffer再次进行写操作
   - 每次写后需要判断Buffer中是否还有数据（是否写完）。**若写完，需要移除SelecionKey中的Buffer附件，避免其占用过多内存，同时还需移除对写事件的关注**
```java
SocketChannel socket = (SocketChannel) key.channel();
// 获得buffer
ByteBuffer buffer = (ByteBuffer) key.attachment();
// 执行写操作
int write = socket.write(buffer);
System.out.println(write);
// 如果已经完成了写操作，需要移除key中的附件，同时不再对写事件感兴趣
if (!buffer.hasRemaining()) {
    key.attach(null);
    key.interestOps(0);
}
```

**整体代码如下**
```java
public class WriteServer {
    public static void main(String[] args) {
        try(ServerSocketChannel server = ServerSocketChannel.open()) {
            server.bind(new InetSocketAddress(8080));
            server.configureBlocking(false);
            Selector selector = Selector.open();
            server.register(selector, SelectionKey.OP_ACCEPT);
            while (true) {
                selector.select();
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> iterator = selectionKeys.iterator();
                while (iterator.hasNext()) {
                    SelectionKey key = iterator.next();
                    // 处理后就移除事件
                    iterator.remove();
                    if (key.isAcceptable()) {
                        // 获得客户端的通道
                        SocketChannel socket = server.accept();
                        // 写入数据
                        StringBuilder builder = new StringBuilder();
                        for(int i = 0; i < 500000000; i++) {
                            builder.append("a");
                        }
                        ByteBuffer buffer = StandardCharsets.UTF_8.encode(builder.toString());
                        // 先执行一次Buffer->Channel的写入，如果未写完，就添加一个可写事件
                        int write = socket.write(buffer);
                        System.out.println(write);
                        // 通道中可能无法放入缓冲区中的所有数据
                        if (buffer.hasRemaining()) {
                            // 注册到Selector中，关注可写事件，并将buffer添加到key的附件中
                            socket.configureBlocking(false);
                            socket.register(selector, SelectionKey.OP_WRITE, buffer);
                        }
                    } else if (key.isWritable()) {
                        SocketChannel socket = (SocketChannel) key.channel();
                        // 获得buffer
                        ByteBuffer buffer = (ByteBuffer) key.attachment();
                        // 执行写操作
                        int write = socket.write(buffer);
                        System.out.println(write);
                        // 如果已经完成了写操作，需要移除key中的附件，同时不再对写事件感兴趣
                        if (!buffer.hasRemaining()) {
                            key.attach(null);
                            key.interestOps(0);
                        }
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
<a name="hzQGa"></a>
## 7、优化
<a name="Cch3V"></a>
### 多线程优化
充分利用多核CPU，分两组选择器

- 单线程配一个选择器（Boss），**专门处理 accept 事件**
- 创建 cpu 核心数的线程（Worker），**每个线程配一个选择器，轮流处理 read 事件**
<a name="ai8w4"></a>
#### 实现思路

- 创建**一个**负责处理Accept事件的Boss线程，与**多个**负责处理Read事件的Worker线程
- **Boss线程**执行的操作
   - 接受并处理Accepet事件，当Accept事件发生后，调用Worker的register(SocketChannel socket)方法，让Worker去处理Read事件，其中需要**根据标识robin去判断将任务分配给哪个Worker**
   - register(SocketChannel socket)方法会**通过同步队列完成Boss线程与Worker线程之间的通信**，让SocketChannel的注册任务被Worker线程执行。添加任务后需要调用selector.wakeup()来唤醒被阻塞的Selector
```java
// 创建固定数量的Worker
Worker[] workers = new Worker[4];
// 用于负载均衡的原子整数
AtomicInteger robin = new AtomicInteger(0);
// 负载均衡，轮询分配Worker
workers[robin.getAndIncrement()% workers.length].register(socket);Copy
```

```java
public void register(final SocketChannel socket) throws IOException {
    // 只启动一次
    if (!started) {
        // 初始化操作
    }
    // 向同步队列中添加SocketChannel的注册事件
    // 在Worker线程中执行注册事件
    queue.add(new Runnable() {
        @Override
        public void run() {
            try {
                socket.register(selector, SelectionKey.OP_READ);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    });
    // 唤醒被阻塞的Selector
    // select类似LockSupport中的park，wakeup的原理类似LockSupport中的unpark
    selector.wakeup();
}
```

- **Worker线程执行**的操作
   - **从同步队列中获取注册任务，并处理Read事件**
<a name="kI3Oz"></a>
#### 实现代码
```java
import edu.ylh.componentAndFile.ByteBufferUtil;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.concurrent.atomic.AtomicInteger;

public class ThreadsServer {
    public static void main(String[] args) {
        try (ServerSocketChannel server = ServerSocketChannel.open()) {
            // 当前线程为Boss线程
            Thread.currentThread().setName("Boss");
            server.bind(new InetSocketAddress(8080));
            // 负责轮询Accept事件的Selector
            Selector boss = Selector.open();
            server.configureBlocking(false);
            server.register(boss, SelectionKey.OP_ACCEPT);
            // 创建固定数量的Worker
            Worker[] workers = new Worker[4];
            // 用于负载均衡的原子整数
            AtomicInteger robin = new AtomicInteger(0);
            for (int i = 0; i < workers.length; i++) {
                workers[i] = new Worker("worker-" + i);
            }
            while (true) {
                boss.select();
                Set<SelectionKey> selectionKeys = boss.selectedKeys();
                Iterator<SelectionKey> iterator = selectionKeys.iterator();
                while (iterator.hasNext()) {
                    SelectionKey key = iterator.next();
                    iterator.remove();
                    // BossSelector负责Accept事件
                    if (key.isAcceptable()) {
                        // 建立连接
                        SocketChannel socket = server.accept();
                        System.out.println("connected...");
                        socket.configureBlocking(false);
                        // socket注册到Worker的Selector中
                        System.out.println("before read...");
                        // 负载均衡，轮询分配Worker
                        workers[robin.getAndIncrement() % workers.length].register(socket);
                        System.out.println("after read...");
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    static class Worker implements Runnable {
        private Thread thread;
        private volatile Selector selector;
        private String name;
        private volatile boolean started = false;
        /**
         * 同步队列，用于Boss线程与Worker线程之间的通信
         */
        private ConcurrentLinkedQueue<Runnable> queue;

        public Worker(String name) {
            this.name = name;
        }

        public void register(final SocketChannel socket) throws IOException {
            // 只启动一次
            if (!started) {
                thread = new Thread(this, name);
                selector = Selector.open();
                queue = new ConcurrentLinkedQueue<>();
                thread.start();
                started = true;
            }

            // 向同步队列中添加SocketChannel的注册事件
            // 在Worker线程中执行注册事件
            queue.add(new Runnable() {
                @Override
                public void run() {
                    try {
                        socket.register(selector, SelectionKey.OP_READ);
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            });
            // 唤醒被阻塞的Selector
            // select类似LockSupport中的park，wakeup的原理类似LockSupport中的unpark
            selector.wakeup();
        }

        @Override
        public void run() {
            while (true) {
                try {
                    selector.select();
                    // 通过同步队列获得任务并运行
                    Runnable task = queue.poll();
                    if (task != null) {
                        // 获得任务，执行注册操作
                        task.run();
                    }
                    Set<SelectionKey> selectionKeys = selector.selectedKeys();
                    Iterator<SelectionKey> iterator = selectionKeys.iterator();
                    while (iterator.hasNext()) {
                        SelectionKey key = iterator.next();
                        iterator.remove();
                        // Worker只负责Read事件
                        if (key.isReadable()) {
                            // 简化处理，省略细节
                            SocketChannel socket = (SocketChannel) key.channel();
                            ByteBuffer buffer = ByteBuffer.allocate(16);
                            socket.read(buffer);
                            buffer.flip();
                            ByteBufferUtil.debugAll(buffer);
                        }
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```
<a name="fG7WG"></a>
### [UDP](https://bright-boy.gitee.io/technical-notes/#/%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B/netty?id=_47-udp)

- UDP 是无连接的，client 发送数据不会管 server 是否开启
- server 这边的 receive 方法会将接收到的数据存入 byte buffer，但如果数据报文超过 buffer 大小，多出来的数据会被默默抛弃

首先启动服务器端
```java
public class UdpServer {
    public static void main(String[] args) {
        try (DatagramChannel channel = DatagramChannel.open()) {
            channel.socket().bind(new InetSocketAddress(9999));
            System.out.println("waiting...");
            ByteBuffer buffer = ByteBuffer.allocate(32);
            channel.receive(buffer);
            buffer.flip();
            debug(buffer);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
输出<br />`waiting...`<br />运行客户端
```java
public class UdpClient {
    public static void main(String[] args) {
        try (DatagramChannel channel = DatagramChannel.open()) {
            ByteBuffer buffer = StandardCharsets.UTF_8.encode("hello");
            InetSocketAddress address = new InetSocketAddress("localhost", 9999);
            channel.send(buffer, address);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
接下来服务器端输出
```java
+-------------------------------------------------+
|  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 68 65 6c 6c 6f                                  |hello           |
+--------+-------------------------------------------------+----------------+
```
<a name="QWwY1"></a>
## [NIO vs BIO](https://bright-boy.gitee.io/technical-notes/#/%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B/netty?id=_5-nio-vs-bio)
<a name="unvAz"></a>
### 5.1 stream vs channel

- stream 不会自动缓冲数据，channel 会利用系统提供的发送缓冲区、接收缓冲区（更为底层）
- stream 仅支持阻塞 API，channel 同时支持阻塞、非阻塞 API，网络 channel 可配合 selector 实现多路复用
- 二者均为全双工，即读写可以同时进行
<a name="Xg6Uf"></a>
### 5.2 IO 模型
同步阻塞、同步非阻塞、同步多路复用、异步阻塞（没有此情况）、异步非阻塞

- 同步：线程自己去获取结果（一个线程）
- 异步：线程自己不去获取结果，而是由其它线程送结果（至少两个线程）

当调用一次 channel.read 或 stream.read 后，会切换至操作系统内核态来完成真正数据读取，而读取又分为两个阶段，分别为：

- 等待数据阶段
- 复制数据阶段

![image.png](https://cdn.nlark.com/yuque/0/2023/png/22271853/1684120242534-546f28c4-35db-4545-90ee-df813a04a53b.png#averageHue=%23dbfbcb&clientId=uce15c22e-5382-4&from=paste&id=u53f43e1d&originHeight=268&originWidth=442&originalType=url&ratio=1&rotation=0&showTitle=false&size=13789&status=done&style=none&taskId=u40efef83-50cb-4a46-a7ec-e079e9dd1be&title=)

- 阻塞 IO![image.png](https://cdn.nlark.com/yuque/0/2023/png/22271853/1684120242395-1940b604-e700-4d75-aa21-c91421b637e3.png#averageHue=%23d3fbcd&clientId=uce15c22e-5382-4&from=paste&id=u7e9bcea2&originHeight=263&originWidth=437&originalType=url&ratio=1&rotation=0&showTitle=false&size=15613&status=done&style=none&taskId=u46e507da-493f-4985-80f6-7bf092c3443&title=)
- 非阻塞 IO![image.png](https://cdn.nlark.com/yuque/0/2023/png/22271853/1684120242531-e9ba27ea-4879-4661-b183-f91d57a1dcae.png#averageHue=%23d3fbcd&clientId=uce15c22e-5382-4&from=paste&id=u193f34e8&originHeight=272&originWidth=441&originalType=url&ratio=1&rotation=0&showTitle=false&size=17522&status=done&style=none&taskId=uda6d11bd-606b-42d5-9f70-96076c49f2e&title=)
- 多路复用![image.png](https://cdn.nlark.com/yuque/0/2023/png/22271853/1684120242549-c47c649b-c2a1-443b-964e-2b22eda4cf56.png#averageHue=%23d2fbcc&clientId=uce15c22e-5382-4&from=paste&id=u4a9fa776&originHeight=281&originWidth=456&originalType=url&ratio=1&rotation=0&showTitle=false&size=17038&status=done&style=none&taskId=ue0714a10-fdc1-4766-8c9f-c44179b04a4&title=)
- 信号驱动
- 异步 IO![image.png](https://cdn.nlark.com/yuque/0/2023/png/22271853/1684120242580-fd96292f-24cd-4070-92fe-b0dd967f5442.png#averageHue=%23d1f9cc&clientId=uce15c22e-5382-4&from=paste&id=ud61a8958&originHeight=270&originWidth=439&originalType=url&ratio=1&rotation=0&showTitle=false&size=18683&status=done&style=none&taskId=u1bf7da08-3a7c-4d0c-a92d-a440608a68a&title=)
- 阻塞 IO vs 多路复用![image.png](https://cdn.nlark.com/yuque/0/2023/png/22271853/1684120243214-7040080d-2cf4-4588-a5dd-90efd3a5b35d.png#averageHue=%23c6debc&clientId=uce15c22e-5382-4&from=paste&id=uc79345f6&originHeight=472&originWidth=448&originalType=url&ratio=1&rotation=0&showTitle=false&size=32109&status=done&style=none&taskId=uf62aec55-785f-4d7f-aaf8-9fee3138bd2&title=)![image.png](https://cdn.nlark.com/yuque/0/2023/png/22271853/1684120243493-fdfcb942-6d0a-47ed-9e4d-57ef8e80709b.png#averageHue=%23e4fdca&clientId=uce15c22e-5382-4&from=paste&id=u460bcabc&originHeight=333&originWidth=432&originalType=url&ratio=1&rotation=0&showTitle=false&size=24374&status=done&style=none&taskId=u25acdd6e-824e-4fa4-9d53-65576575d52&title=)
<a name="gvT4l"></a>
### 5.3 零拷贝
<a name="CDz5X"></a>
#### 传统 IO 问题
传统的 IO 将一个文件通过 socket 写出
```java
File f = new File("helloword/data.txt");
RandomAccessFile file = new RandomAccessFile(file, "r");

byte[] buf = new byte[(int)f.length()];
file.read(buf);

Socket socket = ...;
socket.getOutputStream().write(buf);
```
内部工作流程是这样的：<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/22271853/1684120244511-fa2c2778-ee94-4462-8907-8e4e3bc9d1ff.png#averageHue=%23ebe8ef&clientId=uce15c22e-5382-4&from=paste&id=u4b399f36&originHeight=263&originWidth=492&originalType=url&ratio=1&rotation=0&showTitle=false&size=13834&status=done&style=none&taskId=u210fe696-24fb-401f-97d1-625653a83ab&title=)

1. java 本身并不具备 IO 读写能力，因此 read 方法调用后，要从 java 程序的**用户态**切换至**内核态**，去调用操作系统（Kernel）的读能力，将数据读入**内核缓冲区**。这期间用户线程阻塞，操作系统使用 DMA（Direct Memory Access）来实现文件读，其间也不会使用 cpu**DMA 也可以理解为硬件单元，用来解放 cpu 完成文件 IO**
2. 从**内核态**切换回**用户态**，将数据从**内核缓冲区**读入**用户缓冲区**（即 byte[] buf），这期间 cpu 会参与拷贝，无法利用 DMA
3. 调用 write 方法，这时将数据从**用户缓冲区**（byte[] buf）写入 **socket 缓冲区**，cpu 会参与拷贝
4. 接下来要向网卡写数据，这项能力 java 又不具备，因此又得从**用户态**切换至**内核态**，调用操作系统的写能力，使用 DMA 将 **socket 缓冲区**的数据写入网卡，不会使用 cpu

可以看到中间环节较多，java 的 IO 实际不是物理设备级别的读写，而是缓存的复制，底层的真正读写是操作系统来完成的

- 用户态与内核态的切换发生了 3 次，这个操作比较重量级
- 数据拷贝了共 4 次
<a name="ZdzM5"></a>
#### NIO 优化
通过 DirectByteBuf

- ByteBuffer.allocate(10) HeapByteBuffer 使用的还是 java 内存
- ByteBuffer.allocateDirect(10) DirectByteBuffer 使用的是操作系统内存

![image.png](https://cdn.nlark.com/yuque/0/2023/png/22271853/1684120244561-e1b32313-328a-496c-8d81-14a798938dff.png#averageHue=%23efe6f3&clientId=uce15c22e-5382-4&from=paste&id=ua5e67ded&originHeight=271&originWidth=488&originalType=url&ratio=1&rotation=0&showTitle=false&size=20036&status=done&style=none&taskId=u551ed700-216f-4376-8c24-5644efc7ab5&title=)<br />大部分步骤与优化前相同，不再赘述。唯有一点：java 可以使用 DirectByteBuf 将堆外内存映射到 jvm 内存中来直接访问使用

- 这块内存不受 jvm 垃圾回收的影响，因此内存地址固定，有助于 IO 读写
- java 中的 DirectByteBuf 对象仅维护了此内存的虚引用，内存回收分成两步
   - DirectByteBuf 对象被垃圾回收，将虚引用加入引用队列
   - 通过专门线程访问引用队列，根据虚引用释放堆外内存
- 减少了一次数据拷贝，用户态与内核态的切换次数没有减少

进一步优化（底层采用了 linux 2.1 后提供的 sendFile 方法），java 中对应着两个 channel 调用 transferTo/transferFrom 方法拷贝数据<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/22271853/1684120244519-bca3a123-4343-4a7f-bc22-1ebbf581ca68.png#averageHue=%23e8dff0&clientId=uce15c22e-5382-4&from=paste&id=u7588c51a&originHeight=160&originWidth=483&originalType=url&ratio=1&rotation=0&showTitle=false&size=9305&status=done&style=none&taskId=uc5b220d5-be17-4ff0-a0a7-5aa18235266&title=)

1. java 调用 transferTo 方法后，要从 java 程序的**用户态**切换至**内核态**，使用 DMA将数据读入**内核缓冲区**，不会使用 cpu
2. 数据从**内核缓冲区**传输到 **socket 缓冲区**，cpu 会参与拷贝
3. 最后使用 DMA 将 **socket 缓冲区**的数据写入网卡，不会使用 cpu

可以看到

- 只发生了一次用户态与内核态的切换
- 数据拷贝了 3 次

进一步优化（linux 2.4）<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/22271853/1684120244526-82fe4725-6800-415e-b3d1-1e5d13eeefe1.png#averageHue=%23eae3f1&clientId=uce15c22e-5382-4&from=paste&id=u3fccd3c7&originHeight=195&originWidth=485&originalType=url&ratio=1&rotation=0&showTitle=false&size=10282&status=done&style=none&taskId=u7ca8c3ab-b570-445c-85a7-0d7b448b699&title=)

1. java 调用 transferTo 方法后，要从 java 程序的**用户态**切换至**内核态**，使用 DMA将数据读入**内核缓冲区**，不会使用 cpu
2. 只会将一些 offset 和 length 信息拷入 **socket 缓冲区**，几乎无消耗
3. 使用 DMA 将 **内核缓冲区**的数据写入网卡，不会使用 cpu

整个过程仅只发生了一次用户态与内核态的切换，数据拷贝了 2 次。所谓的【零拷贝】，并不是真正无拷贝，而是在不会拷贝重复数据到 jvm 内存中，零拷贝的优点有

- 更少的用户态与内核态的切换
- 不利用 cpu 计算，减少 cpu 缓存伪共享
- 零拷贝适合小文件传输
<a name="E7Tqh"></a>
### 5.3 AIO
AIO 用来解决数据复制阶段的阻塞问题

- 同步意味着，在进行读写操作时，线程需要等待结果，还是相当于闲置
- 异步意味着，在进行读写操作时，线程不必等待结果，而是将来由操作系统来通过回调方式由另外的线程来获得结果

**异步模型需要底层操作系统（Kernel）提供支持**

- Windows 系统通过 IOCP 实现了真正的异步 IO
- Linux 系统异步 IO 在 2.6 版本引入，但其底层实现还是用多路复用模拟了异步 IO，性能没有优势
<a name="yRVSC"></a>
#### 文件 AIO
先来看看 AsynchronousFileChannel
```java
@Slf4j
    public class AioDemo1 {
        public static void main(String[] args) throws IOException {
            try{
                AsynchronousFileChannel s = 
                    AsynchronousFileChannel.open(
                        Paths.get("1.txt"), StandardOpenOption.READ);
                ByteBuffer buffer = ByteBuffer.allocate(2);
                log.debug("begin...");
                s.read(buffer, 0, null, new CompletionHandler<Integer, ByteBuffer>() {
                    @Override
                    public void completed(Integer result, ByteBuffer attachment) {
                        log.debug("read completed...{}", result);
                        buffer.flip();
                        debug(buffer);
                    }

                    @Override
                    public void failed(Throwable exc, ByteBuffer attachment) {
                        log.debug("read failed...");
                    }
                });

            } catch (IOException e) {
                e.printStackTrace();
            }
            log.debug("do other things...");
            System.in.read();
        }
    }
```
输出
```java
13:44:56 [DEBUG] [main] c.i.aio.AioDemo1 - begin...
13:44:56 [DEBUG] [main] c.i.aio.AioDemo1 - do other things...
13:44:56 [DEBUG] [Thread-5] c.i.aio.AioDemo1 - read completed...2
+-------------------------------------------------+
|  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 61 0d                                           |a.              |
+--------+-------------------------------------------------+----------------+
```
可以看到

- 响应文件读取成功的是另一个线程 Thread-5
- 主线程并没有 IO 操作阻塞
<a name="DIIfF"></a>
#### 💡 守护线程
默认文件 AIO 使用的线程都是守护线程，所以最后要执行 System.in.read() 以避免守护线程意外结束
<a name="BXntv"></a>
#### 网络 AIO
```java
public class AioServer {
    public static void main(String[] args) throws IOException {
        AsynchronousServerSocketChannel ssc = AsynchronousServerSocketChannel.open();
        ssc.bind(new InetSocketAddress(8080));
        ssc.accept(null, new AcceptHandler(ssc));
        System.in.read();
    }

    private static void closeChannel(AsynchronousSocketChannel sc) {
        try {
            System.out.printf("[%s] %s close\n", Thread.currentThread().getName(), sc.getRemoteAddress());
            sc.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private static class ReadHandler implements CompletionHandler<Integer, ByteBuffer> {
        private final AsynchronousSocketChannel sc;

        public ReadHandler(AsynchronousSocketChannel sc) {
            this.sc = sc;
        }

        @Override
        public void completed(Integer result, ByteBuffer attachment) {
            try {
                if (result == -1) {
                    closeChannel(sc);
                    return;
                }
                System.out.printf("[%s] %s read\n", Thread.currentThread().getName(), sc.getRemoteAddress());
                attachment.flip();
                System.out.println(Charset.defaultCharset().decode(attachment));
                attachment.clear();
                // 处理完第一个 read 时，需要再次调用 read 方法来处理下一个 read 事件
                sc.read(attachment, attachment, this);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void failed(Throwable exc, ByteBuffer attachment) {
            closeChannel(sc);
            exc.printStackTrace();
        }
    }

    private static class WriteHandler implements CompletionHandler<Integer, ByteBuffer> {
        private final AsynchronousSocketChannel sc;

        private WriteHandler(AsynchronousSocketChannel sc) {
            this.sc = sc;
        }

        @Override
        public void completed(Integer result, ByteBuffer attachment) {
            // 如果作为附件的 buffer 还有内容，需要再次 write 写出剩余内容
            if (attachment.hasRemaining()) {
                sc.write(attachment);
            }
        }

        @Override
        public void failed(Throwable exc, ByteBuffer attachment) {
            exc.printStackTrace();
            closeChannel(sc);
        }
    }

    private static class AcceptHandler implements CompletionHandler<AsynchronousSocketChannel, Object> {
        private final AsynchronousServerSocketChannel ssc;

        public AcceptHandler(AsynchronousServerSocketChannel ssc) {
            this.ssc = ssc;
        }

        @Override
        public void completed(AsynchronousSocketChannel sc, Object attachment) {
            try {
                System.out.printf("[%s] %s connected\n", Thread.currentThread().getName(), sc.getRemoteAddress());
            } catch (IOException e) {
                e.printStackTrace();
            }
            ByteBuffer buffer = ByteBuffer.allocate(16);
            // 读事件由 ReadHandler 处理
            sc.read(buffer, buffer, new ReadHandler(sc));
            // 写事件由 WriteHandler 处理
            sc.write(Charset.defaultCharset().encode("server hello!"), ByteBuffer.allocate(16), new WriteHandler(sc));
            // 处理完第一个 accpet 时，需要再次调用 accept 方法来处理下一个 accept 事件
            ssc.accept(null, this);
        }

        @Override
        public void failed(Throwable exc, Object attachment) {
            exc.printStackTrace();
        }
    }
}
```
<a name="DGvCI"></a>
# <br />
