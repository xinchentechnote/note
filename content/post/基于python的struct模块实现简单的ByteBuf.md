+++
date = '2025-05-11T18:29:30+08:00'
draft = false
title = '基于python的struct模块实现简单的ByteBuf'
categories = ["Python"]
tags = ["struct", "ByteBuf"]
+++
# 基于python的struct模块实现简单的ByteBuf


### **写在前面**

  在网络编程中需要将消息序列化为二进制序列打包传输。python标准库中的struct模块提供了pack、unpack等函数将基本数据类型转换为对应的bytes数组。使用pack、unpack需要在传参是需要关注字节序（大小端）、格式等，其中字节序有@、=、<、>、！五种，格式约21种，使用成本相对高。所以参考Netty的ByteBuf及Rust的bytes库中的Buf、BufMut为Python简单封装一个类似的ByteBuf。

netty中ByteBuf的基本结构如下：

```
 +-------------------+------------------+------------------+
 | discardable bytes |  readable bytes  |  writable bytes  |
 |                   |     (CONTENT)    |                  |
 +-------------------+------------------+------------------+
 |                   |                  |                  |
 0      <=      readerIndex   <=   writerIndex    <=    capacity

```

### **一、Buf接口设计**
![Buf接口](/images/pasted-image-20220405221600.png)
```
import abc  
import struct  
  
from numpy import longlong  
  
"""  
https://netty.io/4.0/api/io/netty/buffer/ByteBuf.html  
https://docs.oracle.com/javase/8/docs/api/java/nio/ByteBuffer.html  
https://docs.rs/bytes/1.1.0/bytes/  
"""  
  
  
class Buf(metaclass=abc.ABCMeta):  
    class ByteOrder:  
        NATIVE = '@'  
 STD_NATIVE = '='  
 LITTLE_ENDIAN = '<'  
 BIG_ENDIAN = '>'  
 NETWORK = '!'  
  
 PAD_BYTE = 'x'  
 CHAR = 'c'  
 SIGNED_CHAR = 'b'  
 UNSIGNED_CHAR = 'B'  
 BOOLEAN = '?'  
 SHORT = 'h'  
 UNSIGNED_SHORT = 'H'  
 INT = 'i'  
 UNSIGNED_INT = 'I'  
 LONG = 'l'  
 UNSIGNED_LONG = 'L'  
 LONG_LONG = 'q'  
 UNSIGNED_LONG_LONG = 'Q'  
 SSIZE_T = 'n'  
 SIZE_T = 'N'  
 EXPONENT = 'e'  
 FLOAT = 'f'  
 DOUBLE = 'd'  
 CHAR_ARR = 's'  
 CHAR_ARR1 = 'p'  
 VOID = 'P'  
  
 @abc.abstractmethod  
 def readable_bytes_len(self) -> int:  
        pass  
  
 @abc.abstractmethod  
 def to_bytes(self) -> bytearray:  
        pass  
  
 @abc.abstractmethod  
 def write_i8(self, value: int):  
        pass  
  
 @abc.abstractmethod  
 def write_u8(self, value: int):  
        pass  
  
 @abc.abstractmethod  
 def write_bool(self, value: bool):  
        pass  
  
 @abc.abstractmethod  
 def write_i16(self, value: int):  
        pass  
  
 @abc.abstractmethod  
 def write_i16_le(self, value: int):  
        pass  
  
 @abc.abstractmethod  
 def write_u16(self, value: int):  
        pass  
  
 @abc.abstractmethod  
 def write_u16_le(self, value: int):  
        pass  
  
 @abc.abstractmethod  
 def write_i32(self, value: int):  
        pass  
  
 @abc.abstractmethod  
 def write_i32_le(self, value: int):  
        pass  
  
 @abc.abstractmethod  
 def write_u32(self, value: int):  
        pass  
  
 @abc.abstractmethod  
 def write_u32_le(self, value: int):  
        pass  
  
 @abc.abstractmethod  
 def write_i64(self, value: longlong):  
        pass  
  
 @abc.abstractmethod  
 def write_i64_le(self, value: longlong):  
        pass  
  
 @abc.abstractmethod  
 def write_u64(self, value: longlong):  
        pass  
  
 @abc.abstractmethod  
 def write_u64_le(self, value: longlong):  
        pass  
  
 @abc.abstractmethod  
 def write_f32(self, value: float):  
        pass  
  
 @abc.abstractmethod  
 def write_f32_le(self, value: float):  
        pass  
  
 @abc.abstractmethod  
 def write_f64(self, value: float):  
        pass  
  
 @abc.abstractmethod  
 def write_f64_le(self, value: float):  
        pass  
  
 @abc.abstractmethod  
 def write_bytes(self, value: bytes):  
        pass  
  
 @abc.abstractmethod  
 def read_i8(self):  
        pass  
  
 @abc.abstractmethod  
 def read_u8(self):  
        pass  
  
 @abc.abstractmethod  
 def read_bool(self):  
        pass  
  
 @abc.abstractmethod  
 def read_i16(self):  
        pass  
  
 @abc.abstractmethod  
 def read_i16_le(self):  
        pass  
  
 @abc.abstractmethod  
 def read_u16(self):  
        pass  
  
 @abc.abstractmethod  
 def read_u16_le(self):  
        pass  
  
 @abc.abstractmethod  
 def read_i32(self):  
        pass  
  
 @abc.abstractmethod  
 def read_i32_le(self):  
        pass  
  
 @abc.abstractmethod  
 def read_u32(self):  
        pass  
  
 @abc.abstractmethod  
 def read_u32_le(self):  
        pass  
  
 @abc.abstractmethod  
 def read_i64(self):  
        pass  
  
 @abc.abstractmethod  
 def read_i64_le(self):  
        pass  
  
 @abc.abstractmethod  
 def read_u64(self):  
        pass  
  
 @abc.abstractmethod  
 def read_u64_le(self):  
        pass  
  
 @abc.abstractmethod  
 def read_f32(self):  
        pass  
  
 @abc.abstractmethod  
 def read_f32_le(self):  
        pass  
  
 @abc.abstractmethod  
 def read_f64(self):  
        pass  
  
 @abc.abstractmethod  
 def read_f64_le(self):  
        pass  
  
 @abc.abstractmethod  
 def read_bytes(self, length: int) -> bytes:  
        pass  
  

```


### **二、ByteBuf具体实现**

ByteBuf底层使用可以字节数组bytearray作存储，记录分别读写的位置。
![ByteBuf](/note/images/pasted-image-20220405221730.png)

```
import abc  
import struct  
  
from numpy import longlong  
  
  
  
class ByteBuf(Buf):  
  
    def __init__(self, buf: bytearray = None) -> None:  
        if buf is None:  
            self.buf = bytearray()  
            self.write_index = 0  
 else:  
            self.buf = buf  
            self.write_index = len(buf)  
        self.read_index = 0  
  
 def check_readable_bytes_len(self, length: int):  
        if self.readable_bytes_len() < length:  
            raise Exception("readable bytes length must greater than or equal %d" % length)  
  
    def readable_bytes_len(self) -> int:  
        return self.write_index - self.read_index  
  
    def to_bytes(self) -> bytearray:  
        return self.buf  
  
    def write_i8(self, value: int):  
        self.buf += struct.pack(Buf.SIGNED_CHAR, value)  
        self.write_index += 1  
  
 def write_u8(self, value: int):  
        self.buf += struct.pack(Buf.UNSIGNED_CHAR, value)  
        self.write_index += 1  
  
 def write_bool(self, value: bool):  
        self.buf += struct.pack(Buf.BOOLEAN, value)  
        self.write_index += 1  
  
 def write_i16(self, value: int):  
        self.buf += struct.pack(Buf.ByteOrder.BIG_ENDIAN + Buf.SHORT, value)  
        self.write_index += 2  
  
 def write_i16_le(self, value: int):  
        self.buf += struct.pack(Buf.ByteOrder.LITTLE_ENDIAN + Buf.SHORT, value)  
        self.write_index += 2  
  
 def write_u16(self, value: int):  
        self.buf += struct.pack(Buf.ByteOrder.BIG_ENDIAN + Buf.UNSIGNED_SHORT, value)  
        self.write_index += 2  
  
 def write_u16_le(self, value: int):  
        self.buf += struct.pack(Buf.ByteOrder.LITTLE_ENDIAN + Buf.UNSIGNED_SHORT, value)  
        self.write_index += 2  
  
 def write_i32(self, value: int):  
        self.buf += struct.pack(Buf.ByteOrder.BIG_ENDIAN + Buf.INT, value)  
        self.write_index += 4  
  
 def write_i32_le(self, value: int):  
        self.buf += struct.pack(Buf.ByteOrder.LITTLE_ENDIAN + Buf.INT, value)  
        self.write_index += 4  
  
 def write_u32(self, value: int):  
        self.buf += struct.pack(Buf.ByteOrder.BIG_ENDIAN + Buf.UNSIGNED_INT, value)  
        self.write_index += 4  
  
 def write_u32_le(self, value: int):  
        self.buf += struct.pack(Buf.ByteOrder.LITTLE_ENDIAN + Buf.UNSIGNED_INT, value)  
        self.write_index += 4  
  
 def write_i64(self, value: longlong):  
        self.buf += struct.pack(Buf.ByteOrder.BIG_ENDIAN + Buf.LONG_LONG, value)  
        self.write_index += 8  
  
 def write_i64_le(self, value: longlong):  
        self.buf += struct.pack(Buf.ByteOrder.LITTLE_ENDIAN + Buf.LONG_LONG, value)  
        self.write_index += 8  
  
 def write_u64(self, value: longlong):  
        self.buf += struct.pack(Buf.ByteOrder.BIG_ENDIAN + Buf.UNSIGNED_LONG_LONG, value)  
        self.write_index += 8  
  
 def write_u64_le(self, value: longlong):  
        self.buf += struct.pack(Buf.ByteOrder.LITTLE_ENDIAN + Buf.UNSIGNED_LONG_LONG, value)  
        self.write_index += 8  
  
 def write_f32(self, value: float):  
        self.buf += struct.pack(Buf.ByteOrder.BIG_ENDIAN + Buf.FLOAT, value)  
        self.write_index += 4  
  
 def write_f32_le(self, value: float):  
        self.buf += struct.pack(Buf.ByteOrder.LITTLE_ENDIAN + Buf.FLOAT, value)  
        self.write_index += 4  
  
 def write_f64(self, value: float):  
        self.buf += struct.pack(Buf.ByteOrder.BIG_ENDIAN + Buf.DOUBLE, value)  
        self.write_index += 8  
  
 def write_f64_le(self, value: float):  
        self.buf += struct.pack(Buf.ByteOrder.LITTLE_ENDIAN + Buf.DOUBLE, value)  
        self.write_index += 8  
  
 def write_bytes(self, value: bytes):  
        if len(value) > 0:  
            self.buf += value  
            self.write_index += len(value)  
  
    def read_i8(self) -> int:  
        self.check_readable_bytes_len(1)  
        ret = struct.unpack(Buf.SIGNED_CHAR, self.buf[self.read_index:self.read_index + 1])  
        self.read_index += 1  
 return ret[0]  
  
    def read_u8(self):  
        self.check_readable_bytes_len(1)  
        ret = struct.unpack(Buf.UNSIGNED_CHAR, self.buf[self.read_index:self.read_index + 1])  
        self.read_index += 1  
 return ret[0]  
  
    def read_bool(self):  
        self.check_readable_bytes_len(1)  
        ret = struct.unpack(Buf.BOOLEAN, self.buf[self.read_index:self.read_index + 1])  
        self.read_index += 1  
 return ret[0]  
  
    def read_i16(self):  
        self.check_readable_bytes_len(2)  
        ret = struct.unpack(Buf.ByteOrder.BIG_ENDIAN + Buf.SHORT, self.buf[self.read_index:self.read_index + 2])  
        self.read_index += 2  
 return ret[0]  
  
    def read_i16_le(self):  
        self.check_readable_bytes_len(2)  
        ret = struct.unpack(Buf.ByteOrder.LITTLE_ENDIAN + Buf.SHORT, self.buf[self.read_index:self.read_index + 2])  
        self.read_index += 2  
 return ret[0]  
  
    def read_u16(self):  
        self.check_readable_bytes_len(2)  
        ret = struct.unpack(Buf.ByteOrder.BIG_ENDIAN + Buf.UNSIGNED_SHORT,  
 self.buf[self.read_index:self.read_index + 2])  
        self.read_index += 2  
 return ret[0]  
  
    def read_u16_le(self):  
        self.check_readable_bytes_len(2)  
        ret = struct.unpack(Buf.ByteOrder.LITTLE_ENDIAN + Buf.UNSIGNED_SHORT,  
 self.buf[self.read_index:self.read_index + 2])  
        self.read_index += 2  
 return ret[0]  
  
    def read_i32(self):  
        ret = struct.unpack(Buf.ByteOrder.BIG_ENDIAN + Buf.INT,  
 self.buf[self.read_index:self.read_index + 4])  
        self.read_index += 4  
 return ret[0]  
  
    def read_i32_le(self):  
        self.check_readable_bytes_len(4)  
        ret = struct.unpack(Buf.ByteOrder.LITTLE_ENDIAN + Buf.INT,  
 self.buf[self.read_index:self.read_index + 4])  
        self.read_index += 4  
 return ret[0]  
  
    def read_u32(self):  
        self.check_readable_bytes_len(4)  
        ret = struct.unpack(Buf.ByteOrder.BIG_ENDIAN + Buf.UNSIGNED_INT,  
 self.buf[self.read_index:self.read_index + 4])  
        self.read_index += 4  
 return ret[0]  
  
    def read_u32_le(self):  
        self.check_readable_bytes_len(4)  
        ret = struct.unpack(Buf.ByteOrder.LITTLE_ENDIAN + Buf.UNSIGNED_INT,  
 self.buf[self.read_index:self.read_index + 4])  
        self.read_index += 4  
 return ret[0]  
  
    def read_i64(self):  
        self.check_readable_bytes_len(8)  
        ret = struct.unpack(Buf.ByteOrder.BIG_ENDIAN + Buf.LONG_LONG,  
 self.buf[self.read_index:self.read_index + 8])  
        self.read_index += 8  
 return ret[0]  
  
    def read_i64_le(self):  
        self.check_readable_bytes_len(8)  
        ret = struct.unpack(Buf.ByteOrder.LITTLE_ENDIAN + Buf.LONG_LONG,  
 self.buf[self.read_index:self.read_index + 8])  
        self.read_index += 8  
 return ret[0]  
  
    def read_u64(self):  
        self.check_readable_bytes_len(8)  
        ret = struct.unpack(Buf.ByteOrder.BIG_ENDIAN + Buf.UNSIGNED_LONG_LONG,  
 self.buf[self.read_index:self.read_index + 8])  
        self.read_index += 8  
 return ret[0]  
  
    def read_u64_le(self):  
        self.check_readable_bytes_len(8)  
        ret = struct.unpack(Buf.ByteOrder.LITTLE_ENDIAN + Buf.UNSIGNED_LONG_LONG,  
 self.buf[self.read_index:self.read_index + 8])  
        self.read_index += 8  
 return ret[0]  
  
    def read_f32(self):  
        self.check_readable_bytes_len(4)  
        ret = struct.unpack(Buf.ByteOrder.BIG_ENDIAN + Buf.FLOAT,  
 self.buf[self.read_index:self.read_index + 4])  
        self.read_index += 4  
 return ret[0]  
  
    def read_f32_le(self):  
        self.check_readable_bytes_len(4)  
        ret = struct.unpack(Buf.ByteOrder.LITTLE_ENDIAN + Buf.FLOAT,  
 self.buf[self.read_index:self.read_index + 4])  
        self.read_index += 4  
 return ret[0]  
  
    def read_f64(self):  
        self.check_readable_bytes_len(8)  
        ret = struct.unpack(Buf.ByteOrder.BIG_ENDIAN + Buf.DOUBLE,  
 self.buf[self.read_index:self.read_index + 8])  
        self.read_index += 8  
 return ret[0]  
  
    def read_f64_le(self):  
        self.check_readable_bytes_len(8)  
        ret = struct.unpack(Buf.ByteOrder.LITTLE_ENDIAN + Buf.DOUBLE,  
 self.buf[self.read_index:self.read_index + 8])  
        self.read_index += 8  
 return ret[0]  
  
    def read_bytes(self, length: int) -> bytearray:  
        self.check_readable_bytes_len(length)  
        ret = self.buf[self.read_index:self.read_index + length]  
        self.read_index += length  
        return ret
```

### **三、编写单元测试**

```
import os, sys  
  
sys.path.append(os.getcwd())  
from unittest import TestCase  
import pytest  
from buf.byte_buf import ByteBuf  
  
  
class TestByteBuf(TestCase):  
  
    def setUp(self) -> None:  
        self.buf = ByteBuf()  
  
    def test_byte_buf(self) -> int:  
        self.buf.write_i8(127)  
        self.buf.write_i8(127)  
        self.buf.write_i8(1)  
        self.buf.write_f32(1)  
        buf1 = ByteBuf(self.buf.to_bytes())  
        self.assertEqual(7, buf1.readable_bytes_len())  
  
    def test_readable_bytes_len(self) -> int:  
        self.buf.write_i8(127)  
        self.buf.write_i8(127)  
        self.buf.write_i8(1)  
        self.buf.write_f32(1)  
        self.assertEqual(7, self.buf.readable_bytes_len())  
  
    def test_to_bytes(self) -> bytes:  
        self.buf.write_i8(127)  
        self.buf.write_i8(127)  
        self.buf.write_i8(1)  
        self.assertEqual(b'\x7f\x7f\x01', self.buf.to_bytes())  
  
    def test_write_i8(self):  
        self.buf.write_i8(-128)  
        self.buf.write_i8(0)  
        self.buf.write_i8(127)  
        self.assertEqual(-128, self.buf.read_i8())  
        self.assertEqual(0, self.buf.read_i8())  
        self.assertEqual(127, self.buf.read_i8())  
  
    def test_write_i8_failed1(self):  
        with pytest.raises(Exception):  
            self.buf.write_i8(128)  
  
    def test_write_i8_failed2(self):  
        with pytest.raises(Exception):  
            self.buf.write_i8(-129)  
  
    def test_write_i8_failed3(self):  
        with pytest.raises(Exception):  
            self.buf.read_i8()  
  
    def test_write_u8(self):  
        self.buf.write_u8(0)  
        self.buf.write_u8(127)  
        self.buf.write_u8(255)  
        self.assertEqual(0, self.buf.read_u8())  
        self.assertEqual(127, self.buf.read_u8())  
        self.assertEqual(255, self.buf.read_u8())  
  
    def test_write_u8_failed1(self):  
        with pytest.raises(Exception):  
            self.buf.write_u8(-1)  
  
    def test_write_u8_failed2(self):  
        with pytest.raises(Exception):  
            self.buf.write_u8(256)  
  
    def test_write_bool(self):  
        self.buf.write_bool(True)  
        self.buf.write_bool(False)  
        self.buf.write_bool(1)  
        self.buf.write_bool(0)  
        self.buf.write_bool(22222)  
        self.assertTrue(self.buf.read_bool())  
        self.assertFalse(self.buf.read_bool())  
        self.assertTrue(self.buf.read_bool())  
        self.assertFalse(self.buf.read_bool())  
        self.assertTrue(self.buf.read_bool())  
  
    def test_write_i16(self):  
        self.buf.write_i16(-32768)  
        self.buf.write_i16(0)  
        self.buf.write_i16(127)  
        self.buf.write_i16(32767)  
        self.assertEqual(-32768, self.buf.read_i16())  
        self.assertEqual(0, self.buf.read_i16())  
        self.assertEqual(127, self.buf.read_i16())  
        self.assertEqual(32767, self.buf.read_i16())  
  
    def test_write_i16_failed1(self):  
        with pytest.raises(Exception):  
            self.buf.write_i16(-32769)  
  
    def test_write_i16_failed2(self):  
        with pytest.raises(Exception):  
            self.buf.write_i16(32768)  
  
    def test_write_i16_le(self):  
        self.buf.write_i16_le(-32768)  
        self.buf.write_i16_le(0)  
        self.buf.write_i16_le(127)  
        self.buf.write_i16_le(32767)  
        self.buf.write_i16_le(32767)  
        self.assertEqual(-32768, self.buf.read_i16_le())  
        self.assertEqual(0, self.buf.read_i16_le())  
        self.assertEqual(127, self.buf.read_i16_le())  
        self.assertEqual(32767, self.buf.read_i16_le())  
        self.assertNotEqual(32767, self.buf.read_i16())  
  
    def test_write_u16(self):  
        self.buf.write_u16(0)  
        self.buf.write_u16(127)  
        self.buf.write_u16(65535)  
        self.assertEqual(0, self.buf.read_u16())  
        self.assertEqual(127, self.buf.read_u16())  
        self.assertEqual(65535, self.buf.read_u16())  
  
    def test_write_u16_failed1(self):  
        with pytest.raises(Exception):  
            self.buf.write_u16(-1)  
  
    def test_write_u16_failed2(self):  
        with pytest.raises(Exception):  
            self.buf.write_u16(65536)  
  
    def test_write_u16_le(self):  
        self.buf.write_u16_le(0)  
        self.buf.write_u16_le(127)  
        self.buf.write_u16_le(65535)  
        self.buf.write_u16_le(65534)  
        self.assertEqual(0, self.buf.read_u16_le())  
        self.assertEqual(127, self.buf.read_u16_le())  
        self.assertEqual(65535, self.buf.read_u16_le())  
        self.assertNotEqual(65534, self.buf.read_u16())  
  
    def test_write_i32(self):  
        self.buf.write_i32(-2 ** 31)  
        self.buf.write_i32(0)  
        self.buf.write_i32(127)  
        self.buf.write_i32(127)  
        self.buf.write_i32(2 ** 31 - 1)  
        self.assertEqual(-2 ** 31, self.buf.read_i32())  
        self.assertEqual(0, self.buf.read_i32())  
        self.assertEqual(127, self.buf.read_i32())  
        self.assertNotEqual(127, self.buf.read_i32_le())  
        self.assertEqual(2 ** 31 - 1, self.buf.read_i32())  
  
    def test_write_i32_le(self):  
        self.buf.write_i32_le(-2 ** 31)  
        self.buf.write_i32_le(0)  
        self.buf.write_i32_le(127)  
        self.buf.write_i32_le(127)  
        self.buf.write_i32_le(2 ** 31 - 1)  
        self.assertEqual(-2 ** 31, self.buf.read_i32_le())  
        self.assertEqual(0, self.buf.read_i32_le())  
        self.assertEqual(127, self.buf.read_i32_le())  
        self.assertNotEqual(127, self.buf.read_i32())  
        self.assertEqual(2 ** 31 - 1, self.buf.read_i32_le())  
  
    def test_write_u32(self):  
        self.buf.write_u32(0)  
        self.buf.write_u32(127)  
        self.buf.write_u32(127)  
        self.buf.write_u32(2 ** 32 - 1)  
        self.assertEqual(0, self.buf.read_u32())  
        self.assertEqual(127, self.buf.read_u32())  
        self.assertNotEqual(127, self.buf.read_u32_le())  
        self.assertEqual(2 ** 32 - 1, self.buf.read_u32())  
  
    def test_write_u32_failed1(self):  
        with pytest.raises(Exception):  
            self.buf.write_u32(-1)  
  
    def test_write_u32_failed2(self):  
        with pytest.raises(Exception):  
            self.buf.write_u32(2 ** 32)  
  
    def test_write_u32_le(self):  
        self.buf.write_u32_le(0)  
        self.buf.write_u32_le(127)  
        self.buf.write_u32_le(127)  
        self.buf.write_u32_le(2 ** 32 - 1)  
        self.assertEqual(0, self.buf.read_u32_le())  
        self.assertEqual(127, self.buf.read_u32_le())  
        self.assertNotEqual(127, self.buf.read_u32())  
        self.assertEqual(2 ** 32 - 1, self.buf.read_u32_le())  
  
    def test_write_i64(self):  
        self.buf.write_i64(- 2 ** 63)  
        self.buf.write_i64(0)  
        self.buf.write_i64(127)  
        self.buf.write_i64(127)  
        self.buf.write_i64(2 ** 63 - 1)  
        self.assertEqual(-2 ** 63, self.buf.read_i64())  
        self.assertEqual(0, self.buf.read_i64())  
        self.assertEqual(127, self.buf.read_i64())  
        self.assertNotEqual(127, self.buf.read_i64_le())  
        self.assertEqual(2 ** 63 - 1, self.buf.read_i64())  
  
    def test_write_i64_le(self):  
        self.buf.write_i64_le(- 2 ** 63)  
        self.buf.write_i64_le(0)  
        self.buf.write_i64_le(127)  
        self.buf.write_i64_le(127)  
        self.buf.write_i64_le(2 ** 63 - 1)  
        self.assertEqual(-2 ** 63, self.buf.read_i64_le())  
        self.assertEqual(0, self.buf.read_i64_le())  
        self.assertEqual(127, self.buf.read_i64_le())  
        self.assertNotEqual(127, self.buf.read_i64())  
        self.assertEqual(2 ** 63 - 1, self.buf.read_i64_le())  
  
    def test_write_u64(self):  
        self.buf.write_i64_le(- 2 ** 63)  
        self.buf.write_i64_le(0)  
        self.buf.write_i64_le(127)  
        self.buf.write_i64_le(127)  
        self.buf.write_i64_le(2 ** 63 - 1)  
        self.assertEqual(-2 ** 63, self.buf.read_i64_le())  
        self.assertEqual(0, self.buf.read_i64_le())  
        self.assertEqual(127, self.buf.read_i64_le())  
        self.assertNotEqual(127, self.buf.read_i64())  
        self.assertEqual(2 ** 63 - 1, self.buf.read_i64_le())  
  
    def test_write_u64_le(self):  
        self.buf.write_u64_le(0)  
        self.buf.write_u64_le(127)  
        self.buf.write_u64_le(127)  
        self.buf.write_u64_le(2 ** 64 - 1)  
        self.assertEqual(0, self.buf.read_u64_le())  
        self.assertEqual(127, self.buf.read_u64_le())  
        self.assertNotEqual(127, self.buf.read_u64())  
        self.assertEqual(2 ** 64 - 1, self.buf.read_u64_le())  
  
    def test_write_f32(self):  
        self.buf.write_f32(0)  
        self.buf.write_f32(127)  
        self.buf.write_f32(127)  
        self.buf.write_f32(12.0)  
        self.assertEqual(0, self.buf.read_f32())  
        self.assertEqual(127, self.buf.read_f32())  
        self.assertNotEqual(127, self.buf.read_f32_le())  
        self.assertEqual(12.0, self.buf.read_f32())  
  
    def test_write_f32_le(self):  
        self.buf.write_f32_le(0)  
        self.buf.write_f32_le(127)  
        self.buf.write_f32_le(127)  
        self.buf.write_f32_le(12.0)  
        self.assertEqual(0, self.buf.read_f32_le())  
        self.assertEqual(127, self.buf.read_f32_le())  
        self.assertNotEqual(127, self.buf.read_f32())  
        self.assertEqual(12.0, self.buf.read_f32_le())  
  
    def test_write_f64(self):  
        self.buf.write_f64(0)  
        self.buf.write_f64(127)  
        self.buf.write_f64(127)  
        self.buf.write_f64(12.0)  
        self.assertEqual(0, self.buf.read_f64())  
        self.assertEqual(127, self.buf.read_f64())  
        self.assertNotEqual(127, self.buf.read_f64_le())  
        self.assertEqual(12.0, self.buf.read_f64())  
  
    def test_write_f64_le(self):  
        self.buf.write_f64_le(0)  
        self.buf.write_f64_le(127)  
        self.buf.write_f64_le(127)  
        self.buf.write_f64_le(12.0)  
        self.assertEqual(0, self.buf.read_f64_le())  
        self.assertEqual(127, self.buf.read_f64_le())  
        self.assertNotEqual(127, self.buf.read_f64())  
        self.assertEqual(12.0, self.buf.read_f64_le())  
  
    def test_write_bytes(self):  
        self.buf.write_bytes(b'hello')  
        self.assertEqual(b'hel', self.buf.read_bytes(3))  
        self.assertEqual(b'lo', self.buf.read_bytes(2))
```

执行单元测试
```
buf\test_byte_buf.py ..................................                                                                                                                                                  [100%]

============================================================================================= 34 passed in 0.60s ==============================================================================================


```

### **四、总结**：

该ByteBuf不考虑线程安全，仅提供了顺序读写的能力，未来在实际使用过程中根据实际完善，比如随机读写能力，基于多个ByteBuf进行Wrapper组合构建新的ByteBuf等。

参考文档：  
https://netty.io/4.0/api/io/netty/buffer/ByteBuf.html
https://docs.oracle.com/javase/8/docs/api/java/nio/ByteBuffer.html
https://docs.rs/bytes/1.1.0/bytes/

  

### *下期预告*：基于Bytebuf实现对交易所交易接口消息的封装