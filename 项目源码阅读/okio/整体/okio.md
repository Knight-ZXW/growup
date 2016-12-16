# Okio库 组件分析
@(Android 开发之路)

## Segment
　　Segment 是数据操作过程中，保存数据的地方，内部持有一个 限制最大为 **8192 byte**  的byte[]数组用来存储数据。Segment 的结构是 双向链表的结构，所以提供了 **pop** **push** 对结构的操作。Segment 采用链表的结构，内部使用数组来保存存储数据是一个数据操作的这种方案，链表使得插入删除更快，数组可以保证读取更快。Segment 中的优化还有 Segment 数据快的共享 以及 数据的压缩机制。

```java
  /**
   * Call this when the tail and its predecessor may both be less than half
   * full. This will copy data so that segments can be recycled.
   */
  public void compact() { //压缩当前Segment
    if (prev == this) throw new IllegalStateException();
    if (!prev.owner) return; // Cannot compact: prev isn't writable.
    int byteCount = limit - pos;
    int availableByteCount = SIZE - prev.limit + (prev.shared ? 0 : prev.pos);
    if (byteCount > availableByteCount) return; // Cannot compact: not enough writable space.
    writeTo(prev, byteCount); //判断可以则写入
    pop(); // pop当前segment
    SegmentPool.recycle(this); // pool 回收
  }
```
## SegmentPool
　　Segment的创建以及回收。因为Segment本身是链式结构，所以SegmentPool，内部使用 一个  **next** 作为 当前回收池第一个Segment的引用。SegmentPool 内部对 池的大小做了限制， Pool的最大字节是 64kb。不过看 这个 **todo**的注释，貌似也不确定 **MAX_SIZE** 设置为多少合适。

```
/** The maximum number of bytes to pool. */
  // TODO: Is 64 KiB a good maximum size? Do we ever have that many idle segments?
  static final long MAX_SIZE = 64 * 1024; // 64 KiB.
```
　　
## Source 与 Sink
　　类似 **java.io** 中流的概念，流的设计可以层层嵌套，功能加强，**Source** 表示输入流，**Sink** 表示输出流。**BufferedSource** 与 **BufferdSink** 提供了 数据的缓冲处理。**Buffer**的封装后，内部使用 Segment 链表来保存数据，在具体的实现上，okio 还提供了 数据的共享。**Source** 和 **Sink** 抽象了数据的来源，可以是 **磁盘**，**内部**，**网络**等。

### 对Buffer 设计的理解
　　这里，实际上对Buffer 这一层的设计，我只理解了是要做数据的暂存，避免持续的读写直接从管道对应的底层硬件操作。Okio中,Buffer类 实现了 BufferedSource, BufferedSink，也就是Buffer类将Source 和 Sink的操作都集合在一起，而在实际的处理过程中，是由 **RealBufferedSink** 和 **RealBufferedSource**  处理 写入 和输出 的一些不同之处，共同的业务逻辑调用**Buffer**类来实现，（Real类 是 Buffere 的代理类）分别这里贴上一段别人对okio中Buffer这一层设计的理解。
```java
public final class Buffer implements BufferedSource, BufferedSink, Cloneable {
  private static final byte[] DIGITS =
      { '0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'a', 'b', 'c', 'd', 'e', 'f' };
  static final int REPLACEMENT_CHARACTER = '\ufffd';

  Segment head;
  long size;

  public Buffer() {
  }
}  
```
```java
final class RealBufferedSource implements BufferedSource {
  public final Buffer buffer = new Buffer();
  public final Source source;
  boolean closed;

  RealBufferedSource(Source source) {
    if (source == null) throw new NullPointerException("source == null");
    this.source = source;
  }
  //... 一些数据操作
}
```
>Okio对数据的操作需要先从 管道(类似InputStream) 读到 Buffer 里（require），这个过程实际由匿名 Source 完成（提供了超时控制），再把数据从 Buffer 中读出来返回。

>其实我们如果粗略看一下 RealBufferedSource 和 RealBufferedSink 这两个类，我们就会发现，它们读写逻辑的实现都比较绕：读操作都是先把数据从 Source 读到 Buffer，再把数据从 Buffer 读到输出（返回值或传入的输出参数）；写操作都是先把数据从输入写到 Buffer，再把数据从 Buffer 写到 Sink。

>为什么要这么倒腾？ 让我们从功能需求和设计方案来考虑。

>BufferedSource 要提供各种形式的读取操作，还有查找与判等操作。大家可能会想，那我就在实现类中自己实现不就好了吗？干嘛要经过 Buffer 中转呢？这里我们实现的时候，需要考虑效率的问题，而且不仅 BufferedSource 需要高效实现，BufferedSink 也需要高效实现，这两者的高效实现技巧，很大部分都是共通的，所以为了避免同样的逻辑重复两遍，Okio 就直接把读写操作都实现在了 Buffer 这一个类中，这样逻辑更加紧凑，更加内聚。而且还能直接满足我们对于“两用数据缓冲区”的需求：既可以从头部读取数据，也能向尾部写入数据。至于我们单独的读写操作需求，Okio 就为 Buffer 分别提供了委托类：RealBufferedSource 和 RealBufferedSink，实现好 Buffer 之后，它们两者的实现将非常简洁（前者 450 行，后者 250 行）。　　

## ByteString
　　在大多数语言，包括Java中，String **是一个不可变的对象**，ByteString 也是一个 不可变对象，ByteString 对象代表一个 **immutable** 的字节序列，主要的字段是 **data** 以及 **utf8**，**data**即ByteString 构造时传入的 **byte数组**，**utf8** 字段是该字节的 **utf-8**表示 ，这是一个 **lazily computed** 字段，即只有真正需要获取（也仅在第一次，因为byteString的设计是不可变的）才会去做转换。 **ByteString**  是思想是 **以空间换时间**来提高时间上的效率。

　　**ByteString** 的构造函数并没有 clone data，因为这个类内部本身并不会对 data做任何的修改，既然是**immutable**,也就没必要浪费这个空间了。
```
  ByteString(byte[] data) {
    this.data = data; // Trusted internal constructor doesn't clone data.
  }
```

**ByteString** 提供了很多静态帮助方法 类型包括 数据的加密（包括 md5、sha1、等）、数据的截取、大小写转换、数据IO操作等，这些方法的返回都是一个新的 **ByteString**，这也是 immutable 的弊端，在复杂操作过程中会创建大量的对象。


### 用例
　　官网上读取 png 图片的例子
```java
private static final ByteString PNG_HEADER = ByteString.decodeHex("89504e470d0a1a0a");

public void decodePng(InputStream in) throws IOException {
  BufferedSource pngSource = Okio.buffer(Okio.source(in));//inputstream 转化成okio的 Source

  ByteString header = pngSource.readByteString(PNG_HEADER.size());
  if (!header.equals(PNG_HEADER)) {
    throw new IOException("Not a PNG.");
  }

  while (true) {
    Buffer chunk = new Buffer();

    // Each chunk is a length, type, data, and CRC offset.
    int length = pngSource.readInt();
    String type = pngSource.readUtf8(4);
    pngSource.readFully(chunk, length);
    int crc = pngSource.readInt();

    decodeChunk(type, chunk);
    if (type.equals("IEND")) break;
  }

  pngSource.close();
}

private void decodeChunk(String type, Buffer chunk) {
  if (type.equals("IHDR")) {
    int width = chunk.readInt();
    int height = chunk.readInt();
    System.out.printf("%08x: %s %d x %d%n", chunk.size(), type, width, height);
  } else {
    System.out.printf("%08x: %s%n", chunk.size(), type);
  }
}
```

## 例外
### transient 关键字
　　在Java中实现对象的序列化只需要将类实现Serializable接口，那么 这个类对象的所有属性和方法都会自动序列化，但是有时候有些属性时不需要序列化的，这个时候只要将字段使用 **transient** 关键字属性标记即可。
```
public class User implements Serializable{
	private String name;
	private tranisent String pwd;
}
```