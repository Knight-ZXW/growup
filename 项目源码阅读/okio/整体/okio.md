# Okio库 ByteString 源码解析
@(Android 开发之路)

## Segment
　　Segment 是数据操作过程中，保存数据的地方，内部持有一个 限制最大为 **8192 byte**  的byte[]数组用来存储数据。


## ByteString
　　在大多数语言，包括Java中，String **是一个不可变的对象**，ByteString 也是一个 不可变对象，ByteString 类 主要的字段是 **data** 以及 **utf8**，**data**即ByteString 构造时传入的 **byte数组**，**utf8** 字段是该字节的 **utf-8**表示 ，这是一个 **lazily computed** 字段，即只有真正需要获取（也仅在第一次，因为byteString的设计是不可变的）才会去做转换。 **ByteString**  是思想是 **以空间换时间**来提高时间上的效率。

　　**ByteString** 的构造函数并没有 clone data，因为这个类内部本身并不会对 data做任何的修改，既然是**immutable**,也就没必要浪费这个空间了。
```
  ByteString(byte[] data) {
    this.data = data; // Trusted internal constructor doesn't clone data.
  }
```

**ByteString** 提供了很多静态帮助方法 类型包括 数据的加密（包括 md5、sha1、等）、数据的截取、大小写转换、数据IO操作等，这些方法的返回都是一个新的 **ByteString**，这也是 immutable 的弊端，在复杂操作过程中会创建大量的对象。



## 例外
### transient 关键字
　　在Java中实现对象的序列化只需要将类实现Serializable接口，那么 这个类对象的所有属性和方法都会自动序列化，但是有时候有些属性时不需要序列化的，这个时候只要将字段使用 **transient** 关键字属性标记即可。
```
public class User implements Serializable{
	private String name;
	private tranisent String pwd;
}
```