# Koel 媒体库扫描流程：磁盘音频文件 → 歌曲数据模型

## 概述

Koel 的媒体库扫描将磁盘上的音频文件解析并映射为数据库中的 `Song`、`Album`、`Artist`、`Genre` 等 Eloquent 模型。整条流水线的核心思路是：**遍历目录 → 逐文件提取元数据 → 解析关联实体 → 持久化到数据库 → 扫描后清理**。

---

## 一、入口：触发扫描

扫描有三个触发入口：

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

### 3. 文件监听入口：WatchRecordScanner

[WatchRecordScanner](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/WatchRecordScanner.php)

用于 inotifywait 等文件系统监听工具触发的增量扫描：
- 文件删除 → 从数据库删除对应歌曲
- 文件新增/修改 → 调用 `IndividualFileHandler::handle()`
- 目录删除 → 删除该目录下所有歌曲
- 目录新增/修改 → 遍历目录中文件逐个处理

---

## 二、扫描配置

[ScanConfiguration](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Values/Scanning/ScanConfiguration.php#L7-L33)

| 字段 | 含义 |
|------|------|
| `owner` | 新歌曲所属用户 |
| `makePublic` | 新歌曲是否公开 |
| `ignores` | 重新扫描时忽略的标签（仅对已存在文件生效） |
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

并行扫描时，每个子进程拥有独立的 `ScannerCacheStrategy` 缓存实例，互不共享。

---

## 四、单文件处理

### 4.1 IndividualFileHandler

[IndividualFileHandler](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/IndividualFileHandler.php#L12-L37)

```
handle(path, config) → ScanResult
```

对每个文件执行以下步骤：

1. **查库**：`songRepository.findOneByPath(path)` 查找已有记录
2. **第一层跳过判断**：如果歌曲已存在、文件未修改（mtime 未变）、且非 force 模式 → 直接返回 `ScanResult::skipped()`，**不调用 getID3，节省解析开销**
3. **扫描文件**：`fileScanner.scan(path)` 提取元数据 → 得到 `ScanInformation`
4. **写入数据库**：`songService.createOrUpdateSongFromScan(info, config, song)` 创建或更新歌曲
5. **异常捕获**：任何 `Throwable` 都被捕获为 `ScanResult::error()`

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
| `mTime` | `get_mtime($path)` | — |
| `mimeType` | `info.mime_type` | `audio/mpeg` |
| `fileSize` | `File::size($path)` | — |

**标签合并优先级**：`id3v1 < id3v2 < comments < vorbiscomment`（后出现的覆盖先出现的，即 ID3v2 优先于 ID3v1，Vorbis Comment 最高）

**编码修复**：[TagFixer](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Helpers/Encoding/TagFixer.php) 负责修复乱码标签，处理两种情况：
- 非 UTF-8 原始字节 → 尝试从 GB18030/Windows-1252 转换
- 双重乱码（"double mojibake"）→ 逆向还原后重新编码

### 5.3 mtime 的两种读取路径

扫描流程中读取文件 mtime 的地方有两处，使用**不同的函数**，失败时的行为完全不同。

#### 路径 A：IndividualFileHandler 预检查 — `File::lastModified()`

[IndividualFileHandler::handle](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/IndividualFileHandler.php#L25)

```php
if (!$config->force && $song && !$song->isFileModified(File::lastModified($path))) {
```

- 使用 `File::lastModified($path)`，这是 Laravel 对 `filemtime()` 的封装，**失败时抛出异常**
- 异常被外层 `catch (Throwable $e)` 捕获 → 返回 `ScanResult::error($path, $e->getMessage())`
- **结果**：读不到 mtime 的已有文件 → 整个文件被标记为 error，不进入解析阶段

#### 路径 B：ScanInformation 构造 — `get_mtime()`

[ScanInformation::fromGetId3Info](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Values/Scanning/ScanInformation.php#L81) → [get_mtime](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Helpers.php#L120-L126)

```php
function get_mtime(string|SplFileInfo $path): int
{
    $path = is_string($path) ? $path : $path->getPathname();
    return rescue(static fn () => File::lastModified($path)) ?? time();
}
```

- 使用 `rescue()` 包裹 `File::lastModified()`，**失败时不抛异常，返回 `time()`（当前时间戳）**
- **结果**：读不到 mtime 的文件 → mTime 为当前时间 → 必然与数据库中存储的旧 mtime 不同 → 文件被判定为"已修改" → 正常解析并更新

#### 两条路径的关系

路径 A 在路径 B 之前执行。对**已有歌曲**而言：

| 场景 | 路径 A（预检查） | 路径 B（元数据阶段） | 最终结果 |
|------|-----------------|---------------------|---------|
| mtime 正常读取 | 成功跳过或继续 | 不涉及（已跳过）或正常读取 | 正常 |
| mtime 读取失败 | `File::lastModified()` 抛异常 → **error** | 不会到达 | 文件标记为 error，不解析 |
| 新文件（库中无记录） | 预检查不适用（`$song` 为 null） | `get_mtime()` → `time()` | 正常解析，mtime 写入当前时间 |
| force 模式 | 预检查被跳过 | `get_mtime()` → `time()` | 正常解析 |

关键结论：`get_mtime()` 的 `time()` 兜底只在路径 A 不适用时（新文件、force 模式）才会生效。对已有歌曲，如果 `File::lastModified()` 失败，文件在路径 A 就已经被标为 error，根本到不了路径 B。

---

## 六、未变更文件的跳过判定

扫描中有**两层跳过检查**，分别位于不同的调用层级，各自有不同的作用：

### 第一层：IndividualFileHandler 中的预检查

[IndividualFileHandler::handle](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/IndividualFileHandler.php#L25-L27)

```php
if (!$config->force && $song && !$song->isFileModified(File::lastModified($path))) {
    return ScanResult::skipped($path);
}
```

- **目的**：在调用 getID3 解析文件之前快速跳过，避免昂贵的元数据解析开销
- **判定依据**：`$song->mtime !== File::lastModified($path)`
- **命中结果**：返回 `ScanResult::skipped($path)`，该路径会被计入 valid 结果，参与后续清理的保护集合

### 第二层：SongService::createOrUpdateSongFromScan 中的内部检查

[SongService::createOrUpdateSongFromScan](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/SongService.php#L221-L229)

```php
$isFileNew = !$song;
$isFileModified = $song && $song->isFileModified($info->mTime);
$isFileNewOrModified = $isFileNew || $isFileModified;

if (!$isFileNewOrModified && !$config->force) {
    return $song;
}
```

- **目的**：对上传场景（`ScansAndStoresSong` trait 调用）等绕过 `IndividualFileHandler` 的调用方提供同样的跳过保护
- **判定依据**：同样是 mtime 比较，但使用 `ScanInformation` 中的 `mTime` 值
- **命中结果**：直接返回现有 Song 对象，但不计入 skipped 结果（上传场景不需要 skipped 统计）

### Song::isFileModified 方法

[Song::isFileModified](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Models/Song.php#L229-L234)

```php
public function isFileModified(int $lastModified): bool
{
    throw_if($this->isEpisode(), new LogicException('Podcast episodes do not have associated files.'));
    return $this->mtime !== $lastModified;
}
```

- 严格的整数全等比较（`!==`）
- 播客节目（`isEpisode()`）会抛出异常，因为播客节目没有本地文件对应

---

## 七、元数据 → 数据模型映射

### 7.1 SongService::createOrUpdateSongFromScan

[SongService::createOrUpdateSongFromScan](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/SongService.php#L216-L290)

这是元数据到模型映射的核心方法，流程如下：

```
createOrUpdateSongFromScan(info, config, song?) → ?Song
```

#### 步骤 1：判断是否需要处理

见上文"第二层跳过检查"。如果 `$song` 参数为 null，方法内部会再通过 `songRepository->findOneByPath($info->path)` 查一次库。

#### 步骤 2：应用 ignore 规则

- 如果是**新文件** → 忽略 `ignores` 配置，全量写入所有元数据
- 如果是**已有文件更新** → 从数据数组中移除 `ignores` 中指定的键，保留数据库中的原值

可忽略的标签包括：`title`, `album`, `artist`, `albumartist`, `track`, `disc`, `year`, `genre`, `lyrics`, `cover`。

#### 步骤 3：解析关联实体

1. **Artist**：`resolveArtist(owner, artistName)`
   - 通过 `ScannerCacheStrategy`（FIFO 缓存，上限 1000 条）避免重复查询
   - 调用 `Artist::getOrCreate(user, name)` — 按名称查找，不存在则创建
   - 名称去 BOM、trim 后为空则使用 "Unknown Artist"

2. **AlbumArtist**：如果有 `albumArtistName` 则单独解析，否则 fallback 到上面的 Artist

3. **Album**：`resolveAlbum(albumArtist, albumName)`
   - 同样走缓存
   - 调用 `Album::getOrCreate(artist, name)` — 按 `artist_id + name` 查找，不存在则创建

#### 步骤 4：处理封面

- 如果 Album 已有封面且磁盘上的封面文件存在 → 跳过
- 如果 `cover` 不在 ignores 列表中：
  - 有内嵌封面数据 → `AlbumService::storeAlbumCover()` 保存为图片文件
  - 无内嵌封面 → `AlbumService::trySetAlbumCoverFromDirectory()` 在同目录查找 `cover.jpg` / `folder.png` 等文件（结果缓存 1 天）

#### 步骤 5：构建 Song 数据并持久化

从 `ScanInformation` 的 `toArray()` 输出中移除关联字段（`album`, `artist`, `albumartist`, `cover`），补充外键和冗余字段：

```php
$data['album_id']      = $album->id;
$data['artist_id']     = $artist->id;
$data['is_public']     = $config->makePublic;
$data['album_name']    = $album->name;   // 冗余字段，加速查询
$data['artist_name']   = $artist->name;  // 冗余字段，加速查询
// 新文件额外设置:
$data['owner_id']      = $config->owner->id;
```

- **新文件**：`Song::query()->create($data)`
- **已有文件**：`$song->update($data)` — 不修改 `owner_id`（保持原所有者）

#### 步骤 6：同步 Genre

`Song::syncGenres(genre)` 将逗号分隔的流派字符串拆分，查找或创建 `Genre` 记录，然后通过 `genre_song` 中间表同步多对多关联。流派是全局共享的（不绑定用户）。

#### 步骤 7：更新 Album 年份

如果 Album 尚无 year，而 Song 有 year → 更新 Album 的 year。

#### 步骤 8：提取文件夹结构

如果 `config.extractFolderStructure` 为 true → 派发 `ExtractSongFolderStructureJob`，为歌曲创建对应的 `Folder` 记录（仅本地存储驱动）。

---

## 八、Owner 与许可模式对 Artist/Album 归属的影响

Artist 和 Album 的归属逻辑受 Koel 许可证模式（Community vs Plus）影响：

### 8.1 Artist 的归属规则

[Artist::getOrCreate](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Models/Artist.php#L100-L121)

```php
$where = ['name' => $name];
if (License::isPlus()) {
    $where['user_id'] = $user->id;
}
```

| 模式 | 查找键 | 归属 | 说明 |
|------|--------|------|------|
| **Community** | `name` | 全局共享 | 所有用户共用同一个 Artist 记录，但记录本身仍有 `user_id`（第一个创建该 Artist 的用户） |
| **Plus** | `name + user_id` | 用户私有 | 每个用户拥有独立的 Artist 集合，同名艺术家在不同用户下是不同的记录 |

无论哪种模式，新创建的 Artist 始终设置 `user_id = $user->id`（即当前扫描的 owner）。

### 8.2 Album 的归属规则

[Album::getOrCreate](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Models/Album.php#L86-L95)

```php
return static::query()->firstOrCreate([
    'artist_id'   => $artist->id,
    'artist_name' => $artist->name,
    'user_id'     => $artist->user_id,
    'name'        => trim($name) ?: self::UNKNOWN_NAME,
]);
```

- 查找键：`artist_id + name`（不是直接按 user_id 查）
- `user_id` 直接从 Artist 继承
- **在 Community 模式下**：由于 Artist 是共享的，Album 也随之共享
- **在 Plus 模式下**：由于 Artist 是用户私有的，Album 间接地也变成用户私有（因为 artist_id 不同）

### 8.3 命名同步

当 Artist 或 Album 的名称被修改时，对应的 Observer 会将名称同步到所有关联的 Song 记录的冗余字段中（`artist_name` / `album_name`）。

- [ArtistObserver::updated](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Observers/ArtistObserver.php#L22-L31) — 名称变更时同步到 songs 和 albums
- [AlbumObserver::updated](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Observers/AlbumObserver.php#L32-L40) — 名称变更时同步到 songs

---

## 九、扫描后清理：删除路径、事件断链与监听器职责

扫描完成后触发 `MediaScanCompleted` 事件，注册了三个监听器（[EventServiceProvider](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Providers/EventServiceProvider.php#L47-L51)）。三者都实现 `ShouldQueue`，按注册顺序派发到队列：

```php
MediaScanCompleted::class => [
    DeleteNonExistingRecordsPostScan::class,
    PruneLibrary::class,
    WriteScanLog::class,
],
```

这三个监听器的设计不是随意排列的 —— 它们之间存在**隐含的依赖链**和**职责分工**，根源在于两条删除路径的机制完全不同。

### 9.1 两条删除路径：查询级 vs 模型级

在讲监听器之前，先搞清楚 Koel 中删除歌曲的两种方式及其本质差异。

#### 路径 A：查询级批量删除 — `$query->delete()`

代表：`DeleteNonExistingRecordsPostScan` 中的 `Song::deleteWhereValueNotIn()`

[SupportsDeleteWhereValueNotIn::deleteAndUnsearch](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Models/Concerns/SupportsDeleteWhereValueNotIn.php#L65-L72)

```php
private static function deleteAndUnsearch(Builder $query): void
{
    if (in_array(Searchable::class, class_uses_recursive(static::class), true)) {
        $query->unsearchable();
    }
    $query->delete();
}
```

**本质**：调用 Eloquent 查询构造器的 `delete()`，底层直接执行 `DELETE FROM songs WHERE ...` SQL 语句。

**特点**：
- 不实例化模型对象
- **不触发**模型的 `deleting` / `deleted` 事件
- **不触发**模型 Observer（`#[ObservedBy]` 注册的也不触发）
- **不触发** `LibraryChanged` 等业务事件
- 不会级联清理关联文件（封面、转码文件等）
- 性能好，适合大批量删除

#### 路径 B：模型级删除 — `$model->delete()` / `Model::destroy()`

代表：`SongService::deleteSongs()` 中的 `Song::destroy($ids)`

[SongService::deleteSongs](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/SongService.php#L192-L214)

```php
public function deleteSongs(array|string $ids): void
{
    $ids = Arr::wrap($ids);
    $songFiles = Song::query()->findMany($ids)->map(SongFileInfo::fromSong(...));
    $transcodeFiles = $this->transcodeRepository->findBySongIds($ids)->map(TranscodeFileInfo::fromTranscode(...));
    if (Song::destroy($ids) === 0) {
        return;
    }
    Dispatcher::dispatch(new DeleteSongFilesJob($songFiles));
    Dispatcher::dispatch(new DeleteTranscodeFilesJob($transcodeFiles));
    event(new LibraryChanged());
}
```

**本质**：先查出每个模型对象，再逐个调用 `$model->delete()`，走完完整的模型生命周期。

**特点**：
- 实例化每个模型对象
- **会触发**模型的 `deleting` / `deleted` 事件
- **会触发**模型 Observer 的对应方法
- 手动 `event(new LibraryChanged())` 触发后续清理
- 会清理关联文件（音频文件、转码文件）
- 性能开销大，适合小批量用户操作

**关键对比表**：

| 特性 | 查询级删除（扫描清理路径） | 模型级删除（手动删歌路径） |
|------|--------------------------|--------------------------|
| 调用方式 | `$query->delete()` | `Model::destroy($ids)` |
| 模型实例化 | 否 | 是 |
| 模型事件 | 不触发 | 触发 |
| Observer | 不触发 | 触发 |
| LibraryChanged 事件 | 不触发 | 手动触发 |
| 关联文件清理 | 不清理 | 清理（音频+转码） |
| 性能 | 高（批量 SQL） | 低（逐条处理） |
| 适用场景 | 扫描后批量清理 | 用户手动删除 |

### 9.2 DeleteNonExistingRecordsPostScan — 歌曲级清理（第一道）

[DeleteNonExistingRecordsPostScan](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Listeners/DeleteNonExistingRecordsPostScan.php#L18-L30)

**职责**：删除数据库中"磁盘上已不存在"的本地歌曲记录。

```php
$paths = $event->results->valid()
    ->map(static fn (ScanResult $result) => $result->path)
    ->merge($this->songRepository->getAllStoredOnCloud()->pluck('path'))
    ->toArray();

Song::deleteWhereValueNotIn($paths, 'path', static function (Builder $builder): Builder {
    return $builder->whereNull('podcast_id');
});
```

**保护条件**（三类歌曲不会被删除）：

| 类别 | 方式 | 原因 |
|------|------|------|
| 本次扫描 valid 的歌曲 | `$event->results->valid()` | 磁盘上存在，正常保留 |
| 云存储歌曲 | `getAllStoredOnCloud()` | 不在本地磁盘上，不参与扫描 |
| 播客节目 | `whereNull('podcast_id')` 排除 | 播客有自己的生命周期，不由媒体扫描管理 |

**删除后留下的空缺**：
- 歌曲记录被删了，但 Album/Artist 记录还在（变成"空"实体）
- 歌曲对应的转码文件还在磁盘上（不会被清理，因为查询级删除不触发文件清理逻辑）
- 不会触发 `LibraryChanged`，所以后续的 prune 不会自动跟进

这就是为什么需要第二个监听器 `PruneLibrary` —— 它补的就是**查询级删除不触发后续清理**这个空缺。

### 9.3 PruneLibrary — 实体级清理（第二道）

[PruneLibrary](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Listeners/PruneLibrary.php) → [LibraryManager::prune](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/LibraryManager.php#L16-L43)

**职责**：清理"没有任何歌曲关联"的空 Album 和空 Artist。

```php
$albumQuery = Album::query()
    ->leftJoin('songs', 'songs.album_id', '=', 'albums.id')
    ->whereNull('songs.album_id');

$artistQuery = Artist::query()
    ->leftJoin('songs', 'songs.artist_id', '=', 'artists.id')
    ->leftJoin('albums', 'albums.artist_id', '=', 'artists.id')
    ->whereNull('songs.artist_id')
    ->whereNull('albums.artist_id');
```

- **Album 清理条件**：LEFT JOIN songs 后 `songs.album_id IS NULL` —— 没有任何歌曲的专辑
- **Artist 清理条件**：LEFT JOIN songs + LEFT JOIN albums 后两者都为 NULL —— 既没有歌曲也没有专辑的艺人
- 全局清理，不区分用户

**删除方式**：`$albumQuery->delete()` / `$artistQuery->delete()` —— 同样是**查询级删除**。

这意味着 Album/Artist 被 prune 删除时：
- **不触发** `AlbumObserver::deleted()` / `ArtistObserver::deleted()`
- **不会清理** Album 封面文件和 Artist 图片文件（Observer 的 deleted 方法负责删文件）
- 留下的是数据库层面的干净，磁盘上可能残留封面文件

### 9.4 PruneLibrary 为什么要监听两个事件

[EventServiceProvider](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Providers/EventServiceProvider.php#L43-L50)

```php
LibraryChanged::class => [
    PruneLibrary::class,
],

MediaScanCompleted::class => [
    DeleteNonExistingRecordsPostScan::class,
    PruneLibrary::class,
    WriteScanLog::class,
],
```

`PruneLibrary` 同时监听 **`LibraryChanged`** 和 **`MediaScanCompleted`** 两个事件，这是因为两条删除路径的事件链不一样：

| 场景 | 删除路径 | 会不会触发 LibraryChanged | PruneLibrary 怎么被唤醒 |
|------|---------|--------------------------|----------------------|
| 用户手动删歌 | 模型级删除（`destroy`） | 会（`SongService::deleteSongs` 手动 dispatch） | 通过 `LibraryChanged` 事件 |
| 扫描后清理 | 查询级删除（`$query->delete()`） | 不会 | 直接挂在 `MediaScanCompleted` 上 |

**一句话总结**：扫描后的批量删除走"查询级"路线，绕过了 `LibraryChanged`，所以 `PruneLibrary` 必须在 `MediaScanCompleted` 上再挂一份，才能补上这段空缺。

### 9.5 WriteScanLog — 日志记录（第三道）

[WriteScanLog](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Listeners/WriteScanLog.php)

**职责**：将扫描结果写入日志文件。与前两个监听器没有依赖关系，纯记录用途。

根据 `config('koel.sync_log_level')` 决定记录全部结果还是仅记录错误，日志文件写入 `storage/logs/sync-YYYYmmdd-His.log`。

### 9.6 三个监听器的依赖关系图

```
MediaScanCompleted 事件
  │
  ├─ [1] DeleteNonExistingRecordsPostScan  (ShouldQueue)
  │     │
  │     ├─ 职责: 歌曲级删除 — 移除磁盘已不存在的 Song 记录
  │     ├─ 方式: 查询级批量删除 ($query->delete())
  │     ├─ 保护: valid结果 + 云存储歌曲 + 播客节目
  │     ├─ 不触发: 模型事件 / Observer / LibraryChanged
  │     │
  │     └─ 留下的空缺: 孤立 Album/Artist、残留转码文件
  │                          ↓
  ├─ [2] PruneLibrary                        (ShouldQueue)
  │     │
  │     ├─ 职责: 实体级清理 — 移除空 Album 和空 Artist
  │     ├─ 方式: 查询级删除 ($query->delete())
  │     ├─ 不触发: AlbumObserver/ArtistObserver 的 deleted
  │     ├─ 留下的空缺: 残留封面/图片文件
  │     │
  │     └─ 也监听 LibraryChanged 事件 ← 手动删歌走这条
  │
  └─ [3] WriteScanLog                        (ShouldQueue)
        职责: 写入扫描日志（独立，无依赖）
```

**执行顺序的重要性**：`PruneLibrary` 必须在 `DeleteNonExistingRecordsPostScan` 之后执行，否则孤立体尚未产生，prune 不到东西。由于两者都是 `ShouldQueue` 且按注册顺序派发，在单 worker 的同步队列或 FIFO 队列下顺序有保障；多 worker 并发场景下理论上可能乱序，但 prune 是幂等的 —— 早跑了也不会错删，只是可能啥也没删到。

### 9.7 两条删除路径的完整清理链对比

```
【用户手动删歌】
SongService::deleteSongs()
  → Song::destroy($ids)                    模型级删除
  → 触发模型事件 / Observer
  → DeleteSongFilesJob                     删音频文件
  → DeleteTranscodeFilesJob                删转码文件
  → event(new LibraryChanged())            ← 触发后续清理
      → PruneLibrary
          → LibraryManager::prune()        查询级删除空 Album/Artist
          → 不触发 Observer deleted        ← 封面文件残留

【扫描后清理】
MediaScanCompleted 事件
  → DeleteNonExistingRecordsPostScan
      → Song::deleteWhereValueNotIn()      查询级批量删除
      → 不触发模型事件 / Observer
      → 不触发 LibraryChanged              ← 事件链在这里断了
      → 不删关联文件                        ← 转码文件残留
  → PruneLibrary                           ← 直接挂在这里补空缺
      → LibraryManager::prune()            查询级删除空 Album/Artist
      → 不触发 Observer deleted            ← 封面文件残留
  → WriteScanLog
```

---

## 十、缓存策略：FIFO 淘汰

### 10.1 ScannerCacheStrategy

[ScannerCacheStrategy](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/ScannerCacheStrategy.php#L9-L34)

```php
public function remember(string $key, Closure $callback): mixed
{
    if ($this->cache->has($key)) {
        return $this->cache->get($key);
    }

    if ($this->cache->count() >= $this->maxCacheSize) {
        $this->cache->shift();
    }

    $result = $callback();
    $this->cache->put($key, $result);

    return $result;
}
```

**淘汰方式：FIFO（First-In, First-Out）**，不是 LRU。

| 特性 | 说明 |
|------|------|
| 数据结构 | Laravel `Collection`（关联数组，保持插入顺序） |
| 命中行为 | 直接返回值，**不改变元素位置** |
| 淘汰触发 | 当缓存数量达到 `maxCacheSize`（默认 1000）时 |
| 淘汰方式 | `shift()` — 移除**最早插入**的元素（队首） |
| 插入位置 | `put()` — 追加到**末尾**（队尾） |

由于命中不会将元素移到队尾，频繁访问的"热门"实体如果很早就进入缓存，也可能在缓存满时被最早淘汰掉。这是一个简单的 FIFO 实现，而非 LRU。

### 10.2 缓存作用范围

- 顺序扫描：整个扫描过程共享同一个缓存实例
- 并行扫描：每个子进程有独立的缓存实例（进程间不共享）
- 单元测试：使用 `ScannerNoCacheStrategy` 直通策略，保证测试一致性

缓存绑定在 [AppServiceProvider](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Providers/AppServiceProvider.php#L78-L81) 中注册。

---

## 十一、端到端流程图

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
│     └─ Finder → 扩展名过滤 → 只保留音频文件                        │
│                                                                   │
│  2. 选择扫描策略                                                   │
│     ├─ SequentialScanStrategy (jobs=1)  共享缓存                  │
│     └─ ParallelScanStrategy (jobs>1)  每进程独立缓存              │
│        └─ spawn N 个 koel:scan:chunk 子进程                       │
└───────────────────────────┬───────────────────────────────────────┘
                            │
                   逐文件处理 │
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│            IndividualFileHandler::handle(path, config)            │
│                                                                   │
│  1. 查库: songRepository.findOneByPath(path)                      │
│  2. 第一层跳过: 未修改 && !force → SKIPPED（省 getID3 开销）       │
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
│  6. get_mtime() 兜底: 读不到 mtime → 用 time() 当当前时间          │
└───────────────────────────┬───────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│      SongService::createOrUpdateSongFromScan(info, config, song)  │
│                                                                   │
│  1. 第二层跳过检查（为上传场景等非 IndividualFileHandler 调用方）   │
│  2. 应用 ignores 规则（仅对已有文件）                               │
│  3. resolveArtist(owner, artistName) → FIFO 缓存 1000             │
│     └─ Artist::getOrCreate()  Community共享 / Plus用户私有        │
│  4. resolveArtist(owner, albumArtistName) 或 fallback             │
│  5. resolveAlbum(albumArtist, albumName)                          │
│     └─ Album::getOrCreate()  user_id 继承自 Artist                │
│  6. 封面处理: 内嵌封面 → 存储; 无封面 → 目录查找 cover/folder 图   │
│  7. 构建 Song 数据 + 外键 + 冗余字段 (album_name, artist_name)     │
│  8. Song::create() 或 $song->update()（owner 只在创建时设置）      │
│  9. syncGenres() 同步流派（全局共享 Genre）                        │
│ 10. 更新 Album.year                                               │
│ 11. 派发 ExtractSongFolderStructureJob（本地存储）                 │
└───────────────────────────┬───────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│                   扫描后处理 (MediaScanCompleted)                  │
│                                                                   │
│  ├─ DeleteNonExistingRecordsPostScan                              │
│  │    保留: valid结果 + 云存储歌曲 + 播客节目                      │
│  │    删除: path 不在保留列表中 且 非播客的歌曲                     │
│  │    实现: deleteWhereValueNotIn（分 chunk 防 IN 超限）           │
│  │                                                                 │
│  ├─ PruneLibrary → LibraryManager::prune()                        │
│  │    删除: 无歌曲的 Album                                         │
│  │    删除: 无歌曲且无专辑的 Artist                                │
│  │                                                                 │
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
| [IndividualFileHandler](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/IndividualFileHandler.php) | 单文件处理调度（第一层跳过） |
| [FileScanner](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/FileScanner.php) | getID3 元数据提取 |
| [ScanInformation](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Values/Scanning/ScanInformation.php) | 结构化元数据值对象 |
| [SongService](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/SongService.php) | 元数据 → 模型映射与持久化 |
| [AlbumService](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/AlbumService.php) | 封面存储与目录封面查找 |
| [LibraryManager](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/LibraryManager.php) | 库清理（空专辑/空艺人） |
| [TagFixer](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Helpers/Encoding/TagFixer.php) | 标签编码修复 |
| [SimpleLrcReader](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/SimpleLrcReader.php) | LRC 歌词文件读取 |
| [ScannerCacheStrategy](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Services/Scanners/ScannerCacheStrategy.php) | FIFO 缓存（Artist/Album 查询） |
| [Artist](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Models/Artist.php) | Artist 模型与 getOrCreate |
| [ArtistObserver](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Observers/ArtistObserver.php) | Artist 名称同步观察者 |
| [Album](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Models/Album.php) | Album 模型与 getOrCreate |
| [AlbumObserver](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Observers/AlbumObserver.php) | Album 名称同步观察者 |
| [Song](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Models/Song.php) | Song 模型 |
| [Genre](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Models/Genre.php) | Genre 模型与同步 |
| [SupportsDeleteWhereValueNotIn](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Models/Concerns/SupportsDeleteWhereValueNotIn.php) | 批量删除分 chunk trait |
| [ScanConfiguration](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Values/Scanning/ScanConfiguration.php) | 扫描配置值对象 |
| [ScanResult](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Values/Scanning/ScanResult.php) | 单文件扫描结果 |
| [ScanResultCollection](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Values/Scanning/ScanResultCollection.php) | 扫描结果集合 |
| [MediaScanCompleted](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Events/MediaScanCompleted.php) | 扫描完成事件 |
| [LibraryChanged](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Events/LibraryChanged.php) | 库变更事件（手动删歌触发 PruneLibrary） |
| [EventServiceProvider](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Providers/EventServiceProvider.php) | 事件-监听器注册 |
| [DeleteNonExistingRecordsPostScan](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Listeners/DeleteNonExistingRecordsPostScan.php) | 删除无效记录 |
| [PruneLibrary](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Listeners/PruneLibrary.php) | 清理空专辑/空艺人 |
| [WriteScanLog](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Listeners/WriteScanLog.php) | 写入扫描日志 |
| [ExtractSongFolderStructureJob](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Jobs/ExtractSongFolderStructureJob.php) | 提取文件夹结构 |
| [Helpers.php](file:///d:/fz/0601-1/solo-dogfeeding/code/45-koel/app/Helpers.php) | 辅助函数（get_mtime 等） |
