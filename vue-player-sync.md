# 前端播放器状态与后端 API 衔接机制

## 概述

Koel 播放器的状态同步涉及三层协作：Vue 端状态管理、后端流式接口、以及基于双 Token 的会话保持。三者共同确保播放状态在客户端与服务端之间的一致性和安全性。

---

## 一、Vue 端状态管理

### 1.1 状态分层架构

前端播放状态分布在三个 Store 中，各司其职：

| Store | 职责 | 核心文件 |
|-------|------|----------|
| `playableStore` | 管理所有可播放项的元数据与播放状态 | [playableStore.ts](resources/assets/js/stores/playableStore.ts) |
| `queueStore` | 管理播放队列（列表、当前项、上下首） | [queueStore.ts](resources/assets/js/stores/queueStore.ts) |
| `commonStore` | 存储服务端下发的初始播放状态 | [commonStore.ts](resources/assets/js/stores/commonStore.ts) |

### 1.2 播放状态机

每个可播放项（`Playable`）都有一个 `playback_state` 属性，有三种状态：

- `Stopped` — 停止态（默认）
- `Playing` — 播放中
- `Paused` — 已暂停

状态迁移由 [QueuePlaybackService](resources/assets/js/services/QueuePlaybackService.ts) 控制：

```typescript
// 播放时
playable.playback_state = 'Playing'

// 暂停时
queueStore.current!.playback_state = 'Paused'

// 停止时
queueStore.current && (queueStore.current.playback_state = 'Stopped')
```

### 1.3 服务层抽象

播放逻辑通过服务层封装，支持多种播放源：

- [BasePlaybackService](resources/assets/js/services/BasePlaybackService.ts) — 抽象基类，定义播放/暂停/停止等接口
- [QueuePlaybackService](resources/assets/js/services/QueuePlaybackService.ts) — 队列播放实现
- [playbackManager](resources/assets/js/services/playbackManager.ts) — 播放管理器，切换播放源

### 1.4 状态同步触发点

前端主动向后端同步状态的时机：

| 触发时机 | 频率/条件 | API 端点 | 代码位置 |
|---------|----------|----------|----------|
| 播放开始 | 每次切歌 | `PUT /api/queue/playback-status` | [QueuePlaybackService.ts](resources/assets/js/services/QueuePlaybackService.ts) |
| 播放进度 | 每 5 秒 | `PUT /api/queue/playback-status` | [QueuePlaybackService.ts](resources/assets/js/services/QueuePlaybackService.ts) |
| 队列变更 | 队列变化时 | `PUT /api/queue/state` | [queueStore.ts](resources/assets/js/stores/queueStore.ts) |
| 播放计数 | 播放超过 25% 时 | `POST /api/interaction/play` | [playableStore.ts](resources/assets/js/stores/playableStore.ts) |

### 1.5 初始化流程

应用启动时，通过 [FetchInitialDataController](app/Http/Controllers/API/FetchInitialDataController.php) 获取初始数据，其中包含 `queue_state`：

1. 后端 `FetchInitialDataController` 返回 `queue_state`（队列歌曲 + 当前歌曲 + 播放位置）
2. 前端 `commonStore.init()` 接收数据
3. `queueStore.init(savedState)` 恢复队列，将当前歌曲设为 `Paused` 状态
4. 用户点击播放时，`QueuePlaybackService.resume()` 从保存的 `playback_position` 继续播放

```typescript
// queueStore.init 恢复队列状态
init(savedState: QueueState) {
  this.state.playables = playableStore.syncWithVault(savedState.songs)
  if (savedState.current_song) {
    playableStore.syncWithVault(savedState.current_song)[0].playback_state = 'Paused'
  }
}

// resume 时从保存位置继续
if (!this.media.src) {
  this.media.src = playableStore.getSourceUrl(playable)
  this.seekTo(commonStore.state.queue_state.playback_position)
}
```

---

## 二、后端流式接口

### 2.1 播放路由与认证

音频播放路由定义在 [web.base.php](routes/web.base.php) 中，使用独立的 `audio.auth` 中间件：

```php
Route::middleware('audio.auth')->group(static function (): void {
    Route::get('play/{song}/{transcode?}', PlayController::class)->name('song.play');
});
```

### 2.2 播放控制器

[PlayController](app/Http/Controllers/PlayController.php) 是流式播放的入口：

- 授权检查：`$this->authorize('access', $song)`
- 转码判断：根据 `transcode` 参数和用户偏好决定是否转码
- 流式输出：通过 `Streamer` 类返回音频流

```php
public function __invoke(Authenticatable $user, SongPlayRequest $request, Song $song, ?bool $transcode = null)
{
    $this->authorize('access', $song);
    
    $transcodeBitRate = $transcode
        ? (int) filter_var($user->preferences->transcodeQuality, FILTER_SANITIZE_NUMBER_INT)
        : null;

    return (new Streamer(song: $song, config: RequestedStreamingConfig::make(
        transcode: (bool) $transcode,
        bitRate: $transcodeBitRate,
        startTime: (float) $request->time,
    )))->stream();
}
```

### 2.3 流式适配器

[Streamer](app/Services/Streamer/Streamer.php) 使用适配器模式，根据歌曲存储类型和转码需求选择不同的流式实现：

| 适配器 | 适用场景 |
|--------|----------|
| `LocalStreamerAdapter` | 本地存储文件 |
| `S3CompatibleStreamerAdapter` | S3 兼容存储 |
| `SftpStreamerAdapter` | SFTP 存储 |
| `DropboxStreamerAdapter` | Dropbox 存储 |
| `TranscodingStreamerAdapter` | 需要转码时 |
| `PodcastStreamerAdapter` | 播客剧集 |

转码触发条件：
1. 移动端强制转码（`transcode=true`）
2. 浏览器不支持的音频格式
3. FLAC 文件且配置了 `TRANSCODE_FLAC=true`

### 2.4 播放状态持久化

后端通过 [QueueState](app/Models/QueueState.php) 模型持久化播放状态，存储在数据库中：

| 字段 | 说明 |
|------|------|
| `user_id` | 用户 ID |
| `song_ids` | 队列歌曲 ID 数组 |
| `current_song_id` | 当前播放歌曲 ID |
| `playback_position` | 播放位置（秒） |

[QueueService](app/Services/QueueService.php) 提供读写接口：

- `getQueueState($user)` — 获取队列状态（首次调用自动创建空记录）
- `updateQueueState($user, $songIds)` — 更新队列列表
- `updatePlaybackStatus($user, $song, $position)` — 更新当前播放歌曲和位置

对应 API 控制器：
- [QueueStateController](app/Http/Controllers/API/QueueStateController.php) — 队列状态读写
- [UpdatePlaybackStatusController](app/Http/Controllers/API/UpdatePlaybackStatusController.php) — 播放进度更新

---

## 三、会话保持：双 Token 机制

### 3.1 为什么需要双 Token

音频播放使用 `<audio>` 元素的 `src` 属性直接请求音频文件，浏览器会发起 GET 请求，**无法**自定义 `Authorization` 请求头。如果直接把 API Token 放在 URL 里，会有以下风险：

- Token 可能被代理服务器、CDN 日志记录
- 浏览器历史记录可能泄露 Token
-  Referer 头可能把 Token 泄露给第三方网站

因此 Koel 采用**双 Token 机制**（Composite Token）：

### 3.2 CompositeToken 结构

[CompositeToken](app/Values/CompositeToken.php) 包含两个 Token：

| Token 类型 | 权限 | 用途 | 传递方式 |
|-----------|------|------|----------|
| API Token | `*`（全部权限） | API 请求 | `Authorization: Bearer` 请求头 |
| Audio Token | 仅 `audio` 权限 | 音频播放下载 | URL 查询参数 `?t=` |

Audio Token 只有播放和下载音频的权限，即使泄露也无法操作用户数据。

### 3.3 Token 创建与分发

登录时由 [TokenManager](app/Services/Auth/TokenManager.php) 创建双 Token：

```php
public function createCompositeToken(User $user): CompositeToken
{
    $token = CompositeToken::fromAccessTokens(
        api: $this->createToken($user),           // 权限：['*']
        audio: $this->createToken($user, ['audio']), // 权限：['audio']
    );

    // 缓存 API Token 到 Audio Token 的映射，便于注销
    Cache::forever("app.composite-tokens.$token->apiToken", $token->audioToken);

    return $token;
}
```

前端通过 [authService](resources/assets/js/services/authService.ts) 接收并存储两个 Token：

```typescript
setTokensUsingCompositeToken(compositeToken: CompositeToken) {
  this.setApiToken(compositeToken.token)
  this.setAudioToken(compositeToken['audio-token'])
}
```

### 3.4 请求时的 Token 使用

**API 请求**（[http.ts](resources/assets/js/services/http.ts)）：

```typescript
hooks: {
  beforeRequest: [
    request => {
      request.headers.set('Authorization', `Bearer ${authService.getApiToken()}`)
    },
  ],
}
```

**音频播放**（[playableStore.ts](resources/assets/js/stores/playableStore.ts)）：

```typescript
getSourceUrl: (playable: Playable) => {
  return isMobile.any && preferenceStore.transcode_on_mobile
    ? `${commonStore.state.cdn_url}play/${playable.id}/1?t=${authService.getAudioToken()}`
    : `${commonStore.state.cdn_url}play/${playable.id}?t=${authService.getAudioToken()}`
}
```

### 3.5 会话保持链路详解

从 `?t=` 参数到解析为当前用户，完整链路涉及五层：

```
请求到达
  │
  ▼
1. 路由匹配：web 中间件组 + audio.auth 中间件
  │  路由定义在 routes/web.base.php
  │ 使用 Route::middleware('web')->group(...)
  │
  ▼
2. Guard 选择：web guard → token-via-query-parameter 驱动
  │  config/auth.php 中配置：
  │  'web' => ['driver' => 'token-via-query-parameter']
  │
  ▼
3. 自定义 Guard 解析 Token（AuthServiceProvider）
  │  Auth::viaRequest('token-via-query-parameter', function (Request $request): ?User {
  │      $token = $request->get('api_token') ?: $request->get('t');
  │      return app(TokenManager::class)->getUserFromPlainTextToken($token ?: '');
  │  });
  │
  ▼
4. TokenManager 查找用户
  │  PersonalAccessToken::findToken($plainTextToken)?->tokenable
  │  从 personal_access_tokens 表查找 token 记录
  │  通过 tokenable 多态关系获取 User 模型
  │
  ▼
5. AudioAuthenticate 中间件验证权限
  │  abort_unless($request->user()?->tokenCan('audio'), 401);
  │  调用 Sanctum 的 tokenCan() 方法检查 token 能力
  │
  ▼
请求通过，进入 PlayController
```

下面逐层详解每一层的代码实现：

#### 3.5.1 认证配置

[config/auth.php](config/auth.php) 定义了两个 Guard：

```php
'guards' => [
    'web' => [
        'driver' => 'token-via-query-parameter',
        'provider' => 'users',
    ],
    'api' => [
        'driver' => 'sanctum',
        'provider' => 'users',
    ],
],
```

- **web guard**：驱动为 `token-via-query-parameter`（自定义），用于播放等 web 路由
- **api guard**：驱动为 `sanctum`，用于 API 路由
- 默认 guard 是 `api`

播放路由在 `web` 中间件组中，因此使用 `web` guard。

#### 3.5.2 AuthServiceProvider：注册自定义 Guard

[AuthServiceProvider](app/Providers/AuthServiceProvider.php) 的 `boot()` 方法中通过 `Auth::viaRequest()` 注册了自定义认证驱动：

```php
Auth::viaRequest('token-via-query-parameter', static function (Request $request): ?User {
    $token = $request->get('api_token') ?: $request->get('t');

    return app(TokenManager::class)->getUserFromPlainTextToken($token ?: '');
});
```

关键点：
- `viaRequest()` 是 Laravel 的闭包请求认证方式，适合自定义 Guard 的简单方式
- 支持两个参数名：`api_token` 和 `t`（后者用于音频播放）
- 解析逻辑委托给 `TokenManager::getUserFromPlainTextToken()

#### 3.5.3 TokenManager：Token 解析用户

[TokenManager](app/Services/Auth/TokenManager.php) 的 `getUserFromPlainTextToken()` 方法：

```php
public function getUserFromPlainTextToken(#[SensitiveParameter] string $plainTextToken): ?User
{
    return PersonalAccessToken::findToken($plainTextToken)?->tokenable;
}
```

实现细节：
- 使用 Laravel\Sanctum\PersonalAccessToken::findToken() 查找 token 记录
- `findToken()` 会自动处理 token ID 和哈希比对
- 通过 `tokenable` 多态关系获取关联的 User 模型
- 找不到 token 或 token 无效时返回 `null`

Sanctum 的 `findToken()` 内部逻辑（概念性说明）：
- Token 格式为 `ID|plain-text`
- 先按 ID 查询数据库
- 再比对哈希后的明文
- 返回 Token 模型实例

#### 3.5.4 AudioAuthenticate：验证 audio 权限

[AudioAuthenticate](app/Http/Middleware/AudioAuthenticate.php) 中间件：

```php
public function handle(Request $request, Closure $next)
{
    abort_unless($request->user()?->tokenCan('audio'), Response::HTTP_UNAUTHORIZED);

    return $next($request);
}
```

关键点：
- `$request->user()` 触发 Guard 的 user() 方法，触发上述认证流程
- `tokenCan('audio')` 是 Sanctum 提供的能力检查方法
- 检查 Token 的 abilities 字段中是否包含 `audio` 或 `*`
- 无用户或无权限时返回 401

对比 [Authenticate](app/Http/Middleware/Authenticate.php) 中间件（API 认证）：

```php
public function handle(Request $request, Closure $next)
{
    if ($request->user()?->tokenCan('*')) {
        return $next($request);
    }
    // ...
}
```

- API 认证检查 `*` 权限（全权限）
- Audio 认证只检查 `audio` 权限（最小权限）

#### 3.5.5 中间件注册

[bootstrap/app.php](bootstrap/app.php) 中注册中间件别名：

```php
$middleware->alias([
    'audio.auth' => AudioAuthenticate::class,
    'os.auth' => ObjectStorageAuthenticate::class,
]);
```

#### 3.5.6 链路总结

| 层级 | 组件 | 职责 | 关键代码 |
|------|------|------|-----------|
| 1 | 路由层 | web 中间件组 + audio.auth 别名 | [web.base.php](routes/web.base.php) |
| 2 | Guard 配置 | web guard 使用自定义驱动 | [auth.php](config/auth.php) |
| 3 | Guard 实现 | 从查询参数提取 token | [AuthServiceProvider.php](app/Providers/AuthServiceProvider.php) |
| 4 | Token 解析 | 查找 token 记录 → tokenable 关系 | [TokenManager.php](app/Services/Auth/TokenManager.php) |
| 5 | 权限验证 | 检查 token 的 audio 能力 | [AudioAuthenticate.php](app/Http/Middleware/AudioAuthenticate.php) |

### 3.6 Token 注销

登出时同时删除两个 Token（[TokenManager::deleteCompositionToken](app/Services/Auth/TokenManager.php)）：

```php
public function deleteCompositionToken(string $plainTextApiToken): void
{
    $audioToken = Cache::get("app.composite-tokens.$plainTextApiToken");
    if ($audioToken) {
        $this->deleteTokenByPlainTextToken($audioToken);
        Cache::forget("app.composite-tokens.$plainTextApiToken");
    }
    $this->deleteTokenByPlainTextToken($plainTextApiToken);
}
```

---

## 四、三者衔接全景

### 4.1 完整播放流程

```
用户点击播放
    │
    ▼
QueuePlaybackService.play(playable)
    │
    ├─ 设置 playable.playback_state = 'Playing'
    ├─ 更新浏览器 MediaSession
    ├─ socketService 广播播放状态
    │
    ▼
设置 audio.src = getSourceUrl(playable)
    │
    └─ getSourceUrl 拼接 Audio Token 到 URL
        │
        ▼
浏览器发起 GET /play/{song}?t=<audio-token>
    │
    ▼
web 中间件组 → web guard (token-via-query-parameter)
    │
    ├─ AuthServiceProvider 从 ?t= 提取 token
    ├─ TokenManager 通过 PersonalAccessToken 查找用户
    │
    ▼
audio.auth 中间件 (AudioAuthenticate)
    │
    └─ 验证 token 具有 'audio' 能力
    │
    ▼
PlayController::__invoke()
    │
    ├─ authorize('access', $song) 检查歌曲访问权限
    ├─ 判断是否需要转码
    └─ Streamer 选择适配器并返回音频流
        │
        ▼
浏览器播放音频
    │
    ▼
timeupdate 事件（每秒触发，1000ms 节流）
    │
    ├─ 每 5 秒 → PUT /api/queue/playback-status 同步播放位置
    │   （携带 API Token 在 Authorization 头，走 api guard + sanctum 驱动）
    │
    ├─ 播放超过 25% → POST /api/interaction/play 增加播放计数
    │
    └─ 接近结尾（剩余 30 秒）→ 预加载下一首
```

### 4.2 状态同步方向

```
┌─────────────────────┐         初始加载          ┌─────────────────────┐
│                     │ ────────────────────────► │                     │
│  后端 (QueueState)  │   队列+当前歌曲+位置      │  前端 (Vue Store)   │
│                     │ ◄──────────────────────── │                     │
└─────────────────────┘    增量同步（播放时）      └─────────────────────┘
                            5秒一次播放进度
                            队列变更即时同步
                            播放计数达阈值同步
```

### 4.3 认证体系对比

| 场景 | 认证方式 | Guard 驱动 | Token 类型 | 权限范围 |
|------|----------|------------|-----------|----------|
| API 数据请求 | `Authorization: Bearer` Header | sanctum | API Token | 全部权限 |
| 音频流播放 | URL 查询参数 `?t=` | token-via-query-parameter | Audio Token | 仅 audio 权限 |
| 初始数据加载 | API 请求 | sanctum | API Token | 全部权限 |
| 播放进度同步 | API 请求 | sanctum | API Token | 全部权限 |

---

## 五、关键设计要点

### 5.1 安全性

- **最小权限原则**：Audio Token 仅有 `audio` 权限，泄露不影响账号安全
- **传输通道分离**：高权限 Token 走 Header，低权限 Token 走 URL
- **独立认证中间件**：`audio.auth` 只检查 audio 能力，不授予更多权限
- **自定义 Guard**：通过 `Auth::viaRequest()` 实现简洁的查询参数认证

### 5.2 性能与体验

- **渐进加载**：歌曲不一次性全部加载，按需获取（vault 模式）
- **预加载**：当前歌曲剩余 30 秒时预加载下一首
- **淡入淡出**：支持 crossfade，切换歌曲无间隙
- **进度保存**：刷新页面后可从上次位置继续播放

### 5.3 状态一致性

- **前端优先**：UI 状态立即更新，服务端同步是补充
- **服务端权威**：播放计数以服务端返回值为准（`registerPlay` 中更新）
- **定期同步**：每 5 秒同步播放进度，防止刷新丢失过多进度
