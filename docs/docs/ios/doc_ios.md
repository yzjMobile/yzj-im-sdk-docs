# iOS端集成文档


# 导入SDK

## Cocopods导入（推荐）
推荐使用Cocopods集成,Cocopods能简单导入SDK以及所属的依赖库，以及引用关系

安装 
```
sudo gem install -n /usr/local/bin cocoapods
# 安装本地库 
pod setup
```
然后进入项目根目录，执行`pod init`,会生成一个名为Podfile的文件，打开，添加以下行

```
# 依赖的第三方库
  pod 'WCDB.swift', '1.0.7'
  pod 'PromiseKit', '6.5.2'
  pod 'AFNetworking', '3.2.1'
  pod 'JSONModel', '1.8.0'

  # tag 由自己指定
  pod 'YZJIMLib', :git => 'https://github.com/yzjMobile/YZJIMLib-iOS.git', :tag => '0.1.0'
```
保存后执行`pod install`,完成后就已经集成完成

## 手动导入

到[IMSDK](https://github.com/yzjMobile/YZJIMLib-iOS)地址,下载SDK,拖入文件`YZJIMLib.framework、mars.framework`到工程
然后到各个库`WCDB、PromiseKit、AFNetworking、JSONModel`的官网下载对应Framework，再拖入添加到工程


# 初始化配置

引入SDK后需要在调用的文件中使用`import YZJIMLib`来引入SDK， 其中接口主要都使用`YZJIMLib.shared`单例进行调用

## 配置appid
使用云之家提供的AppId进行初始化

```swift 
YZJIMLib.shared.initWithAppId("1234567890")
```

## 配置服务器地址
使用云之家提供的域名和端口号，用于SDK底层通信，也用于多环境切换

```swift
let config = YZJLoginConfig.init()
config.baseUrl = "http://172.20.181.40:19060/"
config.fileUrl = "http://172.20.181.40:10211/"
config.mercUrlLong = "http://172.20.181.40:9210/"
config.mercUrlShort = "http://172.20.181.40:9211/"
YZJIMLib.shared.setConfig(config)
```
## 配置推送服务

### 申请推送证书
[推送voip证书制作](push-doc#推送)

### 工程配置
点击工程target->Capabilities->打开Background Modes->勾选Voice over IP，然后配置推送的回调事件并实现代理的回调方法

```swift
YZJIMLib.shared.setAPNSDelegate(self)

//用户点击推送操作，push对应会话
func apnsDidReceivePushHandle(group: YZJGroup) {

}
```
在Appdelegate内实现推送接收方法
```swift
func application(_ application: UIApplication, didReceiveRemoteNotification userInfo: [AnyHashable : Any]) {

let success = YZJIMLib.shared.handleRemoteNotification(userInfo: userInfo)
if !success {//是SDK不处理的推送类型，需要自行处理

}
}

func application(_ application: UIApplication, didReceive notification: UILocalNotification) {

let success = YZJIMLib.shared.handleLocalNotification(notification: notification)
if !success {//是SDK不处理的推送类型，需要自行处理

}
}
```

# 登录

## 登录接口
用户登录成功后使用服务端返回的token进行IMLib的登录，登录成功后才能发送和接收消息，推荐登录成功后把`token`和`userId`存储在本地，用于下次自动登录
```swift
/// 登录接口
///
/// - **Parameters**:
///  - token: 从im-server获取的登录令牌
///  - completion: succ 成功标识，userId im用户标识，errorMsg 错误信息
public func login(token: String?,
completion: ((_ succ: Bool, _ userId: String?, _ errorMsg: String?) -> Void)?)
```
## 自动登录接口
使用本地存储的`token`和`userId`进行快速登录
```swift
/// 自动登录接口
///
/// - **Parameters**:
///  - token: 从im-server获取的登录令牌
///  - userId: 用户id
func login(token: String, userId: String)
```
## 退出登录接口
退出登录时应该调用`YZJIMLib`的接口来关闭通信
```swift
/// 登出接口
/// 数据库关闭、长连接关闭、推送注销等操作
public func logout()
```
## 被踢下线处理

```swift
//设置登录事件回调操作
YZJIMLib.shared.setLoginDelegate(delegate: self)

func didLogout(errorMsg: String?) {
YZJIMLib.shared.logout()
//退回到登录界面并提示用户被踢下线
window?.rootViewController = YZJLoginViewController()

let alert = UIAlertController.init(title: nil, message: errorMsg ?? "登录失效", preferredStyle: .alert)
alert.addAction(UIAlertAction.init(title: "确定", style: .default, handler: nil))
window?.rootViewController?.present(alert, animated: true, completion: nil)
}
```
# 群组

## 获取会话列表

同步从DB中获取会话列表，不传offset默认会取所有的会话；如果分页需要用offset要指定读取位置

```swift
/// 同步从DB中获取会话
///
/// - **Parameters**:
///  - count: 需要的数量
///  - offset: 用于翻页，不传会取所有的会话，会占用大量时间
public func getGroup(count: Int, offset: Int = -1) -> [YZJGroup]
```
**为防止群组数据变更，列表每次显示的时候应该使用此接口更新一次列表**

## 更新会话列表
当有消息来时底层会主动调用此接口，**为防止消息的延迟在会话列表显示前应该主动调用一次**

```swift
YZJIMLib.shared.updateInternalGroups()
```

## 群组回调操作

监听群组代理回调
```swift
YZJIMLib.shared.setGroupDelegate(self)
```
实现回调方法
```swift
//群组信息变更通知，收到后应该调用getGroup接口更新当前列表所有显示的会话
public func groupListDidChange(changedGroups:[YZJGroup]) 
//群组数据拉取状态变更，用来展示loading
public func groupListLoadingStateDidChange(loading: Bool)
//退出群组回调，收到后应该调用getGroup接口更新当前列表所有显示的会话
public func groupDidExit(groupId: String)
//未读数变更，收到后应该调用getGroup接口更新当前列表所有显示的会话
public func unreadCountChange() 
```
## 创建会话
```swift
///创建群组
///
/// - **Parameters**:
///  - groupName: 会话组名称，不传服务端会自动生成一个会话名
///  - userIds: 用户id数组
///  - completion: 请求回调，失败返回错误原因String,成功返回nil
public func creatGroup(groupName: String?, userIds: [String], completion:((String?, YZJGroup?)->())?)
```

# 消息

## 消息拉取

### 获取某条消息前后的若干条消息
用于聊天界面的上下分页；返回结果不包含传入的msgId；优先取本地数据，如果没有则取网络数据
``` swift
/// - Parameters:
///   - msgId: 目标消息id
///   - direction: 拉取方向，新或旧
///   - count: 拉取的消息数量
///   - groupId: 会话组id
///   - toUserId: 用户id，没有groupId则传该字段
///   - success: 获得到的消息对象集合
///   - error: 错误回调
func fetchMessageFromMsgId(_ msgId: String?,
                         direction: YZJMsgListDirection,
                             count: Int,
                           groupId: String?,
                          toUserId: String?,
                   currentMessages: [YZJMessage]?,
                           success: (([YZJMessage]) -> Void)?,
                             error: ((YZJNetworkErrorCode) -> Void)?) 
```

### 同步一个组从某条消息开始到最新一条的所有消息
用于进入聊天界面后保持数据更新；可能需要同步的消息较多，故提供分页调用

``` swift
/// - Parameters:
///   - msgId: 目标消息id
///   - groupId: 会话组id
///   - toUserId: 用户id，没有groupId则传该字段
///   - includeSelf: 是否包含自己
///   - pageCount: 分页拉取每页条数，默认200条
///   - pageCompletion: 分页拉取每页回调
///   - success: 获得到的整体消息对象集合
///   - error: 错误回调
public func syncGroupFromMsgId(_ msgId: String?,
                               groupId: String?,
                               toUserId: String?,
                               includeSelf: Bool,
                               pageCount: Int = 200,
                               pageCompletion: (([YZJMessage]) -> Void)?,
                               currentMessages: [YZJMessage]?,
                               success: (([YZJMessage]) -> Void)?,
                               error: ((YZJNetworkErrorCode) -> Void)?)
```
## 消息发送

### 发送文本消息
``` swift
/// - Parameters:
///   - groupId: 群组id
///   - toUserId: 消息接收者userId
///   - content: 消息文本内容
let param = YZJTextMessageParam(groupId: groupId,
                                toUserId: toUserId,
                                content: content)
YZJIMLib.shared.send(param, callback: callback)
```
### 发送回复消息
``` swift
/// - Parameters:
///   - replyPersonId: 回复者personId
///   - replyPersonName: 回复者personName
///   - replyMessage: 被回复的消息
let replyParam = YZJTextReplyMessageParam(replyPersonId: replyPersonId,
                                          replyPersonName: replyPersonName,
                                          replyMessage: replyMessage)
let param = YZJTextMessageParam(groupId: groupId,
                                toUserId: toUserId,
                                content: content,
                                textReplyMessageParam: replyParam)
YZJIMLib.shared.send(param, callback: callback)
```
### 发送@提及消息
``` swift
/// - Parameters:
///   - isNotifyAll: 是否提及所有人
///   - beNotifiedPersonIds: 被提及personId数组，isNotifyAll为true时不需要传
let notifyParam = YZJTextNotifyMessageParam(isNotifyAll: isNotifyAll,
                                            beNotifiedPersonIds: beNotifiedPersonIds)
let param = YZJTextMessageParam(groupId: groupId,
                                toUserId: toUserId,
                                content: content,
                                textNotifyMessageParam: replyParam)
YZJIMLib.shared.send(param, callback: callback)                                         
```

### 发送图片消息
``` swift
/// - Parameters:
///   - image: 静态图
///   - isOriginal: 是否原图
let param = YZJImageMessageParam(groupId: groupId,
                                 toUserId: toUserId,
                                 content: content,
                                 image: image,
                                 isOriginal: isOriginal)
YZJIMLib.shared.send(param, callback: callback)  
```

### 发送视频消息
``` swift
/// - Parameters:
///   - videoURL: 视频本地路径
///   - videoData: 视频数据
///   - thumbnail: 视频封面缩略图
///   - duration: 视频时长
let param = YZJVideoMessageParam(groupId: groupId,
                                 toUserId: toUserId,
                                 content: content,
                                 videoURL: videoURL,
                                 thumbnail: thumbnail,
                                 duration: duration)
YZJIMLib.shared.send(param, callback: callback)

or

let param = YZJVideoMessageParam(groupId: groupId,
                                 toUserId: toUserId,
                                 content: content,
                                 videoData: videoData,
                                 thumbnail: thumbnail,
                                 duration: duration)
YZJIMLib.shared.send(param, callback: callback)
```
### 发送音频消息
``` swift
/// - Parameters:
///   - voiceData: 音频数据，建议使用AMR格式
///   - duration: 音频长度
let param = YZJVoiceMessageParam(groupId: groupId,
                                 toUserId: toUserId,
                                 content: content,
                                 voiceData: voiceData,
                                 duration: duration)
YZJIMLib.shared.send(param, callback: callback)
```

### 发送普通事件消息
``` swift
/// - Parameters:
///   - eventKey: 事件key
///   - eventData: 事件数据
let normalParam = YZJEventNormalMessageParam(eventKey: eventKey,
                                             eventData: eventData)
let param = YZJEventMessageParam(groupId: groupId,
                                 toUserId: toUserId,
                                 content: content,
                                 eventNormalMessageParam: withdrawParam)
YZJIMLib.shared.send(param, callback: callback)
```
### 发送撤回消息
``` swift
/// - Parameters:
///   - msgId: 撤回消息id
///   - msgBy: 撤回者userId
///   - originContent: 原始内容
///   - replyMsgId: 如果是回复消息，需把回复消息id传过来
let withdrawParam = YZJEventWithdrawMessageParam(msgId: msgId,
                                                 msgBy: msgBy,
                                                 originContent: originContent,
                                                 replyMsgId: replyMsgId)
let param = YZJEventMessageParam(groupId: groupId,
                                 toUserId: toUserId,
                                 content: content,
                                 eventWithdrawMessageParam: withdrawParam)
YZJIMLib.shared.send(param, callback: callback)
```

### 发送系统表情消息
``` swift
/// - Parameters:
///   - emoticonName: 表情名称
///   - emoticonFileId: 表情文件id
let param = YZJSystemEmoticonMessageParam(groupId: groupId,
                                          toUserId: toUserId,
                                          content: content,
                                          emoticonName: emoticonName,
                                          emoticonFileId: emoticonFileId)
YZJIMLib.shared.send(param, callback: callback)
```

### 发送普通表情（GIF）消息
``` swift
/// - Parameters:
///   - emoticonData: 表情数据
let param = YZJCustomEmoticonMessageParam(groupId: groupId,
                                          toUserId: toUserId,
                                          content: content,
                                          emoticonData: emoticonData)
YZJIMLib.shared.send(param, callback: callback)
```

### 发送文件消息
``` swift
/// - Parameters:
///   - fileId: 文件id
///   - fileName: 文件名称
///   - fileSize: 文件大小
///   - fileExt: 文件后缀
///   - fileType: 文件类型（file、picture、gif、video、voice）
///   - folderId: 文件夹id
///   - folderName: 文件夹名称
///   - paramDict: 其他参数
let param = YZJFileMessageParam(groupId: groupId,
                                toUserId: toUserId,
                                content: content,
                                fileId: fileId,
                                fileName: fileName,
                                fileSize: fileSize,
                                fileExt: fileExt,
                                fileType: fileType,
                                folderId: folderId,
                                folderName: folderName,
                                paramDict: paramDict)
YZJIMLib.shared.send(param, callback: callback)
```

### 消息失败重发
``` swift
/// - Parameters:
///   - message: 需重发的消息
let param = YZJResendMessageParam(groupId: groupId,
                                  toUserId: toUserId,
                                  content: content,
                                  message: message)
YZJIMLib.shared.send(param, callback: callback)
```

# 文件
## 文件下载
``` swift
/// - Parameters:
///   - fileId: 文件id
///   - downloadPath: 文件下载的存储路径
///   - completion: 完成回调，true为下载成功，false为下载失败
YZJIMLib.shared.downloadFile(fileId: fileId,
                             downloadPath: path,
                             completion: completion)
```

## 文件缓存

``` swift
/// - Parameters:
///   - data: 文件数据
///   - key: 消息文件标记，建议传msgId
///   - type: 文件类型
///   - resolutionType: 图片文件分辨率参数（thumbnail、large、original），其他文件类型不需要传
```

### 获取默认消息文件缓存路径
``` swift
YZJMessageFileCache.messageFilePath(forKey: key,
                                    type: type,
                                    resolutionType: resolutionType)
```

### 缓存文件到本地默认路径
``` swift
YZJMessageFileCache.store(data: data,
                          forKey: key,
                          type: type,
                          resolutionType: resolutionType)
```

### 读取本地已缓存文件数据
``` swift
YZJMessageFileCache.cacheData(forKey: key,
                              type: type,
                              resolutionType: resolutionType)
```

### 删除本地已缓存文件
``` swift
YZJMessageFileCache.remove(forKey: key,
                           type: type,
                           resolutionType: resolutionType)
```

