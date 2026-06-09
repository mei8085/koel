# 歌词获取与元数据整合外部链路分析

## 概述

Koel 的歌词获取与元数据整合走两条相对独立但又有协作的链路：
- **歌词链路**：以本地文件内嵌标签和 .lrc 外部文件为主要来源，辅以 AI 工具和人工编辑
- **元数据链路**：以 Last.fm / MusicBrainz 等第三方百科服务为主要来源，获取专辑/艺术家的 Wiki 摘要、封面、曲目列表等信息
- **补充链路**：Spotify 封面补充、iTunes 曲目链接等独立的第三方集成

## 一、歌词获取链路

### 1.1 主要来源

#### 1.1.1 音频文件内嵌歌词（ID3/Vorbis 标签）

扫描阶段通过 `getID3` 库从音频文件的元数据标签中提取歌词。

| 层级 | 文件 | 关键方法/类 |
|------|------|------------|
| 扫描入口 | [FileScanner.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/Scanners/FileScanner.php) | `scan()` |
| 标签解析 | [ScanInformation.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Values/Scanning/ScanInformation.php#L60-L66) | `fromGetId3Info()` |
| 字段映射 | - | `unsynchronised_lyric` / `unsyncedlyrics` / `lyrics` 等多标签兼容 |

歌词标签解析的优先级顺序（后覆盖前）：
1. `tags.id3v1`
2. `tags.id3v2`
3. `comments`
4. `tags.vorbiscomment`

#### 1.1.2 外部 .lrc 同步歌词文件

当音频文件内嵌歌词为空时，尝试读取同名的 `.lrc` 文件。

| 层级 | 文件 | 关键方法/类 |
|------|------|------------|
| LRC 读取 | [SimpleLrcReader.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/SimpleLrcReader.php) | `tryReadForMediaFile()` |
| 路径匹配 | [SimpleLrcReader.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/SimpleLrcReader.php#L21-L32) | `getLrcFilePath()` |

匹配逻辑：将音频文件扩展名替换为 `.lrc` 或 `.LRC`，检查文件是否存在且可读。

#### 1.1.3 AI 助理获取与更新

通过 Laravel AI 工具提供歌词的查询和写入能力。

| 工具 | 文件 | 作用 |
|------|------|------|
| GetLyrics | [GetLyrics.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Ai/Tools/GetLyrics.php) | 读取歌曲歌词（从本地数据库） |
| UpdateSongLyrics | [UpdateSongLyrics.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Ai/Tools/UpdateSongLyrics.php) | 更新/写入歌曲歌词到数据库 |

AI 工具本身不调用外部歌词 API，而是：
- 读取：直接从 `songs.lyrics` 数据库字段读取
- 写入：将 AI 搜索到的歌词（用户授权后）写入 `songs.lyrics` 字段

#### 1.1.4 用户手动编辑

通过 REST API 手动更新歌曲歌词。

| 层级 | 文件 | 关键方法 |
|------|------|----------|
| API 入口 | [SongController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Http/Controllers/API/SongController.php#L47-L69) | `update()` |
| 请求验证 | [SongUpdateRequest.php] | - |
| 业务逻辑 | [SongService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/SongService.php#L40-L108) | `updateSongs()` / `updateSong()` |

### 1.2 歌词数据流转

```
音频文件/外部LRC → 扫描阶段 (FileScanner) → ScanInformation.lyrics
                                              ↓
                                    SongService::createOrUpdateSongFromScan
                                              ↓
                                    songs.lyrics 字段（数据库持久化）
                                              ↓
                    ┌─────────────────────────┼─────────────────────────┐
                    ↓                         ↓                         ↓
            用户手动编辑                AI 工具读取/更新          前端显示播放
            (SongController)          (GetLyrics/               (useLyrics.ts)
                                       UpdateSongLyrics)
```

### 1.3 歌词前端解析

前端通过 `useLyrics` composable 解析歌词，支持两种格式：

| 格式 | 说明 | 解析位置 |
|------|------|----------|
| 纯文本 | 普通逐行歌词 | [useLyrics.ts](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/resources/assets/js/composables/useLyrics.ts) |
| LRC 同步 | 带时间戳 `[mm:ss.xx]` 的同步歌词 | [useLyrics.ts](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/resources/assets/js/composables/useLyrics.ts#L27-L93) |

LRC 解析时，对无时间戳的行会通过前后时间戳插值推断其显示时间。

### 1.4 歌词字段 Cast

数据库存储的歌词在读取时经过 [SongLyricsCast](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Casts/SongLyricsCast.php) 转换：
- 将 `<br>` 标签替换为换行符
- 剥离所有 HTML 标签（保留 LRC 时间戳格式）

## 二、元数据（百科信息）整合链路

元数据整合主要围绕 **专辑** 和 **艺术家** 的补充信息展开，包括 Wiki 摘要、封面图片、曲目列表等。

### 2.1 服务提供者绑定

Encyclopedia 服务的实现采用**优先级选择**模式，在 [AppServiceProvider.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Providers/AppServiceProvider.php#L63-L74) 中绑定：

```
优先级从高到低：
1. LastfmService   ← 配置了 lastfm.key + lastfm.secret 时启用
2. MusicBrainzService ← 配置了 musicbrainz.enabled 时启用
3. NullEncyclopedia  ← 兜底空实现
```

接口契约定义在 [Encyclopedia.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/Contracts/Encyclopedia.php)：
- `getArtistInformation(Artist $artist): ?ArtistInformation`
- `getAlbumInformation(Album $album): ?AlbumInformation`

### 2.2 第三方接口适配层

#### 2.2.1 Last.fm 适配

| 层级 | 文件 | 说明 |
|------|------|------|
| 服务类 | [LastfmService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/Integrations/LastfmService.php) | 实现 Encyclopedia 接口 |
| HTTP 连接器 | [LastfmConnector.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Http/Integrations/Lastfm/LastfmConnector.php) | Saloon 连接器，带签名认证 |
| 专辑信息请求 | [GetAlbumInfoRequest.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Http/Integrations/Lastfm/Requests/GetAlbumInfoRequest.php) | album.getInfo 方法 |
| 艺术家信息请求 | [GetArtistInfoRequest.php] | artist.getInfo 方法 |

返回数据结构（AlbumInformation）：
- `url`：Last.fm 专辑页面
- `cover`：封面图片 URL
- `wiki.summary` / `wiki.full`：Wiki 摘要和全文
- `tracks[]`：曲目列表（title, length, url）

#### 2.2.2 MusicBrainz + Wikipedia 适配

MusicBrainz 本身不提供 Wiki 文本，需要通过 **Pipeline 管道模式** 多级跳转获取：

```
艺术家名称 → MusicBrainz 搜索 → MBID
                              → Wikidata ID
                              → Wikipedia 页面标题
                              → Wikipedia 页面摘要

专辑名称 → MusicBrainz 搜索 → Release MBID + Release Group MBID
                            → (Release MBID) → 专辑曲目列表
                            → (Release Group MBID) → Wikidata ID
                                                    → Wikipedia 页面标题
                                                    → Wikipedia 页面摘要
```

| 管道步骤 | 文件 | 作用 |
|----------|------|------|
| GetMbidForArtist | [GetMbidForArtist.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Pipelines/Encyclopedia/GetMbidForArtist.php) | 搜索艺术家 MBID |
| GetArtistWikidataIdUsingMbid | [GetArtistWikidataIdUsingMbid.php] | 通过 MBID 查 Wikidata ID |
| GetWikipediaPageTitleUsingWikidataId | [GetWikipediaPageTitleUsingWikidataId.php] | 通过 Wikidata ID 查 Wikipedia 标题 |
| GetWikipediaPageSummaryUsingPageTitle | [GetWikipediaPageSummaryUsingPageTitle.php] | 获取 Wikipedia 摘要 |
| GetReleaseAndReleaseGroupMbidsForAlbum | [GetReleaseAndReleaseGroupMbidsForAlbum.php] | 搜索专辑 Release 和 Release Group MBID |
| GetAlbumTracksUsingMbid | [GetAlbumTracksUsingMbid.php] | 获取专辑曲目列表 |
| GetAlbumWikidataIdUsingReleaseGroupMbid | [GetAlbumWikidataIdUsingReleaseGroupMbid.php] | 通过 Release Group MBID 查 Wikidata ID |

相关连接器：
- [MusicBrainzConnector.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Http/Integrations/MusicBrainz/MusicBrainzConnector.php)
- [WikidataConnector.php]
- [WikipediaConnector.php]

#### 2.2.3 Spotify 封面补充（逐段代码分析）

Spotify 不作为主百科服务，而是作为**封面图片的补充来源**。以下是从前端触发到本地存储的完整代码走向。

##### 2.2.3.1 启用条件

配置项：`config/koel.php` 中 `services.spotify.client_id` 和 `services.spotify.client_secret`，对应环境变量 `SPOTIFY_CLIENT_ID` 和 `SPOTIFY_CLIENT_SECRET`。

判断方法在 [SpotifyService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/Integrations/SpotifyService.php#L16-L19)：

```php
public static function enabled(): bool
{
    return config('koel.services.spotify.client_id') && config('koel.services.spotify.client_secret');
}
```

只有两个配置都非空时，Spotify 集成才启用。

##### 2.2.3.2 前端触发入口

触发点有两个组件：
- [AlbumInfo.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/resources/assets/js/components/album/AlbumInfo.vue) — 专辑信息面板
- [ArtistInfo.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/resources/assets/js/components/artist/ArtistInfo.vue) — 艺术家信息面板

两个组件都使用相同的触发模式：**通过 `watch` 监听数据变化**。以 `AlbumInfo.vue` 为例：

```js
const { useMusicBrainz, useLastfm, useSpotify } = useThirdPartyServices()

watch(
  album,
  async () => {
    info.value = null
    // 只要任意一个百科服务启用，就触发获取
    if (useMusicBrainz.value || useLastfm.value || useSpotify.value) {
      loading.value = true
      info.value = await encyclopediaService.fetchForAlbum(album.value)
      loading.value = false
    }
  },
  { immediate: true, deep: true },
)
```

关键点：
- `useSpotify` 来自 `useThirdPartyServices()`，读取 `commonStore.state.uses_spotify`
- `watch` 带有 `immediate: true`，组件挂载时立即执行
- 只要有一个百科服务（Last.fm / MusicBrainz / Spotify）启用，就会触发获取

##### 2.2.3.3 前端服务层（encyclopediaService）

[encyclopediaService.ts](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/resources/assets/js/services/encyclopediaService.ts) 是前端百科服务，带**内存级缓存**：

```js
async fetchForAlbum(album: Album) {
  album = albumStore.syncWithVault(album)[0]
  const cacheKey = ['album.info', album.id, album.name]

  if (cache.has(cacheKey)) {
    return cache.get<AlbumInfo>(cacheKey)
  }

  const info = await http.get<AlbumInfo | null>(`albums/${album.id}/information`)
  info && cache.set(cacheKey, info)

  // 获取到封面后，同步更新 album.cover 和所有相关歌曲的 album_cover
  if (info?.cover) {
    album.cover = info.cover
    playableStore.byAlbum(album).forEach(song => (song.album_cover = info.cover!))
  }

  return info
}
```

前端缓存是 L1 缓存（会话级），避免同一页面多次请求。

##### 2.2.3.4 后端编排层（EncyclopediaService）

[EncyclopediaService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/Integrations/EncyclopediaService.php) 是后端编排门面，带 **L2 应用级缓存（1 周）**。

外层入口 `getAlbumInformation()`：

```php
public function getAlbumInformation(Album $album): ?AlbumInformation
{
    if ($album->is_unknown) {
        return null;
    }

    return rescue(
        fn () => Cache::remember(
            cache_key('album information', $album->name, $album->artist->name),
            now()->addWeek(),
            fn () => $this->fetchAlbumInformation($album),
        ),
        fn () => $this->fetchAlbumInformation($album),
    );
}
```

缓存 key 由 `album information + 专辑名 + 艺术家名` 组成，TTL 1 周。

内层 `fetchAlbumInformation()` 是真正的获取逻辑，包含 **Spotify 补充封面的触发条件**：

```php
private function fetchAlbumInformation(Album $album): AlbumInformation
{
    $info = $this->encyclopedia->getAlbumInformation($album) ?: AlbumInformation::make();

    // 封面触发条件：本地已有封面 → 直接返回
    // 或者：Spotify 未启用 且 百科也没返回封面 → 直接返回
    if ($album->cover || !SpotifyService::enabled() && !$info->cover) {
        return $info;
    }

    $info->cover = rescue(
        function () use ($album, $info): ?string {
            return $this->fetchAndStoreAlbumCover($album, $info) ?? $info->cover;
        },
        static fn () => $info->cover,
    );

    return $info;
}
```

**封面触发条件解读**（`$album->cover || !SpotifyService::enabled() && !$info->cover`）：
- 由于 `&&` 优先级高于 `||`，实际等价于：`$album->cover || (!SpotifyService::enabled() && !$info->cover)`
- 即：如果本地已有封面 → 不调用 Spotify，直接返回
- 或者：如果 Spotify 未启用 **且** 百科也没封面 → 不调用 Spotify，直接返回
- 反过来说，**调用 Spotify 的条件**是：本地无封面 **且** (Spotify 已启用 **或** 百科有封面)

然后 `fetchAndStoreAlbumCover()` 执行实际获取和存储：

```php
private function fetchAndStoreAlbumCover(Album $album, AlbumInformation $info): ?string
{
    // Spotify 启用时用 Spotify 搜索，否则用百科返回的封面
    $coverUrl = SpotifyService::enabled() ? $this->spotifyService->tryGetAlbumCover($album) : $info->cover;

    if (!$coverUrl) {
        return null;
    }

    $fileName = $this->imageStorage->storeImage($coverUrl);
    $album->cover = $fileName;
    $album->save();

    return image_storage_url($fileName);
}
```

##### 2.2.3.5 Spotify 搜索查询

[SpotifyService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/Integrations/SpotifyService.php) 封装了搜索逻辑：

```php
public function tryGetAlbumCover(Album $album): ?string
{
    if (!static::enabled()) {
        return null;
    }

    // 未知专辑/未知艺术家/合辑艺术家 不搜索
    if ($album->is_unknown || $album->artist->is_unknown || $album->artist->is_various) {
        return null;
    }

    // 构造查询：专辑名 artist:艺术家名
    return Arr::get($this->client->search("$album->name artist:{$album->artist->name}", 'album', [
        'limit' => 1,
    ]), 'albums.items.0.images.0.url');
}
```

查询构造使用 Spotify 的 `artist:` 字段语法，精确匹配艺术家。只取第一条结果的最大尺寸封面（`images.0.url`）。

##### 2.2.3.6 Access Token 缓存

[SpotifyClient.php](file:///d:/fz/0508-2\solo-dogfeeding\code\112-koel\app\Http\Integrations\Spotify\SpotifyClient.php) 封装了 `SpotifyWebAPI`，负责 **Access Token 的获取和缓存**。

构造函数中调用 `setAccessToken()`：

```php
public function __construct(
    public SpotifyWebAPI $wrapped,
    private readonly ?Session $session,
    private readonly Cache $cache,
) {
    if (SpotifyService::enabled()) {
        $this->wrapped->setOptions(['return_assoc' => true]);
        rescue($this->setAccessToken(...));
    }
}
```

`setAccessToken()` 方法实现 Token 缓存逻辑：

```php
private function setAccessToken(): void
{
    $token = $this->cache->get(self::ACCESS_TOKEN_CACHE_KEY);

    if (!$token) {
        // 缓存未命中，通过 Client Credentials 模式获取新 Token
        $this->session->requestCredentialsToken();
        $token = $this->session->getAccessToken();

        // Spotify 的 Token 有效期 1 小时，缓存 59 分钟留 1 分钟缓冲
        $this->cache->put(self::ACCESS_TOKEN_CACHE_KEY, $token, 59 * 60);
    }

    $this->wrapped->setAccessToken($token);
}
```

关键点：
- 缓存 key：`spotify.access_token`（常量 `ACCESS_TOKEN_CACHE_KEY`）
- TTL：59 分钟（3540 秒），比 Spotify 官方 1 小时有效期少 1 分钟作为安全缓冲
- 认证模式：Client Credentials Flow（客户端凭证模式），无需用户授权
- Token 获取与业务数据缓存分离，独立管理

Session 的绑定在 [AppServiceProvider.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Providers/AppServiceProvider.php#L54-L61)：

```php
$this->app->bind(SpotifySession::class, static function () {
    return SpotifyService::enabled()
        ? new SpotifySession(
            config('koel.services.spotify.client_id'),
            config('koel.services.spotify.client_secret'),
        )
        : null;
});
```

##### 2.2.3.7 封面本地化存储

封面通过 [ImageStorage](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/Image/ImageStorage.php) 下载到本地：
- 文件名使用 ULID 随机生成
- 格式统一转换为 webp
- 数据库 `albums.cover` / `artists.image` 字段只存文件名

#### 2.2.4 iTunes 曲目链接（逐段代码分析）

iTunes 提供曲目查看 URL（带联盟 ID），不参与百科信息。以下是从前端按钮到后端重定向的完整代码走向。

##### 2.2.4.1 启用条件

配置项：`config/koel.php` 中 `services.itunes.enabled`，对应环境变量 `USE_ITUNES`，**默认为 true**。

```php
'itunes' => [
    'enabled' => env('USE_ITUNES', true),
    'affiliate_id' => '1000lsGu',
    'endpoint' => 'https://itunes.apple.com/search',
],
```

联盟 ID `1000lsGu` 硬编码在配置中，用于追踪从 Koel 到 iTunes Store 的转化。

判断方法在 [ITunesService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/Integrations/ITunesService.php#L16-L19)：

```php
public static function used(): bool
{
    return (bool) config('koel.services.itunes.enabled');
}
```

##### 2.2.4.2 前端按钮组件

[AppleMusicButton.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/resources/assets/js/components/ui/AppleMusicButton.vue) 是一个纯展示组件：

- 渐变背景（`itunes-gradient`）：从红到紫到蓝的 27 度线性渐变
- 内嵌 Apple Music logo 的 SVG
- 接收一个 `url` prop，点击后在新标签页打开

按钮不处理任何业务逻辑，只是一个带样式的 `<a target="_blank">` 链接。

##### 2.2.4.3 按钮显示与 URL 构造

按钮在 [AlbumTrackListItem.vue](file:///d:/fz/0508-2\solo-dogfeeding\code\112-koel\resources\assets\js\components\album\AlbumTrackListItem.vue) 中使用：

```html
<AppleMusicButton v-if="useAppleMusic && !matchedSong" :url="iTunesUrl" />
```

**显示条件**：
- `useAppleMusic`：来自 `useThirdPartyServices()`，即 `commonStore.state.uses_i_tunes`
- `!matchedSong`：本地库中没有匹配的歌曲时才显示（引导用户去 iTunes 购买）

**URL 构造**：

```js
const iTunesUrl = computed(() => {
  return `${window.KOEL.base_url}itunes/song/${album.value.id}?q=${encodeURIComponent(track.value.title)}&api_token=${authService.getApiToken()}`
})
```

URL 组成：
- 路径：`/itunes/song/{album_id}` — web 路由
- 查询参数 `q`：曲目名称（URL 编码）
- 查询参数 `api_token`：用户的 API Token，用于后端鉴权

注意：这是 **web 路由**（不是 API 路由），因为后端会直接 302 重定向到 iTunes 页面。

##### 2.2.4.4 路由定义

路由在 [routes/web.base.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/routes/web.base.php#L37-L39) 中定义：

```php
if (ITunes::used()) {
    Route::get('itunes/song/{album}', ViewSongOnITunesController::class)->name('iTunes.viewSong');
}
```

关键点：
- 路由在 `web` 中间件组内
- 用 `ITunes::used()` 条件判断，未启用时不注册路由
- 单动作控制器（invokable）
- 路由模型绑定：`{album}` 参数自动解析为 Album 模型

##### 2.2.4.5 后端控制器与鉴权

[ViewSongOnITunesController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Http/Controllers/ViewSongOnITunesController.php) 处理请求：

```php
public function __invoke(
    ViewSongOnITunesRequest $request,
    ITunesService $iTunesService,
    TokenManager $tokenManager,
    Album $album,
) {
    // 第一步：api_token 鉴权
    abort_unless((bool) $tokenManager->getUserFromPlainTextToken($request->api_token), Response::HTTP_UNAUTHORIZED);

    // 第二步：调用 iTunes 服务获取曲目 URL
    $url = $iTunesService->getTrackUrl($request->q, $album);
    abort_unless((bool) $url, Response::HTTP_NOT_FOUND, "Koel can't find such a song on iTunes Store.");

    // 第三步：302 重定向到 iTunes 页面
    return redirect($url);
}
```

鉴权方式：
- 不使用 Laravel 默认的 `auth` 中间件
- 手动使用 `TokenManager::getUserFromPlainTextToken()` 验证 `api_token`
- 底层调用 `PersonalAccessToken::findToken()`，即 Laravel Sanctum

请求验证在 [ViewSongOnITunesRequest.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Http/Requests/API/ViewSongOnITunesRequest.php)：
- `q`：必填，搜索词
- `api_token`：必填

##### 2.2.4.6 iTunes Search API 调用

[ITunesService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/Integrations/ITunesService.php) 的 `getTrackUrl()` 方法：

```php
public function getTrackUrl(string $trackName, Album $album): ?string
{
    return rescue(function () use ($trackName, $album): ?string {
        $request = new GetTrackRequest($trackName, $album);

        return Cache::remember(
            cache_key('iTunes track URL', serialize($request->query())),
            now()->addWeek(),
            function () use ($request): ?string {
                $response = $this->connector->send($request)->object();

                if (!$response->resultCount) {
                    return null;
                }

                $trackUrl = $response->results[0]->trackViewUrl;
                $connector = parse_url($trackUrl, PHP_URL_QUERY) ? '&' : '?';

                // 追加联盟 ID
                return $trackUrl . "{$connector}at=" . config('koel.services.itunes.affiliate_id');
            },
        );
    });
}
```

搜索词构造在 [GetTrackRequest.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Http/Integrations/iTunes/Requests/GetTrackRequest.php)：

```php
protected function defaultQuery(): array
{
    $term = $this->trackName;

    if ($this->album->name !== Album::UNKNOWN_NAME) {
        $term .= ' ' . $this->album->name;
    }

    if (
        $this->album->artist->name !== Artist::UNKNOWN_NAME
        && $this->album->artist->name !== Artist::VARIOUS_NAME
    ) {
        $term .= ' ' . $this->album->artist->name;
    }

    return [
        'term' => $term,
        'media' => 'music',
        'entity' => 'song',
        'limit' => 1,
    ];
}
```

搜索词拼接逻辑：
- 基础：曲目名
- 如果专辑名不是 "Unknown Album"，追加专辑名
- 如果艺术家名不是 "Unknown Artist" 且不是 "Various Artists"，追加艺术家名

##### 2.2.4.7 一周缓存策略

iTunes 曲目 URL 的缓存：
- **缓存 key 生成**：`cache_key('iTunes track URL', serialize($request->query()))`
  - 将所有查询参数序列化后作为 key 的一部分
  - 不同的搜索词、不同的专辑都会生成不同的缓存 key
- **TTL**：1 周（`now()->addWeek()`）
- **缓存位置**：[ITunesService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/Integrations/ITunesService.php#L26-L41)

为什么用一周 TTL：
- iTunes 曲目的 URL 是稳定的，不会频繁变化
- 减少对 iTunes Search API 的调用次数
- 一周的平衡：既不会缓存太久导致过时，也不会太频繁调用

##### 2.2.4.8 Saloon HTTP 连接器

iTunes API 调用使用 [Saloon](https://docs.saloon.dev/) HTTP 客户端封装：

- [ITunesConnector.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Http/Integrations/iTunes/ITunesConnector.php) — 连接器，设置 base URL
- [GetTrackRequest.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Http/Integrations/iTunes/Requests/GetTrackRequest.php) — 请求类，定义 endpoint 和 query 参数

连接器使用了两个 trait：
- `AcceptsJson` — 自动设置 Accept: application/json 头
- `AlwaysThrowOnErrors` — HTTP 错误时抛出异常（配合外层 `rescue()` 捕获）

#### 2.2.5 NullEncyclopedia 空对象

[NullEncyclopedia.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/Integrations/NullEncyclopedia.php) 是兜底实现，返回空的 `AlbumInformation` / `ArtistInformation`，避免调用方做空判断。

### 2.3 EncyclopediaService 编排层

[EncyclopediaService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/Integrations/EncyclopediaService.php) 是元数据获取的**编排门面**，负责：

1. **缓存读取**：优先从 Laravel Cache 读取
2. **主数据源调用**：调用绑定的 Encyclopedia 实现（Last.fm 或 MusicBrainz）
3. **封面补充**：如果主数据源无封面，尝试从 Spotify 获取
4. **封面本地化存储**：将远程封面下载到本地存储，更新数据库 `albums.cover` / `artists.image` 字段
5. **失败降级**：`rescue()` 包裹，失败时返回空信息

## 三、缓存策略

### 3.1 后端缓存

#### 3.1.1 百科信息缓存（EncyclopediaService）

| 缓存项 | Key 生成 | TTL | 位置 |
|--------|----------|-----|------|
| 专辑信息 | `album information + 专辑名 + 艺术家名` | 1 周 | [EncyclopediaService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/Integrations/EncyclopediaService.php#L28-L36) |
| 艺术家信息 | `artist information + 艺术家名` | 1 周 | [EncyclopediaService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/Integrations/EncyclopediaService.php#L44-L52) |

缓存策略：`Cache::remember()` + `rescue()` 降级
- 缓存命中直接返回
- 缓存未命中调用第三方 API
- API 调用失败时返回空对象（不写入缓存）

#### 3.1.2 MusicBrainz 管道步骤缓存

使用 [TriesRemember](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Pipelines/Encyclopedia/TriesRemember.php) trait，对每个管道步骤单独缓存。

| 缓存项 | Key 生成 | TTL | 位置 |
|--------|----------|-----|------|
| 艺术家 MBID | `artist mbid + 艺术家名` | 永久 | [GetMbidForArtist.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Pipelines/Encyclopedia/GetMbidForArtist.php#L23-L25) |
| 其他管道步骤 | 各步骤自定义 | 永久/长期 | - |

永久缓存的原因：MBID、Wikidata ID 等标识符是稳定不变的。

#### 3.1.3 iTunes 曲目链接缓存

| 缓存项 | TTL | 位置 |
|--------|-----|------|
| iTunes 曲目 URL | 1 周 | [ITunesService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/Integrations/ITunesService.php#L26-L29) |

#### 3.1.4 扫描阶段内存缓存

扫描时使用 [ScannerCacheStrategy](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/Scanners/ScannerCacheStrategy.php) 做内存级 LRU 缓存（默认 1000 条），避免重复查询艺术家/专辑。

| 缓存项 | 用途 |
|--------|------|
| 艺术家解析 | `resolveArtist()` 中缓存 Artist 实例 |
| 专辑解析 | `resolveAlbum()` 中缓存 Album 实例 |

#### 3.1.5 目录封面缓存

| 缓存项 | TTL | 位置 |
|--------|-----|------|
| 目录封面查找结果 | 1 天 | [AlbumService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/AlbumService.php#L59-L81) |

### 3.2 前端缓存

前端 `encyclopediaService` 使用内存 `cache` 对象做单层缓存：

| 缓存项 | Key | 位置 |
|--------|-----|------|
| 艺术家信息 | `['artist.info', artist.id]` | [encyclopediaService.ts](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/resources/assets/js/services/encyclopediaService.ts#L8-L21) |
| 专辑信息 | `['album.info', album.id, album.name]` | [encyclopediaService.ts](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/resources/assets/js/services/encyclopediaService.ts#L23-L40) |

副作用：获取到封面后，会同步更新 store 中的 `album.cover` / `artist.image`。

## 四、曲目本地字段更新流程

### 4.1 扫描阶段写入（初始/增量扫描）

```
音频文件 → getID3 分析 → ScanInformation 数据对象
                              ↓
                SongService::createOrUpdateSongFromScan
                              ↓
                    新建或更新 songs 表记录
                              ↓
                    同步 genres 关联表
                              ↓
                    同步更新 albums.year 等字段
```

关键文件：
- 入口：[SongService.php::createOrUpdateSongFromScan()](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/SongService.php#L216-L290)
- 歌曲字段：title, artist_id, album_id, lyrics, track, disc, year, length 等
- 忽略机制：`ScanConfiguration.ignores` 配置，非新文件时跳过指定字段更新

歌词字段在扫描时的写入逻辑：
1. 优先取音频文件内嵌歌词标签
2. 为空则尝试读取同名 `.lrc` 文件
3. 都没有则为空字符串

### 4.2 API 编辑更新

```
PUT /api/songs → SongUpdateRequest 验证
                      ↓
            SongService::updateSongs
                      ↓
            事务内逐条 updateSong
                      ↓
            更新 songs 表 + 处理 albums/artists 增减
                      ↓
            同步 genres
```

关键文件：
- 控制器：[SongController.php::update()](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Http/Controllers/API/SongController.php#L47-L69)
- 业务逻辑：[SongService.php::updateSongs()](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/SongService.php#L40-L108)
- DTO：[SongUpdateData.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Values/Song/SongUpdateData.php)

单首 vs 多首更新的区别：
- 单首：空值字段被视为"清空"（lyrics 设为 ''）
- 多首：空值字段被视为"不修改"（保留原值，`??=` 运算符）

### 4.3 AI 工具更新

```
UpdateSongLyrics AI Tool
        ↓
  songResolver 解析歌曲
        ↓
  $song->lyrics = $request['lyrics']
        ↓
  $song->save()
```

位置：[UpdateSongLyrics.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Ai/Tools/UpdateSongLyrics.php#L43-L58)

### 4.4 元数据封面字段更新

当百科服务获取到封面后，会写回本地数据库：

- 专辑封面：`albums.cover` 字段，存储本地文件名
- 艺术家图片：`artists.image` 字段，存储本地文件名

位置：
- [EncyclopediaService.php::fetchAndStoreAlbumCover()](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/Integrations/EncyclopediaService.php#L90-L103)
- [EncyclopediaService.php::fetchAndStoreArtistImage()](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/Integrations/EncyclopediaService.php#L105-L118)

图片通过 [ImageStorage] 服务下载并存储到本地磁盘，数据库仅存文件名。

## 五、协作关系总览

### 5.1 歌词链路协作

```
  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
  │  音频内嵌标签│────▶│  FileScanner │────▶│ songs.lyrics │
  └─────────────┘     └─────────────┘     └──────┬──────┘
         ▲                                         │
  ┌──────┴──────┐                                  │
  │  .lrc 文件   │                                  ▼
  └─────────────┘                           ┌──────────────┐
                                            │ SongLyricsCast │
                                            └──────┬──────┘
  ┌─────────────┐                                 │
  │   AI 工具    │────── 读/写 ───────────────────▶│
  │GetLyrics /  │                                 │
  │UpdateSongLyrics│                              ▼
  └─────────────┘                           ┌──────────────┐
                                            │  useLyrics.ts │
  ┌─────────────┐                           └──────────────┘
  │  API 编辑    │────── 更新 ───────────────────▶│
  │SongController│
  └─────────────┘
```

### 5.2 元数据链路协作

```
  ┌───────────────────────────────────────────────────────────┐
  │                    EncyclopediaService                    │
  │              (编排门面 + 缓存 + 封面补充)                 │
  └─────────────┬───────────────────────────┬─────────────────┘
                ▼                           ▼
  ┌─────────────────────┐     ┌─────────────────────────────┐
  │  Encyclopedia 接口   │     │      SpotifyService         │
  │  (Last.fm / MB / Null)│     │     (封面补充来源)         │
  └─────────┬───────────┘     └───────────────┬─────────────┘
            │                                 │
            ▼                                 ▼
  ┌─────────────────────┐     ┌─────────────────────────────┐
  │ Last.fm / MusicBrainz│     │      albums.cover           │
  │   HTTP API 调用      │────▶│      artists.image          │
  └─────────────────────┘     │  (本地存储文件名)            │
                              └─────────────────────────────┘
```

### 5.3 Spotify 封面完整链路

```
  ┌──────────────────────────────────────────────────────────┐
  │  前端层                                                  │
  │  ┌──────────────────┐     ┌─────────────────────────┐   │
  │  │ AlbumInfo.vue    │────▶│ encyclopediaService.ts  │   │
  │  │ ArtistInfo.vue   │     │ (cache + http 请求)     │   │
  │  └────────┬─────────┘     └───────────┬─────────────┘   │
  │           │  watch album/artist 变化  │ 内存缓存命中？   │
  └───────────┼───────────────────────────┼──────────────────┘
              ▼                           │
  ┌───────────────────────────────────────┘
  │  后端编排层
  │  ┌──────────────────────────────────────────────────┐
  │  │ EncyclopediaService::getAlbumInformation()       │
  │  │   1. 缓存命中？→ 直接返回                         │
  │  │   2. 调用 Encyclopedia 接口获取百科信息          │
  │  │   3. 无封面？→ 调用 SpotifyService::tryGetAlbumCover()
  │  │   4. 有封面？→ ImageStorage 下载到本地            │
  │  │   5. 写入缓存，返回结果                          │
  │  └──────────────────────────────────────────────────┘
  └───────────┬──────────────────────────────────────────┘
              ▼
  ┌──────────────────────────────────────────────────────┐
  │  Spotify API 层                                       │
  │  ┌────────────────────────────────────────────────┐   │
  │  │ SpotifyClient (SpotifyWebAPI 封装)              │   │
  │  │   - Client Credentials 获取 access_token       │   │
  │  │   - access_token 缓存 59 分钟                   │   │
  │  │   - search album/artist → 取 images[0].url     │   │
  │  └────────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────┘
              │
              ▼
  ┌──────────────────────────────────────────────────────┐
  │  本地持久化层                                         │
  │  ┌──────────────────┐  ┌─────────────────────────┐   │
  │  │ ImageStorage     │  │  albums.cover           │   │
  │  │  - ULID 命名     │──▶│  artists.image         │   │
  │  │  - webp 格式     │  │  (数据库字段存文件名)    │   │
  │  └──────────────────┘  └─────────────────────────┘   │
  └──────────────────────────────────────────────────────┘
```

### 5.4 iTunes 曲目链接完整链路

```
  ┌──────────────────────────────────────────────────────────┐
  │  前端层                                                  │
  │  ┌──────────────────────┐                               │
  │  │ AppleMusicButton.vue │                               │
  │  │   - 渐变背景按钮      │                               │
  │  │   - 悬停显示放大图标  │                               │
  │  │   - 显示条件：        │                               │
  │  │     useAppleMusic     │                               │
  │  │     && !matchedSong   │                               │
  │  └──────────┬───────────┘                               │
  │             │ 点击                                       │
  └─────────────┼────────────────────────────────────────────┘
                ▼
  ┌──────────────────────────────────────────────────────────┐
  │  后端 Web 路由层                                          │
  │  ┌────────────────────────────────────────────────────┐  │
  │  │ Route: /iTunes/view-song                           │  │
  │  │   - web 路由（非 api）                             │  │
  │  │   - api_token 鉴权（TokenManager）                 │  │
  │  │   - ViewSongOnITunesController                     │  │
  │  │   - 302 重定向到 iTunes 页面                       │  │
  │  └──────────┬─────────────────────────────────────────┘  │
  └─────────────┼────────────────────────────────────────────┘
                ▼
  ┌──────────────────────────────────────────────────────────┐
  │  iTunes API 层                                            │
  │  ┌────────────────────────────────────────────────────┐  │
  │  │ ITunesService::getTrackUrl()                       │  │
  │  │   - 按查询参数序列化缓存（1周）                     │  │
  │  │   - ITunesConnector (Saloon)                       │  │
  │  │   - GetTrackRequest                               │  │
  │  │   - 搜索词：歌曲名 + 艺术家名                      │  │
  │  │   - 追加联盟 ID：at=1000lsGu                       │  │
  │  └────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────┘
```

### 5.5 缓存层级全景

```
  ┌──────────────────────────────────────────────────────────┐
  │ L1: 前端内存缓存（encyclopediaService.cache）             │
  │   - 会话级，页面刷新即失效                                │
  │   - Key: ['album.info', id, name] / ['artist.info', id]  │
  └───────────────────────────┬──────────────────────────────┘
                              ▼
  ┌──────────────────────────────────────────────────────────┐
  │ L2: 后端 Laravel Cache（应用级缓存）                      │
  │   - 百科信息：1 周   (EncyclopediaService)               │
  │   - MBID/URL：永久  (MusicBrainz Pipeline)               │
  │   - iTunes 链接：1 周  (ITunesService)                   │
  │   - Spotify Token：59 分钟 (SpotifyClient)               │
  │   - 目录封面：1 天  (AlbumService)                       │
  └───────────────────────────┬──────────────────────────────┘
                              ▼
  ┌──────────────────────────────────────────────────────────┐
  │ L3: 扫描期内存缓存（ScannerCacheStrategy）                │
  │   - LRU 1000 条                                          │
  │   - 单次扫描任务内有效                                    │
  │   - Artist / Album 实例缓存                              │
  └───────────────────────────┬──────────────────────────────┘
                              ▼
  ┌──────────────────────────────────────────────────────────┐
  │ L4: 数据库持久化                                          │
  │   - songs.lyrics（歌词文本）                              │
  │   - albums.cover（封面文件名）                            │
  │   - artists.image（艺术家图片文件名）                     │
  │   - songs 其他元数据字段                                  │
  └──────────────────────────────────────────────────────────┘
```

## 六、关键设计决策总结

1. **Encyclopedia 接口 + 优先级绑定**：通过服务容器绑定实现可插拔的百科数据源，Last.fm 优先，MusicBrainz 备选
2. **Pipeline 管道模式**：MusicBrainz 链路采用多级管道串联，每步独立缓存，应对需要多次跳转的数据源
3. **编排门面（EncyclopediaService）**：统一缓存、封面补充、失败降级逻辑，与具体数据源解耦
4. **歌词双层来源**：扫描阶段从文件提取（内嵌标签 + .lrc 文件），运行阶段通过 AI 工具和手动编辑补充
5. **四级缓存架构**：前端内存 → 后端应用缓存 → 扫描期 LRU → 数据库持久化，分层减少第三方 API 调用
6. **Null Object 模式**：NullEncyclopedia 避免调用方做空值判断
7. **封面本地化存储**：第三方封面图片下载到本地，ULID 随机命名 + webp 格式，避免直接引用远程 URL
8. **Spotify 补充封面策略**：百科信息无封面时才调用 Spotify，作为第二来源而非主来源
9. **Spotify Token 独立缓存**：Access Token 与业务数据分离缓存，59 分钟（1小时减1分钟安全缓冲）
10. **iTunes Web 路由设计**：iTunes 跳转走 web 路由而非 API 路由，便于直接 302 重定向到外部页面
11. **iTunes 联盟营销追踪**：URL 追加 `at=1000lsGu` 联盟 ID，流量转化可追踪
12. **前端功能开关驱动**：`uses_spotify` / `useAppleMusic` 等标志控制 UI 显示，后端配置决定前端能力
