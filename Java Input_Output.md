## Java Input/Output

### 1. InputStream/OutputStream
InputStream/OutputStream属于流处理的基类，抽象类。

#### 1.1 InputStream

###### InputStream#Read
```java

/** 抽象Read()
 * 1. 抽象方法，子类需实现
 * 2. read从流中读取下一个字节
 * 3. 返回类型为int，取值在0到255之间。
 * 4. 当读到流结尾的时候，返回值为-1
 * 5. 如果流中没有数据，read方法会阻塞直到数据到来、流关闭、或异常出现
 * 6. 异常出现时，read方法抛出异常，类型为IOException，这是一个受检异常，调用者必须进行处理。
 */
public abstract int read() throws IOException;

/** Read byte[]
 * 1. 读入的字节放入缓冲区参数数组b中，第一个字节存入b[0]，第二个存入b[1]，依次类推。
 * 2. 返回值：实际读入的字节个数。如果刚开始读取时已到流结尾，则返回-1。
 * 3. 最多将b放满，即读入的长度为最大为 b.len
 * 4. 如果流中一个字节都没有，它会阻塞，异常出现时也是抛出IOException。
 * 5. 重载更为通用的方法
 */
public int read(byte b[]) throws IOException {
    return read(b, 0, b.length);
}
public int read(byte b[], int off, int len) throws IOException {
    // code  if 边界判断
    int i = 1;
    try {
        for (; i < len ; i++) {
            c = read();
            if (c == -1) {
                break;
            }
            b[off + i] = (byte)c;
        }
    } catch (IOException ee) {
    }
    return i;
}
```

###### InputStream#Close
```java
/** Close 使用
 * 1. 流读取结束后，将其关闭，以释放相关资源
 * 2. 不管read方法是否抛出了异常，都应该调用close方法
 * 3. close通常应该放在finally语句内
 * 4. close自己可能也会抛出IOException，但通常可以捕获并忽略。
 */
public void close() throws IOException {}
```

###### InputStream#Skip
```java
/** 跳过n字节
 * 1. skip跳过输入流中n个字节
 * 2. 返回值：实际跳过的字节数。假设输入流中剩余的字节个数为m，若m<n，则会将m全部跳过，返回m。若m>n，则跳过n就停止，返回n。
 * 3. 每次读取min(2048,remaining)，放入buffer[]，然后丢弃。其中，remaining为剩余需要跳过的字节数。
 * 4. 读取然后扔掉，实现方式较low。子类往往会提供更为高效的实现，例如FileInputStream会调用本地方法。
 */
public long skip(long n) throws IOException {
    long remaining = n;
    // code ...
    int size = (int)Math.min(MAX_SKIP_BUFFER_SIZE, remaining); // MAX_SKIP_BUFFER_SIZE = 2048;
    byte[] skipBuffer = new byte[size];
    while (remaining > 0) {
        // skipBuffer只是为了让read()顺利执行而存在的方法。其中读取的数据不作处理，即视为直接扔掉。
        nr = read(skipBuffer, 0, (int)Math.min(size, remaining));
        if (nr < 0) {
            break;
        }
        remaining -= nr;
    }
    return n - remaining;
}
```
###### InputStream#(mark/reset/markSupported) 重新读取 
**重新读取：提供先查看后面的内容，然后再重新从头读取。**
比如，处理一个未知的二进制文件，我们不确定它的类型，但可能可以通过流的前几十个字节判断出来，判读出来后，再重置到流开头，交给相应类型的代码进行处理。

```java
/** mark/reset/markSupported
 * 1. 先使用mark方法将当前位置标记下来
 * 2. 在读取了一些字节，希望重新从标记位置读时，调用reset方法
 * 3. mark方法有一个参数readLimit，表示在设置了标记后，能够继续往后读的最多字节数，如果超过了，标记会无效。
 * 4. 流能够将从标记位置开始的字节保存起来，而保存消耗的内存不能无限大，流只保证其小于readLimit
 * 5. 不是所有流都支持mark/reset的，可调用markSupported判断当前流是否支持mark/reset。
 * 6. InpuStream的默认实现是不支持，FileInputStream也不直接支持，但BufferedInputStream和ByteArrayInputStream可以。
 */
public synchronized void mark(int readlimit) {}

public synchronized void reset() throws IOException {
    throw new IOException("mark/reset not supported");
}

public boolean markSupported() {
    return false;
}
```




#### 1.2 OutputStream

######OutputStream# Write
```java
/**
 * 1. 向流中写入一个字节
 * 2. 参数类型虽然是int，但其实只会用到最低的8位。其余的24位被丢弃
 * 3. 子类必须写具体实现
 */
public abstract void write(int b) throws IOException;

/**
 * 1. 向流中写入特定的字节数组 byte b[]
 * 2. 会重载进入write(byte b[], int off, int len)
 */
public void write(byte b[]) throws IOException {
    write(b, 0, b.length);
}

/**
 * 1. 向流中写入特定的字节数组 byte b[]
 * 2. 按顺序写入len个字节，第一个写入b[off]，最后一个是b[off+len-1]
 * 3. 默认实现为循环调用单字节的write方法，较为低效。
 * 4. 子类需要使用高效的方法。例如：FileOutpuStream会调用对应的批量写本地方法
 */
public void write(byte b[], int off, int len) throws IOException {
   // code ... 边界判断
   for (int i = 0 ; i < len ; i++) {
        write(b[off + i]);
    }
}
```
###### OutputStream#Flush
```java
/**
 * 1. flush将缓冲而未实际写的数据进行实际写入
 * 2. 基类OutputStream没有缓冲，flush代码为空
 * 3. BufferedOutputStream中，调用flush会将其缓冲区的内容写到其装饰的流中，并调用该流的flush方法。
 * 4. FileOutputStream中没有缓冲，没有重写flush，调用flush没有任何效果。数据只是传递给操作系统，由操作系统判断什么时间保存到硬盘上。
 */
public void flush() throws IOException {}
```
###### OutputStream#Close
```java
/**
 * 1. close：关闭输出流，释放流占用的系统资源。
 * 2. close一般会先调用flush，然后释放资源
 * 3. 流被关闭后，不能继续执行操作，也不能重新被打开。
 * 4. 同InputStream一样，close一般应该放在finally语句内。
 * 5. 没有具体实现。
 */
public void close() throws IOException {}
```

### 2. FileInputStream/FileOutputStream
输入源和输出目标是文件的流。
#### 2.1 FileInputStream
###### FileInputStream#构造方法
```java
/**
 * 1. 传入参数为文件路径orFile对象，传入路径name会创建File对象
 * 3. new一个FileInputStream对象会实际打开文件，操作系统会分配相关资源。
 * 4. 如果当前用户没有写权限，会抛出异常SecurityException，它是一种RuntimeException。
 * 5. 如果指定的文件是一个已存在的目录，或者由于其他原因不能打开文件，会抛出异常FileNotFoundException，它是IOException的一个子类。
 * 6. 没有缓冲的情况下逐个字节读取性能很低
 */
public FileInputStream(String name) throws FileNotFoundException {
    this(name != null ? new File(name) : null);
}
public FileInputStream(File file) throws FileNotFoundException {
    String name = (file != null ? file.getPath() : null);
    SecurityManager security = System.getSecurityManager();
    if (security != null) {
        security.checkRead(name);
    }
    if (name == null) {
        throw new NullPointerException();
    }
    if (file.isInvalid()) {
        throw new FileNotFoundException("Invalid file path");
    }
    fd = new FileDescriptor();
    isFdOwner = true;
    this.path = name;

    BlockGuard.getThreadPolicy().onReadFromDisk();
    open(name);         // 调用open()打开文件，操作系统会分配相关资源。
    guard.open("close");
}
```
###### FileInputStream#Read
```java
/**
 * 1. Read属于 InputStream Read方法的实现
 * 2. 调用系统的IoBridge.read方法
 */
public int read() throws IOException {
    byte[] b = new byte[1];
    return (read(b, 0, 1) != -1) ? b[0] & 0xff : -1;
}
public int read(byte b[]) throws IOException {
    return read(b, 0, b.length);
}
public int read(byte b[], int off, int len) throws IOException {
    if (closed && len > 0) {
        throw new IOException("Stream Closed");
    }
    tracker.trackIo(len);
    return IoBridge.read(fd, b, off, len);
}
```
###### FileInputStream#Open
```java
/**
 * 1. FileInputStream or FileOutputStream中，open实际会调用native方法。
 * 2. 参数传入路径name、覆盖(overwriting) or 追加(appending)
 * 3. 为了方便instrumentation，在native方法外添加一层包装
 */
private void open(String name, boolean append) throws FileNotFoundException {
    open0(name, append);
}
private native void open0(String name, boolean append) throws FileNotFoundException;
```
###### FileInputStream#getFD 获取FileDescriptor
```java
/**
 * 1. getFD返回值FileDescriptor(文件描述符)，它与操作系统的一些文件内存结构相连，一般我们用不到它。
 * 2. FileDescriptor有个sync方法，确保将操作系统缓冲的数据写到硬盘上。在一定特定情况下，一定需要确保数据写入硬盘，则可以调用该方法。
 * 3. 对比：flush只能将应用程序缓冲的数据写到操作系统
 */
public final FileDescriptor getFD()  throws IOException {
    if (fd != null) {
        return fd;
    }
    throw new IOException();
 }
 FileDescriptor # public native void sync() throws SyncFailedException;
```
#### 2.2 FileOutputStream
###### FileOutputStream#构造方法
```java
/**
 * 1. 传入参数为文件路径name or File对象file，传入路径name会创建File对象
 * 2. append参数指定是追加还是覆盖，true表示追加，没传append参数默认表示覆盖。
 * 3. new一个FileOutputStream对象会调用open()打开文件，操作系统会分配相关资源。
 * 4. 会检查写入权限，如果当前用户没有写权限，会抛出异常SecurityException，它是一种RuntimeException。
 * 5. 如果指定的文件是一个已存在的目录，或者由于其他原因不能打开文件，会抛出异常FileNotFoundException，它是IOException的一个子类。
 */
public FileOutputStream(String name) throws FileNotFoundException {
    this(name != null ? new File(name) : null, false);
}
public FileOutputStream(String name, boolean append) throws FileNotFoundException {
    this(name != null ? new File(name) : null, append);
}
public FileOutputStream(File file) throws FileNotFoundException {
    this(file, false);
}
public FileOutputStream(File file, boolean append) throws FileNotFoundException {
    String name = (file != null ? file.getPath() : null);
    SecurityManager security = System.getSecurityManager();
    if (security != null) {
        security.checkWrite(name);          //检查写入权限
    }
    if (name == null) {
        throw new NullPointerException();
    }
    if (file.isInvalid()) {                             //检查文件是否可用
        throw new FileNotFoundException("Invalid file path");
    }
    this.fd = new FileDescriptor();
    this.append = append;
    this.path = name;
    this.isFdOwner = true;

    BlockGuard.getThreadPolicy().onWriteToDisk();
    open(name, append);         // 调用open()打开文件，操作系统会分配相关资源。
    guard.open("close");
}
```

###### FileOutputStream#Write
```java
/**
 * 1. Write属于 OutputStream write方法的实现
 * 2. 调用系统的write方法
 */
public void write(int b) throws IOException {
    write(new byte[] { (byte) b }, 0, 1);
}
public void write(byte b[]) throws IOException {
    write(b, 0, b.length);
}
public void write(byte b[], int off, int len) throws IOException {
    if (closed && len > 0) {
        throw new IOException("Stream Closed");
    }
    tracker.trackIo(len);
    IoBridge.write(fd, b, off, len);
}
```

### 3. ByteArrayInputStream/ByteArrayOutputStream
输入源和输出目标是字节数组的流。

为什么要将byte数组转换为InputStream呢？
这与容器类中要将数组、单个元素转换为容器接口的原因是类似的，有很多代码是以InputStream/OutputSteam为参数构建的，它们构成了一个协作体系，将byte数组转换为InputStream可以方便的参与这种体系，复用代码。

#### 3.1 ByteArrayOutputStream
###### ByteArrayOutputStream#构造方法
```java
/**
 * 1. ByteArrayOutputStream的输出目标是一个byte数组
 * 2. byte b[]保存在内存中
 * 3. byte b[]默认大小为32，可自动扩容。扩展策略同样是指数扩展，每次至少增加一倍。
 */
public ByteArrayOutputStream() {
    this(32);
}
public ByteArrayOutputStream(int size) {
    if (size < 0) {
        throw new IllegalArgumentException("Negative initial size: "+ size);
    }
    buf = new byte[size];
}
```
###### ByteArrayOutputStream#数组扩容
```java
/**
 * 1. byte b[]默认大小为32，每次write前会检查容量是否足够。
 * 2. 为了兼容某些虚拟机会在数组前添加header，最大容量MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8
 * 3. 容量每次增长一倍
 */
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
private void ensureCapacity(int minCapacity) {
    if (minCapacity - buf.length > 0)
        grow(minCapacity);
}
private void grow(int minCapacity) {
    int oldCapacity = buf.length;
    int newCapacity = oldCapacity << 1;     //指数增加
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    buf = Arrays.copyOf(buf, newCapacity);
}
// 溢出判断处理
private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ? Integer.MAX_VALUE : MAX_ARRAY_SIZE;
}
```
###### ByteArrayOutputStream#Writes
```java
/**
 * 1. write 之前先进行容量检查
 * 2. writeTo(OutputStream out) 可以将数据方便的写到另一个OutputStream
 */
public synchronized void write(int b) {
    ensureCapacity(count + 1);
    buf[count] = (byte) b;
    count += 1;
}
public synchronized void write(byte b[], int off, int len) {
    ensureCapacity(count + len);
    System.arraycopy(b, off, buf, count, len);
    count += len;
}
public synchronized void writeTo(OutputStream out) throws IOException {
    out.write(buf, 0, count);
}
```
###### ByteArrayOutputStream#toString toByteArray
```java
/**
 * 1. 数据转换为字节数组或字符串
 * 2. toString()方法使用系统默认编码
 */
public synchronized byte toByteArray()[] {
    return Arrays.copyOf(buf, count);
}
public synchronized String toString() {
    return new String(buf, 0, count);
}
public synchronized String toString(String charsetName) throws UnsupportedEncodingException {
    return new String(buf, 0, count, charsetName);
}
```
#### 3.2 ByteArrayInputStream
ByteArrayInputStream将byte数组包装为一个输入流，是一种适配器模式。
###### ByteArrayInputStream#构造方法
```java
/**
 * 1. buf中offset开始最大读取length个字节。len不输入，默认为整个buf长度
 * 2. ByteArrayInputStream的所有数据buf都在内存，支持mark/reset重复读取
 */
public ByteArrayInputStream(byte buf[]) {
    this.buf = buf;
    this.pos = 0;
    this.count = buf.length;
}

public ByteArrayInputStream(byte buf[], int offset, int length) {
    this.buf = buf;
    this.pos = offset;
    this.count = Math.min(offset + length, buf.length);
    this.mark = offset;
}
```
read、mark、reset没有很大出入，省略。
### 4. FilterInputStream/FilterOutputStream
流装饰类的父类。DataInputStream/DataOutputStream、BufferedInputStream/BufferedOutputStream属于其子类。
此类很鸡贼，在内部封装了一个InputStream/OutputStream，内部的流操作调用InputStream/OutputStream的对应操作，然后由子类在外边做装饰操作。
#### 4.1 FilterInputStream
```java
protected FilterInputStream(InputStream in) {
    this.in = in;
}
public int read() throws IOException {
    return in.read();
}
```
#### 4.2 FilterOutputStream
```java
public FilterOutputStream(OutputStream out) {
    this.out = out;
}
public void write(int b) throws IOException {
    out.write(b);
}
```

### 5. DataInputStream/DataOutputStream
装饰类，按基本类型（int, double）和字符串而非只是字节读写流。
#### 5.1 DataOutputStream
DataOutputStream实现了DataOutput接口，DataOutput是一个接口，可以以各种基本类型和字符串写入数据。
DataOutputStream会将这些类型的数据转换为其对应的二进制字节

```java
/** 
 * DataOutputStream 部分方法
 */
//写入一个字节，如果值为true，则写入1，否则0
public final void writeBoolean(boolean v) throws IOException {
    out.write(v ? 1 : 0);  
    incCount(1);
}
public final void writeByte(int v) throws IOException {
    out.write(v);
    incCount(1);
}
public final void writeShort(int v) throws IOException {
    out.write((v >>> 8) & 0xFF);
    out.write((v >>> 0) & 0xFF);
    incCount(2);
}
// 写入四个字节，最高位字节先写入，最低位最后写入
public final void writeInt(int v) throws IOException {
    out.write((v >>> 24) & 0xFF);
    out.write((v >>> 16) & 0xFF);
    out.write((v >>>  8) & 0xFF);
    out.write((v >>>  0) & 0xFF);
    incCount(4);
}
```

#### 5.2 DataInputStream
DataInputStream是装饰类基类FilterInputStream的子类，FilterInputStream是InputStream的子类。
DataInputStream实现了DataInput接口，DataInput是一个接口，可以以各种基本类型和字符串读取数据。
在读取时，DataInputStream会先按字节读进来，然后转换为对应的类型。

```java
/** 
 * DataInputStream 部分方法
 */
public final boolean readBoolean() throws IOException {
    int ch = in.read();
    if (ch < 0)
        throw new EOFException();
    return (ch != 0);
}
public final byte readByte() throws IOException {
    int ch = in.read();
    if (ch < 0)
        throw new EOFException();
    return (byte)(ch);
}
public final short readShort() throws IOException {
    readFully(readBuffer, 0, 2);
    return Memory.peekShort(readBuffer, 0, ByteOrder.BIG_ENDIAN);
}
public final int readInt() throws IOException {
    readFully(readBuffer, 0, 4);
    return Memory.peekInt(readBuffer, 0, ByteOrder.BIG_ENDIAN);
}
```

### 6. BufferedInputStream/BufferedOutputStream
装饰类，对输入输出流提供缓冲功能。
FileInputStream/FileOutputStream是没有缓冲的，按单个字节读写时性能比较低。将文件流包装到缓冲流中，可以提升性能，于是有了BufferedInputStream/BufferedOutputStream。
#### 6.1 BufferedInputStream
###### BufferedInputStream#构造方法
```java
/**
 * 1. 类super.(in),持有一个InputStream
 * 2. 类内部有字节数组byte buf[] 的缓冲区，默认大小为8192
 * 3. 采用缓冲区，性能提升。读取时，先从这个缓冲区读，缓冲区读完了再调用包装的流读。
 * 4. 支持mark/reset，可以重复读取
 */
public BufferedInputStream(InputStream in) {
    this(in, DEFAULT_BUFFER_SIZE);
}
public BufferedInputStream(InputStream in, int size) {
    super(in);
    if (size <= 0) {
        throw new IllegalArgumentException("Buffer size <= 0");
    }
    buf = new byte[size];
}
```

#### 6.2 BufferedOutputStream
###### BufferedOutputStream#构造方法
```java
 * 1. 类super.(out),持有一个InputStream
 * 2. 类内部有字节数组byte buf[] 的缓冲区，默认大小为8192
 * 3. 采用缓冲区，性能提升。
 * 4. 支持mark/reset，可以重复读取
 */
public BufferedOutputStream(OutputStream out) {
    this(out, 8192);
}
public BufferedOutputStream(OutputStream out, int size) {
    super(out);
    if (size <= 0) {
        throw new IllegalArgumentException("Buffer size <= 0");
    }
    buf = new byte[size];
}
```
#### 6.3 BufferedInputStream/BufferedOutputStream # 简单使用
在使用FileInputStream/FileOutputStream时，应该几乎总是在它的外面包上对应的缓冲类。

```java
InputStream input = new BufferedInputStream(new FileInputStream("hello.txt"));
OutputStream output =  new BufferedOutputStream(new FileOutputStream("hello.txt"));

DataOutputStream output = new DataOutputStream(new BufferedOutputStream(new FileOutputStream("students.dat")));
DataInputStream input = new DataInputStream(new BufferedInputStream(new FileInputStream("students.dat")));   
```
