---
title: Java IO
tags: 'Java, IO, 源码'
date: 2022-09-10 10:18:36
---

# InputStream
InputStream: 输入流的抽象，提供应用程从内存读取任意字节数（read指针向后偏移），跳过，标记流位置（标记一个索引位置，reset后，read指针会重置为mark的值），流剩余字节数的API。
![data_structure](https://tva1.sinaimg.cn/large/e6c9d24egy1h61dub1vbaj20pq08o3z5.jpg)

![input_stream_class_diagram](https://tva1.sinaimg.cn/large/e6c9d24egy1h63mdvxjnlj20n30degng.jpg)
1. FilterInputStream: 算是装饰者，包装或者聚合了`InputStream`，使用它作为基本的数据源。覆盖`InputStream`所有方法，简单的把方法调用转发给`InputStream`。
``` java   
public abstract class InputStream implements Closeable {
    protected volatile InputStream in;

    /**
     * Creates a <code>FilterInputStream</code>
     * by assigning the  argument <code>in</code>
     * to the field <code>this.in</code> so as
     * to remember it for later use.
     *
     * @param   in   the underlying input stream, or <code>null</code> if
     *          this instance is to be created without an underlying stream.
     */
    protected FilterInputStream(InputStream in) {
        this.in = in;
    }
}    
```
2. ByteArrayInputStream: 内部使用字节数组 `byte buf[]` 作为直接流数据源，每一次read，都从buf读取数据。
3. BufferedInputStream: 内部也使用字节数组 `byte buf[]`，不过它是代理的角色， 实际的数据流是类的聚合字段`InputStream in`，每一次read都是从buf读取数据，buf数据读完，会从`in` 预读取一块数据填充 `buf`。
一般而言，当我们使用File作为数据源的时候，需要配合`BufferedInputStream`使用`new BufferedInputStream(new BufferedInputStream())`，一次读取文件系统的一大块数据（涉及磁盘旋转等耗时物理操作），然后应用程序可以一次一次读取小块数据，提高性能。
4. FileInputStream: 使用文件系统的文件作为流数据源，读取其原始的字节数据，如果要读取字符流，考虑使用`FileReader` 
> A FileInputStream obtains input bytes from a file in a file system. FileInputStream is meant for reading streams of raw bytes such as image data. For reading streams of characters, consider using FileReader.
5. ObjectInputStream: An ObjectInputStream deserializes primitive data and objects previously written using an ObjectOutputStream.
```java
public class ObjectStreamDemo {
        FileOutputStream fos = new FileOutputStream("t.tmp");
        ObjectOutputStream oos = new ObjectOutputStream(fos);
  
        oos.writeInt(12345);
        oos.writeObject("Today");
        oos.writeObject(new Date());
  
        oos.close();
        
        FileInputStream fis = new FileInputStream("t.tmp");
        ObjectInputStream ois = new ObjectInputStream(fis);
  
        int i = ois.readInt();
        String today = (String) ois.readObject();
        Date date = (Date) ois.readObject();
  
        ois.close();
}
```
6. SequenceInputStream: 代表多个`InputStream`的逻辑拼接`Enumeration<? extends InputStream> e`。程序开始时读取数组第一个`InputStream`元素，读完流数据，继续读第二个元素，如此反复。
> A SequenceInputStream represents the logical concatenation of other input streams. It starts out with an ordered collection of input streams and reads from the first one until end of file is reached, whereupon it reads from the second one, and so on, until end of file is reached on the last of the contained input streams.
7. PushbackInputStream: 在`read`的基础上，支持`unread`，程序可以提前读取后续的字节，看看是什么数据，然后`unread`回到原来的状态，决定如何`read`。类似`mark & reset`，不同之处是unread额外提供数据（可能是上一次read的，也可能是别的）， 覆盖reset区域。
8. PipedInputStream: 管道输入流必须和管道输出流连接，一般是一个线程从管道输入流读取数据，这些数据是另一个线程往管道输入流写入。单一线程使用管道输入输出流是不推荐的。PipedInputStream构造时，传入管道输出流，并和它建立连接。
```java
public class PipedInputStream extends InputStream {
    public PipedInputStream(PipedOutputStream src, int pipeSize)
            throws IOException {
         initPipe(pipeSize);
         connect(src);
    }
    
    public void connect(PipedOutputStream src) throws IOException {
        src.connect(this);
    }
}      
```
```java
public class PipedOutputStream extends OutputStream {
    public synchronized void connect(PipedInputStream snk) throws IOException {
        if (snk == null) {
            throw new NullPointerException();
        } else if (sink != null || snk.connected) {
            throw new IOException("Already connected");
        }
        sink = snk;
        snk.in = -1;
        snk.out = 0;
        snk.connected = true;
    }
}    
```
管道输出流写入数据时，向管道输入流发送接收的消息。
```java
public class PipedOutputStream extends OutputStream {
    public void write(int b)  throws IOException {
        if (sink == null) {
            throw new IOException("Pipe not connected");
        }
        sink.receive(b);
    }
}    
```
管道输入流接收管道输出流的数据，保存在 `buffer[in++]`
```java
public class PipedInputStream extends InputStream {
    protected synchronized void receive(int b) throws IOException {
        checkStateForReceive();
        writeSide = Thread.currentThread();
        if (in == out)
            awaitSpace();
        if (in < 0) {
            in = 0;
            out = 0;
        }
        buffer[in++] = (byte)(b & 0xFF);
        if (in >= buffer.length) {
            in = 0;
        }
    }
    
    /**
     * The index of the position in the circular buffer at which the
     * next byte of data will be stored when received from the connected
     * piped output stream. <code>in&lt;0</code> implies the buffer is empty,
     * <code>in==out</code> implies the buffer is full
     * @since   JDK1.1
     */
    protected int in = -1;

    /**
     * The index of the position in the circular buffer at which the next
     * byte of data will be read by this piped input stream.
     * @since   JDK1.1
     */
    protected int out = 0;
}    
```
![](https://tva1.sinaimg.cn/large/e6c9d24egy1h61jgeoakgj20m207e3yy.jpg)
管道输入流维护写入，读出的下标索引，应用程序`read`，就从buffer读取数据`int ret = buffer[out++] & 0xFF;`
```java
public class PipedInputStream extends InputStream {
    public synchronized int read()  throws IOException {
        if (!connected) {
            throw new IOException("Pipe not connected");
        } else if (closedByReader) {
            throw new IOException("Pipe closed");
        } else if (writeSide != null && !writeSide.isAlive()
                   && !closedByWriter && (in < 0)) {
            throw new IOException("Write end dead");
        }

        readSide = Thread.currentThread();
        int trials = 2;
        while (in < 0) {
            if (closedByWriter) {
                /* closed by writer, return EOF */
                return -1;
            }
            if ((writeSide != null) && (!writeSide.isAlive()) && (--trials < 0)) {
                throw new IOException("Pipe broken");
            }
            /* might be a writer waiting */
            notifyAll();
            try {
                wait(1000);
            } catch (InterruptedException ex) {
                throw new java.io.InterruptedIOException();
            }
        }
        int ret = buffer[out++] & 0xFF;
        if (out >= buffer.length) {
            out = 0;
        }
        if (in == out) {
            /* now empty */
            in = -1;
        }

        return ret;
    }
}    
```
9. LineNumberInputStream: 读取字节如果是A carriage-return character or a carriage return followed by a newline character 则lineNumber递增计数。应用程序可以通过`java.io.LineNumberInputStream#getLineNumber`获取行号记录。
```java
public class LineNumberInputStream extends FilterInputStream {
    public int read() throws IOException {
        int c = pushBack;

        if (c != -1) {
            pushBack = -1;
        } else {
            c = in.read();
        }

        switch (c) {
          case '\r':
            pushBack = in.read();
            if (pushBack == '\n') {
                pushBack = -1;
            }
          case '\n':
            lineNumber++;
            return '\n';
        }
        return c;
    }
```
10. DataInputStream: 和ObjectInputStream类似，区别在于它只支持读取基本数据类型。
> A data input stream lets an application read primitive Java data types from an underlying input stream in a machine-independent way. An application uses a data output stream to write data that can later be read by a data input stream.

11. SocketInputStream: socket 通信时，系统分配一个fd 给接收端，用于接收数据。应用程序通过系统读文件的系统调用，就可以读取fd的数据了。
> This stream extends FileInputStream to implement a SocketInputStream. Note that this class should NOT be public.
```java
class SocketInputStream extends FileInputStream {
    int read(byte b[], int off, int length, int timeout) throws IOException {
        int n;

        // EOF already encountered
        if (eof) {
            return -1;
        }
	// ignore

        // acquire file descriptor and do the read
        FileDescriptor fd = impl.acquireFD();
        try {
            n = socketRead(fd, b, off, length, timeout);
            if (n > 0) {
                return n;
            }
        } catch (ConnectionResetException rstExc) {
            gotReset = true;
        } finally {
            impl.releaseFD();
        }

	// ignore
        eof = true;
        return -1;
    }
}    
```
13. CheckedInputStream: 维护一个读取数据的校验和功能，可以验证输入数据的完整性。
> An input stream that also maintains a checksum of the data being read. The checksum can then be used to verify the integrity of the input data.
# OutputStream
![output_stream_class_diagram](https://tva1.sinaimg.cn/large/e6c9d24egy1h63mf0qgkqj20la0dw3zx.jpg)
输出流总体上和输入流类似，只是方向相反。比如`BufferedOutputStream`有字节缓冲区`byte buf[]`，接收应用程序的写入，当缓冲区写满了，就调用装饰的输出流`OutputStream out`。`PrintStream`比较特殊，涉及到`Writer`，我们放到`Writer`部分展开。
```java
class BufferedOutputStream extends FilterOutputStream {
    /**
     * Writes the specified byte to this buffered output stream.
     *
     * @param      b   the byte to be written.
     * @exception  IOException  if an I/O error occurs.
     */
    public synchronized void write(int b) throws IOException {
        if (count >= buf.length) {
            flushBuffer();
        }
        buf[count++] = (byte)b;
    }
}    
```

# Reader
![reader_class_diagram](https://tva1.sinaimg.cn/large/e6c9d24egy1h63mgxpb3oj20nd0c875q.jpg)
一次读取`InputStream`的两个字节，并使用指定的字符集解析成对应字符。Java中一个`char`占用2`byte`，但对于UTF-8字符集，它的编码长度是可变的，用1`byte·来编码常见字符。
> A char represents a character in Java (*). It is 2 bytes large (or 16 bits).
That doesn't necessarily mean that every representation of a character is 2 bytes long. In fact many character encodings only reserve 1 byte for every character (or use 1 byte for the most common characters).

1. InputStreamReader:  作为字节流到字符流的桥接，读取一个或者多个字节，使用字符集解码成对应的字符。使用装饰者模式实现，解码工作由`StreamDecoder`完成。

> An InputStreamReader is a bridge from byte streams to character streams: It reads bytes and decodes them into characters using a specified charset. The charset that it uses may be specified by name or may be given explicitly, or the platform's default charset may be accepted.
Each invocation of one of an InputStreamReader's read() methods may cause one or more bytes to be read from the underlying byte-input stream. To enable the efficient conversion of bytes to characters, more bytes may be read ahead from the underlying stream than are necessary to satisfy the current read operation.
For top efficiency, consider wrapping an InputStreamReader within a BufferedReader. For example:
   `BufferedReader in = new BufferedReader(new InputStreamReader(System.in));`

2. FileReader: 实现很言简意赅和巧妙的令人叹为观止。提供`FileInputStream`给`InputStreamReader`作字节流数据源，就完事了，整个类才三个重载的构造器。
```java 
public class FileReader extends InputStreamReader {
   /**
    * Creates a new <tt>FileReader</tt>, given the name of the
    * file to read from.
    *
    * @param fileName the name of the file to read from
    * @exception  FileNotFoundException  if the named file does not exist,
    *                   is a directory rather than a regular file,
    *                   or for some other reason cannot be opened for
    *                   reading.
    */
    public FileReader(String fileName) throws FileNotFoundException {
        super(new FileInputStream(fileName));
    }
}    
```
3. BufferedReader: 也是使用装饰者模式。内部定义一个缓冲区，一次性读取源字符流一大块字符，以此对外提供字符流。
4. FilterReader: 同样是装饰者模式。里面啥也没做，所有的消息都转发给被装饰者，即内部的`Reader`.
5. PushbackReader: 撤销读的实现很巧妙，使用`char[] buf;` 并且索引`pos`初始为size，unread的字符写在这里，后续的读优先从这里读，里面如果没有数据，再从被装饰的`Reader`读。
```java
public class PushbackReader extends FilterReader {

    /** Pushback buffer */
    private char[] buf;

    /** Current position in buffer */
    private int pos;

    /**
     * Creates a new pushback reader with a pushback buffer of the given size.
     *
     * @param   in   The reader from which characters will be read
     * @param   size The size of the pushback buffer
     * @exception IllegalArgumentException if {@code size <= 0}
     */
    public PushbackReader(Reader in, int size) {
        super(in);
        if (size <= 0) {
            throw new IllegalArgumentException("size <= 0");
        }
        this.buf = new char[size];
        this.pos = size;
    }
}
```    

6. StringReader: 使用字符串作为字符流的数据源。
7. CharArrayReader: 使用字符数组作为字符流的数据源，里面的实现就全都是数组操作了。
8. PipedWriter: 与`PipedInputStream`类似，只是API接口参数从`byte` 换成了 `char`。
7. LineNumberReader: 增加字符流里的行号识别功能。
# Writer
与`Reader`方向相反，和`OutputStream`类似，不再赘述。
![write_class_diagram](https://tva1.sinaimg.cn/large/e6c9d24egy1h63mhs54j9j20in0gbq4c.jpg)

# RandomAccessFile
把文件当成一个`byte[]`，支持同时读写操作，读写指针做相应偏移。
![](https://tva1.sinaimg.cn/large/e6c9d24egy1h63mlwea47j20oi09sq3m.jpg)

![random_class_diagram](https://tva1.sinaimg.cn/large/e6c9d24egy1h63mjpax45j206p05e3yk.jpg)
