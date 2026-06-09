# 歌曲上传、识别与入库流程解析

## 概述

一首歌曲从用户上传到最终出现在专辑下，经历了 **上传接收 → 文件存储 → 重复检测 → 元数据解析 → 艺术家解析 → 专辑解析 → 封面处理 → 数据库写入** 共 8 个核心阶段。整个流程支持同步执行和异步队列两种模式。

---

## 整体流程图

```
用户上传文件
     │
     ▼
[UploadSongController]  上传入口控制器
     │  验证权限、验证文件格式
     ▼
文件移入临时目录 (artifact_path/tmp/{ULID}/)
     │
     ▼
[HandleSongUploadJob]  分发任务（同步/队列）
     │
     ▼
[UploadService::handleUpload]  上传业务主流程
     │
     ├─► [SongStorage::storeUploadedFile]  存储到最终位置
     │
     ├─► [DuplicateUploadService::detectDuplicate]  重复检测
     │
     └─► [ScansAndStoresSong::scanAndStore]  扫描并入库
              │
              ├─► [FileScanner::scan]  ID3 元数据解析
              │     └─ getID3 库分析标签
              │
              └─► [SongService::createOrUpdateSongFromScan]  创建/更新歌曲
                        │
                        ├─ resolveArtist → Artist::getOrCreate
                        │
                        ├─ resolveAlbum  → Album::getOrCreate
                        │
                        ├─ 专辑封面处理
                        │
                        └─ Song::create / Song::update  数据库写入
```

---

## 阶段一：上传入口

### 路由

路由定义在 [routes/api.base.php](file:///d:/fz/0508-2/solo-dogfeeding/code/113-koel/routes/api.base.php#L160-L160)：

```php
Route::post('upload', UploadSongController::class);
```

### 控制器

[UploadSongController](file:///d:/fz/0508-2/solo-dogfeeding/code/113-koel/app/Http/Controllers/API/Upload/UploadSongController.php) 是上传入口，采用单动作控制器（Invokable）模式。

**核心步骤：**

1. **权限校验**：`$this->authorize('upload', User::class)` — 验证用户是否有上传权限
2. **存储驱动校验**：`$storage->assertSupported()` — 确认当前存储驱动可用
3. **请求验证**：使用 [UploadSongRequest](file:///d:/fz/0508-2/solo-dogfeeding/code/113-koel/app/Http/Requests/API/Upload/UploadSongRequest.php) 验证文件格式
   - 规则：`required | file | SupportedAudioFile`
   - `SupportedAudioFile` 规则检查是否为受支持的音频格式
4. **文件暂存**：将上传文件移动到临时目录
   ```php
   $file = $request->file->move(
       artifact_path('tmp/' . Ulid::generate()),
       $request->file->getClientOriginalName(),
   );
   ```
   - 临时路径：`artifact_path/tmp/{ULID}/{原始文件名}`
   - 使用 ULID 生成唯一子目录，避免文件名冲突

5. **任务分发**：通过 `Dispatcher::dispatch(new HandleSongUploadJob(...))` 分发处理任务
   - 同步模式：直接执行，返回 `Song` 模型
   - 异步模式：进入队列，返回 `204 No Content`

6. **响应处理**：
   - 同步成功：返回 `SongUploadResponse`（含歌曲和专辑信息）
   - 重复上传：返回 `409 Conflict` + `DuplicateUploadResource`
   - 媒体路径未设置：返回 `403 Forbidden`
   - 上传失败：返回 `400 Bad Request`

---

## 阶段二：任务处理 Job

[HandleSongUploadJob](file:///d:/fz/0508-2/solo-dogfeeding/code/113-koel/app/Jobs/HandleSongUploadJob.php) 封装了上传处理逻辑，可同步执行也可入队异步执行。

**handle 方法流程：**

1. 调用 `UploadService::handleUpload()` 处理上传
2. 通过 `SongRepository` 和 `AlbumRepository` 加载完整关联数据
3. 广播 `SongUploadResponse` 事件（前端可实时收到通知）
4. 返回 `Song` 模型

---

## 阶段三：上传业务主流程

[UploadService::handleUpload()](file:///d:/fz/0508-2/solo-dogfeeding/code/113-koel/app/Services/Upload/UploadService.php#L28-L54) 是上传的业务核心。

### 核心流程

```php
public function handleUpload(string $filePath, User $uploader): Song
{
    $uploadReference = $this->storage->storeUploadedFile($filePath, $uploader);

    try {
        $this->duplicateUploadService->detectDuplicate(
            $uploadReference->localPath,
            $uploadReference,
            $uploader
        );

        return $this->scanAndStore(...);
    } catch (DuplicateSongUploadException $e) {
        throw $e;
    } catch (Throwable $error) {
        $this->storage->undoUpload($uploadReference);
        throw SongUploadFailedException::make($error);
    } finally {
        if ($this->storage instanceof MustDeleteTemporaryLocalFileAfterUpload) {
            File::delete($uploadReference->localPath);
        }
    }
}
```

**关键设计：**

- **事务安全性**：出错时调用 `undoUpload()` 回滚存储操作
- **临时文件清理**：实现了 `MustDeleteTemporaryLocalFileAfterUpload` 接口的存储驱动（如云存储），在最后清理本地临时文件
- **错误包装**：捕获所有异常并包装为 `SongUploadFailedException`

### UploadReference 值对象

[UploadReference](file:///d:/fz/0508-2/solo-dogfeeding/code/113-koel/app/Values/UploadReference.php) 封装存储位置信息：

- `location`：存储位置标识（存入数据库）
- `localPath`：本地文件路径（用于扫描和清理）

对于本地存储，两者相同；对于云存储，`location` 是云端路径，`localPath` 是本地临时文件。

---

## 阶段四：文件存储

### 存储抽象

[SongStorage](file:///d:/fz/0508-2/solo-dogfeeding/code/113-koel/app/Services/SongStorages/SongStorage.php) 是抽象基类，定义了存储驱动的统一接口：

| 方法 | 作用 |
|------|------|
| `storeUploadedFile()` | 存储上传文件，返回 UploadReference |
| `undoUpload()` | 回滚上传（删除已存储的文件） |
| `delete()` | 删除歌曲文件 |
| `getLocalPath()` | 获取文件的本地路径 |
| `getStorageType()` | 返回存储类型枚举 |
| `testSetup()` | 测试存储配置是否可用 |

**实现的驱动：**
- `LocalStorage` — 本地存储
- `S3CompatibleStorage` — S3 兼容存储
- `S3LambdaStorage` — S3 + Lambda 转码
- `DropboxStorage` — Dropbox
- `SftpStorage` — SFTP

### 本地存储实现

[LocalStorage](file:///d:/fz/0508-2/solo-dogfeeding/code/113-koel/app/Services/SongStorages/LocalStorage.php) 是默认存储驱动。

**存储路径规则：**

- 基础路径：`{media_path}/__KOEL_UPLOADS__${user_id}__/`
- 文件名策略：
  - 无重名：使用原始文件名
  - 有重名：`{6位hash}_{原始文件名}`

**示例：**
```
/music/__KOEL_UPLOADS__$1__/Hello.mp3
/music/__KOEL_UPLOADS__$1__/a1b2c3_Hello.mp3
```

---

## 阶段五：重复检测

[DuplicateUploadService::detectDuplicate()](file:///d:/fz/0508-2/solo-dogfeeding/code/113-koel/app/Services/Upload/DuplicateUploadService.php#L38-L59) 负责检测重复上传。

### 检测逻辑

1. 检查用户偏好 `detectDuplicateUploads`，未开启则跳过
2. 计算文件哈希：`File::hash($filePath)`
3. 通过 `SongRepository::findByHash()` 查询是否存在相同哈希的歌曲
4. 若存在重复：
   - 创建 `DuplicateUpload` 记录
   - 抛出 `DuplicateSongUploadException` 异常

### 重复上传后续处理

用户可以选择：
- `keep()` — 保留（继续入库，删除重复记录）
- `discard()` — 丢弃（删除文件和重复记录）

---

## 阶段六：元数据解析（扫描）

### FileScanner

[FileScanner::scan()](file:///d:/fz/0508-2/solo-dogfeeding/code/113-koel/app/Services/Scanners/FileScanner.php#L19-L41) 使用 `getID3` 库解析音频文件的元数据。

### 解析流程

1. `getID3->analyze($filePath)` — 分析文件，获取原始信息
2. 校验播放时长，空文件抛出异常
3. `getID3->CopyTagsToComments($raw)` — 将标签复制到 comments 数组
4. `ScanInformation::fromGetId3Info()` — 转换为结构化对象
5. 尝试读取同目录 `.lrc` 歌词文件作为补充

### ScanInformation

[ScanInformation](file:///d:/fz/0508-2/solo-dogfeeding/code/113-koel/app/Values/Scanning/ScanInformation.php) 是扫描结果的值对象。

#### 标签提取优先级

标签按以下顺序合并（后者覆盖前者）：
```
id3v1 → id3v2 → comments → vorbiscomment
```

#### 标签字段映射

| 字段 | 标签键名（按优先级） | 默认值 |
|------|---------------------|--------|
| title | `title` | 文件名 |
| albumName | `album` | "Unknown Album" |
| artistName | `artist` | "Unknown Artist" |
| albumArtistName | `albumartist`, `album_artist`, `band` | null |
| track | `track`, `tracknumber`, `track_number` | 0 |
| disc | `discnumber`, `part_of_a_set` | 1 |
| year | `year` | null |
| genre | `genre` | null |
| lyrics | `unsynchronised_lyric`, `unsychronised_lyric`, `unsyncedlyrics`, `lyrics` | null |

#### 专辑艺人特殊规则

- 如果歌曲标记为合辑（`part_of_a_compilation`）且没有专辑艺人，使用 "Various Artists"
- 编码修复：使用 [TagFixer](file:///d:/fz/0508-2/solo-dogfeeding/code/113-koel/app/Helpers/Encoding/TagFixer.php) 处理乱码问题
- HTML 实体解码：`html_entity_decode()`

#### 其他提取字段

- `length` — 播放时长（秒）
- `cover` — 封面图片数据
- `hash` — 文件哈希（用于重复检测）
- `mTime` — 文件修改时间（用于增量扫描）
- `mimeType` — MIME 类型
- `fileSize` — 文件大小（字节）

---

## 阶段七：入库（扫描 & 存储）

`scanAndStore` 方法定义在 [ScansAndStoresSong](file:///d:/fz/0508-2/solo-dogfeeding/code/113-koel/app/Services/Concerns/ScansAndStoresSong.php) trait 中，被 `UploadService` 和 `DuplicateUploadService` 复用。

### ScanConfiguration

[ScanConfiguration](file:///d:/fz/0508-2/solo-dogfeeding/code/113-koel/app/Values/Scanning/ScanConfiguration.php) 是扫描配置值对象：

| 参数 | 说明 | 上传时默认值 |
|------|------|-------------|
| `owner` | 歌曲所属用户 | 上传者 |
| `makePublic` | 是否设为公开 | 用户偏好 `makeUploadsPublic` |
| `ignores` | 忽略的标签（仅更新时生效） | [] |
| `force` | 强制扫描（即使文件未变化） | false |
| `extractFolderStructure` | 是否提取文件夹结构 | 视存储驱动而定 |

### createOrUpdateSongFromScan

[SongService::createOrUpdateSongFromScan()](file:///d:/fz/0508-2/solo-dogfeeding/code/113-koel/app/Services/SongService.php#L216-L290) 是入库的核心方法。

#### 步骤 1：判断是否需要处理

```php
$song ??= $this->songRepository->findOneByPath($info->path);
$isFileNew = !$song;
$isFileModified = $song && $song->isFileModified($info->mTime);
$isFileNewOrModified = $isFileNew || $isFileModified;

if (!$isFileNewOrModified && !$config->force) {
    return $song;
}
```

- 新文件：必须处理
- 已存在且未修改：直接返回（除非 `force=true`）
- 已存在但已修改：更新

#### 步骤 2：解析艺术家

```php
$artist = $this->resolveArtist($config->owner, $data['artist']);
$albumArtist = $data['albumartist']
    ? $this->resolveArtist($config->owner, $data['albumartist'])
    : $artist;
```

`resolveArtist()` 通过缓存 + `Artist::getOrCreate()` 实现：
- 有专辑艺人：使用专辑艺人
- 无专辑艺人：与歌曲艺人相同

**Artist::getOrCreate()** — 查找或创建艺术家：
- Community 版：按名称全局共享
- Plus 版：按用户 + 名称隔离

#### 步骤 3：解析专辑

```php
$album = $this->resolveAlbum($albumArtist, $data['album']);
```

`resolveAlbum()` 同样通过缓存 + `Album::getOrCreate()` 实现。

**Album::getOrCreate()** — 查找或创建专辑：
- 唯一键：`artist_id + name`
- 无专辑名时使用 "Unknown Album"
- 自动关联 `user_id` 和 `artist_name`

#### 步骤 4：专辑封面处理

```php
if (!$hasCover && !in_array('cover', $config->ignores, true)) {
    $coverData = $data['cover']['data'] ?? null;

    if ($coverData) {
        rescue(fn () => $this->albumService->storeAlbumCover($album, $coverData), report: true);
    } else {
        $this->albumService->trySetAlbumCoverFromDirectory($album, dirname($data['path']));
    }
}
```

封面来源优先级：
1. ID3 标签内嵌封面
2. 同目录下的封面文件（`cover.jpg`, `folder.png` 等）
3. 无封面（留空）

目录封面扫描结果会缓存一天，避免重复扫描。

#### 步骤 5：写入歌曲数据

清理无关字段后，组装歌曲数据：
```php
$data = [
    'album_id' => $album->id,
    'artist_id' => $artist->id,
    'is_public' => $config->makePublic,
    'album_name' => $album->name,
    'artist_name' => $artist->name,
    // ... 其他字段：title, length, track, disc, year, mtime, hash, etc.
];
```

**新歌曲**：设置 `owner_id` 后 `Song::create($data)`
**已存在**：`$song->update($data)`

#### 步骤 6：流派同步

```php
if ($genre !== $song->genre) {
    $song->syncGenres($genre);
}
```

流派使用多对多关系，通过 `genre_song` 中间表关联。

#### 步骤 7：专辑年份补充

```php
if (!$album->year && $song->year) {
    $album->update(['year' => $song->year]);
}
```

如果专辑没有年份但歌曲有，从歌曲继承年份。

#### 步骤 8：文件夹结构提取

```php
if ($config->extractFolderStructure) {
    Dispatcher::dispatch(new ExtractSongFolderStructureJob($song));
}
```

本地存储驱动会触发文件夹结构提取任务，用于媒体浏览器的目录视图。

---

## 阶段八：数据库模型

### Song 模型

[Song](file:///d:/fz/0508-2/solo-dogfeeding/code/113-koel/app/Models/Song.php) 是歌曲的 Eloquent 模型。

**关键字段：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | UUID | 主键 |
| `path` | string | 文件存储路径/位置 |
| `storage` | SongStorageType | 存储类型枚举 |
| `owner_id` | int | 所属用户 ID |
| `artist_id` | string | 艺术家 UUID |
| `album_id` | string | 专辑 UUID |
| `title` | string | 歌曲标题 |
| `artist_name` | string | 艺术家名称（冗余） |
| `album_name` | string | 专辑名称（冗余） |
| `length` | float | 播放时长（秒） |
| `track` | int | 曲目号 |
| `disc` | int | 碟号 |
| `year` | int | 年份 |
| `lyrics` | text | 歌词 |
| `mtime` | int | 文件修改时间戳 |
| `hash` | string? | 文件哈希 |
| `mime_type` | string? | MIME 类型 |
| `file_size` | int? | 文件大小（字节） |
| `is_public` | bool | 是否公开 |

**设计要点：**
- 冗余存储 `artist_name` 和 `album_name`，避免关联查询
- 使用 UUID 作为主键
- 自动预加载 `album`, `artist`, `album.artist`, `genres`, `owner` 等关联

### Album 模型

[Album](file:///d:/fz/0508-2/solo-dogfeeding/code/113-koel/app/Models/Album.php) 是专辑模型。

**关键字段：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | ULID | 主键 |
| `artist_id` | string | 专辑艺术家 UUID |
| `user_id` | int | 所属用户 ID |
| `name` | string | 专辑名称 |
| `cover` | string | 封面文件名 |
| `thumbnail` | string? | 缩略图文件名 |
| `year` | int? | 年份 |

**唯一约束：** `artist_id + name` 组合唯一（同一艺术家下专辑名不重复）

### Artist 模型

艺术家模型，类似结构。

---

## 核心设计模式

### 1. 值对象 (Value Object)

使用 `readonly class` 实现不可变值对象：
- `ScanInformation` — 扫描结果
- `ScanConfiguration` — 扫描配置
- `UploadReference` — 上传引用
- `SongFileInfo` — 歌曲文件信息

### 2. Trait 复用

`ScansAndStoresSong` trait 被 `UploadService` 和 `DuplicateUploadService` 共用，提取了"扫描 + 入库"的通用逻辑。

### 3. 存储抽象

`SongStorage` 抽象类定义统一接口，多种驱动实现，符合开闭原则。

### 4. 缓存策略

艺术家和专辑解析使用 `CacheStrategy` 缓存，避免重复查询数据库。

### 5. 同步/异步透明切换

通过 `Dispatcher::dispatch()` 封装，同一个 Job 可同步执行也可入队异步，调用方无需感知。

---

## 异常处理

| 异常 | 触发场景 | HTTP 状态码 |
|------|---------|------------|
| `DuplicateSongUploadException` | 检测到重复上传 | 409 Conflict |
| `MediaPathNotSetException` | 媒体路径未配置 | 403 Forbidden |
| `SongUploadFailedException` | 上传处理失败 | 400 Bad Request |
| `KoelPlusRequiredException` | 存储驱动需要 Koel Plus | - |

---

## 总结

歌曲上传入库是一个典型的 **管道式流程**：文件经过层层处理（验证 → 存储 → 去重 → 解析 → 关联 → 写入），每一层都有明确的职责和边界。关键设计亮点包括：

1. **存储驱动抽象** — 灵活支持本地、S3、Dropbox 等多种存储
2. **值对象封装** — 扫描结果、配置等使用不可变对象传递
3. **事务安全** — 出错时回滚文件存储
4. **缓存优化** — 艺术家/专辑解析使用缓存，减少数据库查询
5. **同步异步统一** — 同一 Job 可同步可异步，架构灵活
