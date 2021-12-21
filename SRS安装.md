# SRS安装 #

## Getting Started ##

下载源码，推荐用CentOS7系统：

	git clone -b 4.0release https://gitee.com/ossrs/srs.git

编译，注意需要切换到srs/trunk目录：

    cd srs/trunk
    ./configure
    make

启动服务器：

	./objs/srs -c conf/srs.conf

检查SRS是否成功启动，可以打开 http://localhost:8080/ ，或者执行命令：

    # 查看SRS的状态
    ./etc/init.d/srs status
    
    # 或者看SRS的日志
    tail -n 30 -f ./objs/srs.log


例如，下面的命令显示SRS正在运行：

```
MB0:trunk $ ./etc/init.d/srs status
SRS(pid 90408) is running.                                 [  OK  ]

MB0:trunk $ tail -n 30 -f ./objs/srs.log
[2021-08-13 10:30:36.634][Trace][90408][12c97232] Hybrid cpu=0.00%,0MB, cid=1,1, timer=61,0,0, clock=0,22,25,0,0,0,0,1,0
```

使用 FFmpeg(点击下载) 或 OBS(点击下载) 推流：

	ffmpeg -re -i ./doc/source.flv -c copy -f flv -y rtmp://localhost/live/livestream

打开下面的页面播放流（若SRS不在本机，请将localhost更换成服务器IP）:

- RTMP (by VLC): rtmp://localhost/live/livestream
- H5(HTTP-FLV): http://localhost:8080/live/livestream.flv
- H5(HLS): http://localhost:8080/live/livestream.m3u8
- H5(WebRTC): webrtc://localhost/live/livestream

Note: 若需要开启WebRTC能力，请将CANDIDATE设置为服务器的外网地址，详细请阅读CANDIDATE。

以下是`./configure`最后的结果：

```
Configure summary:
     --x86-x64
     --prefix=/usr/local/srs --hls=on --hds=off --dvr=on --ssl=on --https=on --ssl-1-0=off --ssl-local=off --sys-ssl=off --transcode=on --ingest=on --stat=on --http-callback=on --http-server=on --stream-caster=on --http-api=on --utest=off --cherrypy=off --srt=off --rtc=on --simulator=off --cxx11=off --cxx14=off --ffmpeg-fit=on --nasm=on --srtp-nasm=on --clean=on --gperf=off --gmc=off --gmd=off --gmp=off --gcp=off --gprof=off --static=off --shared-st=off --shared-srt=off --shared-ffmpeg=off --log-verbose=off --log-info=off --log-trace=on --gcov=off --debug=off --debug-stats=off --cross-build=off --cc=gcc --cxx=g++ --ar=ar --ld=ld --randlib=randlib
HLS is enabled.
Experiment: StreamCaster is enabled.
Warning: HDS is disabled.
Warning: SRT is disabled.
Experiment: RTC is enabled. https://github.com/ossrs/srs/issues/307
Experiment: HTTPS is enabled. https://github.com/ossrs/srs/issues/1657
DVR is enabled.
RTMP complex handshake is enabled
NASM for HTTPS(openssl) and FFmepg is enabled
SRTP-NASM for WebRTC(openssl) is enabled
The transcoding is enabled
The ingesting is enabled.
The http-callback is enabled
Embeded HTTP server for HTTP-FLV/HLS is enabled.
The HTTP API is enabled
Note: The utests are disabled.
Note: The gperf(tcmalloc) is disabled.
Note: The gmc(gperf memory check) is disabled.
Note: The gmd(gperf memory defense) is disabled.
Note: The gmp(gperf memory profile) is disabled.
Note: The gcp(gperf cpu profile) is disabled.
Note: The gprof(GNU profile tool) is disabled.
Note: The cross-build is disabled.
Note: The valgrind is disabled.
Enable module: modules/hls-ingester
Enable module: modules/mp4-parser
```

以下是`make install`的结果：

```
Now mkdir /usr/local/srs
Now make the http root dir
Now copy binary files
Now copy srs conf files
Now copy init.d script files
Now copy systemctl service files

The api installed, to link and start srs, please
For CentOS6:
     sudo ln -sf /usr/local/srs/etc/init.d/srs /etc/init.d/srs &&
     /etc/init.d/srs start
For CentOS7:
     sudo ln -sf /usr/local/srs/etc/init.d/srs /etc/init.d/srs &&
     sudo cp -f /usr/local/srs/usr/lib/systemd/system/srs.service /usr/lib/systemd/system/srs.service && sudo systemctl daemon-reload && sudo systemctl enable srs &&
     sudo systemctl start srs
@see: https://github.com/ossrs/srs/wiki/v3_CN_LinuxService
```