## Bitmap介绍
Bitmap在Android中指的是一张图片，可以是png格式也可以是jpg等其他常见的图片格式。

那么如何加载一个图片呢？     
BitmapFactory类提供了四类方法：decodeFile、decodeResource、decodeStream、decodeByteArray。   
分别用于支持从文件系统、资源、输入流以及字节数组中加载出一个Bitmap对象。 其中decodeFile和decodeResource又间接调用了decodeStream方法。  
