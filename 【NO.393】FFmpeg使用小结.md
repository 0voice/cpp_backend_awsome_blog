# 【NO.393】FFmpeg使用小结

## 1.视频播放器的原理

**封装格式**

作用：视频码流和音频码流按照一定的格式储存在一个文件汇总

**视频编码数据**

作用：将视频像素数据（RGB，YUV等）压缩成为视频码流，从而降低视频的数据量

**音频编码数据**

作用：将音频采样数据（PCM等）压缩成为音频码流，从而降低音频的数据量

**视频像素数据**

作用：保存了屏幕上每一个像素点的像素值

格式：常见的像素数据格式有RGB24, RGB32, YUV420P,YUV422P,YUV444P等。压缩编码中一般使用的是YUV格式的像素数据,最为常见的格式为YUV420P。

特点：视频像素数据体积很大，一般情况下一小时高清视频的RGB24格式的数据体积为:3600*25*1920*1080*3=559.9GB（PS：这里假定帧率为25HZ，取样精度8bit）

**音频采样数据**

作用：保存了音频中每个采样点的值。

特点：音频采样数据体积很大,一般情况下一首4分钟的PCM格式的歌曲体积为:4*60*44100*2*2=42.3MByte
PS:这里假定采样率为44100Hz,采样精度为16bit

1 术语：

什么是影片？其实就是一组（很多张）图片，时间间隔很小的连续展示出来，人们就觉得画面中的人物在动，这就是影片。那电影的实质就是N多张图片的集合。那 每张图片和帧又有什么关系呢？事实上，如果一部影片里面的图片，我们原封不动的全部存起来，空间会很大很大很大，但是如果通过一定的算法（这里不讲相关算 法），把每一张图片压缩（编码_encode）一下，变成 帧。再把帧连起来变成流，再把不同的流放到某个容器里面，这就是我们平常看见的电影文件了，文件 碟中谍4.H264.ACC.mkv，他为什么要这样命名呢？ mkv表达了它的容器是.mkv的，且包含至少两个流，h264的视频流，ACC的音频流。这是一种典型的 牺牲时间来换取空间的做法。

容器(Container)——容器就是一种文件格式，比如flv，mkv等。包含下面5种流以及文件头信息。

流(Stream)——是一种视频数据信息的传输方式，5种流：音频，视频，字幕，附件，数据。

帧(Frame)——帧代表一幅静止的图像，分为I帧，P帧，B帧。

编解码器(Codec)——是对视频进行压缩或者解压缩，CODEC =COde （编码） +DECode（解码）

复用/解复用(mux/demux)——把不同的流按照某种容器的规则放入容器，这种行为叫做复用（mux）

把不同的流从某种容器中解析出来，这种行为叫做解复用(demux)

视频压缩中，每帧代表一幅静止的图像。而在实际压缩时，会采取各种算法减少数据的容量，其中IPB就是最常见的。

（1）I帧表示关键帧，你可以理解为这一帧画面的完整保留；解码时只需要本帧数据就可以完成（因为包含完整画面）

（2）P帧表示的是这一帧跟之前的一个关键帧（或P帧）的差别，解码时需要用之前缓存的画面叠加上本帧定义的差别，生成最终画面。（也就是差别帧，P帧没有完整画面数据，只有与前一帧的画面差别的数据）

（3）B帧是双向差别帧，也就是B帧记录的是本帧与前后帧的差别（具体比较复杂，有4种情况），换言之，要解码B帧，不仅要取得之前的缓存画面，还要解码之后的画面，通过前后画面的与本帧数据的叠加取得最终的画面。B帧压缩率高，但是解码时CPU会比较累。

从上面的解释看，我们知道I和P的解码算法比较简单，资源占用也比较少，I只要自己完成就行了，P呢，也只需要解码器把前一个画面缓存一下，遇到P时就使用之前缓存的画面就好了，如果视频流只有I和P，解码器可以不管后面的数据，边读边解码，线性前进，大家很舒服。但网络上的电影很多都采用了B帧，因为B帧记录的是前后帧的差别，比P帧能节约更多的空间，但这样一来，文件小了，解码器就麻烦了，因为在解码时，不仅要 用之前缓存的画面，还要知道下一个I或者P的画面（也就是说要预读预解码），而且，B帧不能简单地丢掉，因为B帧其实也包含了画面信息，如果简单丢掉，并 用之前的画面简单重复，就会造成画面卡（其实就是丢帧了），并且由于网络上的电影为了节约空间，往往使用相当多的B帧，B帧用的多，对不支持B帧的播放器 就造成更大的困扰，画面也就越卡。 一般平均来说，I的压缩率是7（跟JPG差不多），P是20，B可以达到50，可见使用B帧能节省大量空间，节省出来的空间可以用来保存多一些I帧，这样在相同码率下，可以提供更好的画质。

附: [编解码过程](https://link.zhihu.com/?target=http%3A//blog.csdn.net/abcjennifer/article/details/6577934)

![img](https://pic4.zhimg.com/80/v2-d5de0a11ab01fa5ed2f3b99b41f73beb_720w.webp)

\1. 注册所有容器格式和CODEC:av_register_all()

\2. 打开文件:av_open_input_file()

\3. 从文件中提取流信息:av_find_stream_info()

\4. 穷举所有的流，查找其中种类为CODEC_TYPE_VIDEO

\5. 查找对应的解码器:avcodec_find_decoder()

\6. 打开编解码器:avcodec_open()

\7. 为解码帧分配内存:avcodec_alloc_frame()

\8. 不停地从码流中提取出帧数据:av_read_frame()

\9. 判断帧的类型，对于视频帧调用:avcodec_decode_video()

\10. 解码完后，释放解码器:avcodec_close()

\11. 关闭输入文件:av_close_input_file()

------

## 2. 简介（[http://derekzhan.iteye.com/blog/1989274](https://link.zhihu.com/?target=http%3A//derekzhan.iteye.com/blog/1989274)）

**资料**

FFmpeg官网： [http://www.ffmpeg.org](https://link.zhihu.com/?target=http%3A//www.ffmpeg.org)

FFmpeg doc : [http://www.ffmpeg.org/documentation.html](https://link.zhihu.com/?target=http%3A//www.ffmpeg.org/documentation.html)

FFmpeg wiki : [https://trac.ffmpeg.org/wiki](https://link.zhihu.com/?target=https%3A//trac.ffmpeg.org/wiki)

FFmpeg基础: [http://wenku.baidu.com/view/296eefcaf90f76c661371af1.html](https://link.zhihu.com/?target=http%3A//wenku.baidu.com/view/296eefcaf90f76c661371af1.html)

FFmpeg的名称来自MPEG视频编码标准，前面的“FF”代表“Fast Forward”，FFmpeg是一套可以用来记录、转换数字音频、视频，并能将其转化为流的开源计算机程序。可以轻易地实现多种视频格式之间的相互转换。FFmpeg的用户有Google，Facebook，Youtube，优酷，爱奇艺，土豆等。

**组成**

1、libavformat：用于各种音视频封装格式的生成和解析，包括获取解码所需信息以生成解码上下文结构和读取音视频帧等功能，包含demuxers和muxer库；

2、libavcodec：用于各种类型声音/图像编解码；

3、libavutil：包含一些公共的工具函数；

4、libswscale：用于视频场景比例缩放、色彩映射转换；

5、libpostproc：用于后期效果处理；

6、ffmpeg：是一个命令行工具，用来对视频文件转换格式，也支持对电视卡实时编码；

7、ffsever：是一个HTTP多媒体实时广播流服务器，支持时光平移；

8、ffplay：是一个简单的播放器，使用ffmpeg 库解析和解码，通过SDL显示；

**2.1 过滤器(Filter)**

在多媒体处理中，filter的意思是被编码到输出文件之前用来修改输入文件内容的一个软件工具。如：视频翻转，旋转，缩放等。

语法：[input_link_label1][input_link_label2]… filter_name=parameters [output_link_label1][output_link_label2]…

过滤器图link label ：是标记过滤器的输入或输出的名称

（1）.视频过滤器 -vf

如testsrc视频按顺时针方向旋转90度　　ffplay -f lavfi -i testsrc -vf transpose=1

如testsrc视频水平翻转(左右翻转)　　ffplay -f lavfi -i testsrc -vf hflip

（2）.音频过滤器 -af

实现慢速播放，声音速度是原始速度的50%　　ffplay p629100.mp3 -af atempo=0.5

（3）如何实现顺时针旋转90度并水平翻转？

过滤器链（Filterchain）

基本语法 Filterchain = 逗号分隔的一组filter

语法：“filter1,filter2,filter3,…filterN-2,filterN-1,filterN”

顺时针旋转90度并水平翻转 ffplay -f lavfi -i testsrc -vf transpose=1,hflip

（4）如何实现水平翻转视频和源视频进行比较？

方法一: 过滤器链（Filterchain）

第一步： 源视频宽度扩大两倍　　ffmpeg -i jidu.mp4 -t 10 -vf pad=2*iw output.mp4

第二步：源视频水平翻转　　ffmpeg -i jidu.mp4 -t 10 -vf hflip output2.mp4

第三步：水平翻转视频覆盖output.mp4　　ffmpeg -i output.mp4 -i output2.mp4 -filter_complex overlay=w compare.mp4

方法二：过滤器图（Filtergraph）

基本语法 :Filtergraph = 分号分隔的一组filterchain

“filterchain1;filterchain2;…filterchainN-1;filterchainN”

用ffplay直接观看结果：fplay -f lavfi -i testsrc -vf split[a][b];[a]pad=2*iw[1];[b]hflip[2];[1][2]overlay=w

F1: split过滤器创建两个输入文件的拷贝并标记为[a],[b]

F2: [a]作为pad过滤器的输入，pad过滤器产生2倍宽度并输出到[1].

F3: [b]作为hflip过滤器的输入，vflip过滤器水平翻转视频并输出到[2].

F4: 用overlay过滤器把 [2]覆盖到[1]的旁边.

**2.2 选择媒体流**

一些多媒体容器比如AVI，mkv，mp4等，可以包含不同种类的多个流，如何从容器中抽取各种流呢？

语法：-map file_number:stream_type[:stream_number]

这有一些特别流符号的说明：

1、-map 0 选择第一个文件的所有流

2、-map i:v 从文件序号i(index)中获取所有视频流， -map i:a 获取所有音频流，-map i:s 获取所有字幕流等等。

3、特殊参数-an,-vn,-sn分别排除所有的音频，视频，字幕流。注意:文件序号和流序号从0开始计数。

**2.3 查看帮助**

可用的bit流 ：ffmpeg –bsfs

可用的编解码器：ffmpeg –codecs

可用的解码器：ffmpeg –decoders

可用的编码器：ffmpeg –encoders

可用的过滤器：ffmpeg –filters

可用的视频格式：ffmpeg –formats

可用的声道布局：ffmpeg –layouts

可用的license：ffmpeg –L

可用的像素格式：ffmpeg –pix_fmts

可用的协议：ffmpeg -protocals

**2.4 码率、帧率和文件大小**

码率和帧率是视频文件的最重要的基本特征，对于他们的特有设置会决定视频质量。如果我们知道码率和时长那么可以很容易计算出输出文件的大小。

帧率：帧率也叫帧频率，帧率是视频文件中每一秒的帧数，肉眼想看到连续移动图像至少需要15帧。 fps

码率：比特率(也叫码率，数据率)是一个确定整体视频/音频质量的参数，秒为单位处理的字节数，码率和视频质量成正比，在视频文件中中比特率用bps来表达

**帧率**

1、用 -r 参数设置帧率 （fps 每秒传输帧数(Frames Per Second)）

ffmpeg –i input –r fps output

2、用fps filter设置帧率

ffmpeg -i clip.mpg -vf fps=fps=25 clip.webm

例如设置帧率为29.97fps，下面三种方式具有相同的结果：

ffmpeg -i input.avi -r 29.97 output.mpg

ffmpeg -i input.avi -r 30000/1001 output.mpg

ffmpeg -i input.avi -r netsc output.mpg



**码率、文件大小**

设置码率 –b 参数　　ffmpeg -i film.avi -b 1.5M film.mp4

音频：-b:a 视频： - b:v　　设置视频码率为1500kbps　　ffmpeg -i input.avi -b:v 1500k output.mp4

**控制输出文件大小**

-fs (file size首字母缩写) 　　ffmpeg -i input.avi -fs 1024K output.mp4

计算输出文件大小　　(视频码率+音频码率) * 时长 /8 = 文件大小K

------

## 3. 调整视频分辨率

**3.1 调整视频分辨率**

用-s参数设置视频分辨率，参数值wxh，w宽度单位是像素，h高度单位是像素　　　　ffmpeg -i input_file -s 320x240 output_file

**3.2 预定义的视频尺寸**

下面两条命令有相同效果

ffmpeg -i input.avi -s 640x480 output.avi

ffmpeg -i input.avi -s vga output.avi

Scale filter调整分辨率 ，Scale filter的优点是可以使用一些额外的参数 ， 语法： Scale=width:height[:interl={1|-1}]

下面两条命令有相同效果

ffmpeg -i input.mpg -s 320x240 output.mp4

ffmpeg -i input.mpg -vf scale=320:240 output.mp4

对输入视频成比例缩放

改变为源视频一半大小　 　ffmpeg -i input.mpg -vf scale=iw/2:ih/2 output.mp4

改变为原视频的90%大小 ffmpeg -i input.mpg -vf scale=iw*0.9:ih*0.9 output.mp4

在未知视频的分辨率时，保证调整的分辨率与源视频有相同的横纵比。

例如宽度固定400，高度成比例：

ffmpeg -i input.avi -vf scale=400:400/a

ffmpeg -i input.avi -vf scale=400:-1

相反地，高度固定300，宽度成比例：

ffmpeg -i input.avi -vf scale=-1:300

ffmpeg -i input.avi -vf scale=300*a:300

------

## 4. 裁剪/填充视频

**4.1 裁剪视频crop filter**

从输入文件中选取你想要的矩形区域到输出文件中,常见用来去视频黑边。 语法：crop:ow[:oh[:x[:y:[:keep_aspect]]]]

裁剪输入视频的左三分之一，中间三分之一，右三分之一:

ffmpeg -i input -vf crop=iw/3:ih :0:0 output

ffmpeg -i input -vf crop=iw/3:ih :iw/3:0 output

ffmpeg -i input -vf crop=iw/3:ih :iw/3*2:0 output

裁剪帧的中心, 当我们想裁剪区域在帧的中间时，裁剪filter可以跳过输入x和y值，他们的默认值是

Xdefault = ( input width - output width)/2 , Ydefault = ( input height - output height)/2

ffmpeg -i input_file -v crop=w:h output_file

裁剪中间一半区域： ffmpeg -i input.avi -vf crop=iw/2:ih/2 output.avi

比较裁剪后的视频和源视频比较

ffplay -i jidu.mp4 -vf split[a][b];[a]drawbox=x=(iw-300)/2:(ih-300)/2:w=300:h=300:c=yellow[A];[A]pad=2*iw[C];[b]crop=300:300:(iw-300)/2:(ih-300)/2[B];　　

[C][B]overlay=w*2.4:40

自动检测裁剪区域 , cropdetect filter 自动检测黑边区域

ffplay jidu.mp4 -vf cropdetect

然后用检测到的值来裁剪视频

ffplay jidu.mp4 –vf crop=672:272:0:54

填充视频(pad) , 在视频帧上增加一快额外额区域，经常用在播放的时候显示不同的横纵比　　语法：pad=width[:height:[:x[:y:[:color]]]]

创建一个30个像素的粉色宽度来包围一个SVGA尺寸的图片： ffmpeg -i photo.jpg -vf pad=860:660:30:30:pink framed_photo.jpg

同理可以制作testsrc视频用30个像素粉色包围视频:　　ffplay -f lavfi -i testsrc -vf pad=iw+60:ih+60:30:30:pink

4:3到16:9---------- 一些设备只能播放16:9的横纵比，4:3的横纵比必须在水平方向的两边填充成16:9，高度被保持，宽度等于高度乘以16/9，x（输入文件水平位移）值由表达式(output_width - input_width)/2来计算。

4：3到16:9的通用命令是：ffmpeg -i input -vf pad=ih*16/9:ih :(ow-iw)/2:0:color output

举例： ffplay -f lavfi -i testsrc -vf pad=ih*16/9:ih:(ow-iw)/2:0:pink

16:9到4:3----------为了用4:3的横纵比来显示16:9的横纵比，填充输入文件的垂直两边，宽度保持不变，高度是宽度的3/4，y值（输入文件的垂直偏移量）是由一个表达式（output_height-input_height）/2计算出来的。

16:9到4:3的通用命令：ffmpeg -i input -vf pad=iw :iw*3/4:0:(oh-ih)/2:color output

举例：ffplay -f lavfi -i testsrc=size=320x180 -vf pad=iw:iw*3/4:0:(oh-ih)/2:pink

------

## 5. 翻转和旋转

**翻转**

水平翻转语法： -vf hflip：ffplay -f lavfi -i testsrc -vf hflip

垂直翻转语法：-vf vflip：ffplay -f lavfi -i testsrc -vf vflip

**旋转**

语法：transpose={0,1,2,3}

　　0:逆时针旋转90°然后垂直翻转

　　1:顺时针旋转90°

　　2:逆时针旋转90°

　　3:顺时针旋转90°然后水平翻转

------

## 6. 模糊，锐化

模糊, 语法：boxblur=luma_r:luma_p[:chroma_r:chram_p[:alpha_r:alpha_p]]　, fplay -f lavfi -i testsrc -vf boxblur=1:10:4:10

注意：luma_r和alpha_r半径取值范围是0~min(w,h)/2, chroma_r半径的取值范围是0~min(cw/ch)/2

锐化, 语法：-vf unsharp=l_msize_x:l_msize_y:l_amount:c_msize_x:c_msize_y:c_amount, 所有的参数是可选的，默认值是5:5:1.0:5:5:0.0

l_msize_x:水平亮度矩阵，取值范围3-13，默认值为5

l_msize_y:垂直亮度矩阵，取值范围3-13，默认值为5

l_amount:亮度强度，取值范围-2.0-5.0，负数为模糊效果，默认值1.0

c_msize_x:水平色彩矩阵，取值范围3-13，默认值5

c_msize_y:垂直色彩矩阵，取值范围3-13，默认值5

c_amount:色彩强度，取值范围-2.0-5.0，负数为模糊效果，默认值0.0

**举例**

使用默认值，亮度矩阵为5x5和亮度值为1.0　　ffmpeg -i input -vf unsharp output.mp4

高斯模糊效果(比较强的模糊)：　　ffplay -f lavfi -i testsrc -vf unsharp=13:13:-2

------

## **7. 覆盖（画中画）**

覆盖, 语法：overlay[=x[:y], 所有的参数都是可选，默认值都是0

**举例**

Logo在左上角:ffmpeg -i pair.mp4 -i logo.png -filter_complex overlay pair1.mp4

右上角：ffmpeg -i pair.mp4 -i logo.png -filter_complex overlay=W-w pair2.mp4

左下角：ffmpeg -i pair.mp4 -i logo.png -filter_complex overlay=0:H-h pair2.mp4

右下角： ffmpeg -i pair.mp4 -i logo.png -filter_complex overlay=W-w:H-h pair2.mp4

删除logo,语法：-vf delogo=x:y:w:h[:t[:show]]

x:y 离左上角的坐标

w:h logo的宽和高

t: 矩形边缘的厚度默认值4

show：若设置为1有一个绿色的矩形，默认值0.

ffplay -i jidu.mp4 -vf delogo=50:51:60:60:100:0

------

## 8. 添加文本

语法：drawtext=fontfile=font_f:text=text1[:p3=v3[:p4=v4[…]]]

**常用的参数值**

x：离左上角的横坐标，y: 离左上角的纵坐标，fontcolor：字体颜色，fontsize：字体大小，text:文本内容

textfile:文本文件，t：时间戳，单位秒，n:帧数开始位置为0，draw/enable:控制文件显示，若值为0不显示，1显示，可以使用函数

**简单用法**

1、在左上角添加Welcome文字

ffplay -f lavfi -i color=c=white -vf drawtext=fontfile=arial.ttf:text=Welcom

2、在中央添加Good day

ffplay -f lavfi -i color=c=white -vf drawtext="fontfile=arial.ttf:text='Goodday':x=(w-tw)/2:y=(h-th)/2"

3、设置字体颜色和大小

ffplay -f lavfi -i color=c=white -vf drawtext="fontfile=arial.ttf:text='Happy Holidays':x=(w-tw)/2:y=(h-th)/2:fontcolor=green:fontsize=30"

**动态文本**

用 t (时间秒)变量实现动态文本

1、顶部水平滚动

ffplay -i jidu.mp4 -vf drawtext="fontfile=arial.ttf:text='Dynamic RTL text':x=w-t*50:fontcolor=darkorange:fontsize=30"

2、底部水平滚动

ffplay -i jidu.mp4 -vf drawtext="fontfile=arial.ttf:textfile=textfile.txt:x=w-t*50:y=h-th:fontcolor=darkorange:fontsize=30"

3、垂直从下往上滚动

ffplay jidu.mp4 -vf drawtext="textfile=textfile:fontfile=arial.ttf:x=(w-tw)/2:y=h-t*100:fontcolor=white:fontsize=30“

4.实现右上角显示当前时间？

**动态文本**

在右上角显示当前时间 localtime

ffplay jidu.mp4 -vf drawtext="fontfile=arial.ttf:x=w-tw:fontcolor=white:fontsize=30:text='%{localtime\:%H\\\:%M\\\:%S}'“

每隔3秒显示一次当前时间

ffplay jidu.mp4 -vf drawtext="fontfile=arial.ttf:x=w-tw:fontcolor=white:fontsize=30:text='%{localtime\:%H\\\:%M\\\:%S}':enable=lt(mod(t\,3)\,1)"

------

## **9. 图片处理**

图片支持: FFmpeg支持绝大多数图片处理, 除LJPEG（无损JPEG）之外，其他都能被解码，除了EXR,PIC,PTX之外，所有的都能被编码。

截取一张图片使用 –ss(seek from start)参数:ffmpeg -ss 01:23:45 -i jidu.mp4 image.jpg

从视频中生成GIF图片:ffmpeg -i jidu.mp4 -t 10 -pix_fmt rgb24 jidu.gif

转换视频为图片（每帧一张图):ffmpeg -i clip.avi frame%4d.jpg

图片转换为视频:ffmpeg -f image2 -i img%4d.jpg -r 25 video.mp4

裁剪:ffmpeg -f lavfi -i rgbtestsrc -vf crop=150:150 crop_rg.png

填充:ffmpeg -f lavfi -i smptebars -vf pad=360:280:20:20:orange pad_smpte.jpg

翻转:ffmpeg -i orange.jpg -vf hflip orange_hfilp.jpg

　　ffmpeg -i orange.jpg -vf vflip orange_vfilp.jpg

旋转:ffmpeg -i image.png -vf transpose=1 image_rotated.png

覆盖:ffmpeg -f lavfi -i rgbtestsrc -s 400x300 rgb .png

　　ffmpeg -f lavfi -i smptebars smpte.png

　　 ffmpeg -i rgb .png -i smpte.png -filter_complex overlay= (W-w)/2:(H-h)/2 rgb_smpte.png

------

## **10.其他高级技巧**

**屏幕录像**

显示设备名称:ffmpeg -list_devices 1 -f dshow -i dummy

调用摄像头:ffplay -f dshow -i video="Integrated Camera"

保存为文件:ffmpeg -y -f dshow -s 320x240 -r 25 -i video="Integrated Camera" -b:v 800K -vcodec mpeg4 new.mp4

添加字幕subtitles:语法 –vf subtitles=file, ffmpeg -i jidu.mp4 -vf subtitles=rgb.srt output.mp4

视频颤抖:ffplay –i jidu.mp4 -vf crop=in_w/2:in_h/2:(in_w-out_w)/2+((in_w-out_w)/2)*sin(n/10):(in_h-out_h)/2 +((in_h-out_h)/2)*sin(n/7)

色彩平衡:ffplay -i jidu.mp4 -vf curves=vintage

色彩变幻:fplay -i jidu.mp4 -vf hue="H=2*PI*t: s=sin(2*PI*t)+1“

彩色转换黑白:ffplay -i jidu.mp4 -vf lutyuv="u=128:v=128"

**设置音频视频播放速度:**

3倍视频播放视频: ffplay -i jidu.mp4 -vf setpts=PTS/3

?速度播放视频:ffplay -i jidu.mp4 -vf setpts=PTS/(3/4)

2倍速度播放音频:ffplay -i speech.mp3 -af atempo=2

**截图**

每隔一秒截一张图:　　ffmpeg -i input.flv -f image2 -vf fps=fps=1 out%d.png

每隔20秒截一张图:　　ffmpeg -i input.flv -f image2 -vf fps=fps=1/20 out%d.png

多张截图合并到一个文件里（2x3） ?每隔一千帧(秒数=1000/fps25)即40s截一张图

ffmpeg? -i jidu.mp4 -frames 3 -vf "select=not(mod(n\,1000)),scale=320:240,tile=2x3" out.png

**马赛克视频**

用多个输入文件创建一个马赛克视频：

ffmpeg -i jidu.mp4 -i jidu.flv -i "Day By Day SBS.mp4" -i "Dangerous.mp4" -filter_complex "nullsrc=size=640x480 [base]; [0:v] setpts=PTS-STARTPTS, scale=320x240 [upperleft]; [1:v] setpts=PTS-STARTPTS, scale=320x240 [upperright]; [2:v] setpts=PTS-STARTPTS, scale=320x240 [lowerleft]; [3:v] setpts=PTS-STARTPTS, scale=320x240 [lowerright]; [base][upperleft] overlay=shortest=1 [tmp1]; [tmp1][upperright] overlay=shortest=1:x=320 [tmp2]; [tmp2][lowerleft] overlay=shortest=1:y=240 [tmp3]; [tmp3][lowerright] overlay=shortest=1:x=320:y=240" -c:v libx264 output.mkv

**Logo动态移动**

2秒后logo从左到右移动：ffplay -i jidu.mp4 -vf movie=logo.png[logo];[in][logo]overlay=x='if(gte(t\,2)\,((t-2)*80)-w\,NAN)':y=0

2秒后logo从左到右移动后停止在左上角:ffplay -i jidu.mp4 -vf movie=logo.png[logo];[in][logo]overlay=x='if(gte(((t-2)*80)-w\,W)\,0\,((t-2)*80)-w)':y=0

每隔10秒交替出现logo:ffmpeg -y -t 60 -i jidu.mp4 -i logo.png -i logo2.png -filter_complex "overlay=x=if(lt(mod(t\,20)\,10)\,10\,NAN ):y=10,overlay=x=if(gt(mod(t\,20)\,10)\,W-w-10\,NAN ) :y=10" overlay.mp4

原文作者：零声音视频开发

原文链接：https://zhuanlan.zhihu.com/p/451247703