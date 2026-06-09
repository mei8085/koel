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

从 `?t=` 参数到解析为当前用户，完整链路涉及 Guard 驱动、Token 解析、权限验证三层核心环节。下面沿着 Laravel 框架层深入追踪各组件的实际调用路径与关联。

#### 3.5.1 双 Guard 架构：配置与实际生效

[config/auth.php](config/auth.php) 定义了两个认证 Guard，但**配置中的默认值 ≠ 实际生效的 Guard**。

**配置文件定义**：

```php
// config/auth.php
'defaults' => [
    'guard' => 'api',      // 配置中的默认 Guard
    'passwords' => 'users',
],

'guards' => [
    'web' => [
        'driver' => 'token-via-query-parameter',  // 自定义驱动
        'provider' => 'users',
    ],
    'api' => [
        'driver' => 'sanctum',                    // Sanctum 官方驱动
        'provider' => 'users',
    ],
],
```

**实际生效规则**：

Laravel 的设计约定是「路由所在的中间件组决定默认 Guard」：

| 路由中间件组 | 实际默认 Guard | 驱动 | Token 提取方式 | 典型场景 |
|-------------|--------------|------|---------------|----------|
| `web` 组 | `web` guard | `token-via-query-parameter` | 查询参数 `api_token` / `t` | 播放、下载、Last.fm 回调 |
| `api` 组 | `api` guard | `sanctum` | `Authorization: Bearer` Header | 数据读写、状态同步 |

> **验证依据**：[LastfmTest.php](tests/Feature/LastfmTest.php) 第 40 行 Mock `TokenManager::getUserFromPlainTextToken()` 被期望调用，证明 web 路由上自定义驱动确实参与了认证。

**核心设计洞察**：
- 两个 Guard 不是「两套独立认证体系」，而是「同一套 Token 存储 + 两种 Token 提取方式」
- 两者底层都依赖 Sanctum 的 `PersonalAccessToken` 模型
- 差异仅在于「从哪里提取 Token 字符串」，验证逻辑完全一致
- 都能设置 `currentAccessToken`，因此都支持 `tokenCan()` 权限检查

#### 3.5.2 $request->user() 的框架层调用链

`$request->user()` 是认证链路的总入口。沿着 Laravel 框架层向下追踪，调用链如下：

```
Illuminate\Http\Request::user($guard = null)
    │
    ├─ 调用 $this->userResolver 闭包
    │   （由框架在服务容器初始化时注册）
    │
    ▼
    userResolver($guard)
    │
    ├─ 调用 auth() 辅助函数 → 获取 AuthManager 实例
    │
    ▼
    auth()->guard($guard)->user()
    │
    ├─ $guard = null → 使用默认 Guard
    │   （由路由中间件组决定）
    │
    ▼
    defaultGuard()->user()
    │
    └─ 根据 Guard 驱动类型执行不同逻辑：
       ├─ web guard → RequestGuard → 执行 viaRequest 闭包
       └─ api guard → Sanctum Guard → 执行 Sanctum 认证逻辑
```

**关键细节说明**：

1. **userResolver 的注入**：Laravel 在 `AuthServiceProvider` 中通过 `$request->setUserResolver()` 将用户解析器闭包注入到 Request 实例。

2. **默认 Guard 的动态切换**：当请求进入 web 中间件组时，框架将默认 Guard 设置为 `web`；进入 api 中间件组时设置为 `api`。这解释了为什么同一段 `$request->user()` 代码在不同路由上行为不同。

3. **惰性求值**：`$request->user()` 采用惰性求值，首次调用时才执行认证逻辑，结果会被 Guard 缓存，后续调用直接返回缓存的用户。

**对 AudioAuthenticate 的影响**：

[AudioAuthenticate](app/Http/Middleware/AudioAuthenticate.php) 第 13 行：

```php
abort_unless($request->user()?->tokenCan('audio'), Response::HTTP_UNAUTHORIZED);
```

由于播放路由位于 `web` 中间件组内，这里的 `$request->user()` 使用 **web Guard**，走 `token-via-query-parameter` 驱动，从 `?t=` 参数提取 Token。

#### 3.5.3 自定义驱动：token-via-query-parameter

`web` Guard 的驱动由 [AuthServiceProvider](app/Providers/AuthServiceProvider.php) 第 20 行通过 `Auth::viaRequest()` 注册：

```php
Auth::viaRequest('token-via-query-parameter', static function (Request $request): ?User {
    $token = $request->get('api_token') ?: $request->get('t');

    return app(TokenManager::class)->getUserFromPlainTextToken($token ?: '');
});
```

**viaRequest 的工作原理**：

| 层级 | 组件 | 说明 |
|------|------|------|
| 注册层 | `Auth::viaRequest($name, $callback)` | 在 AuthManager 中注册一个自定义驱动 |
| 实例化层 | `RequestGuard` | Laravel 为 viaRequest 驱动创建的 Guard 实例 |
| 执行层 | 闭包函数 | 调用 `guard()->user()` 时执行，接收 `$request`，返回 User 或 null |

**驱动逻辑拆解**：
1. 从请求查询参数中尝试获取 `api_token`，若无则尝试 `t`
2. 若两者都没有，传空字符串给 `getUserFromPlainTextToken()`
3. `TokenManager` 负责实际的 Token 查找和用户解析
4. 找到则返回 User 模型，找不到返回 `null`

#### 3.5.4 Token 解析与 currentAccessToken 的绑定来源

这是理解认证链路最关键的一环：**自定义驱动返回的 User 模型上，currentAccessToken 是从哪里来的？**

**TokenManager 的解析方法** [TokenManager.php](app/Services/Auth/TokenManager.php) 第 54 行：

```php
public function getUserFromPlainTextToken(#[SensitiveParameter] string $plainTextToken): ?User
{
    return PersonalAccessToken::findToken($plainTextToken)?->tokenable;
}
```

看似简单的一行代码，实际包含了「Token 查找 → 用户获取 → Token 自动绑定」三步。

**逐阶段追踪**：

**第一阶段：PersonalAccessToken::findToken()**

- 接收明文 Token，格式为 `{id}|{plain_text}`
- 解析出 ID，查询 `personal_access_tokens` 表
- 使用 SHA-256 哈希比对明文验证
- 验证通过返回 `PersonalAccessToken` Eloquent 模型

**第二阶段：访问 ->tokenable 属性**

这是自动绑定的关键。`tokenable` 是 Sanctum 定义的「多态关联」，其访问器（getter）中包含了绑定逻辑：

```
$token->tokenable
    │
    ├─ 触发 Eloquent 多态关联加载
    │
    ├─ 获取关联的 User 模型
    │
    └─ 在返回 User 之前：
        $user->withAccessToken($token)
        │
        └─ 设置 $user->currentAccessToken = $token
```

**withAccessToken 方法**由 `HasApiTokens` trait 提供，它将当前 Token 设置到 User 模型的 `currentAccessToken` 属性上。

**关键结论**：
> `currentAccessToken` 的绑定发生在 `PersonalAccessToken` 模型的 `tokenable` 关联访问器中。**无论通过什么方式获取到 PersonalAccessToken，只要访问它的 tokenable 属性，就会自动将自身绑定为 User 的 currentAccessToken。**

这就是为什么自定义 viaRequest 驱动返回的 User 也能正确使用 `tokenCan()` 的原因 —— 因为 `tokenable` 关系自带绑定逻辑。

#### 3.5.5 tokenCan 的工作原理：与 Guard 无关

`tokenCan()` 的工作完全不依赖于 Guard，它的唯一依赖是 `currentAccessToken` 属性。

**定义与实现**：

`tokenCan()` 由 `Laravel\Sanctum\HasApiTokens` trait 提供，[User 模型](app/Models/User.php) 第 64 行引入了该 trait：

```php
use HasApiTokens;
```

**执行逻辑**：

```
$user->tokenCan('audio')
    │
    ├─ 读取 $this->currentAccessToken
    │
    ├─ 若 currentAccessToken 为空 → 返回 false
    │
    ├─ 检查 abilities 是否包含 '*' → 是则返回 true
    │
    └─ 检查 abilities 是否包含指定能力 → 返回结果
```

**与 Guard 的关系**：

| Guard | 认证方式 | 是否设置 currentAccessToken | tokenCan 是否可用 |
|-------|---------|--------------------------|------------------|
| web guard（自定义驱动） | 查询参数 Token + PersonalAccessToken + tokenable | ✅ 通过 tokenable 自动绑定 | ✅ 可用 |
| api guard（Sanctum 驱动） | Header Token + Sanctum Guard | ✅ Sanctum Guard 显式设置 | ✅ 可用 |

**设计意义**：
- `tokenCan()` 与 Guard 解耦，只要 User 模型上设置了 `currentAccessToken`，就能检查权限
- 这使得「双 Guard 架构」成为可能：两种不同的认证入口，共享同一套权限检查机制
- 权限数据存储在 `personal_access_tokens.abilities` 字段（JSON 类型）

**权限对比表**：

| Token 类型 | abilities | tokenCan('audio') | tokenCan('*') | 用途 |
|-----------|-----------|-------------------|---------------|------|
| API Token | `['*']` | ✅ 通过 | ✅ 通过 | API 请求、全权限操作 |
| Audio Token | `['audio']` | ✅ 通过 | ❌ 不通过 | 仅播放/下载音频 |

这是双 Token 机制的安全性基础：Audio Token 即使泄露，也无法访问 API 或操作用户数据。

#### 3.5.6 三层 Authenticate 角色澄清

Koel 代码中存在三个不同层级的「Authenticate」概念，极易混淆。下面逐一澄清。

**概念辨析表**：

| 层级 | 类名 | 别名 | 注册位置 | 检查逻辑 | 适用场景 | 参与播放认证？ |
|------|------|------|---------|----------|----------|--------------|
| 框架层 | `Illuminate\Auth\Middleware\Authenticate` | `auth` | 框架默认 | 支持多 Guard，抛出 AuthenticationException | API 路由、需全权限的 web 路由 | ❌ 不参与 |
| 应用层 | `App\Http\Middleware\Authenticate` | （未注册） | — | `$request->user()?->tokenCan('*')` | 可能为遗留/备用方案 | ❌ 不参与 |
| 业务层 | `App\Http\Middleware\AudioAuthenticate` | `audio.auth` | bootstrap/app.php | `$request->user()?->tokenCan('audio')` | 播放、下载等音频路由 | ✅ 参与 |

**详细分析**：

**1. 框架层 Authenticate（`auth` 别名）**

- Laravel 框架自带的标准认证中间件
- [bootstrap/app.php](bootstrap/app.php) 第 47 行 `$middleware->redirectGuestsTo('/')` 配置了其重定向路径
- 由 [AuthTest.php](tests/Feature/AuthTest.php) 第 58 行注释「Laravel 12's Authenticate middleware」印证
- 支持多 Guard 参数：`auth:web`、`auth:api`，不传则用默认 Guard
- 用于 API 路由和 Last.fm 等需要全权限认证的 web 路由

**2. 应用层 Authenticate（自定义）**

- 文件：[app/Http/Middleware/Authenticate.php](app/Http/Middleware/Authenticate.php)
- 未在 `bootstrap/app.php` 中注册为 `auth` 别名
- 实现简化：直接检查 `tokenCan('*')`，不支持多 Guard 参数
- **状态判断**：当前为未激活的备用/遗留代码，`auth` 别名实际指向框架版

**3. 业务层 AudioAuthenticate（`audio.auth` 别名）**

- 文件：[app/Http/Middleware/AudioAuthenticate.php](app/Http/Middleware/AudioAuthenticate.php)
- 别名 `audio.auth`，在 [bootstrap/app.php](bootstrap/app.php) 第 41 行显式注册
- 仅检查 `audio` 权限，专用于播放、下载等音频路由
- 是播放路由认证链路上**唯一**的认证中间件

**播放路由的中间件执行路径**：

```
GET /play/{song}?t=xxx
    │
    ▼
web 中间件组（StartSession、CSRF、SubstituteBindings 等）
    │  ← 此时默认 Guard 已切换为 web
    ▼
audio.auth 中间件（AudioAuthenticate::handle）
    │  ← 唯一的认证检查点
    │  1. $request->user() → web Guard → 自定义驱动
    │  2. tokenCan('audio') → 检查权限
    ▼
PlayController
```

**结论**：`app/Http/Middleware/Authenticate.php` 与播放路由认证**完全无关**。播放路由的认证仅通过 `audio.auth` → `AudioAuthenticate` 完成。

#### 3.5.7 完整调用链全景：为什么带 t 参数的请求能通过 AudioAuthenticate

将前面所有分析串联起来，带 `?t=` 参数的播放请求能通过 AudioAuthenticate 认证，完整链路如下：

```
                        请求进入
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│  1. 路由匹配阶段                                    │
│     路由：routes/web.base.php 中的播放路由           │
│     中间件组：web + audio.auth                      │
└─────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│  2. web 中间件组执行                                │
│     ├─ StartSession、VerifyCsrfToken 等            │
│     └─ 默认 Guard 切换为 web                        │
└─────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│  3. audio.auth 中间件（AudioAuthenticate）执行      │
│     调用 $request->user()?->tokenCan('audio')      │
└─────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│  4. $request->user() 触发认证                       │
│     ├─ userResolver 闭包                            │
│     └─ auth()->guard('web')->user()                │
└─────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│  5. web Guard 驱动执行                              │
│     驱动名：token-via-query-parameter               │
│     位置：AuthServiceProvider::viaRequest 闭包       │
│     操作：从 $request->get('t') 提取 Token          │
└─────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│  6. TokenManager::getUserFromPlainTextToken()       │
│     调用 PersonalAccessToken::findToken($token)     │
│     ├─ 解析 {id}|{plain_text} 格式                  │
│     ├─ 按 ID 查 personal_access_tokens 表           │
│     └─ SHA-256 哈希比对验证                         │
└─────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│  7. 访问 tokenable 多态关系                         │
│     ├─ 获取关联的 User 模型                          │
│     └─ 自动绑定：$user->withAccessToken($token)     │
│        （Sanctum 在 tokenable 访问器中自动调用）     │
│        → 设置 $user->currentAccessToken             │
└─────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│  8. tokenCan('audio') 权限检查                      │
│     ├─ 读取 currentAccessToken->abilities          │
│     ├─ 检查是否包含 'audio' 或 '*'                  │
│     └─ Audio Token abilities = ['audio'] → 通过     │
└─────────────────────────────────────────────────────┘
                          │
                          ▼
                    认证通过 → PlayController
```

**各层组件职责对照表**：

| 层级 | 组件 | 职责 | 关键文件 |
|------|------|------|----------|
| 路由层 | web 中间件组 + audio.auth 别名 | 路由分组与中间件分配 | [web.base.php](routes/web.base.php) |
| Guard 层 | web Guard + token-via-query-parameter 驱动 | 定义认证方式与 Token 提取逻辑 | [auth.php](config/auth.php)、[AuthServiceProvider.php](app/Providers/AuthServiceProvider.php) |
| Token 层 | TokenManager + PersonalAccessToken | Token 查找、哈希验证、用户解析 | [TokenManager.php](app/Services/Auth/TokenManager.php) |
| 模型层 | User + HasApiTokens + tokenable 自动绑定 | 用户模型 + 自动绑定当前 Token + 权限检查 | [User.php](app/Models/User.php) |
| 中间件层 | AudioAuthenticate | 认证拦截 + audio 权限检查 | [AudioAuthenticate.php](app/Http/Middleware/AudioAuthenticate.php) |
| 注册层 | bootstrap/app.php | 中间件别名注册 | [bootstrap/app.php](bootstrap/app.php) |

**核心结论**：
1. **双 Guard 本质**：同一套 PersonalAccessToken 存储 + 两种 Token 提取方式
2. **Guard 选择**：由路由所在中间件组决定，web 路由用 web Guard，api 路由用 api Guard
3. **Token 自动绑定**：通过 `PersonalAccessToken->tokenable` 关系访问用户时，Sanctum 会自动将 Token 绑定为 `currentAccessToken`
4. **tokenCan 与 Guard 无关**：只要 `currentAccessToken` 被设置，就能用 `tokenCan()` 检查权限
5. **Authenticate.php 不参与播放认证**：`app/Http/Middleware/Authenticate.php` 不参与播放路由，播放认证只走 `audio.auth` 中间件

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
