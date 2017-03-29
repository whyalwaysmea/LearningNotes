我们知道，在java中，一个byte 就是一个字节，也就是八个二进制位；而4个二进制位就可以表示一个十六进制位，所以一个byte可以转化为2个十六进制位。

```java
public static String toHex(byte b) {  
    // 返回为无符号整数基数为16的整数参数的字符串表示形式
    String result = Integer.toHexString(b & 0xFF);  
    if (result.length() == 1) {  
        result = '0' + result;  
    }  
    return result;  
}  
```
