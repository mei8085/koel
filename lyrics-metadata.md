# 歌词获取与元数据整合外部链路分析

## 概述

Koel 的歌词获取与元数据整合走两条相对独立但又有协作的链路：
- **歌词链路**：以本地文件内嵌标签和 .lrc 外部文件为主要来源，辅以 AI 工具和人工编辑
- **元数据链路**：以 Last.fm / MusicBrainz 等第三方百科服务为主要来源，获取专辑/艺术家的 Wiki 摘要、封面、曲目列表等信息

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

#### 2.2.3 Spotify 封面补充

Spotify 不作为主百科服务，而是作为**封面图片的补充来源**。

| 文件 | 方法 | 作用 |
|------|------|------|
| [SpotifyService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/Integrations/SpotifyService.php) | `tryGetAlbumCover()` | 搜索专辑封面 |
| [SpotifyService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/Integrations/SpotifyService.php) | `tryGetArtistImage()` | 搜索艺术家图片 |
| [SpotifyClient.php] | `search()` | Spotify Web API 搜索 |

调用时机：在 [EncyclopediaService](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/Integrations/EncyclopediaService.php#L54-L70) 的 `fetchAlbumInformation` 和 `fetchArtistInformation` 中，当主百科服务没有返回封面且本地无封面时，尝试从 Spotify 获取。

#### 2.2.4 iTunes 曲目链接

iTunes 提供曲目查看 URL（带联盟 ID），不参与百科信息。

| 文件 | 方法 | 作用 |
|------|------|------|
| [ITunesService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/112-koel/app/Services/Integrations/ITunesService.php) | `getTrackUrl()` | 获取 iTunes 曲目页面链接 |

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

### 5.3 缓存层级

```
  ┌──────────────────────────────────────────────────────┐
  │ 前端 encyclopediaService.cache（内存，会话级）        │
  └───────────────────────┬──────────────────────────────┘
                          ▼
  ┌──────────────────────────────────────────────────────┐
  │ 后端 Laravel Cache（文件/Redis，配置级 TTL）          │
  │  - 专辑/艺术家百科信息：1 周                          │
  │  - MBID / Wikidata ID：永久                           │
  │  - iTunes 链接：1 周                                 │
  │  - 目录封面：1 天                                    │
  └───────────────────────┬──────────────────────────────┘
                          ▼
  ┌──────────────────────────────────────────────────────┐
  │ 数据库持久化                                          │
  │  - songs.lyrics（歌词字段）                           │
  │  - albums.cover（封面文件名）                         │
  │  - artists.image（艺术家图片文件名）                  │
  │  - songs 其他元数据字段                               │
  └──────────────────────────────────────────────────────┘
```

## 六、关键设计决策总结

1. **Encyclopedia 接口 + 优先级绑定**：通过服务容器绑定实现可插拔的百科数据源，Last.fm 优先，MusicBrainz 备选
2. **Pipeline 管道模式**：MusicBrainz 链路采用多级管道串联，每步独立缓存，应对需要多次跳转的数据源
3. **编排门面（EncyclopediaService）**：统一缓存、封面补充、失败降级逻辑，与具体数据源解耦
4. **歌词双层来源**：扫描阶段从文件提取（内嵌标签 + .lrc 文件），运行阶段通过 AI 工具和手动编辑补充
5. **多级缓存**：前端内存缓存 → 后端应用缓存（分级 TTL） → 数据库持久化，减少第三方 API 调用
6. **Null Object 模式**：NullEncyclopedia 避免调用方做空值判断
7. **封面本地化**：第三方封面图片下载到本地存储，避免直接引用远程 URL
