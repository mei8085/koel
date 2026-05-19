# Koel 音频播放请求代码实现分析

## 1. 路由入口与请求生命周期

### 1.1 路由定义

**文件**: `routes/web.base.php:47-61`

```php
Route::middleware('audio.auth')->group(static function (): void {
    Route::get('play/{song}/{transcode?}', PlayController::class)->name('song.play');
    // ... 其他音频相关路由
});
```

**路由参数说明**:
- `{song}`: 歌曲 UUID，通过 Laravel 路由模型绑定自动注入 `Song` 模型
- `{transcode?}`: 可选布尔参数，用于强制转码（移动端场景）

### 1.2 请求生命周期概览

```
HTTP Request
    ↓
[bootstrap/app.php] 中间件注册
    ↓
[AudioAuthenticate] audio.auth 中间件
    ↓
[Route Model Binding] Song $song 注入
    ↓
[PlayController::__invoke] 控制器入口
    ↓
[SongPolicy::access] 权限校验
    ↓
[Streamer::resolveAdapter] 适配器选择
    ├─ 转码路径 → TranscodingStreamerAdapter
    └─ 直出路径 → Local/S3/SFTP/Dropbox 适配器
        ↓
[StreamerAdapter::stream] 实际流式输出
```

---

## 2. 权限校验代码实现

### 2.1 第一层: audio.auth 中间件

**文件**: `app/Http/Middleware/AudioAuthenticate.php:9-16`

```php
class AudioAuthenticate
{
    public function handle(Request $request, Closure $next)
    {
        abort_unless($request->user()?->tokenCan('audio'), Response::HTTP_UNAUTHORIZED);
        return $next($request);
    }
}
```

**代码分析**:
- 使用 `$request->user()` 获取当前认证用户
- `tokenCan('audio')` 检查 Sanctum Token 是否具备 `audio` 能力
- 校验失败直接返回 `401 Unauthorized`
- 中间件别名注册在 `bootstrap/app.php:41`

### 2.2 第二层: 控制器授权

**文件**: `app/Http/Controllers/PlayController.php:18-34`

```php
public function __invoke(Authenticatable $user, SongPlayRequest $request, Song $song, ?bool $transcode = null)
{
    $this->authorize('access', $song);  // 触发 SongPolicy
    
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

**代码分析**:
- `$this->authorize('access', $song)` 调用 Laravel 授权系统
- `SongPlayRequest` 是一个空的 Request 类，仅用于类型约束
- `RequestedStreamingConfig` 是只读值对象 (readonly class)
- 强制转码时使用用户偏好的转码质量，否则为 null（由 Streamer 自动判断）

### 2.3 第三层: SongPolicy 业务逻辑

**文件**: `app/Policies/SongPolicy.php:10-35`

```php
class SongPolicy
{
    public function access(User $user, Song $song): bool
    {
        return License::isCommunity() || $song->accessibleBy($user);
    }

    public function download(User $user, Song $song): bool
    {
        return $this->access($user, $song);
    }
}
```

**文件**: `app/Models/Song.php:132-144`

```php
public function accessibleBy(User $user): bool
{
    if ($this->isEpisode()) {
        return $user->subscribedToPodcast($this->podcast);
    }
    return $this->is_public || $this->ownedBy($user);
}

public function ownedBy(User $user): bool
{
    return $this->owner->id === $user->id;
}
```

**分支条件详解**:

| 分支条件 | 代码位置 | 说明 |
|---------|---------|------|
| `License::isCommunity()` | SongPolicy:14 | Community 版本所有用户均可访问所有歌曲 |
| `$this->isEpisode()` | Song:134 | 播客节目走独立权限逻辑 |
| `$user->subscribedToPodcast()` | Song:135 | 检查用户是否订阅了该播客 |
| `$this->is_public` | Song:138 | Plus 版本中歌曲可设置为公开 |
| `$this->ownedBy($user)` | Song:138 | 歌曲所有者始终可访问 |

---

## 3. 直出与转码路径切换逻辑

### 3.1 Streamer 适配器解析核心

**文件**: `app/Services/Streamer/Streamer.php:17-86`

```php
class Streamer
{
    public function __construct(
        private readonly Song $song,
        private ?StreamerAdapter $adapter = null,
        private readonly ?RequestedStreamingConfig $config = null,
    ) {
        $this->adapter ??= $this->resolveAdapter();
    }

    private function resolveAdapter(): StreamerAdapter
    {
        throw_unless($this->song->storage->supported(), KoelPlusRequiredException::class);

        if ($this->song->isEpisode()) {
            return app(PodcastStreamerAdapter::class);
        }

        if ($this->config?->transcode || self::shouldTranscode($this->song)) {
            return app(TranscodingStreamerAdapter::class);
        }

        return match ($this->song->storage) {
            SongStorageType::LOCAL => app(LocalStreamerAdapter::class),
            SongStorageType::SFTP => app(SftpStreamerAdapter::class),
            SongStorageType::S3, SongStorageType::S3_LAMBDA => app(S3CompatibleStreamerAdapter::class),
            SongStorageType::DROPBOX => app(DropboxStreamerAdapter::class),
        };
    }
```

**适配器选择优先级** (从高到低):
1. 存储类型不支持 → 抛出 `KoelPlusRequiredException`
2. 是播客节目 → `PodcastStreamerAdapter`
3. 强制转码或需要转码 → `TranscodingStreamerAdapter`
4. 按存储类型分派到对应适配器

### 3.2 转码判断逻辑

**文件**: `app/Services/Streamer/Streamer.php:64-86`

```php
private static function shouldTranscode(Song $song): bool
{
    if ($song->isEpisode()) {
        return false;
    }

    if (!self::hasValidFfmpegInstallation()) {
        return false;
    }

    if (
        in_array($song->mime_type, ['audio/flac', 'audio/x-flac'], true) 
        && config('koel.streaming.transcode_flac')
    ) {
        return true;
    }

    return in_array($song->mime_type, config('koel.streaming.transcode_required_mime_types', []), true);
}

private static function hasValidFfmpegInstallation(): bool
{
    return app()->runningUnitTests() || is_executable(config('koel.streaming.ffmpeg_path'));
}
```

**转码决策真值表**:

| 条件 | 结果 | 说明 |
|-----|------|------|
| 是播客节目 | ❌ 不转码 | 播客始终直出 |
| FFmpeg 不可用 | ❌ 不转码 | 无法转码 |
| 强制转码 (`config?->transcode`) | ✅ 转码 | URL 参数控制 |
| FLAC + `transcode_flac=true` | ✅ 转码 | 默认开启 |
| MIME 在转码列表中 | ✅ 转码 | 浏览器不支持的格式 |
| 其他情况 | ❌ 不转码 | 直接流式输出 |

### 3.3 本地流式适配器选择

**文件**: `app/Providers/StreamerServiceProvider.php:11-22`

```php
public function register(): void
{
    $this->app->bind(LocalStreamerAdapter::class, function (): LocalStreamerAdapter {
        return match (config('koel.streaming.method')) {
            'x-sendfile' => $this->app->make(XSendFileStreamerAdapter::class),
            'x-accel-redirect' => $this->app->make(XAccelRedirectStreamerAdapter::class),
            default => $this->app->make(PhpStreamerAdapter::class),
        };
    });
}
```

**三种适配器对比**:

| 适配器 | 代码文件 | 实现方式 | 性能 |
|-------|---------|---------|------|
| PhpStreamerAdapter | `PhpStreamerAdapter.php:9-20` | PHP 读取文件 + `exit()` | 一般 |
| XSendFileStreamerAdapter | `XSendFileStreamerAdapter.php:8-24` | Apache `X-Sendfile` 头 | 高 |
| XAccelRedirectStreamerAdapter | `XAccelRedirectStreamerAdapter.php:9-31` | Nginx `X-Accel-Redirect` 头 | 最高 |

### 3.4 转码流程实现

**文件**: `app/Services/Transcoding/LocalTranscodingStrategy.php:10-44`

```php
public function getTranscodeLocation(Song $song, int $bitRate): string
{
    $transcode = $this->findTranscodeBySongAndBitRate($song, $bitRate);

    if ($transcode?->isValid()) {
        return $transcode->location;
    }

    if ($transcode) {
        File::delete($transcode->location);
    }

    $destination = artifact_path(sprintf('transcodes/%d/%s.m4a', $bitRate, Ulid::generate()));
    $this->transcoder->transcode($song->path, $destination, $bitRate);

    $this->createOrUpdateTranscode(
        $song,
        $destination,
        $bitRate,
        File::hash($destination),
        File::size($destination),
    );

    return $destination;
}
```

**转码缓存机制**:
1. 通过 `song_id + bit_rate` 查询 `transcodes` 表
2. `isValid()` 校验文件 hash 是否匹配
3. 无效则删除旧文件，重新转码
4. 转码后存储 hash 和 file_size 用于后续校验

**文件**: `app/Services/Transcoding/Transcoder.php:19-46`

```php
public function transcode(string $source, string $destination, int $bitRate): void
{
    setlocale(LC_CTYPE, 'en_US.UTF-8');
    File::ensureDirectoryExists(dirname($destination));

    $process = $this->transcodeTimeout ? Process::timeout($this->transcodeTimeout) : Process::forever();

    $result = $process->run([
        $this->ffmpegPath,
        '-nostdin',
        '-i', $source,
        '-vn',
        '-c:a', 'aac',
        '-b:a', "{$bitRate}k",
        '-threads', '0',
        '-movflags', '+faststart',
        '-y',
        $destination,
    ]);

    throw_if($result->failed(), new TranscodingFailedException($result->errorOutput()));
}
```

**FFmpeg 参数详解**:
- `-nostdin`: 禁用标准输入，防止交互式等待
- `-vn`: 去除视频流
- `-c:a aac`: AAC 音频编码器
- `-b:a {bitRate}k`: 目标码率
- `-threads 0`: 自动线程数
- `-movflags +faststart`: moov atom 前置，优化流式播放
- `-y`: 覆盖已存在文件

---

## 4. Range Request 处理实现

### 4.1 核心 Trait

**文件**: `app/Services/Streamer/Adapters/Concerns/StreamsLocalPath.php:18-43`

```php
trait StreamsLocalPath
{
    private function streamLocalPath(string $path): void
    {
        try {
            $rangeHeader = get_request_header('Range');

            // Safari 兼容: "bytes=0-1" 探测请求转为 "bytes=0-"
            $rangeHeader = $rangeHeader === 'bytes=0-1' ? 'bytes=0-' : $rangeHeader;

            $rangeSet = RangeSet::createFromHeader($rangeHeader);
            $resource = new FileResource($path, File::mimeType($path));
            (new ResourceServlet($resource))->sendResource($rangeSet);
        } catch (InvalidRangeHeaderException) {
            abort(Response::HTTP_BAD_REQUEST);
        } catch (UnsatisfiableRangeException) {
            abort(Response::HTTP_REQUESTED_RANGE_NOT_SATISFIABLE);
        } catch (NonExistentFileException) {
            abort(Response::HTTP_NOT_FOUND);
        } catch (UnreadableFileException) {
            abort(Response::HTTP_INTERNAL_SERVER_ERROR);
        } catch (SendFileFailureException $e) {
            abort_unless(headers_sent(), Response::HTTP_INTERNAL_SERVER_ERROR);
            echo "An error occurred while attempting to send the requested resource: {$e->getMessage()}";
        }
    }
}
```

### 4.2 关键代码分析

**Safari 兼容处理**:
```php
$rangeHeader = $rangeHeader === 'bytes=0-1' ? 'bytes=0-' : $rangeHeader;
```
- Safari 会先发送 `bytes=0-1` 探测请求确认服务器支持 Range
- 如果严格按范围返回 2 字节数据，Safari 可能中断播放
- 转换为 `bytes=0-` 返回整个文件，保证流式播放正常

**异常处理分支**:

| 异常 | HTTP 状态 | 触发场景 |
|-----|----------|---------|
| `InvalidRangeHeaderException` | 400 | Range 头格式错误，如 `bytes=abc` |
| `UnsatisfiableRangeException` | 416 | 请求范围超出文件大小，如文件 1MB 请求 `bytes=2000000-` |
| `NonExistentFileException` | 404 | 文件路径不存在 |
| `UnreadableFileException` | 500 | 文件权限不足或 IO 错误 |
| `SendFileFailureException` | 特殊处理 | 发送过程中出错，检查头是否已发送 |

**SendFileFailureException 特殊处理**:
```php
catch (SendFileFailureException $e) {
    abort_unless(headers_sent(), Response::HTTP_INTERNAL_SERVER_ERROR);
    echo "An error occurred...";
}
```
- 如果 HTTP 头已发送（`headers_sent() === true`），无法再发送 500 状态码
- 直接输出错误信息到响应体

### 4.3 使用的第三方库

Koel 使用 `daverandom/resume` 库处理 Range 请求，核心类:
- `RangeSet`: 解析和表示 Range 头
- `FileResource`: 包装本地文件资源
- `ResourceServlet`: 实际发送响应，处理 206 Partial Content

---

## 5. 中断播放与边界处理

### 5.1 PhpStreamerAdapter 的 exit() 调用

**文件**: `app/Services/Streamer/Adapters/PhpStreamerAdapter.php:13-20`

```php
public function stream(Song $song, ?RequestedStreamingConfig $config = null): void
{
    $this->streamLocalPath($song->storage_metadata->getPath());
    exit();  // 关键: 显式退出
}
```

**为什么需要 exit()**:
1. Laravel 的后中间件 (post-middleware) 可能尝试发送额外的 HTTP 头
2. 终止中间件 (terminable middleware) 可能执行清理操作
3. 流式输出过程中 HTTP 头已经发送，后续任何 `header()` 调用都会导致 "headers already sent" 错误
4. `exit()` 确保 PHP 立即终止，不执行后续逻辑

**代价**:
- 终止中间件不会执行（如日志记录、统计等）
- Laravel 的响应后处理逻辑被跳过

### 5.2 X-Sendfile / X-Accel-Redirect 的 exit()

**文件**: `app/Services/Streamer/Adapters/XSendFileStreamerAdapter.php:13-24`

```php
public function stream(Song $song, ?RequestedStreamingConfig $config = null): never
{
    $path = $song->storage_metadata->getPath();
    $contentType = 'audio/' . pathinfo($path, PATHINFO_EXTENSION);
    header("X-Sendfile: $path");
    header("Content-Type: $contentType");
    header('Content-Disposition: inline; filename="' . basename($path) . '"');
    exit();
}
```

**返回类型 `never`**:
- PHP 8.1+ 新增类型，表示函数永远不会返回
- 静态分析工具可以识别此函数后的代码是死代码

### 5.3 转码过程中断处理

**转码流程中的中断点**:
1. **FFmpeg 执行中**: `Process::run()` 是同步阻塞调用
2. **客户端断开**: PHP 会收到 `SIGPIPE` 信号，但 FFmpeg 子进程可能继续运行
3. **超时**: 受 `transcode_timeout` 配置限制（默认 300 秒）

**潜在问题**:
- 用户在转码过程中频繁切换歌曲会导致多个 FFmpeg 进程累积
- 转码完成后即使客户端已断开，文件仍会被缓存（下次可用）

### 5.4 错误报告抑制

**文件**: `app/Services/Streamer/Streamer.php:47-54`

```php
public function stream(): mixed
{
    @error_reporting(0);  // 禁用所有错误报告
    return $this->adapter->stream($this->song, $this->config);
}
```

**原因**:
- 流式输出是二进制数据，任何 PHP 警告、通知都会破坏数据流
- 例如 `fread()` 遇到网络中断会产生警告，混入音频数据导致播放失败
- `@` 错误控制符抑制 `error_reporting(0)` 本身可能产生的错误

### 5.5 云存储重定向的中断处理

**文件**: `app/Services/Streamer/Adapters/S3CompatibleStreamerAdapter.php:17-22`

```php
public function stream(Song $song, ?RequestedStreamingConfig $config = null): Redirector|RedirectResponse
{
    $this->storage->assertSupported();
    return redirect($this->storage->getPresignedUrl($song->storage_metadata->getPath()));
}
```

**特点**:
- 返回 `302 Found` 重定向到 S3 预签名 URL
- 后续的流式传输由浏览器直接与 S3 交互
- Koel 不感知播放中断，也无法处理 Range 请求
- 预签名 URL 有过期时间（通常 15-60 分钟）

---

## 6. 配置与环境变量

### 6.1 streaming 配置块

**文件**: `config/koel.php:40-122`

| 配置项 | 环境变量 | 默认值 | 说明 |
|-------|---------|--------|------|
| `bitrate` | `TRANSCODE_BIT_RATE` | 128 | 转码目标码率 (kbps) |
| `method` | `STREAMING_METHOD` | null | 流式方法: x-sendfile/x-accel-redirect/php |
| `ffmpeg_path` | `FFMPEG_PATH` | 自动探测 | FFmpeg 可执行路径 |
| `transcode_flac` | `TRANSCODE_FLAC` | true | 是否转码 FLAC |
| `transcode_timeout` | `TRANSCODE_TIMEOUT` | 300 | 转码超时 (秒) |

### 6.2 需要转码的 MIME 类型

**文件**: `config/koel.php:91-121`

```php
'transcode_required_mime_types' => [
    'audio/vorbis',          // Ogg Vorbis
    'audio/x-flac',          // FLAC (alternate)
    'audio/amr',             // AMR
    'audio/ac3',             // Dolby AC-3
    'audio/dts',             // DTS
    'audio/vnd.rn-realaudio', // RealAudio
    'audio/x-ms-wma',        // WMA
    'video/x-ms-asf',        // WMA detected as ASF
    'audio/basic',           // µ-law
    'audio/vnd.wave',        // WAV
    'audio/aiff',            // AIFF
    'audio/x-aiff',          // AIFF (alternate)
    'audio/x-m4a',           // ALAC (not AAC)
    'audio/x-matroska',      // MKA
    'audio/x-ape',           // APE
    'audio/tta',             // TTA
    'audio/x-wavpack',       // WavPack
    'audio/x-optimfrog',     // OptimFROG
    'audio/x-shorten',       // Shorten
    'audio/x-lpac',          // LPAC
    'audio/x-dsd',           // DSD
    'audio/x-speex',         // Speex
    'audio/x-dss',           // DSS
    'audio/x-audible',       // Audible
    'audio/x-twinvq',        // TwinVQ
    'audio/vqf',             // TwinVQ (alternate)
    'audio/x-musepack',      // Musepack
    'audio/x-monkeys-audio', // APE (alternate)
    'audio/x-voc',           // Creative VOC
],
```

---

## 7. 完整调用链代码路径

### 7.1 直出路径 (本地 MP3)

```
1. routes/web.base.php:48 → GET play/{song}
2. app/Http/Middleware/AudioAuthenticate.php:13 → tokenCan('audio')
3. app/Http/Controllers/PlayController.php:20 → authorize('access', $song)
4. app/Policies/SongPolicy.php:14 → License::isCommunity() || accessibleBy()
5. app/Services/Streamer/Streamer.php:39 → LocalStreamerAdapter
6. app/Providers/StreamerServiceProvider.php:19 → PhpStreamerAdapter
7. app/Services/Streamer/Adapters/PhpStreamerAdapter.php:15 → streamLocalPath()
8. app/Services/Streamer/Adapters/Concerns/StreamsLocalPath.php:28 → RangeSet::createFromHeader()
9. daverandom/resume → 发送 206 Partial Content
10. app/Services/Streamer/Adapters/PhpStreamerAdapter.php:19 → exit()
```

### 7.2 转码路径 (本地 FLAC)

```
1. routes/web.base.php:48 → GET play/{song}
2. app/Http/Middleware/AudioAuthenticate.php:13 → tokenCan('audio')
3. app/Http/Controllers/PlayController.php:20 → authorize('access', $song)
4. app/Policies/SongPolicy.php:14 → License::isCommunity() || accessibleBy()
5. app/Services/Streamer/Streamer.php:35 → shouldTranscode() = true (FLAC)
6. app/Services/Streamer/Adapters/TranscodingStreamerAdapter.php:16 → stream()
7. app/Services/Transcoding/TranscodeStrategyFactory.php:12 → LocalTranscodingStrategy
8. app/Services/Transcoding/LocalTranscodingStrategy.php:14 → 查找缓存
9. app/Services/Transcoding/LocalTranscodingStrategy.php:27 → FFmpeg 转码
10. app/Services/Transcoding/LocalTranscodingStrategy.php:30 → 保存转码记录
11. app/Services/Streamer/Adapters/TranscodingStreamerAdapter.php:32 → streamLocalPath()
12. app/Services/Streamer/Adapters/Concerns/StreamsLocalPath.php:28 → RangeSet::createFromHeader()
13. daverandom/resume → 发送 206 Partial Content
```

### 7.3 S3 直出路径

```
1. routes/web.base.php:48 → GET play/{song}
2. app/Http/Middleware/AudioAuthenticate.php:13 → tokenCan('audio')
3. app/Http/Controllers/PlayController.php:20 → authorize('access', $song)
4. app/Policies/SongPolicy.php:14 → License::isCommunity() || accessibleBy()
5. app/Services/Streamer/Streamer.php:42 → S3CompatibleStreamerAdapter
6. app/Services/Streamer/Adapters/S3CompatibleStreamerAdapter.php:21 → getPresignedUrl()
7. Laravel redirect() → 302 Found 到 S3 URL
8. 浏览器直接从 S3 下载播放
```

---

## 8. 关键设计决策总结

| 决策 | 代码位置 | 原因 | 代价 |
|-----|---------|------|------|
| 流式前禁用错误报告 | Streamer.php:51 | 防止 PHP 警告破坏二进制流 | 调试困难 |
| PhpStreamer 调用 exit() | PhpStreamerAdapter.php:19 | 防止 Laravel 后中间件发送额外头 | 终止中间件不执行 |
| Safari Range 头转换 | StreamsLocalPath.php:26 | 兼容 Safari 的探测请求 | 与标准略有差异 |
| 转码文件永久缓存 | LocalTranscodingStrategy.php:16 | 节省 CPU，提升体验 | 占用磁盘空间 |
| FLAC 默认转码 | config/koel.php:44 | 浏览器对 FLAC 支持不一致 | 损失音质，增加延迟 |
