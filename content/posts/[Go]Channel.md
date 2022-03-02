---
title: "[Go]Channel"
date: 2022-03-02T15:02:49+08:00
draft: false
---

# 操作不同状态channel的结果

|操作|channel状态|结果|备注|
|-|-|-|-|
|Write|nil|panic|
||打开不满|写入成功|
||打开满|deadlock|
||close|panic|send on closed channel|
|Read|nil|panic|goroutine 1 [chan receive (nil chan)]|
||打开空|deadlock|
||打开有数据|输出数据|
||close|如果有数据就将数据输出|
|close|nil|panic|close of nil channel|
||打开|正常|
||关闭|panic|close of closed channel|