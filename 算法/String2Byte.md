我们知道，在java中，一个byte 就是一个字节，也就是八个二进制位；而4个二进制位就可以表示一个十六进制位，所以一个byte可以转化为2个十六进制位。

在java中，byte的范围是-128~127

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

```java
public static byte[] hexStringToBytes(String hexString) {
	if (hexString == null || hexString.equals("")) {
		return null;
	}
	hexString = hexString.toUpperCase();
	int length = hexString.length() / 2;
	char[] hexChars = hexString.toCharArray();
	byte[] d = new byte[length];
	for (int i = 0; i < length; i++) {
		int pos = i * 2;
		d[i] = (byte) (charToByte(hexChars[pos]) << 4 | charToByte(hexChars[pos + 1]));
	}
	return d;
}

private static byte charToByte(char c) {
	return (byte) "0123456789ABCDEF".indexOf(c);
}
```
