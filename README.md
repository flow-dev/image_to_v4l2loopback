# image_to_v4l2loopback [![Build Status](https://travis-ci.org/lucasw/image_to_v4l2loopback.svg?branch=master)](https://travis-ci.org/lucasw/image_to_v4l2loopback) [![Coverage Status](https://coveralls.io/repos/lucasw/image_to_v4l2loopback/badge.svg?branch=master)](https://coveralls.io/r/lucasw/image_to_v4l2loopback?branch=master)

ROS node for streaming an image topic to  a video capture device. Mostly
based on this:

* [virtual_camera](https://github.com/czw90130/virtual_camera)

## dev

Setup a workspace:

```bash
cd ~/catkin_ws/src  # or wherever catkin_ws is
git clone https://github.com/lucasw/image_to_v4l2loopback
cd ..
catkin build image_to_v4l2loopback
```

## video devices

Stream node needs to write to a video capture device, which varies depending on
your system.

### linux

Uses [v4l2-loopback](https://github.com/umlaeute/v4l2loopback):

Secure boot uefi will prevent the modprobe from working, try https://askubuntu.com/questions/762254/why-do-i-get-required-key-not-available-when-install-3rd-party-kernel-modules

```bash
sudo apt install mokutil
sudo mokutil --disable-validation
sudo reboot
```

```bash
# don't bother this version is too old
# sudo apt-get install v4l2loopback-*
```

But the ubuntu v4l2loopback is old if it doesn't work, try

```bash
git clone git@github.com:umlaeute/v4l2loopback
cd v4l2loopback
make
sudo insmod v4l2loopback.ko exclusive_caps=1 video_nr=1 card_label="Fake"
```
See https://github.com/umlaeute/v4l2loopback for more instructions


```
sudo modprobe v4l2loopback video_nr=1
v4l2-ctl -D -d /dev/video1
Driver Info (not using libv4l2):
    Driver name   : v4l2 loopback
    Card type     : Dummy video device (0x0000)
    Bus info      : v4l2loopback:0
    Driver version: 0.8.0
    Capabilities  : 0x05000003
        Video Capture
        Video Output
        Read/Write
        Streaming
```

## usage

### stream

Typically just:

```bash
rosrun image_to_v4l2loopback stream _device:=/dev/video1 _width:=640 _height:=480 _fourcc:=YV12 image:=/my_camera/image
```

That resolution will then be locked in, need to rmmod and restart the module then set the resolution again.

where:

* `/dev/video1` target device
* `640x480` target size
* `YV12` target [pixel format](http://en.wikipedia.org/wiki/FourCC)
* `/my_camera/image` the source `sensor_msgs/Image` topic re-mapped to `image`

For more:

```bash
rosrun image_to_v4l2loopback stream --help
```

cheese works, but guvcview does not.

Google hangouts says no camera found.

zoom-client works.

## tests

```bash
v4l2-ctl -D -d /dev/video1
Driver Info (not using libv4l2):
    Driver name   : v4l2 loopback
    Card type     : Dummy video device (0x0000)
    Bus info      : v4l2loopback:0
    Driver version: 0.8.0
    Capabilities  : 0x05000003
        Video Capture
        Video Output
        Read/Write
        Streaming
ROS_VIRTUAL_CAM_STREAM_TEST_DEVICE=/dev/video1 catkin_make run_tests
```

If you want coverage:

```bash
catkin_make -DCMAKE_BUILD_TYPE=Debug -DCOVERAGE=ON
ROS_VIRTUAL_CAM_STREAM_TEST_DEVICE=/dev/video1 catkin_make run_tests
lcov --path . --directory . --capture --output-file coverage.info
lcov --remove coverage.info 'tests/*' '/usr/*' '/opt/*' --output-file coverage.info
lcov --list coverage.info
```

## From image _to_v4l2loopback to momo

### catkin build

```bash
cd ~/catkin_ws/src
git clone https://github.com/flow-dev/image_to_v4l2loopback
cd ..
catkin build image_to_v4l2loopback
```

### install v4l2loopback

```bash
sudo apt-get install v4l2loopback-* # ->便利ツール等まとめて一旦インストール
git clone https://github.com/umlaeute/v4l2loopback.git # ->v4l2loopbackだけ最新持ってきてmake
cd v4l2loopback
make
sudo insmod v4l2loopback.ko exclusive_caps=1 video_nr=1 card_label="Fake" # ->insmodでv4l2loopbackを/dev/video1として実体化

# insmodを取り消すときは...
sudo rmmod v4l2loopback
```

### /dev/video1の仮想化ができてるか確認

```bash
# /dev/video1の仮想化ができてるか確認
v4l2-ctl -D -d /dev/video1
Driver Info (not using libv4l2):
    Driver name   : v4l2 loopback
    Card type     : Dummy video device (0x0000)
    Bus info      : v4l2loopback:0
    ******
```

### roslaunchでimage_to_v4l2loopbackを起動

```bash
roslaunch image_to_v4l2loopback loopback.launch
```

* loopback.launchの解説
* 先に配信したいtopicを起動した後で,loopback.launchした(あまり順番は関係なさそうだが)

```xml
<?xml version="1.0"?>
<launch>
  <arg name="device" default="/dev/video1" />
  <arg name="width" default="640" />
  <arg name="height" default="480" />
  <! -- monoはデフォルトだと"YUYV"しか受け付けない　TODO:momo側の設定確認 -->
  <! -- width,height,formatの変換をopencvでやってるので遅いかも　TODO:image_to_v4l2loopbackのsrc確認 -->
  <arg name="format" default="YUYV" />

  <node name="image_loopback" pkg="image_to_v4l2loopback" type="stream" output="screen" required="true" >
    <! -- 任意のTopic名をremapすればOK. sensor/Image型のみ配信可能 -->
    <remap from="image" to="/live_ar/chroma_key" />
    <param name="device" value="$(arg device)" />
    <param name="width" value="$(arg width)" />
    <param name="height" value="$(arg height)" />
    <param name="format" value="$(arg format)" />
  </node>
</launch>
```

### momoの起動

* 以下のXavierNX用を持ってきた
* https://github.com/shiguredo/momo/releases/download/2020.8/momo-2020.8_ubuntu-18.04_armv8_jetson_xavier.tar.gz
* 圧縮を展開したのち

```bash
# momoを/dev/video1指定して起動
./momo --no-audio-device --video-device /dev/video1 test
```

```bash
# chromeでtest.html立ち上げ
localhost:8080/html/test.html
# あとはH264なりVP9なりを選択して"conect"でブラウザに画像出る
```

### RasPi3 memo

```bash
sudo apt-get install v4l2loopback-*
sudo apt-get remove v4l2loopback-dims #古い

git clone https://github.com/umlaeute/v4l2loopback.git # ->v4l2loopbackだけ最新持ってきてmake
cd v4l2loopback
make
sudo insmod v4l2loopback.ko exclusive_caps=1 video_nr=1 card_label="Fake" # ->insmodでv4l2loopbackを/dev/video1として実体化

git clone https://github.com/gonzalo/gphoto2-updater.git
cd gphoto2-updater
sudo ./gphoto2-updater.sh
# develop versionをインストール選択する 1)

gphoto2 --stdout --capture-movie | ffmpeg -i - -vcodec rawvideo -pix_fmt yuv420p -threads 0 -f v4l2 /dev/video1

./momo --no-audio-device --video-device /dev/video1 test

localhost:8080/html/test.html
```



以上
