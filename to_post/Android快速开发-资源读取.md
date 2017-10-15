---
title: Android快速开发-资源读取  
date: 2017-08-15 23:06:34
comments: true
categories: Android
tags: [Android源码]
---

>项目中频繁使用的一些资源读取方式，做快速开发时遗忘查阅。持续更新...

### Tip1 拼接drawable名字获取图片
```
public Bitmap getBitmapByName(String name) {  
        ApplicationInfo appInfo = getApplicationInfo();  
        int resID = getResources().getIdentifier(name, "drawable", appInfo.packageName);  
        return BitmapFactory.decodeResource(getResources(), resID);  
    }  
```  
 
### Tip2 获取/res/values/arrays.xml下的数据
```
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string-array name="texts">
        <item>text1</item>
        <item>text2</item>
        <item>text3</item>
    </string-array>
</resources>
String strs[] = getResources().getStringArray(R.array.flavors);
```

### Tip3 获取简单纯颜色Drawable
```
除了写shape，也可以直接写
<?xml version="1.0" encoding="UTF-8"?>
<resources>
    <drawable name="red_rect">#F00</drawable>
</resources>
ColorDrawable myDraw = (ColorDrawable)getResources().getDrawable(R.drawable.red_rect);
```

### Tip4 点9图技巧及获取
```
左上拉伸一像素,右下限定拉伸域
NinePatchDrawable bitmapFlag = (NinePatchDrawable)getResources().getDrawable(R.drawable.nicePic);
```

### Tip5 加载xml动画
```
    public static Animation loadAnimation(Context context, int id){
        Animation myAnimation= AnimationUtils.loadAnimation(this,R.anim.animation);
    }
```

### Tip6 加载原始文件   
*  **/res/raw**  
该目录下，受Android平台约束，不会编译成二进制文件但会生成R文件，访问方式通过资源ID或者直接读取目录下文件数据。  

``` 
 	private String readStream(InputStream is) {
        try {
            ByteArrayOutputStream bo = new ByteArrayOutputStream();
             int i = is.read();
             while(i != -1) {
                 bo.write(i);
                 i = is.read();
            }   
             return bo.toString();
        } catch (IOException e) {
            return null;
        }
     }

	readStream(getResources().openRawResource(R.raw.rawtext)

     public String getFromRaw(String fileName){   
         String res = "";  
          
         try{ 
             InputStream in = getResources().openRawResource(R.raw.test1);   
             int length = in.available();        
             byte [] buffer = new byte[length];         
             in.read(buffer);          
             res = EncodingUtils.getString(buffer, "UTF-8");     
             in.close();
         }catch(Exception e){  
              e.printStackTrace();          
         }  
          
         return res ;  
     }

```
*  **/assets**   
该目录却不受Android限制，不会编译成二进制文件且只能通过文件名访问，不能通过ID

```
直接读取/assets下txt文件
AssetManager assets = getAssets(); 
((TextView)findViewById(R.id.textview)).setText(readStream(assets.open("assetsText.txt")));

读取/assets下文件的路径
String filePath = "file:///android_asset/文件名";
或者先读取为输入流再转化为string格式
InputStream abpath = getClass().getResourceAsStream("/assets/文件名");
private byte[] InputStreamToByte(InputStream is) throws IOException {
    ByteArrayOutputStream bytestream = new ByteArrayOutputStream();
    int ch;
          
     while ((ch = is.read()) != -1) {
         bytestream.write(ch);
     }
          
     byte imgdata[] = bytestream.toByteArray();
     bytestream.close();
     return imgdata;
}
String filePath = new String(InputStreamToByte(is));
```


