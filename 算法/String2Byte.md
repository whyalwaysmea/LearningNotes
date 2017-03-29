我们知道，在java中，一个byte 就是一个字节，也就是八个二进制位；而4个二进制位就可以表示一个十六进制位，所以一个byte可以转化为2个十六进制位。

在java中，byte的范围是-128~127

```java
/**
 * 16进制字符串转换为字符串
 *  
 * @param s
 * @return
 */  
public static String hexStringToString(String s) {  
    if (s == null || s.equals("")) {  
        return null;  
    }  
    s = s.replace(" ", "");  
    byte[] baKeyword = new byte[s.length() / 2];  
    for (int i = 0; i < baKeyword.length; i++) {  
        try {  
            baKeyword[i] = (byte) (0xff & Integer.parseInt(  
                    s.substring(i * 2, i * 2 + 2), 16));  
        } catch (Exception e) {  
            e.printStackTrace();  
        }  
    }  
    try {  
        s = new String(baKeyword, "gbk");  
        new String();  
    } catch (Exception e1) {  
        e1.printStackTrace();  
    }  
    return s;  
}  

/**
 * 字符串转十六进制字符串
 * @param s
 * @return
 */
public static String stringToHexString(String s) {  
    String str = "";  
    for (int i = 0; i < s.length(); i++) {  
        int ch = (int) s.charAt(i);  
        String s4 = Integer.toHexString(ch);  
        str = str + s4;  
    }  
    return str;  
}  

/**
 * 十六进制字符串转byte数组
 * @param hexString
 * @return
 */
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
