#SDK - IM + AppStore 集成指南

## 一、SDK 简介

SDK 是一个企业级 Android 开发套件，包含 **即时通讯（IM）** 和 **应用市场（AppStore）** 两大核心模块，所有功能封装在 `omm-lib` 中，第三方只需引入单个 AAR 即可快速集成完整能力。

### 1.1 模块概览

| 模块 | 说明 | 技术栈 |
|------|------|--------|
| **IM** | 基于 XMPP 协议的即时通讯系统，支持单聊、群聊、好友管理 | Smack 4.4.6 + SQLite |
| **AppStore** | 企业级应用市场，支持应用展示、下载、安装、更新 | RxJava2 + Retrofit2 + LitePal + Glide |

### 1.2 核心特性

- ✅ **一键集成** — 单个 AAR 文件包含所有功能，无需额外配置
- ✅ **多源依赖** — 支持 JitPack、Nexus、GitLab、本地 AAR 四种集成方式
- ✅ **数据持久化** — 内置 SQLite 数据库，自动管理聊天记录和应用信息
- ✅ **权限封装** — 模块已声明所需权限，宿主无需重复配置

---

## 二、集成方式

`omm-lib` 提供了以下四种集成方式，任选其一即可。

### 2.1 方式一：GitHub + JitPack（公开仓库）

适用于 GitHub 公开仓库，无需认证。

**前置条件：创建 GitHub Release**

1. 访问 https://github.com/Bean-V/WorkUp-SDK/releases
2. 点击 **Create a new release**
3. 选择 Tag version（如 `v1.0.0`），填写 Release title 和描述
4. 点击 **Publish release**
5. 访问 https://jitpack.io/#Bean-V/WorkUp-SDK 触发构建
6. 等待构建完成（首次约 5-10 分钟）

> ⚠️ 必须先创建 Release 并在 JitPack 构建成功，才能添加依赖

**1. 添加仓库**

在项目根目录 `build.gradle` 中：

```gradle
allprojects {
    repositories {
        google()
        mavenCentral()
        maven { url 'https://jitpack.io' }
    }
}
```

**2. 添加依赖**

在 App 模块 `build.gradle` 中：

```gradle
dependencies {
    implementation 'com.github.Bean-V.WorkUp-SDK:omm-lib:1.0.0'
}
```

---

### 2.2 方式二：Nexus Maven（企业内网）

适用于企业内部网络，私有部署。

**1. 添加仓库**

```gradle
allprojects {
    repositories {
        maven {
            url = "http://192.168.88.218:8403/repository/workup-sdk-releases/"
            allowInsecureProtocol = true
        }
    }
}
```

**2. 添加依赖**

```gradle
dependencies {
    implementation 'com.oort.workup-sdk:omm-lib:1.0.0'
}
```

---

### 2.3 方式三：GitLab Maven（备用方案）

适用于使用 GitLab Raw URL 访问独立的 Maven Git 仓库。

**1. 添加仓库**

```gradle
allprojects {
    repositories {
        maven {
            url = "http://192.168.88.125/zhangzhijun/workup-sdk-maven/raw/master"
            allowInsecureProtocol = true

            // 如果仓库是私有的，需要添加认证
            credentials(HttpHeaderCredentials) {
                name = "Private-Token"
                value = "your-gitlab-token"
            }

            authentication {
                header(HttpHeaderAuthentication)
            }
        }
    }
}
```

**2. 添加依赖**

```gradle
dependencies {
    implementation 'com.oort.workup-sdk:omm-lib:1.0.0'
}
```

---

### 2.4 方式四：本地 AAR 文件（离线集成）

直接使用 `omm-lib-1.0.0.aar` 文件进行本地集成。

**1. 放置 AAR 文件**

将 `omm-lib-1.0.0.aar` 复制到 App 模块的 `libs/` 目录下。

**2. 添加依赖**

```gradle
// 在 App 模块 build.gradle 中
repositories {
    flatDir {
        dirs 'libs'
    }
}

dependencies {
    implementation files('libs/omm-lib-1.0.0.aar')
    // 或
    // implementation(name: 'omm-lib-1.0.0', ext: 'aar')
}
```

---

### 2.5 前置依赖

使用远程依赖时，以下模块会通过 POM 自动引入。使用本地 AAR 时需确保宿主已引入：

| 依赖模块 | 说明 |
|---------|------|
| `:core` | 基础框架（用户信息、网络常量等） |
| `:ooortCloudDisk` | 云盘模块（AppStore 内打开云盘时使用） |

---

### 2.6 所需权限

模块已在 `AndroidManifest.xml` 中声明以下权限，宿主**无需重复声明**：

```xml
<!-- IM 模块权限 -->
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.WAKE_LOCK" />
<uses-permission android:name="android.permission.VIBRATE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />

<!-- AppStore 模块权限 -->
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES" />
<uses-permission android:name="android.permission.REQUEST_DELETE_PACKAGES" />
```

宿主 App 需在**运行时**主动申请存储权限和安装权限（模块已封装权限检查逻辑）。

---

## 三、IM 模块集成

### 3.1 初始化与登录连接

IM 功能通过启动 `CoreService` 来初始化 XMPP 连接。

**注意：** 启动前需要先通过 REST API 登录获取 `userId` 和 `accessToken`，并确保 `User` 信息已写入本地。

```java
// 1. 确保用户已登录（通过业务 API 获取）
// User 对象需要包含 userId、nickName、accessToken 等

// 2. 启动 CoreService
Intent intent = CoreService.getIntent(context, userId, password, nickName);
ContextCompat.startForegroundService(context, intent);
```

### 3.2 监听连接状态

通过 `ListenerManager` 注册连接状态监听：

```java
// 注册认证状态监听
ListenerManager.getInstance().addAuthStateChangeListener(new AuthStateListener() {
    @Override
    public void onAuthStateChange(int authState) {
        switch (authState) {
            case AuthStateListener.AUTH_STATE_NOT:      // 1 - 未登录
                break;
            case AuthStateListener.AUTH_STATE_ING:      // 2 - 登录中
                break;
            case AuthStateListener.AUTH_STATE_SUCCESS:  // 3 - 已认证
                break;
        }
    }
});

// 移除监听（在不需要时调用）
ListenerManager.getInstance().removeAuthStateChangeListener(listener);
```

### 3.3 退出登录

```java
// 方法一：退出并销毁服务
Intent intent = new Intent(context, CoreService.class);
context.stopService(intent);

// 方法二：通过 Service 内部方法（需持有 CoreService 引用）
coreService.logout();
```

---

### 3.4 消息收发

#### 发送消息

```java
// 获取 CoreService 引用（通过绑定服务获取 CoreServiceBinder）
CoreService coreService = ...;

// 构造 ChatMessage
ChatMessage message = new ChatMessage();
message.setType(XmppMessage.TYPE_TEXT);      // 文本消息
message.setContent("Hello, World!");
message.setFromUserId(currentUserId);
message.setFromUserName(currentNickName);
message.setToUserId(targetUserId);
message.setPacketId(UUID.randomUUID().toString().replaceAll("-", ""));
message.setDoubleTimeSend(TimeUtils.sk_time_current_time_double());

// 发送单聊消息
coreService.sendChatMessage(targetUserId, message);
```

**支持的常用消息类型：**

| 类型常量 | 值 | 说明 |
|---|---|---|
| `XmppMessage.TYPE_TEXT` | 1 | 文本消息 |
| `XmppMessage.TYPE_IMAGE` | 2 | 图片消息 |
| `XmppMessage.TYPE_VOICE` | 3 | 语音消息 |
| `XmppMessage.TYPE_LOCATION` | 4 | 位置消息 |
| `XmppMessage.TYPE_VIDEO` | 6 | 视频消息 |
| `XmppMessage.TYPE_CARD` | 8 | 名片 |
| `XmppMessage.TYPE_FILE` | 9 | 文件消息 |
| `XmppMessage.TYPE_READ` | 26 | 已读回执 |
| `XmppMessage.TYPE_BACK` | 202 | 撤回消息 |
| `XmppMessage.TYPE_RED` | 28 | 红包消息 |
| `XmppMessage.TYPE_TRANSFER` | 29 | 转账消息 |

#### 接收消息

注册消息监听器：

```java
ListenerManager.getInstance().addChatMessageListener(new ChatMessageListener() {
    @Override
    public void onMessageSendStateChange(int messageState, String msgId) {
        // 消息发送状态回调
        // messageState: 0=发送中, 1=发送成功, 2=发送失败
    }

    @Override
    public boolean onNewMessage(String fromUserId, ChatMessage message, boolean isGroupMsg) {
        // 新消息回调
        // fromUserId: 发送方用户ID
        // message: 消息实体
        // isGroupMsg: 是否为群消息
        // return: true=已处理消息(不再发出通知)
        return false;
    }
});

// 移除监听
ListenerManager.getInstance().removeChatMessageListener(listener);
```

#### 发送已读回执

```java
// 发送已读消息（通过广播）
Intent readIntent = new Intent(OtherBroadcast.Read);
readIntent.putExtra("packetId", messagePacketId);
readIntent.putExtra("isGroup", false);          // false=单聊 true=群聊
readIntent.putExtra("friendId", fromUserId);
readIntent.putExtra("fromUserName", fromUserName);
context.sendBroadcast(readIntent);
```

#### 撤回消息

撤回消息通过发送 `TYPE_BACK = 202` 类型的消息实现：

```java
ChatMessage backMessage = new ChatMessage();
backMessage.setType(XmppMessage.TYPE_BACK);
backMessage.setFromUserId(currentUserId);
backMessage.setFromUserName(currentNickName);
backMessage.setToUserId(targetUserId);
backMessage.setContent(originalPacketId);       // 被撤回消息的 packetId
backMessage.setPacketId(UUID.randomUUID().toString().replaceAll("-", ""));
backMessage.setDoubleTimeSend(TimeUtils.sk_time_current_time_double());

coreService.sendChatMessage(targetUserId, backMessage);
```

---

### 3.5 群聊功能

#### 创建群聊

```java
// 创建群聊房间
String roomId = coreService.createMucRoom(roomName);

// 或通过指定ID创建
boolean success = coreService.createMucRoomId(roomId);
```

#### 加入/退出群聊

```java
// 加入群聊
coreService.joinMucChat(roomId, lastSeconds);

// 退出群聊
coreService.exitMucChat(roomId);
```

#### 发送群消息

```java
ChatMessage message = new ChatMessage();
message.setType(XmppMessage.TYPE_TEXT);
message.setContent("群聊消息");
message.setFromUserId(currentUserId);
message.setFromUserName(currentNickName);
message.setToUserId(roomId);
message.setPacketId(UUID.randomUUID().toString().replaceAll("-", ""));
message.setDoubleTimeSend(TimeUtils.sk_time_current_time_double());

coreService.sendMucChatMessage(roomId, message);
```

#### 群聊监听

```java
ListenerManager.getInstance().addMucListener(new MucListener() {
    @Override
    public void onDeleteMucRoom(String toUserId) {
        // 群组被删除
    }

    @Override
    public void onMyBeDelete(String toUserId) {
        // 我被踢出群组
    }

    @Override
    public void onNickNameChange(String toUserId, String changedUserId, String changedName) {
        // 群内昵称改变
    }

    @Override
    public void onMyVoiceBanned(String toUserId, int time) {
        // 我被禁言（time=禁言时长，-1=解除禁言）
    }
});
```

---

### 3.6 通讯录/好友管理

#### 数据模型

**Friend（好友）**

```java
// 关键字段
friend.getUserId();         // 用户ID
friend.getNickName();       // 昵称
friend.getRemarkName();     // 备注名
friend.getDescription();    // 签名
friend.getStatus();         // 关系状态（见下方）
friend.getRoomFlag();       // 0=好友 1=群聊
friend.getUnReadNum();      // 未读消息数
friend.getContent();        // 最后一条消息内容
friend.getTimeSend();       // 最后一条消息时间
friend.getTopTime();        // 置顶时间（0=未置顶）
friend.getOfflineNoPushMsg(); // 消息免打扰 0=关闭 1=开启
```

**Friend 关系状态（status）：**

| 状态值 | 说明 |
|---|---|
| `Friend.STATUS_BLACKLIST` (-1) | 黑名单 |
| `Friend.STATUS_UNKNOW` (0) | 陌生人 |
| `Friend.STATUS_ATTENTION` (1) | 关注 |
| `Friend.STATUS_FRIEND` (2) | 好友 |
| `Friend.STATUS_SYSTEM` (8) | 系统号 |

**NewFriendMessage（新朋友消息）**

```java
// 关键字段
newFriendMessage.getType();     // 消息类型
newFriendMessage.getUserId();   // 对方用户ID
newFriendMessage.getNickName(); // 对方昵称
newFriendMessage.getContent();  // 招呼内容
newFriendMessage.getState();    // 状态
```

**NewFriendMessage 类型（type）：**

| 类型常量 | 值 | 说明 |
|---|---|---|
| `XmppMessage.TYPE_SAYHELLO` | 500 | 打招呼/添加好友 |
| `XmppMessage.TYPE_PASS` | 501 | 同意加好友 |
| `XmppMessage.TYPE_FEEDBACK` | 502 | 回话 |
| `XmppMessage.TYPE_FRIEND` | 508 | 直接成为好友 |
| `XmppMessage.TYPE_BLACK` | 507 | 黑名单 |
| `XmppMessage.TYPE_REFUSED` | 509 | 取消黑名单 |
| `XmppMessage.TYPE_DELALL` | 505 | 彻底删除 |

#### 获取好友列表

好友数据存储在本地数据库中，通过 `FriendDao` 获取：

```java
// 获取当前用户的所有好友
List<Friend> friendList = FriendDao.getInstance().getFriends(ownerId);

// 获取好友列表（仅好友关系，status=2）
List<Friend> friends = new ArrayList<>();
for (Friend f : friendList) {
    if (f.getStatus() == Friend.STATUS_FRIEND) {
        friends.add(f);
    }
}

// 获取最近聊天的好友列表（用于消息界面）
List<Friend> recentList = FriendDao.getInstance().getNearlyFriendMsg(ownerId);
```

#### 更新好友关系

通过 `FriendHelper` 工具类更新本地好友关系：

```java
// 从服务器获取用户信息后，更新本地好友关系
boolean changed = FriendHelper.updateFriendRelationship(loginUserId, user);

// 添加好友后的额外操作
FriendHelper.addFriendExtraOperation(loginUserId, friendId);

// 拉黑操作
FriendHelper.addBlacklistExtraOperation(loginUserId, friendId);

// 删除好友/取消关注
FriendHelper.removeAttentionOrFollow(ownerId, friendId);
```

#### 发送好友请求

通过 `CoreService` 发送新朋友消息：

```java
// 构造新朋友消息
NewFriendMessage newFriendMsg = NewFriendMessage.createWillSendMessage(
    currentUser,                    // 当前用户 User 对象
    XmppMessage.TYPE_SAYHELLO,     // 消息类型（打招呼）
    "你好，我是" + currentUser.getNickName(), // 招呼内容
    targetUserId,                   // 目标用户ID
    targetNickName,                 // 目标用户昵称
    targetCompanyId                 // 目标公司ID
);

// 发送
coreService.sendNewFriendMessage(targetUserId, newFriendMsg);
```

#### 监听好友请求

```java
ListenerManager.getInstance().addNewFriendListener(new NewFriendListener() {
    @Override
    public void onNewFriendSendStateChange(String toUserId, NewFriendMessage message, int messageState) {
        // 好友请求发送状态回调
        // messageState: 0=发送中, 1=发送成功, 2=发送失败
    }

    @Override
    public boolean onNewFriend(NewFriendMessage message) {
        // 收到新朋友消息
        // return: true=已处理（不再弹出通知）
        return false;
    }
});

// 移除监听
ListenerManager.getInstance().removeNewFriendListener(listener);
```

---

### 3.7 数据库操作

IM 模块使用本地 SQLite 数据库存储数据，核心 DAO 类：

| DAO 类 | 说明 |
|---|---|
| `FriendDao` | 好友/联系人表操作 |
| `ChatMessageDao` | 聊天消息表操作 |
| `NewFriendDao` | 新朋友消息表操作 |
| `RoomMemberDao` | 群成员表操作 |
| `ContactDao` | 手机联系人表操作 |

**常用 DAO 查询示例：**

```java
// 查询聊天记录
List<ChatMessage> messages = ChatMessageDao.getInstance().queryChatMessage(
    loginUserId, friendId, pageIndex, pageSize
);

// 查询未读消息数
int unreadCount = FriendDao.getInstance().getUnReadMessageTotal(loginUserId);

// 更新好友备注
FriendDao.getInstance().updateRemarkName(loginUserId, friendId, newRemark);

// 删除好友
FriendDao.getInstance().deleteFriend(loginUserId, friendId);
```

---

### 3.8 群组专用 ID 常量

系统内置了一些特殊 ID，用于系统消息、新朋友等功能：

```java
Friend.ID_SYSTEM_MESSAGE = "10000";       // 系统消息
Friend.ID_NEW_FRIEND_MESSAGE = "10001";   // 新朋友
Friend.ID_BLOG_MESSAGE = "10002";         // 商务圈消息
Friend.ID_INTERVIEW_MESSAGE = "10004";    // 面试中心
Friend.ID_SYSTEM_NOTIFICATION = "10005";  // 系统通知
```

---

## 四、AppStore 模块集成

### 4.1 初始化

#### 设置 Application

在你的 `Application.onCreate()` 中：

```java
AppStoreInit.setApplication(this);
```

#### 初始化数据

用户登录成功后，调用 `initData()` 初始化应用市场数据（安装记录、模块列表）：

```java
AppStoreInit.getInstance().initData(token, uuid);
```

参数说明：

| 参数 | 类型 | 说明 | 获取方式 |
|------|------|------|---------|
| `token` | String | 用户身份令牌 | 登录接口返回 |
| `uuid` | String | 用户唯一标识 | `UserInfo.getOort_uuid()` |

#### 配置 API 地址

后端接口地址通过 `Constant.BASE_URL` 配置，该常量位于 `:core` 模块中。确保在项目初始化时已正确设置。

```java
// 示例：在 Application 中设置
Constant.BASE_URL = "https://your-server.com/";
```

---

### 4.2 页面跳转 API

#### 打开应用详情页

```java
// AppInfo 可从列表接口获取或自行构建
AppDetailedActivity.actionStart(context, appInfo);
```

`AppInfo` 关键字段说明：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `uid` | String | 是 | 应用唯一标识 |
| `applabel` | String | 是 | 应用显示名称 |
| `apppackage` | String | 是 | 应用包名 |
| `versioncode` | int | 是 | 版本号 |
| `terminal` | int | 是 | 终端类型（0=APK / 1=H5 / 2=Web） |
| `apk_url` | String | 按需 | APK 下载地址（terminal=0 时必填） |
| `icon_url` | String | 按需 | 图标地址 |
| `oneword` | String | 否 | 一句话简介 |
| `apply_status` | int | 否 | 申请状态（0=不可用/1=可用/2=可申请/3=已申请） |

#### 打开搜索页

```java
Intent intent = new Intent(context, SearchActivity.class);
context.startActivity(intent);
```

#### 打开应用管理页

```java
// 方式一：通过 Intent Action
Intent intent = new Intent("com.oortcloud.appstore.app.appManager");
context.startActivity(intent);

// 方式二：直接跳转 Activity
Intent intent = new Intent(context, AppManagerActivity.class);
context.startActivity(intent);
```

#### 打开分类全部应用页

```java
Intent intent = new Intent(context, TypeAppAllActivity.class);
context.startActivity(intent);
```

---

### 4.3 网络请求 API

所有 API 通过 `HttpRequestCenter` 调用，返回 `Observable<String>`，由调用方自行订阅解析。

#### 应用列表

```java
// 获取推荐页数据
HttpRequestCenter.postRecommend().subscribe(observer);

// 获取分类列表
HttpRequestCenter.postClassifyList().subscribe(observer);

// 获取分类下应用列表（分页）
HttpRequestCenter.postClassifyAppMore(classifyUid, pageNum, pageSize).subscribe(observer);

// 搜索应用
HttpRequestCenter.postSearch("关键词").subscribe(observer);

// 当月最新应用
HttpRequestCenter.monthNewApp(pageNum, pageSize).subscribe(observer);

// 我的应用
HttpRequestCenter.myApp(pageNum, pageSize).subscribe(observer);
```

#### 安装管理

```java
// 记录应用安装
HttpRequestCenter.appInstall(label, packageName, classify, uid, versionCode, terminal)
    .subscribe(observer);

// 获取已安装应用列表
HttpRequestCenter.appInstallList(classify).subscribe(observer);

// 获取可更新应用列表
HttpRequestCenter.appUpdateList().subscribe(observer);

// 检测版本更新（返回最新版本信息）
HttpRequestCenter.verifyversioncode(packageName, currentVersionCode).subscribe(observer);
```

#### 评论评分

```java
// 获取评论列表
HttpRequestCenter.replySystemList(pageNum, appUid).subscribe(observer);

// 获取评分
HttpRequestCenter.getGrade(appUid).subscribe(observer);

// 发表评论
HttpRequestCenter.replySystemAdd(content, parentId, replyType, appUid).subscribe(observer);

// 评分
HttpRequestCenter.replySystemGrade(replyTargetId, score).subscribe(observer);
```

#### 响应格式

所有接口统一返回 JSON 格式（由 `Result<T>` 封装）：

```json
{
    "code": 200,
    "message": "success",
    "data": { ... }
}
```

| 常用 code | 含义 |
|-----------|------|
| 200 | 成功 |
| 50010 | 已是最新版本（版本校验场景） |
| 50011 | 有新版本可用 |

---

### 4.4 下载与安装

#### 直接触发下载安装

```java
// 初始化下载管理器
DownloadManager downloadManager = DownloadManager.getInstance();

// 创建下载监听
DownloadListener listener = new DownloadListener(appInfo, progressButton);

// 启动下载
downloadManager.startDownload(appInfo, listener);
```

#### 通过 AppEventUtil 自动处理

`AppEventUtil.onClick()` 封装了完整的应用处理逻辑，会根据 `terminal` 类型自动执行下载/安装/打开：

```java
// 适用于应用列表/详情页的点击事件
AppEventUtil.onClick(appInfo, downloadListener);
```

其内部处理逻辑：

| terminal | 行为 |
|----------|------|
| 0 (APK) | 检查本地是否已安装 → 检查 APK 文件是否存在 → 下载 → 安装 |
| 1 (H5) | 检查 ZIP 文件是否存在 → 下载 → 解压 → 打开 |
| 2 (Web) | 直接通过 WebView 打开 URL |
| 3 (PC) | 提示"手机端不能使用" |

---

## 五、完整集成示例

### 5.1 IM 完整示例

```java
// ========== 1. 登录后启动 IM 服务 ==========
Intent intent = CoreService.getIntent(context, userId, password, nickName);
ContextCompat.startForegroundService(context, intent);

// 绑定服务获取 CoreService 引用
bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE);

// ========== 2. 注册监听 ==========
// 连接状态
ListenerManager.getInstance().addAuthStateChangeListener(authListener);
// 消息监听
ListenerManager.getInstance().addChatMessageListener(msgListener);
// 好友请求监听
ListenerManager.getInstance().addNewFriendListener(newFriendListener);

// ========== 3. 发送消息 ==========
ChatMessage msg = new ChatMessage();
msg.setType(XmppMessage.TYPE_TEXT);
msg.setContent(messageContent);
msg.setFromUserId(userId);
msg.setFromUserName(nickName);
msg.setToUserId(targetUserId);
msg.setPacketId(UUID.randomUUID().toString().replaceAll("-", ""));
msg.setDoubleTimeSend(System.currentTimeMillis() / 1000.0);
coreService.sendChatMessage(targetUserId, msg);

// ========== 4. 发送好友请求 ==========
NewFriendMessage nfm = NewFriendMessage.createWillSendMessage(
    user, XmppMessage.TYPE_SAYHELLO, "你好", 
    friendId, friendName, companyId
);
coreService.sendNewFriendMessage(friendId, nfm);

// ========== 5. 退出登录时 ==========
ListenerManager.getInstance().removeAuthStateChangeListener(authListener);
ListenerManager.getInstance().removeChatMessageListener(msgListener);
ListenerManager.getInstance().removeNewFriendListener(newFriendListener);
context.stopService(new Intent(context, CoreService.class));
```

### 5.2 AppStore 完整示例

```java
// ========== 1. Application 初始化 ==========
@Override
public void onCreate() {
    super.onCreate();
    AppStoreInit.setApplication(this);
    Constant.BASE_URL = "https://your-server.com/";
}

// ========== 2. 用户登录后初始化数据 ==========
// 假设用户登录成功，获取到 token 和 uuid
String token = loginResponse.getToken();
String uuid = UserInfo.getOort_uuid();
AppStoreInit.getInstance().initData(token, uuid);

// ========== 3. 打开应用市场首页 ==========
Intent intent = new Intent(context, MainActivity.class);
context.startActivity(intent);

// ========== 4. 打开应用详情页 ==========
AppInfo appInfo = getAppInfoFromList(); // 从列表接口获取
AppDetailedActivity.actionStart(context, appInfo);

// ========== 5. 切换用户时重新初始化 ==========
// 退出当前用户
AppStoreInit.getInstance().clearData();

// 新用户登录
String newToken = newLoginResponse.getToken();
String newUuid = newUserInfo.getOort_uuid();
AppStoreInit.getInstance().initData(newToken, newUuid);
```

---

## 六、注意事项

### 6.1 IM 模块注意事项

1. **必须在 REST API 登录成功后再启动 `CoreService`**，XMPP 登录依赖 `userId` 和 `accessToken`。
2. **CoreService 是前台服务**，Android 8.0+ 需使用 `ContextCompat.startForegroundService()` 启动。
3. **消息加密**：IM SDK 支持 3DES/AES/DH+AES 多种加密方式，`Friend.getEncryptType()` 标识加密类型（0=明文, 1=3DES, 2=AES, 3=DH+AES）。
4. **消息过期**：`ChatMessage.getDeleteTime()` 标识消息的自动销毁时间，可配合 `chatRecordTimeOut` 字段实现消息限时销毁。
5. **多点登录**：通过 `TYPE_SEND_ONLINE_STATUS (200)` 协议实现多设备在线状态同步。
6. **本地消息持久化**：所有消息和好友关系均存储在本地 SQLite 数据库中，SDK 内部自动处理。

### 6.2 AppStore 模块注意事项

1. **初始化时提示 token/uuid 为空？**  
   确保在调用 `initData()` 之前，用户已成功登录，且 `FastSharedPreferences` 中已保存 `token` 信息。

2. **应用详情页打开后立即关闭？**  
   检查 `appInfo` 对象是否为 null。详情页启动时要求传入非空的 `AppInfo` 实例。

3. **下载后无法安装 APK？**  
   Android 8+ 需要引导用户开启「安装未知应用」权限；Android 11+ 还可能需申请 `MANAGE_EXTERNAL_STORAGE` 权限。模块已内置相关逻辑，用户按提示操作即可。

4. **切换用户后数据未刷新？**  
   切换用户后需重新调用 `AppStoreInit.getInstance().initData(newToken, newUuid)` 刷新本地数据。

5. **模块使用了系统签名？**  
   模块 Manifest 中声明了 `android:sharedUserId="android.uid.system"`，如需卸载或独立使用该功能，请移除该属性。

---

## 七、常见问题

### Q1: 如何同时使用 IM 和 AppStore？

A: 只需引入一次 `omm-lib`，两个模块会自动集成。分别按照各自的初始化流程配置即可。

### Q2: 本地 AAR 集成时缺少依赖怎么办？

A: 确保宿主项目已引入 `:core` 和 `:ooortCloudDisk` 模块，或使用远程依赖方式自动引入。

### Q3: IM 消息收不到怎么办？

A: 检查以下几点：
- 确认 `CoreService` 已启动且连接成功（监听 `AUTH_STATE_SUCCESS`）
- 确认已注册消息监听器
- 检查网络连接是否正常
- 查看 Logcat 中是否有 XMPP 相关错误日志

### Q4: AppStore 下载失败怎么办？

A: 检查以下几点：
- 确认已申请存储权限和安装权限
- 检查 `Constant.BASE_URL` 是否正确配置
- 确认 `AppStoreInit.initData()` 已调用且 token/uuid 有效
- 查看 Logcat 中是否有网络请求错误

### Q5: 如何自定义 UI 样式？

A: IM 和 AppStore 模块均使用默认主题，如需自定义样式，可在宿主项目中覆盖相关资源文件或继承 Activity 后修改布局。

---

## 八、技术支持

如有问题，请联系技术支持团队或查阅以下资源：

- 📖 [GitHub 仓库](https://github.com/Bean-V/WorkUp-SDK)
- 📝 [Release 发布页](https://github.com/Bean-V/WorkUp-SDK/releases)
-  [Issue 反馈](https://github.com/Bean-V/WorkUp-SDK/issues)

---

**文档版本**: v1.0.0  
**最后更新**: 2026-07-10  
**适用 SDK 版本**: omm-lib 1.0.0+
