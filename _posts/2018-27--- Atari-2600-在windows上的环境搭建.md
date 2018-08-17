#前言
昨天看到了敖厂长的视频，[->【烂尾的游戏冒险(雅达利寻剑)】<-](https://www.bilibili.com/video/av26725241), 里面提到了烂尾的雅达利寻剑游戏，敖厂长在片尾振臂一呼，要在雅达利2600上自行补完这个烂尾的游戏。
我非常敬佩于敖厂长的学习能力，吐槽能力和实践能力，所以就花了些时间在自己的电脑上构筑了一个雅达利2600的编程环境。希望对小伙伴们有帮助。

#事前准备
1. bB环境 （batari Basic） 
    下载链接：http://7800.8bitdev.org/index.php/Batari_basic
  
  在Atari 2600上编程需要的类库。
  解压至某一路径即可（路径不要有中文和空格），放置备用。

2. visual bB 环境
     下载链接：http://atariage.com/forums/topic/123849-visual-bb-1-0-a-new-ide-for-batari-basic/

在Atari 2600上编程时候的IDE
 解压至某一路径即可（路径不要有中文和空格），放置备用。

3. 编译Stella模拟器
    3-1， 下载Visual C++ 2017 Community   ，请去官网下载。
              理论上其他的C++ 平台也行，但是需要修改平台工具集，比较麻烦。
              Visual C++ 2017 Community  比较大，需要一些耐心
    3-2， 从github上down下来最新代码。
            连接：https://github.com/stella-emu/stella
    3-3， 下载SDL ， 
            下载链接：http://www.libsdl.org/download-2.0.php
             我下载的是 SDL2-devel-2.0.8-VC.zip，解压后，将文件夹名修改为 SDL，下一级目录应该是include，lib.    将这个SDL文件夹复制粘贴到 3-2下载后的代码的 .src\windows 下，

    3-4， 使用 VC++ 2017 打开 .src\windows 下的 Stella.sln ，然后点击编译。
          这个过程会很慢，请等待。
    3-5， 编译完成之后    
          对于32位系统，复制SDL\lib\x86\SDL2.dll 到 Release 或者 Debug
          对于64位系统，复制 SDL\lib\x64\SDL2.dll 到 x64\Release 或者 x64\Debug

4.  配置 visual bB 的 IDE，如图：**注意一定要勾选红框**
      ![配置图](https://upload-images.jianshu.io/upload_images/3611412-e9ff60105f5155fb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

5. 新建一个工程，将样例代码并将样例代码粘贴进去。
    样例代码的连接：http://atariage.com/forums/topic/109288-code-snippets-samples-for-bb-beginners/
  ![样例代码](https://upload-images.jianshu.io/upload_images/3611412-09857c24ebc6906b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

6. 点击编译，再点击run

#效果
![样例1_DrawSprite.bas](https://upload-images.jianshu.io/upload_images/3611412-9c80dd44b1517c1f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![样例16_MoveSpritewithEnemySpriteFollowing.bas](https://upload-images.jianshu.io/upload_images/3611412-cfbdbf9dad9c82bc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

大功告成，就等敖厂长的github上上传工程了。

#参考链接
https://stella-emu.github.io/docs/index.html  《Stella：A multi-platform Atari 2600 VCS emulator》
https://stella-emu.github.io/development.html 《如何编译Stella》
http://atariage.com/forums/topic/109288-code-snippets-samples-for-bb-beginners/ 《如何在雅达利平台上编程》
