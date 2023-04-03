## WebRTC服务搭建(使用SRS)

# SRS4

目前SRS对WebRTC的支持进度如下：

- SRS4.0.14，支持了RTMP推流，WebRTC播放。
- SRS4.0.76，支持了WebRTC推流，WebRTC播放。
- SRS4.0.95，支持了WebRTC推流，RTMP播放。

Srs4已经支持WebRTC的推流和拉流了。

```
CANDIDATE="49.4.27.42"
docker run --rm -it -p 1935:1935 -p 1985:1985 -p 8080:8080 \
    --env CANDIDATE=$CANDIDATE -p 8000:8000/udp \
    registry.cn-hangzhou.aliyuncs.com/ossrs/srs:4 ./objs/srs -c conf/docker.conf
```

注意如果RTMP转WebRTC流播放，必须使用配置文件[`rtmp2rtc.conf`](https://github.com/ossrs/srs/issues/2728#rtmp2rtc-cn-guide)

```
CANDIDATE="49.4.27.42"
docker run --rm -it -p 1935:1935 -p 1985:1985 -p 8080:8080 \
    --env CANDIDATE=$CANDIDATE -p 8000:8000/udp \
    registry.cn-hangzhou.aliyuncs.com/ossrs/srs:4 ./objs/srs -c conf/rtmp2rtc.conf
```

端口说明

| 端口      | 作用             |
| :-------- | :--------------- |
| 8080(tcp) | 网页访问对应端口 |
| 1985(tcp) | RTC接口对应端口  |
| 1935(tcp) | RTMP端口         |
| 8000(udp) | WebRTC端口       |

注意

> 一定要开放这几个端口。

查看端口是否畅通

TCP端口

```
telnet 49.4.27.42 8080
telnet 49.4.27.42 1985
telnet 49.4.27.42 1935
```

UDP端口

```
# 服务端
yum install netcat -y
nc -ul 8000

# 客户端
yum install netcat -y
nc -u 49.4.27.42 8000
```

我们可以访问

[http://49.4.27.42:8080](http://49.4.27.42:8080/)

WebRTC推流地址

http://49.4.27.42:8080/players/rtc_publisher.html

WebRTC播流地址

http://49.4.27.42:8080/players/rtc_player.html

播放WebRTC是不需要https，但是推送WebRTC是必须要https，否则我们获取不了媒体。

# Nginx反代

配置域名和HTTPS

```
upstream live_xhkjt {   
  server 127.0.0.1:8080;
}  

upstream live_rtc {   
  server 127.0.0.1:1985;
}  

server {
  listen 80;
  server_name live.xhkjedu.com;

  client_max_body_size  200m;
  listen 443  ssl; 
  ssl_certificate   /etc/nginx/cert/xhkjedu.pem;
  ssl_certificate_key  /etc/nginx/cert/xhkjedu.key;
  ssl_session_timeout 5m;
  ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_prefer_server_ciphers on;      


  location / {
    proxy_pass http://live_xhkjt/;  
    proxy_set_header Host $host;  
    proxy_set_header X-Real-IP $remote_addr;  
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;    
  }

  location /rtc/ {
    proxy_pass http://live_rtc/rtc/;  
    proxy_set_header Host $host;  
    proxy_set_header X-Real-IP $remote_addr;  
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;     
  }
}
```

访问地址

https://live.xhkjedu.com/

原来的代码逻辑是根据推流地址变换的，所以不能正常使用

示例推流代码

https://gitee.com/psvmc/srs-push.git

# 推流

## 使用官方Docker推流

使用官方推荐的docker推流

```
docker run --rm registry.cn-hangzhou.aliyuncs.com/ossrs/srs:encoder ffmpeg -stream_loop -1 -re -i ./doc/source.200kbps.768x320.flv -c copy -f flv rtmp://49.4.27.42/live/livestream
```

## WebRTC推流

注意以下情况才能正常推流

- 本地HTTP，地址为localhost或127.0.0.1
- 公网 HTTPS

或者

HTTP在Chrome浏览器快捷方式中添加

```
--ignore-certificate-errors --allow-running-insecure-content --unsafely-treat-insecure-origin-as-secure="http://49.4.27.42:8080"
```

调试

```
chrome://webrtc-internals
```

示例推流代码

https://gitee.com/psvmc/srs-push.git

这个是我稍微修改后的代码，主要为了便于理解。

注意

> 当使用SRS这样的媒体中转服务器时，是不再需要stun和turn服务器的。
>
> 因为我们中转服务器就在公网上，并且推流是单向的从客户端到服务器所以就不需要stun服务器了。

示例代码：

```
function SrsRtcPublisherAsync () {
  var self = {};

  self.pc = new RTCPeerConnection(null);
  self.stream = new MediaStream();
  self.constraints = {
    audio: true,
    video: {
      width: { ideal: 480, max: 576 }
    }
  };

  self.publish = async function (streamUrl, apiUrl) {
    self.pc.addTransceiver("audio", { direction: "sendonly" });
    self.pc.addTransceiver("video", { direction: "sendonly" });

    if (
      !navigator.mediaDevices &&
      window.location.protocol === "http:" &&
      window.location.hostname !== "localhost"
    ) {
      throw new SrsError(
        "HttpsRequiredError",
        `Please use HTTPS or localhost to publish, read https://github.com/ossrs/srs/issues/2762#issuecomment-983147576`
      );
    }
    var stream = await navigator.mediaDevices.getUserMedia(self.constraints);

    // @see https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/addStream#Migrating_to_addTrack
    stream.getTracks().forEach(function (track) {
      self.pc.addTrack(track);
      self.ontrack && self.ontrack({ track: track });
    });

    var offer = await self.pc.createOffer();
    await self.pc.setLocalDescription(offer);
    var data = {
      tid: getId(),
      streamurl: streamUrl,
      clientip: null,
      sdp: offer.sdp
    };
    var session = await rtc_req(apiUrl + "/rtc/v1/publish/", data);
    await self.pc.setRemoteDescription(
      new RTCSessionDescription({ type: "answer", sdp: session.sdp })
    );
    return session;
  };

  // Close the publisher.
  self.close = function () {
    self.pc && self.pc.close();
    self.pc = null;
  };

  // The callback when got local stream.
  // @see https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/addStream#Migrating_to_addTrack
  self.ontrack = function (event) {
    // Add track to stream of SDK.
    self.stream.addTrack(event.track);
  };


  return self;
}
```

从代码中我们可以看几个关键的步骤

```
self.pc = new RTCPeerConnection(null);
// 添加媒体流
self.pc.addTrack(track);
var offer = await self.pc.createOffer();

// 本地描述
await self.pc.setLocalDescription(offer);
// 远程描述
await self.pc.setRemoteDescription(
  new RTCSessionDescription({type: "answer", sdp: session.sdp})
);
```

而其中的远程描述就是请求SRS的服务器接口获取的。这样就建立好连接了，是不是很简单。

## 本地推流

下载FFMPEG

https://ffmpeg.org/download.html#build-windows

添加到环境变量

> D:\Tools\ffmpeg\bin

相关测试视频文件

链接：https://pan.baidu.com/s/1RIJOAwgVDdEV0UMHNaxVjQ
提取码：psvm

### 文件推流

```
ffmpeg -re -i ./source.200kbps.768x320.flv -c copy -f flv -y rtmp://49.4.27.42/live/livestream
```

### 本地桌面推流

桌面整体推流

```
ffmpeg -f gdigrab -i desktop -framerate 30 -vcodec libx264 -b:v 512k -preset ultrafast -pix_fmt yuv422p -f flv rtmp://49.4.27.42/live/livestream
```

多屏只推送主屏

```
ffmpeg -f gdigrab -s 1920x1080 -i desktop -framerate 30 -vcodec libx264 -b:v 512k -preset ultrafast -pix_fmt yuv422p -f flv rtmp://49.4.27.42/live/livestream
```

注意

> `-s 1920x1080`表示分辨率

### 摄像头麦克风推流

获取设备名称

```
ffmpeg -list_devices true -f dshow -i dummy
```

我这获取到的

> 麦克风：麦克风 (Realtek Audio)
>
> 摄像头：Integrated Webcam

查看摄像头视频

```
ffplay -f dshow -i video="Integrated Webcam"
```

摄像头推流

```
ffmpeg -f dshow -i video="Integrated Webcam" -vcodec libx264 -b:v 512k  -preset:v ultrafast -f flv rtmp://49.4.27.42/live/livestream
```

摄像头+麦克风推流

```
ffmpeg -f dshow -i video="Integrated Webcam":audio="麦克风 (Realtek Audio)" -vcodec libx264 -b:v 512k -b:a 16k -r 25 -acodec copy -preset ultrafast -f flv rtmp://49.4.27.42/live/livestream
```

# 流播放

## RTMP流

VLC播放器

https://www.videolan.org/vlc/

播放地址

> rtmp://49.4.27.42/live/livestream

## FLV流

可以通个这个播放

http://49.4.27.42:8080/players/srs_player.html

其中播放地址

> http://49.4.27.42:8080/live/livestream.flv

# 录制

```
vhost __defaultVhost__ {
    dvr {
        enabled         on;
        dvr_path        /data/rtmplive/[app]/[stream]/[2006]/[01]_[02]/[15]_[04]_[05]_[999].flv;
        dvr_plan        session;
        dvr_duration    30;
        dvr_wait_keyframe on;
        time_jitter    full;
    }
}
```

默认我们按照上文配置就可以保证推流的时候自动录制，但是并不是所有的流都要录制

我们可以配置多个vhost，根据域名的不同来区分是否录制

```
vhost live1.psvmc.cn {
  dvr {
        enabled         on;
        dvr_path        /data/rtmplive/[app]/[stream]/[2006]/[01]_[02]/[15]_[04]_[05]_[999].flv;
        dvr_plan        session;
        dvr_duration    30;
        dvr_wait_keyframe on;
        time_jitter    full;
    }
}

vhost live2.psvmc.cn {
}

vhost __defaultVhost__ {
}
```

[# live - webrtc](https://www.psvmc.cn/tags/live-webrtc/)

[WebRTC服务搭建之Kurento](https://www.psvmc.cn/article/2021-01-20-webrtc.html)