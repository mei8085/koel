# 音乐访问权限分析

## 概述

Koel 的音乐访问权限采用**多层防御**架构，从路由中间件到模型策略再到查询作用域，逐层对用户能否听到某首歌、访问某个播放列表进行拦截。权限体系的核心变量是**许可证类型**（Community 社区版 vs Plus 高级版）——社区版所有歌曲全局可见，高级版则引入了歌曲公私属性、播放列表协作、组织隔离等精细化控制。

---

## 第一层：路由中间件（认证与令牌能力）

所有音乐相关请求在进入控制器前，必须先通过认证中间件。

### 1.1 `auth` 中间件

**文件**：[Authenticate.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Http/Middleware/Authenticate.php)

用于 API 路由组（`routes/api.base.php` 第 107 行），检查用户是否已通过 Sanctum 认证且令牌拥有 `*` 能力（全能权限）。

```php
if ($request->user()?->tokenCan('*')) {
    return $next($request);
}
```

**拦截点**：所有 API 数据接口（歌曲列表、播放列表 CRUD、收藏等）。
**失败响应**：AJAX/JSON 请求返回 401，否则重定向到首页。

### 1.2 `audio.auth` 中间件

**文件**：[AudioAuthenticate.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Http/Middleware/AudioAuthenticate.php)

用于 Web 音频流路由组（`routes/web.base.php` 第 47 行），检查令牌是否拥有 `audio` 能力。

```php
abort_unless($request->user()?->tokenCan('audio'), Response::HTTP_UNAUTHORIZED);
```

**拦截点**：
- 歌曲播放流：`GET /play/{song}/{transcode?}` — [PlayController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Http/Controllers/PlayController.php)
- 电台流：`GET /radio/stream/{radioStation}`
- 下载接口（如启用）

> 为什么分成 `auth` 和 `audio.auth` 两套？
> `auth` 用于数据 API（需要 `*` 能力），`audio.auth` 用于音频流（仅需 `audio` 能力）。这样可以颁发只能听音乐、不能修改数据的受限令牌。

### 1.3 `RestrictPlusFeatures` 中间件

**文件**：[RestrictPlusFeatures.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Http/Middleware/RestrictPlusFeatures.php)

全局附加在 `api` 和 `web` 中间件组上（`bootstrap/app.php` 第 28-38 行）。它检查控制器方法上的 `#[RequiresPlus]` 注解，社区版下直接拒绝访问。

---

## 第二层：控制器授权（Policy 门）

通过中间件后，控制器方法会调用 `$this->authorize()` 进行精细化权限检查，委托给对应的 Policy 类。

### 2.1 歌曲访问策略 — SongPolicy

**文件**：[SongPolicy.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Policies/SongPolicy.php)

| 方法 | 判定逻辑 |
|------|---------|
| `access` | 社区版 → `true`；高级版 → `$song->accessibleBy($user)` |
| `own` | `$song->ownedBy($user)` |
| `edit` | 社区版 → 需 `MANAGE_SONGS` 权限；高级版 → `accessibleBy` |
| `delete` | 社区版 → 需 `MANAGE_SONGS` 权限；高级版 → `ownedBy` |
| `download` | 同 `access` |

**歌曲模型层面的判定** — [Song.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Models/Song.php#L132-L144)：

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

**高级版歌曲可访问条件**（满足任一即可）：
1. 歌曲是公开的（`is_public = true`）且其所有者与当前用户同属一个组织
2. 歌曲的所有者就是当前用户
3. 播客剧集：用户已订阅该播客

### 2.2 播放列表访问策略 — PlaylistPolicy

**文件**：[PlaylistPolicy.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Policies/PlaylistPolicy.php)

| 方法 | 判定逻辑 |
|------|---------|
| `access` | `own($user, $playlist) || $playlist->hasCollaborator($user)` |
| `own` | `$playlist->ownedBy($user)` |
| `edit` / `delete` | 仅所有者 |
| `collaborate` | 所有者或协作者 |
| `download` | 同 `access` |
| `inviteCollaborators` | Plus 版 + 所有者 + 非智能播放列表 |

**播放列表归属判定** — [Playlist.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Models/Playlist.php#L119-L122)：

```php
public function ownedBy(User $user): bool
{
    return $this->owner->is($user);
}
```

播放列表通过多对多关系 `users` 关联用户，`pivot.role` 区分 `owner` 和 `collaborator`。`owner` 属性通过 `users` 集合中 `role=owner` 的记录动态获取。

**协作者判定** — [ManagesCollaborators.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Models/Concerns/Playlists/ManagesCollaborators.php#L16-L19)：

```php
public function hasCollaborator(User $collaborator): bool
{
    return $this->collaborators->contains($collaborator->is(...));
}
```

---

## 第三层：Repository 查询作用域（数据可见性过滤）

Policy 只检查单个对象的访问权限。当查询列表时（如获取所有歌曲、获取播放列表中的歌曲），由 **Repository + Builder 作用域** 在 SQL 层面过滤不可见数据，这是最核心的批量数据权限控制层。

### 3.1 SongBuilder 的 `accessible()` 作用域

**文件**：[SongBuilder.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Builders/SongBuilder.php#L66-L108)

```php
public function accessible(): self
{
    if (License::isCommunity()) {
        return $this; // 社区版：全可见，不过滤
    }

    // 高级版：按以下逻辑过滤
    // 1. 播客剧集 → 用户必须已订阅
    // 2. 普通歌曲 →
    //    - 用户未开启"包含公开媒体"：仅自己拥有的
    //    - 用户开启了"包含公开媒体"：自己拥有的 + 同组织其他用户的公开歌曲
}
```

**高级版歌曲可见性完整逻辑**：

```
歌曲可见 = 是播客剧集 AND 用户已订阅该播客
       OR 是普通歌曲 AND (
           (用户 preferences.includePublicMedia = false AND 歌曲 owner_id = user.id)
           OR
           (用户 preferences.includePublicMedia = true AND (
               歌曲 owner_id = user.id
               OR
               (歌曲 is_public = true AND 歌曲所有者的 organization_id = user.organization_id
                AND 歌曲所有者不是自己)
           ))
       )
```

### 3.2 `withUserContext()` — 标准查询入口

**文件**：[SongBuilder.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Builders/SongBuilder.php#L110-L119)

```php
public function withUserContext(
    bool $includeFavoriteStatus = true,
    bool $favoritesOnly = false,
    bool $includePlayCount = true,
): self {
    return $this
        ->accessible()  // 关键：先应用可见性过滤
        ->when($includeFavoriteStatus, ...)
        ->when($includePlayCount, ...);
}
```

`SongRepository` 中几乎所有查询方法（`paginate`、`getByAlbum`、`getByPlaylist`、`getFavorites` 等）都通过 `withUserContext()` 执行，确保返回的歌曲始终在当前用户权限范围内。

### 3.3 播放列表维度的歌曲过滤

**文件**：[SongRepository.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Repositories/SongRepository.php#L199-L258)

`getByPlaylist()` 获取播放列表中的歌曲时，同样使用 `Song::query(user: $scopedUser)->withUserContext()`，即**即使歌曲在播放列表中，如果用户不可见该歌曲，也不会出现在列表里**。

但注意：对于协作播放列表，协作者添加的歌曲会被自动设为公开（`is_public = true`），确保其他协作者可以看到。

### 3.4 播放列表可见性过滤

**文件**：[PlaylistRepository.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Repositories/PlaylistRepository.php#L28-L31)

```php
private function accessibleByUser(User $user): BelongsToMany
{
    return License::isCommunity() ? $user->ownedPlaylists() : $user->playlists();
}
```

- 社区版：只列出用户自己拥有的播放列表
- 高级版：列出用户关联的所有播放列表（拥有 + 协作）

---

## 第四层：协作播放列表与歌曲公开化机制

高级版支持播放列表协作，这会触发歌曲可见性的级联变化。

### 4.1 协作时自动公开歌曲

**文件**：[PlaylistService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Services/Playlist/PlaylistService.php#L146-L149)

```php
public function makePlaylistContentPublic(Playlist $playlist): void
{
    $playlist->playables()->where('is_public', false)->update(['is_public' => true]);
}
```

**触发时机**：
1. 已有协作者的播放列表中添加新歌曲时（`addPlayablesToPlaylist`）
2. 新协作者加入播放列表时（通过事件监听 [MakePlaylistSongsPublic.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Listeners/MakePlaylistSongsPublic.php) 监听 `NewPlaylistCollaboratorJoined` 事件）

### 4.2 协作播放列表的"双重访问"保证

协作播放列表中的歌曲需要同时满足两个层面的可见性：
1. **播放列表层面**：用户必须是所有者或协作者（由 `PlaylistPolicy@collaborate` 检查）
2. **歌曲层面**：歌曲必须对用户可见（由 `SongBuilder::accessible()` 过滤）

因为加入协作时歌曲已被自动设为 `is_public = true`，且协作者通常同属一个组织，所以歌曲层的可见性检查会自动通过。

---

## 完整请求流程示例

### 场景：用户播放一首歌曲

```
请求 GET /play/{song}
  ↓
1. audio.auth 中间件 → 检查令牌是否有 `audio`能力
   失败 → 401 Unauthorized
  ↓
2. 路由模型绑定 → 加载 Song 模型
  ↓
3. PlayController::__invoke
   → $this->authorize('access', $song)
   → SongPolicy@access
   → 社区版？通过
   → 高级版？$song->accessibleBy($user)
     → 是播客？检查订阅
     → 是普通歌曲？检查 is_public 或 ownedBy
   失败 → 403 Forbidden
  ↓
4. Streamer 流式输出音频
```

### 场景：用户获取某个播放列表的歌曲

```
请求 GET /api/playlists/{playlist}/songs
  ↓
1. auth 中间件 → 检查令牌 `*` 能力
   失败 → 401
  ↓
2. 路由模型绑定 → 加载 Playlist
  ↓
3. PlaylistSongController@index
   → 智能播放列表？authorize('own', $playlist)
   → 普通播放列表？authorize('collaborate', $playlist)
   → PlaylistPolicy 检查用户角色（owner / collaborator）
   失败 → 403
  ↓
4. SongRepository::getByPlaylist($playlist, $user)
   → Song::query(user: $user)->withUserContext()
   → SongBuilder::accessible() 在 SQL 层过滤不可见歌曲
   → 返回的一定是用户有权听到的歌曲
  ↓
5. 返回 SongResource 集合
```

### 场景：用户获取所有歌曲列表

```
请求 GET /api/songs
  ↓
1. auth 中间件 → 401
  ↓
2. SongController@index
   → SongRepository::paginate(scopedUser: $user)
   → Song::query(user: $user)->withUserContext()
   → accessible() 作用域在 SQL 层过滤
     社区版：不过滤，全部返回
     高级版：按 is_public + ownedBy + organization + 播客订阅过滤
  ↓
3. 返回分页结果
```

---

## 权限层级汇总表

| 层级 | 机制 | 位置 | 检查内容 | 失败响应 |
|------|------|------|---------|---------|
| L1 | 认证中间件 | `auth` / `audio.auth` | 用户是否登录、令牌能力 | 401 |
| L1.5 | Plus 特性限制 | `RestrictPlusFeatures` | 社区版不能用 Plus 功能 | 402 / 403 |
| L2 | 控制器 Policy 授权 | `$this->authorize()` | 单个对象的访问/编辑/删除权 | 403 |
| L3 | 查询作用域 | `SongBuilder::accessible()` | 列表查询时批量过滤不可见数据 | 数据被静默过滤 |
| L4 | 协作级联 | `makePlaylistContentPublic` | 协作时自动公开歌曲 | N/A（主动变更） |

---

## 关键设计要点

1. **社区版 vs 高级版的本质区别**：社区版所有歌曲全局共享，权限问题简化为"是否登录"；高级版才存在真正的歌曲级访问控制。

2. **双重检查机制**：单个资源用 Policy（有明确的 403 错误），列表查询用 Builder 作用域（静默过滤）。两者互为补充，确保数据不会泄漏。

3. **播放列表 ≠ 歌曲访问权**：能看到播放列表，不代表能听到列表里的每首歌。但协作播放列表通过自动公开歌曲机制，保证了协作者之间歌曲的互通。

4. **组织边界**：高级版的公开歌曲不是全局公开，而是限定在同一 `organization_id` 内。跨组织的用户即使歌曲设为公开也互不可见。

5. **令牌能力分离**：数据操作（API）需要 `*` 能力，音频流只需要 `audio` 能力，支持最小权限原则。
