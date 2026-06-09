# 音乐权限分析

## 一、权限体系分层

Koel 的音乐访问权限分为四层，自上而下逐层拦截：

| 层级 | 机制 | 作用范围 | 失败表现 |
|------|------|---------|---------|
| L1 | 认证中间件 | 路由入口 | 401 Unauthorized |
| L2 | Policy 授权 | 控制器方法（单对象） | 403 Forbidden |
| L3 | 查询作用域 | Repository 列表查询 | 数据被静默过滤 |
| L4 | 协作级联 | 播放列表协作时 | 主动变更歌曲属性 |

许可证类型是核心分支条件：社区版（Community）所有歌曲全局可见，高级版（Plus）才有精细化的歌曲级访问控制。

---

## 二、第一层：认证中间件

### 2.1 `auth` 中间件

**类**：`App\Http\Middleware\Authenticate`

用于 API 路由组，检查用户是否通过 Sanctum 认证且令牌拥有 `*` 能力。

```php
if ($request->user()?->tokenCan('*')) {
    return $next($request);
}
```

**适用路由**：所有 `api.base.php` 中 `Route::middleware('auth')` 分组内的接口，包括歌曲列表、播放列表 CRUD、收藏等。

### 2.2 `audio.auth` 中间件

**类**：`App\Http\Middleware\AudioAuthenticate`

用于 Web 音频流路由组，检查令牌是否拥有 `audio` 能力。

```php
abort_unless($request->user()?->tokenCan('audio'), Response::HTTP_UNAUTHORIZED);
```

**适用路由**：
- 歌曲播放流 `GET /play/{song}/{transcode?}`
- 电台流 `GET /radio/stream/{radioStation}`
- 下载接口（配置开启时）

**两套中间件的区别**：API 数据操作需要 `*` 能力，音频流只需要 `audio` 能力，支持颁发仅可播放不可修改的受限令牌。

### 2.3 `RestrictPlusFeatures` 中间件

**类**：`App\Http\Middleware\RestrictPlusFeatures`

全局附加在 `api` 和 `web` 中间件组上。检查控制器方法的 `#[RequiresPlus]` 注解，社区版下拒绝访问。

---

## 三、第二层：Policy 授权（单曲粒度）

### 3.1 SongPolicy

**类**：`App\Policies\SongPolicy`

| 方法 | 社区版逻辑 | 高级版逻辑 |
|------|-----------|-----------|
| `access` | `true` | `$song->accessibleBy($user)` |
| `own` | `$song->ownedBy($user)` | `$song->ownedBy($user)` |
| `edit` | 需 `MANAGE_SONGS` 权限 | `$song->accessibleBy($user)` |
| `delete` | 需 `MANAGE_SONGS` 权限 | `$song->ownedBy($user)` |
| `download` | 同 `access` | 同 `access` |

### 3.2 `Song::accessibleBy()` 方法

**类**：`App\Models\Song`

```php
public function accessibleBy(User $user): bool
{
    if ($this->isEpisode()) {
        return $user->subscribedToPodcast($this->podcast);
    }

    return $this->is_public || $this->ownedBy($user);
}
```

**判定规则**（高级版，满足任一即可）：
1. 播客剧集：用户已订阅该播客
2. 普通歌曲：`is_public = true` 或 `ownedBy($user)`

**注意**：`accessibleBy()` 方法不检查组织隔离，也不检查 `includePublicMedia` 用户偏好。公开歌曲对所有登录用户可见。

### 3.3 `Song::ownedBy()` 方法

```php
public function ownedBy(User $user): bool
{
    return $this->owner->id === $user->id;
}
```

### 3.4 歌曲编辑授权边界

高级版下，`SongPolicy@edit` 的判定条件等于 `accessibleBy()`，即：

- 所有者可以编辑自己的歌曲
- 任何公开歌曲也可以被编辑
- 删除权限严格限制为所有者（`ownedBy`）

### 3.5 PlaylistPolicy

**类**：`App\Policies\PlaylistPolicy`

| 方法 | 判定逻辑 |
|------|---------|
| `access` | `own() \|\| hasCollaborator()` |
| `own` | `$playlist->ownedBy($user)` |
| `edit` / `delete` | 仅所有者 |
| `collaborate` | 所有者或协作者 |
| `download` | 同 `access` |
| `inviteCollaborators` | Plus 版 + 所有者 + 非智能播放列表 |

### 3.6 播放列表归属与协作

**类**：`App\Models\Playlist`

```php
public function ownedBy(User $user): bool
{
    return $this->owner->is($user);
}
```

播放列表通过多对多关系 `users` 关联用户，`pivot.role` 区分 `owner` 和 `collaborator`。`owner` 属性通过 `users` 集合中 `role=owner` 的记录动态获取。

**协作者判定**（`App\Models\Concerns\Playlists\ManagesCollaborators`）：

```php
public function hasCollaborator(User $collaborator): bool
{
    return $this->collaborators->contains($collaborator->is(...));
}
```

---

## 四、第三层：查询作用域（列表粒度）

### 4.1 `SongBuilder::accessible()` 作用域

**类**：`App\Builders\SongBuilder`

社区版下直接返回，不过滤任何数据。高级版下应用以下过滤逻辑：

**播客剧集**：必须属于用户已订阅的播客。

**普通歌曲**：
- 若 `$user->preferences->includePublicMedia = false`：仅返回用户自己拥有的歌曲
- 若 `$user->preferences->includePublicMedia = true`：返回用户自己的歌曲 + 同组织其他用户的公开歌曲

公开歌曲的 SQL 过滤条件：
```sql
songs.is_public = true
AND owner.organization_id = user.organization_id
AND songs.owner_id <> user.id
```

**关键差异**：`SongBuilder::accessible()` 比 `Song::accessibleBy()` 多了两层限制：
1. 组织隔离：公开歌曲仅限同组织可见
2. 用户偏好：`includePublicMedia` 关闭时不显示任何公开歌曲

### 4.2 `withUserContext()` 标准查询入口

```php
public function withUserContext(
    bool $includeFavoriteStatus = true,
    bool $favoritesOnly = false,
    bool $includePlayCount = true,
): self {
    return $this
        ->accessible()
        ->when($includeFavoriteStatus, ...)
        ->when($includePlayCount, ...);
}
```

`SongRepository` 的所有查询方法均通过 `withUserContext()` 执行，确保返回的歌曲在当前用户可见范围内。

### 4.3 受影响的查询方法

`App\Repositories\SongRepository` 中以下方法均应用 `accessible()` 过滤：

- `paginate()` — 歌曲列表分页
- `getOne()` — 单曲详情
- `getMany()` — 批量获取（按 ID）
- `getByAlbum()` — 专辑下的歌曲
- `getByArtist()` — 艺术家下的歌曲
- `getByPlaylist()` — 播放列表中的歌曲
- `getFavorites()` — 收藏的歌曲
- `getRecentlyPlayed()` / `getMostPlayed()` / `getLeastPlayed()` — 播放统计
- `getByGenre()` — 流派下的歌曲
- `search()` — 搜索结果
- `getUnderPaths()` — 按文件夹路径获取

### 4.4 播放列表可见性过滤

**类**：`App\Repositories\PlaylistRepository`

```php
private function accessibleByUser(User $user): BelongsToMany
{
    return License::isCommunity() ? $user->ownedPlaylists() : $user->playlists();
}
```

- 社区版：只列出用户自己拥有的播放列表
- 高级版：列出用户关联的所有播放列表（拥有 + 协作）

---

## 五、第四层：协作播放列表与歌曲公开化

### 5.1 `makePlaylistContentPublic()` 方法

**类**：`App\Services\Playlist\PlaylistService`

```php
public function makePlaylistContentPublic(Playlist $playlist): void
{
    $playlist->playables()->where('is_public', false)->update(['is_public' => true]);
}
```

将播放列表中所有私有歌曲批量设为公开。

### 5.2 触发时机

1. **添加歌曲时**：在 `addPlayablesToPlaylist()` 中，若播放列表已有协作者，则立即将新加入的歌曲设为公开。

2. **新协作者加入时**：通过 `NewPlaylistCollaboratorJoined` 事件触发 `MakePlaylistSongsPublic` 监听器，异步将播放列表内所有歌曲设为公开。

### 5.3 协作判定条件

```php
public function isPlaylistCollaborative(Playlist $playlist): bool
{
    return once(
        static fn () => !$playlist->is_smart && LicenseFacade::isPlus() && $playlist->collaborators->isNotEmpty(),
    );
}
```

需同时满足：非智能播放列表、Plus 许可证、至少有一个协作者。

---

## 六、三条关键路径的拦截流程

### 6.1 路径一：单曲播放

**入口**：`App\Http\Controllers\PlayController`

```
GET /play/{song}/{transcode?}
  ↓
① audio.auth 中间件
   检查：令牌有 `audio` 能力
   失败 → 401
  ↓
② 路由模型绑定加载 Song
  ↓
③ $this->authorize('access', $song)
   → SongPolicy@access
   → Song::accessibleBy()
   检查：is_public || ownedBy（无组织隔离）
   失败 → 403
  ↓
④ Streamer 流式输出（无额外权限检查）
```

### 6.2 路径二：歌曲列表查询

**入口**：`App\Http\Controllers\API\SongController@index`

```
GET /api/songs
  ↓
① auth 中间件
   检查：令牌有 `*` 能力
   失败 → 401
  ↓
② SongRepository::paginate(scopedUser: $user)
   → Song::query(user: $user)->withUserContext()
   → SongBuilder::accessible()
   检查：组织隔离 + includePublicMedia 偏好
   结果：SQL 层过滤，不可见数据不返回
```

### 6.3 路径三：播放列表歌曲加载

**入口**：`App\Http\Controllers\API\PlaylistSongController@index`

```
GET /api/playlists/{playlist}/songs
  ↓
① auth 中间件 → 401
  ↓
② 路由模型绑定加载 Playlist
  ↓
③ 第一层：播放列表权限检查
   智能播放列表 → authorize('own', $playlist)
   普通播放列表 → authorize('collaborate', $playlist)
   → PlaylistPolicy 检查用户角色
   失败 → 403
  ↓
④ 第二层：歌曲可见性过滤
   SongRepository::getByPlaylist($playlist, $user)
   → Song::query(user: $user)->withUserContext()
   → SongBuilder::accessible()
   检查：组织隔离 + includePublicMedia 偏好
   结果：不可见歌曲被静默过滤
```

**注**：播放列表权限和歌曲可见性是两层独立检查。用户能访问播放列表，不代表能听到列表中的每首歌。

---

## 七、两套访问判定的差异汇总

| 对比维度 | `Song::accessibleBy()`（模型方法） | `SongBuilder::accessible()`（查询作用域） |
|---------|-----------------------------------|-------------------------------------------|
| 调用层 | Policy（单对象授权） | Repository（列表查询） |
| 组织隔离 | 无 | 有（同组织内可见） |
| `includePublicMedia` 偏好 | 不影响 | 影响（关闭则不显示公开歌） |
| 跨组织公开歌曲 | 可访问 | 不可见 |
| 播客剧集检查 | 有 | 有 |

### 不一致性的具体表现

1. **跨组织公开歌曲**：直接播放可听（走 `accessibleBy()`），但在歌曲列表、搜索结果、播放列表视图中不可见（走 `accessible()`）。

2. **`includePublicMedia` 偏好**：仅影响列表展示，不影响单曲播放和详情接口的授权判断。

3. **歌曲详情接口的双重检查**：`SongController@show` 先调用 `authorize('access', $song)`（`accessibleBy()`，无组织过滤），再调用 `songRepository->getOne()`（`accessible()`，有组织过滤）。跨组织公开歌曲在详情接口会因第二步查询返回 404。

---

## 八、关键文件

| 文件 | 作用 |
|------|------|
| `app/Models/Song.php` | 歌曲模型，`accessibleBy()` / `ownedBy()` |
| `app/Builders/SongBuilder.php` | 歌曲查询构造器，`accessible()` / `withUserContext()` |
| `app/Policies/SongPolicy.php` | 歌曲授权策略 |
| `app/Policies/PlaylistPolicy.php` | 播放列表授权策略 |
| `app/Repositories/SongRepository.php` | 歌曲仓储，所有列表查询入口 |
| `app/Repositories/PlaylistRepository.php` | 播放列表仓储 |
| `app/Http/Controllers/PlayController.php` | 歌曲播放控制器 |
| `app/Http/Controllers/API/SongController.php` | 歌曲 API 控制器 |
| `app/Http/Controllers/API/PlaylistSongController.php` | 播放列表歌曲控制器 |
| `app/Services/Playlist/PlaylistService.php` | 播放列表服务，`makePlaylistContentPublic()` |
| `app/Http/Middleware/Authenticate.php` | API 认证中间件 |
| `app/Http/Middleware/AudioAuthenticate.php` | 音频认证中间件 |
| `app/Listeners/MakePlaylistSongsPublic.php` | 协作者加入事件监听器 |
