---
title: 利用Arduino搭建传感器网络(二) ——搭建自组织网络
date: 2018-06-29 19:08:31
tags:
    - Arduino
    - 物联网
    - 自组织网络
categories:
    - 物联网
---

# 简介
本文中，我们将利用Arduino开发板结合无线串口模块来模拟自组织网络中的节点，使用电脑作为控制节点来控制组织网络中的通信，为了便于观察和控制各个节点之间的通信状态，我在此利用Qt开发了一款可视化的通信控制小软件。
# 准备
硬件设备:
1、DL-20 无线串口模块若干(内为已写有无线串口通信固件的CC2530芯片);
2、Arduino 开发板若干;
3、USB 转 TTL 串口模块一个。
软件环境:
1、Arduino IDE
2、Qt5
# 实现方法
##　硬件连接
1. 连接硬件设备,将 DL-20 无线串口模块分别连接到 Arduino 开发板的串口引脚上。如下图所示:
![图片挂掉了！！！](/images/iot_arduino_01_00.jpg)
2. 将一个 DL-20 无线串口模块连接到 USB 转 TTL 串口模块上并插入电脑。
## 软件设计
### 设计思路
1. 由于硬件的限制,无法直接使用 Zigbee 协议进行通信,因此在本设计方案中,我们通过在应用层模拟 Zigbee 协议的核心功能来实现星形无线传感器网络和多跳无线自组织传感器网络。
2. 在本设计方案中,节点分为两类,一类是基于 Arduino 开发板的无线传感器节点,一类是运行在 PC 上的图形化界面控制节点,网络中所有传感器节点与控制节点之间的拓扑结构构成星形网络,网络中所有传感器节点相互之间的拓扑结构形成网状的多跳自组织无线传感器网络。
3. 为了实现节点之间的寻址功能,必须首先为不同的节点分配不同的地址,在本设计方案中,为了简化设计,在程序中使用一字节的节点编号为不同节点赋予不同的静态地址。通信时以节点编号作为通信地址。
4. 为了实现路由功能,在本实验中,每个传感器节点都会维护一张路由表,路由表中记录了网络中所有节点之间的连接情况。当一个节点收到来自其他节点的转发请求时,节点会根据数据包的目的地址在自己的路由表中搜索能够到达该节点的路径,当找到一条路径后,就回溯该路径,找到该路径中位于本节点的下一跳节点的地址,并将数据包发送给下一跳节点。如果节点在路由表中没有找到可以到达目的节点的路径,就向控制节点报告一个错误信息,并丢弃该数据包。
5. 为了实现传感器节点之间的自组织功能,每个传感器节点会定时向外广播自己的路由表,该节点附近的节点接收到路由广播后,首先将广播帧的源节点与自己之间的通路修改为连通,然后根据广播帧中的路由信息更新自己的路由表。
6. 为了当有节点从网络中断开时网络中可以即时的发现该路由拓扑结构的变化,每个节点维护一个定时任务,当一段时间内该节点没有接收到来自邻居节点的广播信息时,就认为该邻居结点已经掉线,将本节点与该邻居节点的连通状况置为断开,之后通过路由广播将该拓扑信息的变化告知网络中的其他节点。
7. 由于实验时所有节点的距离都很近,所有节点相互之间都可以通信,为了演示某些节点之间没有直接连接时的路由转发功能,在路由表的基础上增加了一张路由控制表,控制节点可以修改某些链路的连接状况,并将路由控制表广播告知所有传感器节点。
8. 为了演示传感器节点间的数据转发功能,在控制节点中添加了一个数据传输控制指令,可以指定某一传感器节点向另一节点发送一条指定的数据。
9. 数据传输协议设计如下:
MAC层协议：

| 名称   | 长度    |    备注                 |
|:------|:------:|:------------------------|
|帧头       | 1byte  |取值0xFE                 |
|源MAC地址  |  1byte |此处为了简便,取值直接使用数据发送节点的ID|
|目的MAC地址 |  1byte |取值为数据帧目的节点MAC地址(即目的节点 ID)|
|帧编号     |1byte   |从 0 开始累计,到 256 归 0 |
|MAC层控制字 |1 byte  |MAC 层应答控制,保留未使用 |
|帧长      |1byte   |数据区长度                |
|数据区    |帧长长度  |来自上层的目标数据        |
|校验位    |1byte   |从帧头开始取和,对 256 取余,再按位取反|

网络层协议：

|名称     | 长度   |    备注               |
|:-------|:------:|:------------------------|
|源IP地址   | 1byte  |取值为发送节点网络地址(此处简化为发送节点 ID)|
|目的IP地址 | 1byte  | 取值为目的节点网络地址 |
|网络层控制字| 1byte  | 网络层应答控制,保留不使用 |
| 网络层控制指令 |1byte| 0xA5 = 广播帧<br>0x01 = 数据传输<br>0x30 = 路由控制指令<br>0x50 = 数据传输控制指令,控制特定节点向某节点发送数据|

### 算法实现
传感器节点的详细程序设计如下:
``` c
/*节点相关宏定义*/
#define NID  04
//节点编号，nodeID
#define NCT  0    //控制节点
#define NBC  255  //广播号
#define NSZ  5    //允许的最大节点数，nodeSize
#define FML  64   //协议栈最大帧长度

#define MXTMR 8  //链路状态计时器最大值
#define BUFSZ 128 //读数据缓冲区长度

enum CMD{
  Broadcast = 0xA5, //广播路由表作为心跳包
  SendData  = 0x01, //数据传输
  LinkCtrl  = 0x30, //路由表控制命令
  SendCtrl = 0x50,  //控制发送
};

struct RouteNode{
  bool ctrlAble;  //链路控制器
  byte timer;     //链路状态计数器
};

//路由树节点
struct SearchNode{
  byte nodeId;
  SearchNode *parent;
  SearchNode *bother;
  SearchNode *sons;
  SearchNode(byte id){
    nodeId = id;
    parent = NULL;
    bother = NULL;
    sons = NULL;
  }
};

RouteNode RouteTable[NSZ][NSZ];  //路由表记录

byte FrameNo = 0;  //frame No
byte RCtr = 0; //网络层控制字
byte MACCtr = 0; //MAC控制字

byte readBuff[BUFSZ];
byte buffLength;

void setup() {
  //初始化路由表和路由控制表
  for(int i = 0; i < NSZ; ++i){
    for(int j = 0; j < NSZ; ++j){
      RouteTable[i][j].ctrlAble = true;
      RouteTable[i][j].timer = 0;
    }
  }
  buffLength = 0;
  Serial.begin(9600);//设置波特率
}

void loop() {
  for(int i = 0; i < 4; i++){
  while(receive());
  delay(500);
  }
  
  routeCount();
  broadcast();
  delay(2000);
}

void broadcast()
{
  byte arr[NSZ*NSZ];
  for(int i = 0; i < NSZ; ++i){
    for(int j = 0; j < NSZ; ++j){
      if(RouteTable[i][j].ctrlAble){
        arr[i*NSZ+j] = RouteTable[i][j].timer;
      }else{
        arr[i*NSZ+j] = 0;
      }
    }
  }
  sendTo(arr,NSZ*NSZ, NBC, CMD::Broadcast);
}

bool sendTo(byte *data, byte Rlen, byte NDS, byte cmd)
{
  byte RFrame[56];
  RFrame[0] = NID;    //相当于ip协议中的源ip地址
  RFrame[1] = NDS;    //相当于ip协议中的目的ip地址
  RFrame[2] = RCtr;   //网络层控制字
  RFrame[3] = cmd;    //命令
  for(int i = 0; i < Rlen; ++i){
    RFrame[4+i] = data[i];
  }
  byte NodeDst = findWay(NDS);
  if(NodeDst == byte(0xFE)){
    return false;
  }
  structFrame(RFrame, Rlen+4, NodeDst);
  return true;
}

byte findWay(byte NDS)
{
  if(NDS == NBC || NDS == NCT){
    return NDS;
  }else{
    byte dnode = search(NDS);
    if(dnode == byte(0xFE)){
      byte info[] = "[Error]:0 connot find way to 0\n";
      info[8] = '0'+ NID;
      info[29] = '0'+NDS;
      sendTo(info, 31, 0, CMD::SendData);
    }
    return dnode;
  }  
  return byte(0xFE);
}

bool structFrame(byte *data, byte len, byte NodeDst)
{
  if(len > 56){
    return false;
  }
  byte frame[64];
  frame[0] = 0xFE;                //帧头
  frame[1] = NID;                 //相当于源MAC地址
  frame[2] = NodeDst;             //相当于目的MAC地址
  frame[3] = FrameNo;             //帧号高位
  frame[4] = MACCtr;              //控制字
  frame[5] = len;                 //帧长
  for(int i = 0; i < len; ++i){
    frame[6+i] = data[i];
  }
  frame[6+len] = getCheck(frame, len+6);

  while(receive());
  Serial.write(frame, len+7);
  
  ++ FrameNo;
  return true;
}

byte getCheck(byte *frame, byte len)
{
  byte checkdata = 0;
  for(int i = 0; i < len; ++i){
    checkdata += frame[i];
  }
  return (~checkdata);
}

bool receive()
{
  int len = Serial.available();
  if(len > 0){
    Serial.readBytes(&(readBuff[buffLength]), len);
    buffLength += len;
    parseBuf();
    return true;
  }
  return false;
}

//从缓冲区中寻找一个帧
void parseBuf()
{
  byte *frame;
  int frameBegin = 0, frameLength = 0;
  while(frameBegin + 5 < buffLength &&  readBuff[frameBegin] != 0xFE) { frameBegin++; }
  if(frameBegin + 5 < buffLength){
    frameLength = readBuff[frameBegin+5]+7;
    if(frameBegin+frameLength <= BUFSZ){
      frame = new byte[frameLength];
      memcpy(frame, readBuff, frameLength);
      frameBegin += frameLength;
    }
  }
  
  cleanBuff(frameBegin); 
  parseFrame(frame, frameLength); //帧内数据解析
  delete frame;
}

//清理读数据缓冲区，删除已被解析的数据
void cleanBuff(int frameBegin)  
{
  buffLength -= frameBegin;
  memcpy(readBuff, &(readBuff[frameBegin]), buffLength);
}

void parseFrame(byte *frame, int frameLength)
{
  if(frame[2] != NID && frame[2] != NBC){
    //MAC目的地址不是本节点目的地址或广播地址就忽略
    return;
  }else if(frame[frameLength-1] != getCheck(frame, frameLength-1)){
    //byte info[] = "[Error]:0 check error!";
    //info[8] = '0'+ NID;
    //sendTo(info, 22, 0, CMD::SendData);
    return;
  }
  
  switch(frame[9]){
  case CMD::Broadcast:
    rcvBC(frame, frameLength);
    break;
  case CMD::SendData:
    rcvSD(frame, frameLength);
    break;
  case CMD::LinkCtrl:
    rcvCN(frame, frameLength);
    break;
  case CMD::SendCtrl:
    rcvNT(frame, frameLength);
    break;
  }
}

void rcvBC(byte *frame, int frameLength)
{
  if(RouteTable[NID-1][frame[1]-1].ctrlAble){
    RouteTable[NID-1][frame[1]-1].timer = MXTMR;
    RouteTable[frame[1]-1][NID-1].timer = MXTMR;
  }
  for(int i = 0; i < NSZ; ++i){
    for(int j = 0; j < NSZ; ++j){
      if(frame[i*NSZ+j+10] > RouteTable[i][j].timer){
        RouteTable[i][j].timer = frame[i*NSZ+j+10];
      }
    }
  }
}

void rcvSD(byte *frame, int frameLength)
{
  if(frame[7] == NID){
    //如果本机就是目的节点，向控制主机报告收到数据
    byte *info = new byte[13+frame[5]];
    memcpy(info, "[info]:0 rcv:", 13);
    memcpy(&(info[13]), &(frame[9]), frame[5]);
    info[7] = '0'+NID;
    sendTo(info, 13+frame[5], 0, SendData);
    delete info;
  }else{
    byte NDS = findWay(frame[7]);
    if(NDS != -1){
      structFrame(&(frame[6]), frame[5], NDS);
    }
  }
}

void rcvCN(byte *frame, int frameLength)
{
  for(int i = 0; i < NSZ; ++i){
    for(int j = 0; j < NSZ; ++j){
      RouteTable[i][j].ctrlAble = bool(frame[i*NSZ+j+10]);
      if(!RouteTable[i][j].ctrlAble){
        RouteTable[i][j].timer = 0;
      }
    }
  }
}

void rcvNT(byte *frame, int frameLength)
{
  sendTo(&(frame[11]), frame[5]-5, frame[10], CMD::SendData);
}

byte search(byte target)
{
  SearchNode *root = new SearchNode(NID);
  byte result = doSearch(root, target);
  clearTree(root);
  return result;
}

byte doSearch(SearchNode *parent, byte target)
{
  byte result = 0xFE;
  SearchNode *frontBother = NULL;
  for(int i = 0; i < NSZ; ++i){
    
    //判断路由链路是否可达
    if(RouteTable[parent->nodeId-1][i].timer>0 && RouteTable[parent->nodeId-1][i].ctrlAble){

      //环路检测，防止出现环路
      bool flag = true;
      for(SearchNode *tmp = parent; tmp != NULL && flag; tmp = tmp->parent){
        if(tmp->nodeId == i+1){
          flag = false;
        }
      }

      //加入子/兄节点
      if(flag){
        SearchNode *tmp = new SearchNode(i+1);
        tmp->parent = parent;
        if(frontBother == NULL){
          parent->sons = tmp;
        }else{
          frontBother->bother = tmp;
        }
        frontBother = tmp;
      }
    }
  }

  //搜索子节点
  for(frontBother = parent->sons; frontBother != NULL; frontBother = frontBother->bother){
    if(frontBother->nodeId == target){
      result = getNextNode(frontBother);
    }else{
      result = doSearch(frontBother, target);
    }
    if(result != byte(0xFE)){
      break;
    }
  }
  return result;
}

//返回下一跳目标节点
byte getNextNode(SearchNode *leaf)
{
  SearchNode *tmp  = leaf;
  while(tmp->parent->parent != NULL){
    tmp = tmp->parent;
  }
  return tmp->nodeId;
}

void clearTree(SearchNode *parent)
{
  for(SearchNode *frontBother = parent->sons; frontBother != NULL; frontBother = frontBother->bother){
    clearTree(frontBother);
  }
  delete parent;
}

void routeCount()
{
  //若链路计时器到0，认为链路已断开
  for(int i = 0; i < NSZ; ++i){
    for(int j = 0; j < NSZ; ++j){
      if(RouteTable[i][j].timer > 0){
        RouteTable[i][j].timer -= 1;
      }
    }
  }
}

void serialEvent()
{
  receive();
}
```
控制节点的源码较多，在此不做详细说明，但我将相关源码放在了后文中，以供参考。

# 运行及实验
修改上述代码中的 NID 的值,编译并下载到不同的 Arduino 开发板中,打开控制节点软件,点击串口连接,;连接上串口,即可在控制节点中查看到自组织网络的自组织过程,并可以控制传感器节点间的拓扑结构,并可通过控制传输来观察数据转发过程。如以下各图所示。
(其中,0 号节点为控制节点,即 PC,以黄色表示,连线为红色表示传感器节点与控制节点的连接,连线为黑色表示传感器节点间链路可达,连线为绿色表示该链路上正在传输数据)
两个传感器节点:
![图片挂掉了！！！](/images/iot_arduino_01_01.jpg)
三个传感器节点:
![图片挂掉了！！！](/images/iot_arduino_01_02.jpg)
四个传感器节点:
![图片挂掉了！！！](/images/iot_arduino_01_03.jpg)
上图中 2 号节点与 3 号节点之间直接不可达,链路处于断开状态,经过短时间的自组织链路同步以后,该链路连接自动建立:
![图片挂掉了！！！](/images/iot_arduino_01_04.jpg)
正在传输数据时的状态:
![图片挂掉了！！！](/images/iot_arduino_01_05.jpg)
可以看到,上图中,1 号节点与 2 号节点间链路为绿色,表示链路上正在转发数据。
# 总结
本文中,通过综合运用星形网络和自组织网络,搭建了一个易于控制和观察的无线自组织传感器网络,且将通信过程进行了可视化。但由于协议设计得较为简陋,通信不可靠,极易发生通信数据出错等问题。
# 源码
[ArduinoOrgNet](https://github.com/dhmemi/ArduinoOrgNet)

