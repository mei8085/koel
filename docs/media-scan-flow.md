# Koel 媒体库扫描流程：磁盘音频文件 → 歌曲数据模型

## 概述

Koel 的媒体库扫描将磁盘上的音频文件解析并映射为数据库中的 `Song`、`Album`、`Artist`、`Genre` 等 Eloquent 模型。整条流水线的核心思路是：**遍历目录 → 逐文件提取元数据 → 解析关联实体 → 持久化到数据库**。

---

## 一、入口：触发扫描

扫描有两个触发入口：

### 1. CLI 入口：`koel:scan` 命令

[ScanCommand](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Console/Commands/ScanCommand.php#L21-L79)

```
php artisan koel:scan [--owner=] [--private] [--ignore=] [--force] [--jobs=]
```

- 从 `Setting` 读取 `media_path`，构建 `ScanConfiguration`
- 如果传入 watch record 参数（用于 inotify 文件监听），走 `WatchRecordScanner`；否则走 `DirectoryScanner`

### 2. Web 入口：更新媒体路径时自动扫描

[UpdateMediaPathController](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Http/Controllers/API/Settings/UpdateMediaPathController.php#L24-L34)

用户在 Settings 页面设置媒体路径后，后端调用 `DirectoryScanner::scan()` 自动触发一次扫描。

---

## 二、扫描配置

[ScanConfiguration](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Values/Scanning/ScanConfiguration.php#L7-L33)

| 字段 | 含义 |
|------|------|
| `owner` | 新歌曲所属用户 |
| `makePublic` | 新歌曲是否公开 |
| `ignores` | 重新扫描时忽略的标签（如 title, album, artist 等） |
| `force` | 是否强制重新扫描未修改的文件 |
| `extractFolderStructure` | 是否提取文件夹结构（本地存储为 true，云存储为 false） |

---

## 三、目录遍历与文件发现

### 3.1 DirectoryScanner

[DirectoryScanner](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/DirectoryScanner.php#L28-L49)

```
scan(directory, config, jobs=1) → ScanResultCollection
```

流程：

1. 调用 `setSystemRequirements()` 设置 `time_limit` 和 `memory_limit`
2. 调用 `gatherFiles(directory)` 获取所有匹配的音频文件
3. 根据并行度选择策略：
   - `jobs > 1` → `ParallelScanStrategy`（多进程）
   - `jobs = 1` → `SequentialScanStrategy`（单进程遍历）
4. 扫描完成后触发 `MediaScanCompleted` 事件

### 3.2 gatherFiles — 音频文件过滤

[Scanner::gatherFiles](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/Scanner.php#L23-L36)

使用 Symfony `Finder` 组件遍历目录：

- `ignoreUnreadableDirs()` — 跳过不可读目录
- `ignoreDotFiles()` — 根据配置 `koel.ignore_dot_files` 决定是否忽略隐藏文件/目录
- `followLinks()` — 跟随符号链接
- `name(regex)` — 按扩展名正则过滤：`/\.(mp3|ogg|flac|m4a|...)$/i`

扩展名列表来自 `collect_accepted_audio_extensions()`，该函数读取 `config('koel.streaming.supported_mime_types')` 中的所有值并展平去重，当前支持约 30 种格式（MP3, AAC, OGG, FLAC, WMA, AIFF, APE, WavPack, DSD 等）。

### 3.3 并行扫描 vs 顺序扫描

**顺序扫描**：[SequentialScanStrategy](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/Strategies/SequentialScanStrategy.php#L9-L30)

逐文件调用 `IndividualFileHandler::handle()`，每个文件返回一个 `ScanResult`。

**并行扫描**：[ParallelScanStrategy](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/Strategies/ParallelScanStrategy.php#L15-L209)

将文件列表均分为 N 个 chunk，每个 chunk 写入临时 JSON manifest，然后 spawn N 个子进程执行 `koel:scan:chunk` 命令。子进程内部复用 `SequentialScanStrategy`，每扫描完一个文件就向 stdout 输出一行 JSON 结果。主进程通过轮询子进程 stdout 收集结果。

[ScanChunkCommand](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Console/Commands/ScanChunkCommand.php#L13-L72) 是并行扫描的内部辅助命令，对用户隐藏。

---

## 四、单文件处理

### 4.1 IndividualFileHandler

[IndividualFileHandler](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/IndividualFileHandler.php#L12-L37)

```
handle(path, config) → ScanResult
```

对每个文件执行以下步骤：

1. **查库**：`songRepository.findOneByPath(path)` 查找已有记录
2. **跳过判断**：如果歌曲已存在、文件未修改（mtime 未变）、且非 force 模式 → 返回 `ScanResult::skipped()`
3. **扫描文件**：`fileScanner.scan(path)` 提取元数据 → 得到 `ScanInformation`
4. **写入数据库**：`songService.createOrUpdateSongFromScan(info, config, song)` 创建或更新歌曲
5. **异常捕获**：任何 `Throwable` 都被捕获为 `ScanResult::error()`

### 4.2 WatchRecordScanner（文件监听场景）

[WatchRecordScanner](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/WatchRecordScanner.php)

用于 inotifywait 等文件系统监听工具触发的增量扫描：
- 文件删除 → 从数据库删除对应歌曲
- 文件新增/修改 → 调用 `IndividualFileHandler::handle()`
- 目录删除 → 删除该目录下所有歌曲
- 目录新增/修改 → 遍历目录中文件逐个处理

---

## 五、音频元数据提取

### 5.1 FileScanner

[FileScanner](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/FileScanner.php#L12-L41)

```
scan(path) → ScanInformation
```

核心步骤：

1. 调用 `getID3->analyze(filePath)` 解析文件，得到原始标签数据 `$raw`
2. 校验：如果 `playtime_seconds` 不存在或 getID3 报错 → 抛出 `RuntimeException`（标记为无效文件）
3. 调用 `getID3->CopyTagsToComments($raw)` 将 ID3v1/v2 等标签统一到 `comments` 数组
4. 调用 `ScanInformation::fromGetId3Info($raw, $filePath)` 构建结构化元数据对象
5. 歌词补充：如果音频文件内嵌歌词为空，尝试读取同名 `.lrc` 文件（通过 `SimpleLrcReader`）

### 5.2 ScanInformation — 元数据结构化

[ScanInformation::fromGetId3Info](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Values/Scanning/ScanInformation.php#L34-L85)

将 getID3 原始数据映射为结构化字段：

| 字段 | 提取逻辑 | 默认值 |
|------|----------|--------|
| `title` | `tags.title` | 文件名（不含扩展名） |
| `albumName` | `tags.album` | `Album::UNKNOWN_NAME`（"Unknown Album"） |
| `artistName` | `tags.artist` | `Artist::UNKNOWN_NAME`（"Unknown Artist"） |
| `albumArtistName` | `tags.albumartist` / `album_artist` / `band`，若标记为合辑则为 "Various Artists" | 空 |
| `track` | `tags.track` / `tracknumber` / `track_number` | 0 |
| `disc` | `tags.discnumber` / `part_of_a_set` | 1 |
| `year` | `tags.year` | null |
| `genre` | `tags.genre` | 空 |
| `lyrics` | `tags.unsynchronised_lyric` / `lyrics`，再尝试同名 .lrc 文件 | 空 |
| `length` | `info.playtime_seconds` | — |
| `cover` | `comments.cover` / `comments.picture` | 空 |
| `path` | 文件真实路径 | — |
| `hash` | `File::hash($path)`（文件内容哈希） | — |
| `mTime` | `File::lastModified($path)` | — |
| `mimeType` | `info.mime_type` | `audio/mpeg` |
| `fileSize` | `File::size($path)` | — |

**标签合并优先级**：`id3v1 < id3v2 < comments < vorbiscomment`（后出现的覆盖先出现的，即 ID3v2 优先于 ID3v1）

**编码修复**：[TagFixer](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Helpers/Encoding/TagFixer.php) 负责修复乱码标签，处理两种情况：
- 非 UTF-8 原始字节 → 尝试从 GB18030/Windows-1252 转换
- 双重乱码（"double mojibake"）→ 逆向还原后重新编码

---

## 六、元数据 → 数据模型映射

### 6.1 SongService::createOrUpdateSongFromScan

[SongService::createOrUpdateSongFromScan](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/SongService.php#L216-L290)

这是元数据到模型映射的核心方法，流程如下：

```
createOrUpdateSongFromScan(info, config, song?) → ?Song
```

#### 步骤 1：判断是否需要处理

- 如果歌曲已存在且文件未修改、非 force 模式 → 直接返回现有 Song
- 新文件或已修改文件 → 继续处理

#### 步骤 2：应用 ignore 规则

- 如果是**新文件** → 忽略 `ignores` 配置，全量写入
- 如果是**已有文件更新** → 从数据中移除 `ignores` 中指定的标签键，保留原有值

#### 步骤 3：解析关联实体

1. **Artist**：`resolveArtist(owner, artistName)`
   - 通过 `ScannerCacheStrategy`（LRU 缓存，上限 1000 条）避免重复查询
   - 调用 `Artist::getOrCreate(user, name)` — 按名称查找，不存在则创建
   - 名称去 BOM、trim 后为空则使用 "Unknown Artist"
   - Plus 许可证下 Artist 绑定 user_id

2. **AlbumArtist**：如果有 `albumArtistName` 则解析，否则 fallback 到 Artist

3. **Album**：`resolveAlbum(albumArtist, albumName)`
   - 同样走缓存
   - 调用 `Album::getOrCreate(artist, name)` — 按 `artist_id + name` 查找，不存在则创建

#### 步骤 4：处理封面

- 如果 Album 已有封面且文件存在 → 跳过
- 如果 `cover` 不在 ignores 列表中：
  - 音频内嵌封面数据 → `AlbumService::storeAlbumCover()` 保存为图片文件
  - 无内嵌封面 → `AlbumService::trySetAlbumCoverFromDirectory()` 在同目录查找 `cover.jpg` / `folder.png` 等文件

#### 步骤 5：构建 Song 数据并持久化

从 `ScanInformation` 的 `toArray()` 输出中移除关联字段（`album`, `artist`, `albumartist`, `cover`），补充外键：

```php
$data['album_id']      = $album->id;
$data['artist_id']     = $artist->id;
$data['is_public']     = $config->makePublic;
$data['album_name']    = $album->name;
$data['artist_name']   = $artist->name;
// 新文件额外设置:
$data['owner_id']      = $config->owner->id;
```

- **新文件**：`Song::query()->create($data)`
- **已有文件**：`$song->update($data)`

#### 步骤 6：同步 Genre

`Song::syncGenres(genre)` 将逗号分隔的流派字符串拆分，查找或创建 `Genre` 记录，然后通过 `songs_genres` 中间表同步关联。

#### 步骤 7：更新 Album 年份

如果 Album 尚无 year，而 Song 有 year → 更新 Album 的 year。

#### 步骤 8：提取文件夹结构

如果 `config.extractFolderStructure` 为 true → 派发 `ExtractSongFolderStructureJob`，为歌曲创建对应的 `Folder` 记录。

---

## 七、关联模型创建细节

### 7.1 Artist::getOrCreate

[Artist::getOrCreate](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Models/Artist.php#L100-L121)

- 去除 BOM 和空白，名称为空时使用 "Unknown Artist" 常量
- Community 许可证下，所有用户共享 Artist（仅按 name 查找）
- Plus 许可证下，Artist 绑定 user_id（按 name + user_id 查找）

### 7.2 Album::getOrCreate

[Album::getOrCreate](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Models/Album.php#L86-L95)

- 按 `artist_id + artist_name + user_id + name` 查找，不存在则创建
- 名称为空时使用 "Unknown Album" 常量

### 7.3 Genre::get

[Genre::get](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Models/Genre.php#L56-L62)

- 按 name 查找，不存在则创建

---

## 八、扫描后处理

扫描完成后触发 `MediaScanCompleted` 事件（[MediaScanCompleted](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Events/MediaScanCompleted.php)），注册的监听器有：

| 监听器 | 作用 |
|--------|------|
| [DeleteNonExistingRecordsPostScan](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Listeners/DeleteNonExistingRecordsPostScan.php) | 删除数据库中路径不在本次扫描结果中的歌曲（已从磁盘删除的文件） |
| [PruneLibrary](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Listeners/PruneLibrary.php) | 清理空 Album 和空 Artist |
| [WriteScanLog](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Listeners/WriteScanLog.php) | 将扫描日志写入 `storage/logs/sync-*.log` |

---

## 九、缓存策略

[ScannerCacheStrategy](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/ScannerCacheStrategy.php) — 带大小限制（默认 1000）的 LRU 内存缓存，用于在单次扫描中缓存 `Artist` 和 `Album` 的查询/创建结果，避免对同一艺术家/专辑重复查库。

并行扫描时每个子进程独立缓存；顺序扫描时共享同一缓存实例。

[ScannerNoCacheStrategy](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/ScannerNoCacheStrategy.php) — 无缓存直通策略，每次都执行回调。

---

## 十、端到端流程图

```
┌───────────────────────────────────────────────────────────────────┐
│                         触发扫描                                  │
│  CLI: koel:scan  │  Web: UpdateMediaPathController               │
│  WatchRecordScanner (inotify)                                    │
└───────────────────────────┬───────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│              DirectoryScanner::scan(mediaPath, config, jobs)      │
│                                                                   │
│  1. gatherFiles(mediaPath)                                        │
│     └─ Finder → 按扩展名过滤 → 只保留音频文件                      │
│                                                                   │
│  2. 选择扫描策略                                                   │
│     ├─ SequentialScanStrategy (jobs=1)                            │
│     └─ ParallelScanStrategy (jobs>1)                              │
│        └─ spawn N 个 koel:scan:chunk 子进程                       │
└───────────────────────────┬───────────────────────────────────────┘
                            │
                   逐文件处理 │
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│            IndividualFileHandler::handle(path, config)             │
│                                                                   │
│  1. 查库: songRepository.findOneByPath(path)                      │
│  2. 跳过判断: 未修改 && !force → SKIPPED                          │
│  3. FileScanner::scan(path) → ScanInformation                     │
│  4. SongService::createOrUpdateSongFromScan(info, config, song)   │
└───────────────────────────┬───────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│                  FileScanner::scan(path)                          │
│                                                                   │
│  1. getID3->analyze() → raw 数据                                  │
│  2. 校验 playtime_seconds 和错误                                   │
│  3. getID3->CopyTagsToComments()                                  │
│  4. ScanInformation::fromGetId3Info() → 结构化元数据               │
│     ├─ 标签合并: id3v1 < id3v2 < comments < vorbiscomment         │
│     ├─ TagFixer::fix() 编码修复 (GB18030/Windows-1252)            │
│     └─ 默认值填充 (Unknown Album/Artist, 文件名作标题等)           │
│  5. SimpleLrcReader 尝试读取同名 .lrc 歌词文件                     │
└───────────────────────────┬───────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│      SongService::createOrUpdateSongFromScan(info, config, song)  │
│                                                                   │
│  1. 判断新文件/已修改 → 继续；未修改 → 返回                        │
│  2. 应用 ignores 规则（仅对已有文件）                               │
│  3. resolveArtist(owner, artistName)                              │
│     └─ Artist::getOrCreate() → 缓存加速                          │
│  4. resolveArtist(owner, albumArtistName) 或 fallback              │
│  5. resolveAlbum(albumArtist, albumName)                          │
│     └─ Album::getOrCreate() → 缓存加速                           │
│  6. 封面处理: 内嵌封面 → 存储; 无封面 → 目录查找 cover/folder 图   │
│  7. 构建 Song 数据 + 外键 (album_id, artist_id, owner_id 等)      │
│  8. Song::create() 或 $song->update()                             │
│  9. syncGenres() 同步流派                                         │
│ 10. 更新 Album.year                                               │
│ 11. 派发 ExtractSongFolderStructureJob                            │
└───────────────────────────┬───────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│                   扫描后处理 (MediaScanCompleted)                  │
│                                                                   │
│  ├─ DeleteNonExistingRecordsPostScan: 删除磁盘已不存在的歌曲       │
│  ├─ PruneLibrary: 清理空 Album 和空 Artist                        │
│  └─ WriteScanLog: 写入同步日志                                    │
└───────────────────────────────────────────────────────────────────┘
```

---

## 关键源文件索引

| 文件 | 职责 |
|------|------|
| [ScanCommand](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Console/Commands/ScanCommand.php) | CLI 扫描入口 |
| [ScanChunkCommand](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Console/Commands/ScanChunkCommand.php) | 并行扫描子进程入口 |
| [UpdateMediaPathController](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Http/Controllers/API/Settings/UpdateMediaPathController.php) | Web 扫描入口 |
| [DirectoryScanner](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/DirectoryScanner.php) | 目录扫描编排器 |
| [Scanner](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/Scanner.php) | 基类：文件收集与系统设置 |
| [SequentialScanStrategy](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/Strategies/SequentialScanStrategy.php) | 顺序扫描策略 |
| [ParallelScanStrategy](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/Strategies/ParallelScanStrategy.php) | 并行扫描策略 |
| [WatchRecordScanner](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/WatchRecordScanner.php) | 文件监听增量扫描 |
| [IndividualFileHandler](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/IndividualFileHandler.php) | 单文件处理调度 |
| [FileScanner](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/FileScanner.php) | getID3 元数据提取 |
| [ScanInformation](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Values/Scanning/ScanInformation.php) | 结构化元数据值对象 |
| [SongService](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/SongService.php) | 元数据 → 模型映射与持久化 |
| [AlbumService](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/AlbumService.php) | 封面存储与目录封面查找 |
| [TagFixer](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Helpers/Encoding/TagFixer.php) | 标签编码修复 |
| [SimpleLrcReader](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/SimpleLrcReader.php) | LRC 歌词文件读取 |
| [ScannerCacheStrategy](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/ScannerCacheStrategy.php) | 扫描期间 Artist/Album 查询缓存 |
| [Artist](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Models/Artist.php) | Artist 模型与 getOrCreate |
| [Album](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Models/Album.php) | Album 模型与 getOrCreate |
| [Song](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Models/Song.php) | Song 模型 |
| [Genre](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Models/Genre.php) | Genre 模型与同步 |
| [ScanConfiguration](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Values/Scanning/ScanConfiguration.php) | 扫描配置值对象 |
| [ScanResult](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Values/Scanning/ScanResult.php) | 单文件扫描结果 |
| [ScanResultCollection](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Values/Scanning/ScanResultCollection.php) | 扫描结果集合 |
| [MediaScanCompleted](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Events/MediaScanCompleted.php) | 扫描完成事件 |
| [DeleteNonExistingRecordsPostScan](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Listeners/DeleteNonExistingRecordsPostScan.php) | 删除无效记录 |
| [WriteScanLog](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Listeners/WriteScanLog.php) | 写入扫描日志 |
| [ExtractSongFolderStructureJob](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Jobs/ExtractSongFolderStructureJob.php) | 提取文件夹结构 |
