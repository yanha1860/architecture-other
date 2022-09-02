

## 背景

​		大多数小公司因成本因素，不会选择自建服务，而是直接使用现成的SaaS服务（阿里云、腾讯云、七牛云），所以保障用户的音视频体验更多的是依靠云厂商的服务质量。

​		**看似接入方便、成本低，其实存在很多问题：**

- 流量小，没有话语权，遇到问题不能很快得到答复和解决。

- CDN调度优先级低，经常被调度到偏远地区，用户投诉视频卡顿。

- 高峰期云厂商CDN节点负载高，即使用了HttpDNS直连也不一定能保障线路最优。

  **这里对影响用户观看体验的问题进行了梳理：**
![image](https://user-images.githubusercontent.com/5134790/188067377-408b6106-8331-4ede-b122-37a51642207d.png)
# 如何改善

- **提升音视频服务的可观测性**，类似RUM，采集播放视频过程中的相关指标（收集卡顿日志）
- **优化视频资源**，保障清晰度的情况下尽量减小传输体积，客户端网络自适应分辨率（HLS&H265&FMP4）
- **服务端智能调度（接入多家CDN厂商）**
  - 基于大数据对卡顿指标的分析结果，自己做调度
  - 服务端下发多条线路，当卡顿指标满足一定条件时进行切换（先横向切换厂商、后纵向切换分布率）

# 方案

## 提升音视频服务的可观测性：卡顿监控

#### 卡顿数据上报策略的抽象定义

![image](https://user-images.githubusercontent.com/5134790/188067404-626061b0-f2dd-4921-8180-77feef4e2080.png)

###### ExoPlayer实现

![image](https://user-images.githubusercontent.com/5134790/188067459-3e595c75-9775-44e7-aaf2-16003d1a108b.png)

###### AVPlayer实现

![image](https://user-images.githubusercontent.com/5134790/188067491-f117c9e6-dbea-4f7e-b8d0-4343cf20fbac.png)

###### 监控大盘（部分内容）

![image](https://user-images.githubusercontent.com/5134790/188067531-ccd0ba9d-e4ea-4679-a7f4-55fb5d81c4c6.png)

![image](https://user-images.githubusercontent.com/5134790/188067556-5f74220b-c301-4ae5-b133-ae0005aaa3e5.png)

![image](https://user-images.githubusercontent.com/5134790/188067596-41058eeb-ee27-4465-995d-b74ff05f4119.png)

## **优化视频资源**：HLS+H265+FMP4
#### 不同视频格式的对比

![image](https://user-images.githubusercontent.com/5134790/188095385-dee649d8-9211-4e4d-9b8f-36e391206e70.png)

#### 多分辨率自适应、多线路冗余（支持CDN线路故障自动切换，参见：HTTP Live Streaming协议草案at 2009/10/05）

![image](https://user-images.githubusercontent.com/5134790/188067645-d5ecc951-41f8-4cb7-ae6f-0be98edde90b.png)

#### 实现效果

![image](https://user-images.githubusercontent.com/5134790/188067675-35416a35-e9c9-4a73-97e7-67d89c9847e5.png)

## 智能调度

#### 目标

- 改善用户因自身网络环境的原因导致的卡顿现象，例如用户使用阿里云的CDN线路，就是经常会发生卡顿，但切换到腾讯云，可能就不会。
- 改善用户因硬件性能问题导致的卡顿现象，部分设备较老，播放分辨率较高的视频容易掉帧（卡顿），需要对不同设备做适配。

#### 简单描述下实现方式

- 基于卡顿大数据的调度（用户）
  1. 每日定期收集前一天的用户观看每一厂商的每一分辨率&码率的平均卡顿时长
  2. 过滤出平均卡顿时长大于5s的数据
  3. 统计出每一个用户的每一个厂商的每一个分辨率&码率的**卡顿**数
  4. 统计出每一个用户的每一个厂商的每一个分辨率&码率的**播放**数
  5. 基于上面的数据进行卡顿分数计算，最终会算出一个最佳的适配列表。
  6. 后续服务端下发资源时，会参考每一个用户的”用户厂商适配表“，进行优先级排序。
- 基于卡顿大数据的调度（地区）
  - 解析时间段内卡顿数据的ip地址，得出所属省份及城市后进行统计。
  - 基于每个厂商在不同地区的卡顿数量，最终会算出一个地区最佳适配列表。
  - 后续服务端下发资源时，如果该用户没有对应的”用户厂商适配表“，那么就使用”地区厂商适配表“，进行优先级排序。

#### 策略

![image](https://user-images.githubusercontent.com/5134790/188067762-e4f7ddfd-f03f-49ed-8505-7fc57db9f2c8.png)

#### 数据定义

![image](https://user-images.githubusercontent.com/5134790/188093858-d8d72c1d-15d5-4f0f-986b-de6b81179ce0.png)

#### 核心流程

![image](https://user-images.githubusercontent.com/5134790/188094165-713e848a-8cdf-4d9d-be0f-b9139aa2aca6.png)

![image](https://user-images.githubusercontent.com/5134790/188094338-d88277ef-f469-468b-8cdb-9681a844e641.png)

![image](https://user-images.githubusercontent.com/5134790/188094372-3a19ebc4-2e78-4d46-9d76-a720e4884450.png)

## 衡量指标

![image](https://user-images.githubusercontent.com/5134790/188094579-4377b44d-0fd1-4674-93ea-9a0b4000a388.png)
