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

从 `?t=` 参数到解析为当前用户，完整链路涉及 Guard 驱动、Token 解析、权限验证三层核心环节。下面逐层分析各组件的职责与关联。

#### 3.5.1 双 Guard 架构

[config/auth.php](config/auth.php) 定义了两个独立的认证 Guard，分别服务于不同场景：

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

| Guard 名称 | 驱动类型 | 用途 | Token 来源 |
|-----------|----------|------|-----------|
| `web` | `token-via-query-parameter`（自定义） | 播放、下载、Last.fm 回调等 Web 路由 | URL 查询参数 `api_token` 或 `t` |
| `api` | `sanctum`（默认） | API 数据接口 | `Authorization: Bearer` 请求头 |

- 默认 Guard 为 `api`
- 两个 Guard 共享同一个 `users` Provider（Eloquent User 模型）
- 两者底层都依赖 Laravel Sanctum 的 `PersonalAccessToken` 模型存储 Token

**关键区别**：两个 Guard 的核心差异在于「如何从请求中提取 Token」不同，但最终都通过 `PersonalAccessToken` 表查询 Token 记录并通过 `tokenable` 多态关系获取用户。

#### 3.5.2 自定义 Guard 驱动：token-via-query-parameter

[AuthServiceProvider](app/Providers/AuthServiceProvider.php) 的 `boot()` 方法中通过 `Auth::viaRequest()` 注册了自定义认证驱动：

```php
Auth::viaRequest('token-via-query-parameter', static function (Request $request): ?User {
    $token = $request->get('api_token') ?: $request->get('t');

    return app(TokenManager::class)->getUserFromPlainTextToken($token ?: '');
});
```

**实现细节**：
- `viaRequest()` 是 Laravel 提供的「请求级闭包认证」方式，用于快速实现自定义 Guard 驱动
- 支持两个参数名：`api_token`（Last.fm 回调等场景）和 `t`（音频播放场景）
- Token 解析逻辑完全委托给 `TokenManager::getUserFromPlainTextToken()`
- 解析失败时返回 `null`，表示未认证

**与 Sanctum 的关系**：
- 该自定义驱动仅负责「从请求中提取 Token」
- 实际的 Token 查找和用户解析仍然使用 Sanctum 的 `PersonalAccessToken` 模型
- 这是一种「轻量级自定义 Guard + Sanctum 底层存储」的混合架构

#### 3.5.3 TokenManager：Token 解析用户

[TokenManager](app/Services/Auth/TokenManager.php) 是 Token 操作的核心服务，负责 Token 的创建、删除和用户解析。

`getUserFromPlainTextToken()` 方法是认证链路的关键环节：

```php
public function getUserFromPlainTextToken(#[SensitiveParameter] string $plainTextToken): ?User
{
    return PersonalAccessToken::findToken($plainTextToken)?->tokenable;
}
```

**执行流程**：
1. 调用 `PersonalAccessToken::findToken()` 查找 Token 记录
2. `findToken()` 内部解析 Token 格式（`ID|plain-text`）
3. 按 ID 查询数据库，比对哈希后的明文
4. 通过 `tokenable` 多态关系获取关联的 User 模型
5. 找不到 Token 或 Token 无效时返回 `null`

**Sanctum 的 findToken() 内部逻辑**：
- Token 格式为 `{id}|{plain_text}`
- 先按 ID 查询 `personal_access_tokens` 表
- 使用 SHA-256 哈希比对明文
- 返回 `PersonalAccessToken` 模型实例

#### 3.5.4 AudioAuthenticate 的调用链

[AudioAuthenticate](app/Http/Middleware/AudioAuthenticate.php) 是音频播放的认证中间件，别名 `audio.auth`。

```php
public function handle(Request $request, Closure $next)
{
    abort_unless($request->user()?->tokenCan('audio'), Response::HTTP_UNAUTHORIZED);

    return $next($request);
}
```

**调用链分解**：

**第一步：`$request->user()` 触发认证**

`$request->user()` 是整个认证链路的触发点。它内部调用 `auth()->user()`，使用默认 Guard（`api`，Sanctum 驱动）。

但对于携带 `?t=` 或 `?api_token=` 的 Web 路由请求，实际认证通过 `web` Guard 的 `token-via-query-parameter` 驱动生效（从 Last.fm 测试的 TokenManager 被调用可以验证）。

> **验证依据**：[LastfmTest.php](tests/Feature/LastfmTest.php) 中 Mock `TokenManager::getUserFromPlainTextToken()` 被期望调用，证明自定义驱动参与了认证过程。

**第二步：`tokenCan('audio')` 权限检查**

`tokenCan()` 是 Sanctum `HasApiTokens` trait 提供的方法，用于检查当前认证 Token 的能力（abilities）。

它检查 `personal_access_tokens.abilities` 字段中是否包含指定能力或 `*`（通配符）。

对于 Audio Token，abilities 为 `['audio']`；对于 API Token，abilities 为 `['*']`。

**与 Authenticate 中间件的对比**：

| 中间件 | 检查的能力 | 适用场景 |
|--------|------------|----------|
| `AudioAuthenticate` | `audio` | 播放、下载等音频相关路由 |
| `Authenticate`（`auth` 别名） | `*`（全权限） | API 数据接口 |

#### 3.5.5 Authenticate.php 是否参与播放路由认证？

**结论：播放路由认证过程中，`app/Http/Middleware/Authenticate.php` **不参与**。

**原因分析**：

1. **路由中间件不同**：
   - 播放路由使用 `audio.auth` 中间件（`AudioAuthenticate` 类）
   - 播放路由定义在 [routes/web.base.php](routes/web.base.php) 第 47 行，使用 `Route::middleware('audio.auth')->group(...)` 中
   - `auth` 中间件（无论哪个 Authenticate）仅用于需要全权限认证的路由（如 Last.fm 连接）

2. **Koel 的自定义 Authenticate 角色存疑**：
   - [app/Http/Middleware/Authenticate.php](app/Http/Middleware/Authenticate.php) 文件存在
   - 但在 [bootstrap/app.php](bootstrap/app.php) 中未注册为 `auth` 别名
   - `bootstrap/app.php` 仅注册了 `audio.auth` 和 `os.auth` 两个别名
   - `auth` 中间件别名由 Laravel 框架默认提供

3. **框架 Authenticate 的证据**：
   - [AuthTest.php](tests/Feature/AuthTest.php) 第 58 行注释提到「Laravel 12's Authenticate middleware」
   - `bootstrap/app.php` 中 `$middleware->redirectGuestsTo('/')` 是配置框架 Authenticate 中间件的重定向路径
   - 这表明 `auth` 中间件别名指向框架的 `Illuminate\Auth\Middleware\Authenticate`

4. **Koel 自定义 Authenticate 的现状**：
   - 自定义 `App\Http\Middleware\Authenticate` 类存在但未显式注册为 `auth` 别名
   - 其实现与框架 Authenticate 类似但更简化（检查 `tokenCan('*')`）
   - 可能为遗留代码或替代方案，当前是否被 `auth` 别名实际生效

**对播放路由的影响**：
- 播放路由不经过 `auth` 中间件
- 播放路由经过 `audio.auth` 中间件
- 所以无论是哪个 Authenticate 都不参与播放路由的认证过程

#### 3.5.6 tokenCan 的工作原理

`tokenCan()` 方法由 Sanctum 的 `HasApiTokens` trait 提供，定义在 `Laravel\Sanctum\HasApiTokens` 中。

**User 模型中的使用**：
- [User.php](app/Models/User.php) 第 23 行引入 `HasApiTokens` trait
- 第 64 行 `use HasApiTokens;`

**工作原理**：

1. **currentAccessToken**：当用户通过 Token 认证后，Sanctum 会将当前使用的 Token 设置到 User 模型的 `currentAccessToken` 属性上

2. **tokenCan() 方法**：
   - 检查 `currentAccessToken` 是否存在
   - 调用 Token 模型的 `can()` 方法检查 abilities
   - 支持 `*` 通配符（表示所有权限）

3. **abilities 存储**：
   - 存储在 `personal_access_tokens.abilities` 字段（JSON 类型）
   - 创建 Token 时指定，如 `['audio']` 或 `['*']`

**与 Guard 的关系**：
- `tokenCan()` 方法与使用哪个 Guard 无关
- 只要是通过 Sanctum Token 认证的用户（无论是通过哪个 Guard 驱动），都可以使用 `tokenCan()` 检查权限
- 这是因为两个 Guard 最终都使用 `PersonalAccessToken` 模型，认证后都会设置 `currentAccessToken`

#### 3.5.7 完整认证调用链总结

**播放路由（`?t=` 参数）的完整认证链路**：

```
GET /play/{song}?t=<audio-token>
    │
    ▼
1. 路由匹配：web 中间件组 + audio.auth 中间件
    │  routes/web.base.php 中定义
    │  Route::middleware('web')->group(...)
    │  内含 Route::middleware('audio.auth')->group(...)
    │
    ▼
2. 触发认证：AudioAuthenticate::handle() 调用 $request->user()
    │
    ▼
3. Guard 解析：web guard → token-via-query-parameter 驱动
    │  AuthServiceProvider 中通过 Auth::viaRequest() 注册
    │  从 ?t= 或 ?api_token= 提取 Token
    │
    ▼
4. Token 查找：TokenManager::getUserFromPlainTextToken()
    │  PersonalAccessToken::findToken($plainTextToken)
    │  解析 Token ID + 哈希比对
    │
    ▼
5. 用户获取：通过 tokenable 多态关系 → User 模型
    │  Sanctum 设置 currentAccessToken 属性
    │
    ▼
6. 权限验证：tokenCan('audio')
    │  检查 abilities 包含 'audio' 或 '*'
    │
    ▼
7. 请求通过 → PlayController
```

**各层组件对照表**：

| 层级 | 组件 | 职责 | 关键文件 |
|------|------|------|----------|
| 路由层 | web 中间件组 + audio.auth 别名 | 路由分组与中间件分配 | [web.base.php](routes/web.base.php) |
| Guard 配置 | web guard → token-via-query-parameter 驱动 | 定义 Guard 与驱动的映射 | [auth.php](config/auth.php) |
| 驱动实现 | AuthServiceProvider + viaRequest 闭包 | 从查询参数提取 Token | [AuthServiceProvider.php](app/Providers/AuthServiceProvider.php) |
| Token 解析 | TokenManager + PersonalAccessToken | 查找 Token 记录 → tokenable 关系获取用户 | [TokenManager.php](app/Services/Auth/TokenManager.php) |
| 权限验证 | AudioAuthenticate + tokenCan() | 检查 Token 的 audio 能力 | [AudioAuthenticate.php](app/Http/Middleware/AudioAuthenticate.php) |
| 中间件注册 | bootstrap/app.php | 注册 audio.auth 等中间件别名 | [bootstrap/app.php](bootstrap/app.php) |

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
