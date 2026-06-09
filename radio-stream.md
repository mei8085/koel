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
1. `radioStationStore.current` = 第一个 playback_state !== 'Stopped' 的电台，否则 null


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

## 六、状态接管的容易误解点详解

这是最容易产生困惑的部分：电台暂停后到底算不算"当前项"？切回队列时显示什么？轮询什么时候清？本节逐一拆解。

### 6.1 Paused 与 Stopped 的边界

首先明确两个核心判定逻辑：

**电台 current 的判定**：
**关键代码**：[radioStationStore.ts#L61-L63](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/stores/radioStationStore.ts#L61-L63)

```typescript
get current() {
  return this.state.stations.find(station => station.playback_state !== 'Stopped') || null
}
```

**队列 current 的判定**：
**关键代码**：[queueStore.ts#L148-L152](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/stores/queueStore.ts#L148-L152)

```typescript
get current() {
  return this.all.find(({ playback_state }) => playback_state !== 'Stopped') || playableStore.findPlaying()
}
```

**关键结论**：
- 两者的 current 判定逻辑完全一致：**只要不是 Stopped 就算 current**
- Paused 状态的电台 / 歌曲，仍然被视为各自体系内的 current
- 只有 Stopped 状态才会从 current 中排除

### 6.2 电台 stop() ≠ Stopped

这是最容易踩的坑。电台的 `stop()` 方法实际上直接调用 `pause()`：

**关键代码**：[RadioPlaybackService.ts#L18-L20](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/services/RadioPlaybackService.ts#L18-L20)

```typescript
public async stop() {
  return this.pause()
}
```

**pause() 的行为**：
**关键代码**：[RadioPlaybackService.ts#L43-L59](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/services/RadioPlaybackService.ts#L43-L59)

```typescript
public async pause() {
  radioStationStore.stopPolling()      // 停止轮询 + 清空 nowPlaying

  use(radioStationStore.current, station => {
    station.playback_state = 'Paused' // 注意：是 Paused，不是 Stopped
    socketService.broadcast('SOCKET_STREAMABLE', station)
  })

  if (this.media) {
    this.media.pause()
    this.media.currentTime = 0
    this.media.removeAttribute('src') // 清空媒体源
  }
}
```

**重要结论**：
- 调用 `radioPlayback.stop()` 后，电台状态是 **Paused**，不是 Stopped
- 因此，**暂停后的电台仍然是 radioStationStore.current**
- 只有在播放另一个电台时，前一个电台才会被显式设为 Stopped

### 6.3 从电台切到队列：完整状态迁移

下面是用户点击播放一首本地歌曲时，状态变化的完整时序：

```
初始状态：
  - 电台A Playing（radioStationStore.current = 电台A）
  - 歌曲X Stopped（queueStore.current = 歌曲X？不，Stopped 不算）
  - currentStreamable = 电台A

用户点击播放歌曲X
    │
    ▼
playback('queue').play(songX)
    │
    ▼
playbackManager.usePlayback('queue')
    │
    ├─► radioPlayback.deactivate()           ◄── 第一步：停用电台服务
    │     ├─► 解绑媒体事件监听器
    │     └─► radioPlayback.stop()
    │           └─► pause()
    │                 ├─► stopPolling()       ◄── 停止轮询 + 清空 nowPlaying
    │                 ├─► 电台A.playback_state = Paused
    │                 └─► media.pause() + 移除 src
    │
    │  【关键点 1】此时：
    │  - 电台A状态 = Paused → 仍然是 radioStationStore.current
    │  - 但 radioStationStore.current 的引用没变（同一个对象）
    │  - 所以电台 watcher 不触发（浅比较：引用相同 = 值未变）
    │
    ├─► queuePlayback.activate()              ◄── 第二步：激活队列服务
    │     └─► 绑定媒体事件 + 设置音量 + MediaSession
    │
    └─► queuePlayback.play(songX)             ◄── 第三步：播放歌曲
          ├─► 前一首歌曲.playback_state = Stopped（如果有的话）
          ├─► songX.playback_state = Playing
          │
          │  【关键点 2】此时：
          │  - queueStore.current 变化了（songX 变成 Playing）
          │  - 队列 watcher 触发 → currentStreamable = songX
          │
          ├─► 设置 media.src
          └─► media.play()

最终状态：
  - 电台A Paused（仍然是 radioStationStore.current）
  - 歌曲X Playing（queueStore.current = 歌曲X）
  - currentStreamable = 歌曲X ◄── 底部栏显示歌曲信息
```

**为什么底部栏正确显示了歌曲？**
- 不是因为电台被"挤掉"了，而是因为队列的 current 变化触发了队列 watcher
- 电台那边的 current 引用没变，所以电台 watcher 没触发，没有覆盖
- 最终结果是 **队列 watcher 的更新生效**，显示歌曲信息

### 6.4 从队列切回电台：完整状态迁移

播放电台前，代码会先把当前电台设为 Stopped，再将目标电台设为 Playing。这个模式与队列播放保持一致。

**关键代码**：[RadioPlaybackService.ts#L7-L16](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/services/RadioPlaybackService.ts#L7-L16)

```typescript
public async play(station: RadioStation) {
  use(radioStationStore.current, station => (station.playback_state = 'Stopped'))
  station.playback_state = 'Playing'
  ...
}
```

**完整时序**：

```
初始状态：
  - 歌曲X Playing（queueStore.current = 歌曲X）
  - 电台A Paused（radioStationStore.current = 电台A）
  - currentStreamable = 歌曲X

用户点击播放电台A
    │
    ▼
playback('radio').play(stationA)
    │
    ▼
playbackManager.usePlayback('radio')
    │
    ├─► queuePlayback.deactivate()           ◄── 第一步：停用队列服务
    │     ├─► 解绑媒体事件
    │     └─► queuePlayback.stop()
    │           └─► songX.playback_state = Stopped
    │
    │  【关键点 1】此时：
    │  - queueStore.current 变化（歌曲X Stopped 了）
    │  - 队列 watcher 触发 → currentStreamable = 下一首 / undefined
    │
    ├─► radioPlayback.activate()              ◄── 第二步：激活电台服务
    │
    └─► radioPlayback.play(stationA)          ◄── 第三步：播放电台
          │
          ├─► radioStationStore.current 是电台A（Paused）
          ├─► stationA.playback_state = Stopped  ◄── 先停止当前电台
          │     │
          │     │  【关键点 2】同步执行中，状态瞬间变成 null
          │     │  - 但 watcher 是批处理的，不会立即触发
          │     │
          ├─► stationA.playback_state = Playing  ◄── 再设为播放状态
          │     │
          │     │  【关键点 3】同步执行完毕，最终状态是电台A
          │     │  - 引用没变，watcher 批处理后比较结果相同 → 不触发
          │     │  - ⚠️ 这种情况下 currentStreamable 不会被电台 watcher 更新
          │     │
          ├─► 设置 media.src
          └─► media.play() + startPolling()

最终状态：
  - 电台A Playing（radioStationStore.current = 电台A）
  - 歌曲X Stopped
  - currentStreamable = 队列的新值（下一首 / undefined）⚠️
```

> **重要修正**：之前的"先 Stopped 再 Playing 确保 watcher 触发"的说法是不准确的。
> 由于 Vue watch 的批处理机制，同一 tick 内的连续状态变化只会比较**最终值**和**旧值**。
> 如果初始和最终都是同一个电台对象（引用相同），即使中间经过了 null，watcher 也不会触发。
> 详见下一节「Vue watch 批处理机制与状态切换分析」。

### 6.5 Vue watch 批处理机制与状态切换分析

这是最容易产生误解的核心点，需要结合 Vue 的响应式机制来理解。

#### 6.5.1 Vue watch 的批处理与比较规则

**关键规则**：

1. **批处理**：Vue 的 watch 默认使用 `flush: 'pre'`，在同一个同步 tick 内发生的多次状态变化会被合并，只在下一个微任务阶段执行一次回调。
2. **浅比较**：默认使用 === 比较新值和旧值，如果是同一个对象引用，即使内部属性变了，也认为值没变，不触发回调。
3. **最终值比较**：批处理后比较的是**最终状态值**和**上一次回调时的旧值**，中间状态会被完全忽略。

**关键代码**：[App.vue#L137-L149](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/App.vue#L137-L149)

```typescript
watch(
  () => queueStore.current,
  song => (currentStreamable.value = song),
)

watch(
  () => radioStationStore.current,
  station => {
    if (station) {
      currentStreamable.value = station
    }
  },
)
```
两个 watcher 的 source 都是 getter 函数。每次 getter 执行时，会访问响应式数据（state.stations 数组和每个 station 的 playback_state 属性），Vue 会收集这些依赖。当依赖变化时，watcher 被标记为 dirty，在下一个 tick 重新执行 getter 获取新值。

#### 6.5.2 三种场景下的 watcher 触发分析

我们分别分析三种典型场景下，`radioStationStore.current` 的 watcher 是否会触发。

---

**场景一：切换电台（stationA  stationB）**

```
初始值：stationA (Playing)
  
stationA.playback_state = 'Stopped'
   current 瞬时 = null     （中间状态，被批处理忽略）
  
stationB.playback_state = 'Playing'
   current 最终 = stationB
  
微任务阶段执行 watcher：
  新值 = stationB
  旧值 = stationA
  比较：stationB !== stationA      触发 
```
**结论**： **会触发**。引用变了，新旧值不同。

---

**场景二：首次播放电台（null  stationA）**

```
初始值：null（没有非 Stopped 的电台）
  
stationA.playback_state = 'Playing'
   current 最终 = stationA
  
微任务阶段执行 watcher：
  新值 = stationA
  旧值 = null
  比较：stationA !== null      触发 
```
**结论**： **会触发**。从 null 到对象，引用变了。

---

**场景三：恢复播放同一电台（Paused  Playing）**

```
初始值：stationA (Paused)   （Paused  Stopped，所以 current 是 stationA）
  
stationA.playback_state = 'Stopped'
   current 瞬时 = null     （中间状态，被批处理忽略）
  
stationA.playback_state = 'Playing'
   current 最终 = stationA  （还是同一个对象引用）
  
微任务阶段执行 watcher：
  新值 = stationA
  旧值 = stationA（上一次回调时缓存的值）
  比较：stationA === stationA    不触发 
```
**结论**： **不会触发**。初始和最终是同一个对象引用，浅比较认为值没变。

> **重要**：即使中间经历了 
ull，由于批处理机制，中间状态会被丢弃，只比较首尾。
#### 6.5.3 "先 Stopped 再 Playing"的真正目的

既然"先 Stopped 再 Playing"不能确保 watcher 触发（场景三下无效），那为什么代码要这么写？

**真正的目的**：

1. **统一的播放前清理模式**：这是队列播放和电台播放共用的模式  播放新内容前，先把当前正在播放 / 暂停的内容设为 Stopped，保证任何时刻只有一个"当前项"。

   **关键代码**：[QueuePlaybackService.ts#L113-L117](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/services/QueuePlaybackService.ts#L113-L117)
   ``typescript
   if (queueStore.current) {
     queueStore.current.playback_state = 'Stopped'
   }
   playable.playback_state = 'Playing'
   ``

   电台播放也是完全一样的模式：
   **关键代码**：[RadioPlaybackService.ts#L7-L10](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/services/RadioPlaybackService.ts#L7-L10)
   ``typescript
   public async play(station: RadioStation) {
     use(radioStationStore.current, station => (station.playback_state = 'Stopped'))
     station.playback_state = 'Playing'
   ``

2. **切换电台时能正确工作**：在切换电台（场景一）和首次播放（场景二）时，这个模式确实能让 watcher 正确触发。只有恢复同一电台（场景三）时不触发。

3. **副作用可接受**：场景三下 watcher 不触发通常不构成问题，因为：
   - 如果用户一直在电台页面暂停 / 播放，currentStreamable 本来就是电台
   - 如果用户从队列切回电台，见下一节的分析

#### 6.5.4 从队列切回电台时的显示问题

这是最容易出问题的边界场景：电台A 暂停中  切到队列播放歌曲X  又切回电台A。

**关键分析**：

### 6.6 轮询清理的时机

轮询（polling）的启动和停止是对称的：

| 操作 | 轮询状态 | nowPlaying 值 |
|------|----------|---------------|
| `play(station)` | 启动（15秒间隔） | 立即拉取一次 |
| `pause()` | 停止 | 清空为 null |
| `stop()` | 停止（因为 stop → pause） | 清空为 null |
| `deactivate()` | 停止（deactivate → stop → pause） | 清空为 null |

**关键代码**：[radioStationStore.ts#L91-L104](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/stores/radioStationStore.ts#L91-L104)

```typescript
startPolling(station: RadioStation) {
  this.stopPolling()               // 先清旧的
  this.fetchNowPlaying(station)    // 立即拉一次
  this._pollTimer = setInterval(..., POLL_INTERVAL)
},

stopPolling() {
  if (this._pollTimer) {
    clearInterval(this._pollTimer)
    this._pollTimer = null
  }
  this.nowPlaying.value = null     // 同时清空显示值
},
```

**重要结论**：
- 只要电台进入 Paused 状态（无论是主动暂停还是被 deactivate），**轮询立即停止，元数据立即清空**
- 所以暂停电台后，底部栏的"正在播放"行会从歌曲名切换回电台描述

### 6.7 底部播放栏的显示逻辑

底部播放栏通过 `currentStreamable` 统一渲染，但内部会根据类型分流：

**关键代码**：[app-footer/index.vue#L60-L61](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/components/layout/app-footer/index.vue#L60-L61)

```vue
<RadioStationInfo v-if="isRadio" />
<SongInfo v-else />
```

```typescript
const isRadio = computed(() => currentStreamable.value && isRadioStation(currentStreamable.value))
```

**显示判断链**：
1. `currentStreamable` 有值吗？ → 没有则显示空状态 / 默认占位
2. 有值的话，是电台类型吗？ → 是则渲染 `FooterRadioStationInfo`
3. 否则渲染 `FooterPlayableInfo`（歌曲 / 播客）

**播放按钮的行为**：
**关键代码**：[FooterPlayButton.vue#L76-L88](file:///d:/fz/0508-2/solo-dogfeeding/code/111-koel/resources/assets/js/components/ui/FooterPlayButton.vue#L76-L88)

```typescript
const toggle = async () => {
  if (!streamable.value) {
    await initiatePlayback()   // 没有当前项时，随机播放队列
    return
  }

  if (isRadio.value) {
    await playback('radio').toggle()   // 电台调用电台服务的 toggle
    return
  }

  await playback('queue').toggle()     // 歌曲调用队列服务的 toggle
}
```

**注意**：点击播放按钮时，会根据当前 streamable 的类型选择对应的播放服务。这意味着：
- 如果底部栏显示的是电台，点暂停 / 播放走的是 RadioPlaybackService
- 如果显示的是歌曲，点暂停 / 播放走的是 QueuePlaybackService
- 不会出现"显示着电台但用队列服务播放"的错乱


### 6.8 播放状态与 currentStreamable 对应关系表

为了便于复核，下表列出了电台和队列各种状态组合下，currentStreamable 的最终值。

**判定规则（可复核）：**

1. 
adioStationStore.current = 第一个 playback_state !== 'Stopped' 的电台，否则 null
2. queueStore.current = 第一个 playback_state !== 'Stopped' 的歌曲，否则 fallback 查找
3. 队列 watcher 先创建，无条件赋值；电台 watcher 后创建，仅当 station 为真值时覆盖
4. 同一 tick 内的状态变化被 Vue 批处理，watcher 只比较最终值与上一次回调的旧值
5. 同引用对象的属性变化不触发 watcher（浅比较）

**状态对应关系表：**

| # | 电台状态 | 队列状态 | radioStationStore.current | queueStore.current | currentStreamable | 底部栏显示 | 说明 |
|---|---------|---------|---------------------------|-------------------|-------------------|-----------|------|
| 1 | Stopped | Stopped | null | null / fallback | null / fallback | 空/上次歌曲 | 都没在播放 |
| 2 | Playing | Stopped | stationA | null | stationA | 电台信息 | 只有电台在播 |
| 3 | Paused | Stopped | stationA | null | stationA | 电台信息（暂停） | Paused  Stopped，电台仍是 current |
| 4 | Stopped | Playing | null | songX | songX | 歌曲信息 | 只有队列在播 |
| 5 | Stopped | Paused | null | songX | songX | 歌曲信息（暂停） | 队列 Paused 也算 current |
| 6 | Playing | Paused | stationA | songX | stationA | 电台信息 | 两者都有状态，电台 watcher 后执行覆盖 |
| 7 | Paused | Playing | stationA | songX | songX | 歌曲信息 | 队列状态后变化，队列 watcher 后触发；电台 watcher 因同引用不触发 |
| 8 | Paused | Paused | stationA | songX | 取决于谁最后触发 | 最后变化的那个 | 两者都 Paused，显示最后一次状态变化对应的内容 |

**关键边界场景复核：**

| 场景 | 行为 | 是否符合预期 |
|------|------|-------------|
| 电台播放中  暂停 | 底部栏仍显示电台 |  是 |
| 电台暂停  切到队列播放 | 底部栏变成歌曲 |  是 |
| 队列播放  切回同一暂停电台 | 底部栏可能还显示歌曲 |  边界情况（同引用 watcher 不触发） |
| 队列播放  切到不同电台 | 底部栏变成电台 |  是 |
| 电台播放  切到队列 | 底部栏变成歌曲 |  是 |
| 电台停止  队列有歌曲 | 自动回落显示歌曲 |  是（有条件覆盖设计） |

---
### 6.9 容易误解点速查表

| 问题 | 答案 | 原因 |
|------|------|------|
| 电台暂停后还是 current 吗？ | **是** | Paused ≠ Stopped，current 判定只排除 Stopped |
| 电台 stop() 后状态是？ | **Paused** | stop() 内部直接调用 pause() |
| 从电台切到队列，电台状态是？ | **Paused** | deactivate → stop → pause |
| 从电台切到队列，底部栏显示什么？ | **队列歌曲** | 队列 watcher 触发，更新了 currentStreamable |
| 为什么电台暂停了还显示歌曲？ | 因为你切到了队列播放，底部栏显示的是队列的 current | 两个体系各有 current，最终显示哪个由 watcher 触发顺序和优先级决定 |
| 电台暂停后轮询还在吗？ | **不在了** | pause() 里调用了 stopPolling() |
| 电台暂停后还显示歌曲名吗？ | **不显示** | stopPolling() 把 nowPlaying 清空了，显示电台描述 |
| 为什么 play() 要先设 Stopped？ | 统一播放前清理，停止其他电台 | 与队列播放模式一致；切换电台时引用变化触发 watcher，但同一电台恢复时不触发（批处理+浅比较） |
| 同一电台 Paused→Playing，watcher 触发吗？ | **不触发** | 同一个对象引用，Vue 批处理后浅比较相等 |
| 从队列切回同一暂停电台，显示正确吗？ | **不一定** | 电台 watcher 可能不触发，底部栏可能停留在队列内容 |
| 电台和队列都有 Paused 的项，显示谁？ | 取决于谁的 watcher 最后一次触发 | 通常是后发生状态变化的那个；电台 watcher 只有有值时才覆盖 |

---

## 七、关键设计决策总结

| 决策点 | 方案 | 理由 |
|--------|------|------|
| 播放服务架构 | 双服务独立实现 + 管理器切换 | 电台流与队列播放行为差异大（无进度、无切歌、无队列），强行复用增加复杂度 |
| 元数据获取 | 服务端解析 + 缓存 + 前端轮询 | 浏览器无法直接读取 ICY 元数据（被音频引擎吞掉），必须由服务端代理解析 |
| 流代理策略 | 后端统一代理转发 | 规避浏览器 CORS 限制，同时支持元数据解析和鉴权 |
| 状态优先级 | 电台状态覆盖队列状态 | 电台播放时应显示电台信息，而非队列中的歌曲 |
| 轮询间隔 | 15 秒 | 平衡时效性与请求量，电台歌曲切换通常不会太频繁 |
| 缓存 TTL | 10 分钟 | 防止电台流断开后元数据永久残留，同时支持短时间重连后快速恢复 |

---

## 八、相关文件索引

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
