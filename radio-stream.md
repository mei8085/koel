# 电台流媒体接入与播放接管逻辑梳理

## 一、整体架构概览

Koel 的电台播放与本地曲库采用 **双播放服务 + 统一管理层** 的架构设计。两种播放模式共享同一个 HTMLMediaElement，但由各自的 PlaybackService 控制，通过 `playbackManager` 进行切换和协调。

```
┌──────────────────────────────────────────────────────────┐
│                     playbackManager                      │
│  (统一入口，负责服务切换、事件解绑、media element 接管)    │
└───────────┬──────────────────────────┬───────────────────┘
            │                          │
            ▼                          ▼
┌────────────────────┐    ┌────────────────────────────┐
│ QueuePlaybackService│    │   RadioPlaybackService     │
│  (本地曲库/队列播放) │    │    (电台流媒体播放)         │
└────────────────────┘    └────────────────────────────┘
            │                          │
            ▼                          ▼
┌────────────────────┐    ┌────────────────────────────┐
│   queueStore       │    │    radioStationStore       │
│  (播放队列状态)     │    │   (电台列表+当前播放状态)   │
└────────────────────┘    └────────────────────────────┘
                                      │
                                      ▼
                            ┌──────────────────┐
                            │  后端代理服务    │
                            │ (ICY元数据解析)  │
                            └──────────────────┘
```

---

## 二、外部流媒体接入流程

### 2.1 前端播放源 URL 构造

前端通过 `radioStationStore.getSourceUrl()` 构造电台流的播放地址，请求经过 Koel 后端代理，而不是直接连接电台服务器。

**关键代码**：[radioStationStore.ts#L55-L57](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/stores/radioStationStore.ts#L55-L57)

```typescript
getSourceUrl: (station: RadioStation) => {
  return `${commonStore.state.cdn_url}radio/stream/${station.id}?t=${authService.getAudioToken()}`
}
```

**设计要点**：
- URL 格式：`/radio/stream/{stationId}?t={audioToken}`
- 携带音频 Token 用于鉴权
- 所有流媒体请求统一经过后端代理层

### 2.2 后端流代理入口

`StreamRadioController` 是电台流的 HTTP 入口，负责鉴权后将请求转发给 `RadioStreamService`。

**关键代码**：[StreamRadioController.php#L10-L20](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/app/Http/Controllers/StreamRadioController.php#L10-L20)

```php
class StreamRadioController extends Controller
{
    public function __invoke(Authenticatable $user, RadioStation $radioStation, RadioStreamService $radioStreamService)
    {
        $this->authorize('access', $radioStation);
        return $radioStreamService->stream($radioStation);
    }
}
```

### 2.3 流媒体服务与 ICY 元数据

`RadioStreamService` 是流处理的核心，它通过 `RadioStreamProxy` 打开远程电台流，并根据流是否支持 ICY 元数据选择不同的代理策略。

**关键代码**：[RadioStreamService.php#L8-L42](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/app/Services/Radio/RadioStreamService.php#L8-L42)

```php
class RadioStreamService
{
    public function stream(RadioStation $station): StreamedResponse
    {
        return new StreamedResponse(function () use ($station) {
            $stream = $this->proxy->openStream($station->url);

            $metaHeaders = stream_get_meta_data($stream);
            $icyMetaInt = $this->proxy->extractIcyMetaInt($metaHeaders['wrapper_data'] ?? []);

            if ($icyMetaInt === 0) {
                // 不支持 ICY 元数据，直接透传
                $this->proxy->proxyRawStream($stream);
                return;
            }

            // 支持 ICY 元数据，边代理边解析元数据
            $this->proxy->proxyWithMetadata($stream, $station, $icyMetaInt);
        }, headers: [
            'Content-Type' => 'audio/mpeg',
            'Cache-Control' => 'no-cache, no-store',
            'Connection' => 'close',
        ]);
    }
}
```

### 2.4 ICY 元数据解析机制

#### 什么是 ICY 元数据？

ICY 是 SHOUTcast/Icecast 流媒体协议的元数据标准。元数据以固定间隔（`icy-metaint` 字节）插入音频流中，包含当前播放的歌曲标题等信息。

#### 流打开与元数据请求

**关键代码**：[RadioStreamProxy.php#L15-L37](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/app/Services/Radio/RadioStreamProxy.php#L15-L37)

```php
public function openStream(string $url)
{
    $context = stream_context_create([
        'http' => [
            'header' => "Icy-MetaData: 1\r\n", // 请求服务器发送 ICY 元数据
            'timeout' => 5,
        ],
    ]);

    // 带 ICY 请求失败则降级为普通连接
    $stream = fopen($url, 'r', false, $context);
    if ($stream !== false) {
        return $stream;
    }
    return fopen($url, 'r');
}
```

#### 带元数据的流代理

`proxyWithMetadata()` 方法逐块读取音频流，当读取到 `icyMetaInt` 字节后，下一个字节是元数据长度（乘以 16），随后是元数据块。解析后将元数据剥离，只把纯净的音频数据输出给客户端。

**关键代码**：[RadioStreamProxy.php#L73-L97](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/app/Services/Radio/RadioStreamProxy.php#L73-L97)

```php
public function proxyWithMetadata($stream, RadioStation $station, int $icyMetaInt): void
{
    $bytesUntilMeta = $icyMetaInt;

    while (!feof($stream) && !connection_aborted()) {
        $chunkSize = min($bytesUntilMeta, 8192);
        $data = fread($stream, $chunkSize);

        echo $data;  // 输出纯净音频
        flush();
        $bytesUntilMeta -= strlen($data);

        if ($bytesUntilMeta === 0) {
            $this->processMetadataBlock($stream, $station); // 解析元数据块
            $bytesUntilMeta = $icyMetaInt;
        }
    }
}
```

#### 元数据块解析

**关键代码**：[RadioStreamProxy.php#L104-L129](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/app/Services/Radio/RadioStreamProxy.php#L104-L129)

```php
private function processMetadataBlock($stream, RadioStation $station): void
{
    $lengthByte = fread($stream, 1);
    $metadataLength = ord($lengthByte) * 16; // 长度字节 × 16 = 实际元数据字节数

    if ($metadataLength === 0) {
        return;
    }

    $metadataBlock = fread($stream, $metadataLength);
    $parsed = RadioStreamMetadata::parseIcyBlock($metadataBlock);

    if ($parsed['stream_title'] !== null) {
        RadioStreamMetadata::cache($station, $parsed['stream_title']); // 缓存元数据
    }
}
```

ICY 元数据格式示例：`StreamTitle='Artist - Song Title';StreamUrl='http://...';`

**关键代码**：[RadioStreamMetadata.php#L16-L29](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/app/Services/Radio/RadioStreamMetadata.php#L16-L29)

```php
public static function parseIcyBlock(string $metadataBlock): array
{
    $result = ['stream_title' => null];

    if (preg_match("/StreamTitle='(.*?)';/", $metadataBlock, $matches)) {
        $title = trim($matches[1]);
        if ($title !== '') {
            $result['stream_title'] = $title;
        }
    }

    return $result;
}
```

---

## 三、播放队列融合与切换逻辑

### 3.1 双播放服务架构

Koel 采用 **两套独立的播放服务**，分别处理本地队列播放和电台播放：

| 服务 | 对应类 | 职责 | 状态存储 |
|------|--------|------|----------|
| 队列播放 | `QueuePlaybackService` | 本地歌曲/播客队列播放，支持上一首/下一首/进度拖动等 | `queueStore` |
| 电台播放 | `RadioPlaybackService` | 电台流播放，不支持进度控制和切歌 | `radioStationStore` |

两套服务都继承自 `BasePlaybackService`，共享同一套媒体事件监听机制和音量控制。

**关键代码**：[BasePlaybackService.ts#L6-L121](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/services/BasePlaybackService.ts#L6-L121)

### 3.2 playbackManager：播放服务的切换中枢

`playbackManager` 是播放服务的统一管理层，负责在两套服务之间进行切换。

**关键代码**：[playbackManager.ts#L12-L57](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/services/playbackManager.ts#L12-L57)

```typescript
export const playbackManager = {
  _currentService: null as BasePlaybackService | null,

  usePlayback(type: 'queue' | 'radio', mediaElement?: HTMLMediaElement) {
    // 停用所有其他播放服务
    for (const key in playbackServiceMap) {
      if (key !== type) {
        playbackServiceMap[key].deactivate()
      }
    }

    this._currentService = playbackServiceMap[type]
    return playbackServiceMap[type].activate(mediaElement ?? ...)
  },
}
```

**切换流程**：
1. 调用 `deactivate()` 停用其他服务（解绑事件 + 停止播放）
2. 调用 `activate()` 激活目标服务（绑定事件 + 设置音量 + 注册 MediaSession）
3. 更新 `_currentService` 指向

### 3.3 `playback()` 工厂函数

通过 `playback(type)` 函数可以便捷地获取对应播放服务实例。调用时会自动触发切换逻辑。

**使用示例**：
```typescript
// 获取队列播放服务（默认）
playback('queue').play(song)

// 获取电台播放服务（会自动停用队列服务）
playback('radio').play(station)
```

### 3.4 触发电台播放的入口

#### 从电台卡片播放

**关键代码**：[RadioStationCard.vue#L44-L50](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/components/radio/RadioStationCard.vue#L44-L50)

```typescript
const togglePlay = () => {
  if (station.value.playback_state === 'Playing') {
    playback('radio').stop()
  } else {
    playback('radio').play(station.value)
  }
}
```

#### 从底部播放栏播放

**关键代码**：[FooterPlayButton.vue#L76-L88](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/components/ui/FooterPlayButton.vue#L76-L88)

```typescript
const toggle = async () => {
  if (!streamable.value) {
    await initiatePlayback()
    return
  }

  if (isRadio.value) {
    await playback('radio').toggle()
    return
  }

  await playback('queue').toggle()
}
```

### 3.5 电台播放启动流程

**关键代码**：[RadioPlaybackService.ts#L7-L16](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/services/RadioPlaybackService.ts#L7-L16)

```typescript
public async play(station: RadioStation) {
  // 标记上一个电台为停止状态
  use(radioStationStore.current, station => (station.playback_state = 'Stopped'))

  station.playback_state = 'Playing'
  this.media.src = radioStationStore.getSourceUrl(station)
  await this.media.play()

  radioStationStore.startPolling(station)  // 开始元数据轮询
  socketService.broadcast('SOCKET_STREAMABLE', station)
}
```

**注意**：当调用 `playback('radio')` 时，`playbackManager` 会先调用 `queuePlayback.deactivate()`，这会触发 `QueuePlaybackService.stop()`，将队列中的歌曲标记为 Stopped 状态。

### 3.6 反向切换：从电台切回队列

当用户点击播放一首本地歌曲时，调用 `playback('queue').play(song)` 会触发：
1. `radioPlayback.deactivate()` → 调用 `RadioPlaybackService.stop()` → `pause()`
2. 电台轮询停止，元数据清空
3. 队列播放服务接管 media element

**关键代码**：[RadioPlaybackService.ts#L43-L59](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/services/RadioPlaybackService.ts#L43-L59)

```typescript
public async pause() {
  radioStationStore.stopPolling()

  use(radioStationStore.current, station => {
    station.playback_state = 'Paused'
    socketService.broadcast('SOCKET_STREAMABLE', station)
  })

  // 停止并重置媒体源
  if (this.media) {
    this.media.pause()
    this.media.currentTime = 0
    this.media.removeAttribute('src')
  }
}
```

### 3.7 currentStreamable：统一的"当前播放"状态

为了让 UI 层无需关心当前是哪种播放模式，Koel 在 `App.vue` 中通过两个 watcher 将两种来源的"当前播放"统一到 `currentStreamable` 中。

**关键代码**：[App.vue#L137-L149](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/App.vue#L137-L149)

```typescript
// 队列中的当前歌曲 → currentStreamable
watch(
  () => queueStore.current,
  song => (currentStreamable.value = song),
)

// 当前播放的电台 → currentStreamable（电台优先覆盖）
watch(
  () => radioStationStore.current,
  station => {
    if (station) {
      currentStreamable.value = station
    }
  },
)
```

**设计要点**：
- 电台 watcher 在队列 watcher 之后定义，且只有当电台存在时才覆盖
- 这意味着 **电台播放优先级高于队列播放**
- `currentStreamable` 通过 `provide/inject` 注入给所有子组件
- 底部播放栏、全屏模式、通知等 UI 都基于 `currentStreamable` 渲染

### 3.8 UI 层的类型感知

底部播放栏通过 `isRadioStation()` 类型守卫判断当前播放类型，渲染不同的信息组件：

**关键代码**：[app-footer/index.vue#L60-L61](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/components/layout/app-footer/index.vue#L60-L61)

```vue
<RadioStationInfo v-if="isRadio" />
<SongInfo v-else />
```

```typescript
const isRadio = computed(() => currentStreamable.value && isRadioStation(currentStreamable.value))
```

---

## 四、元数据回写机制

电台的"当前播放"元数据（如正在播放的歌曲名）采用 **服务端缓存 + 前端轮询** 的方式回写给用户。

### 4.1 服务端缓存

当 `RadioStreamProxy` 解析到 ICY 元数据中的 `StreamTitle` 时，会将其缓存到 Laravel Cache 中，TTL 为 10 分钟。

**关键代码**：[RadioStreamMetadata.php#L34-L41](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/app/Services/Radio/RadioStreamMetadata.php#L34-L41)

```php
public static function cache(RadioStation $station, string $streamTitle): void
{
    Cache::put(
        self::cacheKey($station),
        ['stream_title' => $streamTitle, 'updated_at' => now()->toISOString()],
        now()->addMinutes(10),
    );
}
```

缓存键格式：`radio.metadata.{stationId}`

### 4.2 元数据查询接口

前端通过 `RadioStationNowPlayingController` 获取缓存的元数据。

**关键代码**：[RadioStationNowPlayingController.php#L12-L22](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/app/Http/Controllers/API/RadioStationNowPlayingController.php#L12-L22)

```php
class RadioStationNowPlayingController extends Controller
{
    public function __invoke(Authenticatable $user, RadioStation $radioStation): JsonResponse
    {
        $this->authorize('access', $radioStation);
        return response()->json(RadioStreamMetadata::getCached($radioStation));
    }
}
```

响应格式：
```json
{
  "stream_title": "Artist - Song Title",
  "updated_at": "2025-01-01T12:00:00Z"
}
```

### 4.3 前端轮询机制

电台开始播放后，前端启动定时轮询获取最新的"正在播放"元数据，轮询间隔为 15 秒。

**关键代码**：[radioStationStore.ts#L91-L104](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/stores/radioStationStore.ts#L91-L104)

```typescript
const POLL_INTERVAL = 15_000

startPolling(station: RadioStation) {
  this.stopPolling()
  this.fetchNowPlaying(station) // 立即拉取一次
  this._pollTimer = setInterval(() => this.fetchNowPlaying(station), POLL_INTERVAL)
},

stopPolling() {
  if (this._pollTimer) {
    clearInterval(this._pollTimer)
    this._pollTimer = null
  }
  this.nowPlaying.value = null
},
```

### 4.4 元数据获取

**关键代码**：[radioStationStore.ts#L82-L89](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/stores/radioStationStore.ts#L82-L89)

```typescript
async fetchNowPlaying(station: RadioStation) {
  try {
    const response = await http.get<NowPlayingResponse>(`radio/stations/${station.id}/now-playing`)
    this.nowPlaying.value = response.stream_title
  } catch (e: unknown) {
    logger.error('Failed to fetch now-playing metadata', e)
  }
},
```

### 4.5 UI 展示

元数据展示在底部播放栏的电台信息区域，位于电台名称下方。

**关键代码**：[FooterRadioStationInfo.vue#L8-L13](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/components/layout/app-footer/FooterRadioStationInfo.vue#L8-L13)

```vue
<h3 class="title truncate">{{ station.name }}</h3>
<p v-if="nowPlaying" class="truncate text-k-text-secondary">
  {{ nowPlaying }}
</p>
<p v-else class="truncate">{{ station.description }}</p>
```

**显示策略**：
- 有 `nowPlaying` 数据时显示当前播放内容
- 没有时显示电台描述作为兜底

---

## 五、完整接力时序

### 5.1 点击播放电台的完整链路

```
用户点击电台卡片
      │
      ▼
RadioStationCard.togglePlay()
      │
      ▼
playback('radio').play(station)
      │
      ▼
playbackManager.usePlayback('radio')
      │
      ├─► queuePlayback.deactivate()   ◄── 停用队列服务
      │     ├─► 解绑媒体事件
      │     └─► queuePlayback.stop()  ◄── 停止当前歌曲
      │
      ├─► radioPlayback.activate()    ◄── 激活电台服务
      │     ├─► 绑定媒体事件
      │     ├─► 设置音量
      │     └─► 注册 MediaSession
      │
      └─► radioPlayback.play(station)
            ├─► 标记前一电台为 Stopped
            ├─► 标记当前电台为 Playing
            ├─► 设置 media.src = /radio/stream/{id}?t=token
            ├─► media.play()
            ├─► radioStationStore.startPolling()  ◄── 启动元数据轮询
            └─► socket 广播 SOCKET_STREAMABLE
```

### 5.2 元数据回写链路

```
电台流服务器
      │
      ▼
RadioStreamProxy.proxyWithMetadata()
      │
      ├─► 逐块读取音频数据
      ├─► 输出纯净音频给浏览器
      │
      └─► 遇到 ICY 元数据块时
            │
            ▼
      processMetadataBlock()
            │
            ├─► 读取元数据长度字节
            ├─► 读取元数据块
            ├─► RadioStreamMetadata.parseIcyBlock() 解析 StreamTitle
            └─► RadioStreamMetadata.cache() 写入缓存
                                    │
                                    ▼
前端轮询 /radio/stations/{id}/now-playing
                                    │
                                    ▼
                      radioStationStore.nowPlaying 更新
                                    │
                                    ▼
                          FooterRadioStationInfo 展示
```

### 5.3 切回本地曲库的链路

```
用户点击播放某首歌曲
      │
      ▼
playback('queue').play(song)
      │
      ▼
playbackManager.usePlayback('queue')
      │
      ├─► radioPlayback.deactivate()   ◄── 停用电台服务
      │     ├─► 解绑媒体事件
      │     └─► radioPlayback.stop() → pause()
      │           ├─► radioStationStore.stopPolling()  ◄── 停止轮询
      │           ├─► 电台状态设为 Paused
      │           ├─► media.pause()
      │           ├─► media.removeAttribute('src')
      │           └─► socket 广播状态
      │
      └─► queuePlayback.activate() + play(song)
            └─► currentStreamable 更新为该歌曲
```

---

## 六、关键设计决策总结

| 决策点 | 方案 | 理由 |
|--------|------|------|
| 播放服务架构 | 双服务独立实现 + 管理器切换 | 电台流与队列播放行为差异大（无进度、无切歌、无队列），强行复用增加复杂度 |
| 元数据获取 | 服务端解析 + 缓存 + 前端轮询 | 浏览器无法直接读取 ICY 元数据（被音频引擎吞掉），必须由服务端代理解析 |
| 流代理策略 | 后端统一代理转发 | 规避浏览器 CORS 限制，同时支持元数据解析和鉴权 |
| 状态优先级 | 电台状态覆盖队列状态 | 电台播放时应显示电台信息，而非队列中的歌曲 |
| 轮询间隔 | 15 秒 | 平衡时效性与请求量，电台歌曲切换通常不会太频繁 |
| 缓存 TTL | 10 分钟 | 防止电台流断开后元数据永久残留，同时支持短时间重连后快速恢复 |

---

## 七、相关文件索引

| 模块 | 文件路径 |
|------|----------|
| 播放管理 | [playbackManager.ts](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/services/playbackManager.ts) |
| 基类服务 | [BasePlaybackService.ts](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/services/BasePlaybackService.ts) |
| 队列播放 | [QueuePlaybackService.ts](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/services/QueuePlaybackService.ts) |
| 电台播放 | [RadioPlaybackService.ts](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/services/RadioPlaybackService.ts) |
| 电台状态 | [radioStationStore.ts](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/stores/radioStationStore.ts) |
| 队列状态 | [queueStore.ts](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/stores/queueStore.ts) |
| 应用根组件 | [App.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/App.vue) |
| 底部播放栏 | [app-footer/index.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/components/layout/app-footer/index.vue) |
| 后端流控制器 | [StreamRadioController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/app/Http/Controllers/StreamRadioController.php) |
| 流服务 | [RadioStreamService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/app/Services/Radio/RadioStreamService.php) |
| 流代理 | [RadioStreamProxy.php](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/app/Services/Radio/RadioStreamProxy.php) |
| 元数据处理 | [RadioStreamMetadata.php](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/app/Services/Radio/RadioStreamMetadata.php) |
| 元数据接口 | [RadioStationNowPlayingController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/app/Http/Controllers/API/RadioStationNowPlayingController.php) |
