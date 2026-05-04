# IO

`inputStream` 是从源头(文件)读取数据到内存中

`outputStream`是将数据写入到文件中.

`BufferedInputStream`<font style="color:rgb(44, 62, 80);">从源头（通常是文件）读取数据（字节信息）到内存的过程中不会一个字节一个字节的读取，而是会先将读取到的字节存放在缓存区，并从内部缓冲区中单独读取字节。这样大幅减少了 IO 次数，提高了读取效率</font>

<code><font style="color:rgb(44, 62, 80);">BuffdOutputStream</font></code><font style="color:rgb(44, 62, 80);"> 将数据（字节信息）写入到目的地（通常是文件）的过程中不会一个字节一个字节的写入，而是会先将要写入的字节存放在缓存区，并从内部缓冲区中单独写入字节。这样大幅减少了 IO 次数，提高了读取效率</font>


> 更新: 2024-08-08 08:23:49  
> 原文: <https://www.yuque.com/alice-hv75k/mtczog/ylmteg9vh5ea3tax>