# 高级版公开歌曲权限规则分析

## 概述

Koel 高级版（Plus License）的公开歌曲（`is_public = true`）在不同访问路径下遵循**两套不同的判定规则**，存在显著的权限差异。理解这些差异对于排查"用户为什么听不到这首歌"或"这首歌为什么出现在列表里"至关重要。

两套核心判定逻辑分别是：
- **模型方法 `Song::accessibleBy()`**：用于单曲授权（Policy 层）
- **查询作用域 `SongBuilder::accessible()`**：用于列表查询（Repository 层）

两者在**组织隔离**、**用户偏好**等维度上规则不一致。

---

## 一、两套判定规则的详细对比

### 1.1 `Song::accessibleBy()` — 模型方法（单曲授权用）

**文件**：[Song.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Models/Song.php#L132-L139)

```php
public function accessibleBy(User $user): bool
{
    if ($this->isEpisode()) {
        return $user->subscribedToPodcast($this->podcast);
    }

    return $this->is_public || $this->ownedBy($user);
}
```

**公开歌曲判定**：只要 `is_public = true` 就返回 `true`。

| 检查项 | 是否涉及 |
|--------|---------|
| 组织隔离（`organization_id`） | ❌ 不检查 |
| 用户偏好（`includePublicMedia`） | ❌ 不检查 |
| 歌曲所有者（`ownedBy`） | 只需满足任一 |
| 播客订阅 | ✅ 播客剧集单独检查 |

**结论**：`accessibleBy()` 对公开歌曲是**全局放行**的，不限制组织。

---

### 1.2 `SongBuilder::accessible()` — 查询作用域（列表查询用）

**文件**：[SongBuilder.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Builders/SongBuilder.php#L66-L108)

```php
public function accessible(): self
{
    if (License::isCommunity()) {
        return $this;
    }

    // ... 播客剧集检查 ...

    return $this->where(function (self $query): void {
        $query
            ->whereNotNull('songs.podcast_id')
            ->orWhere(function (self $q2) {
                if (!$this->user->preferences->includePublicMedia) {
                    return $q2->whereBelongsTo($this->user, 'owner');
                }

                return $q2->where(function (self $q3): void {
                    $q3->whereBelongsTo($this->user, 'owner')->orWhere(function (self $q4): void {
                        $q4->where('songs.is_public', true)->whereHas('owner', fn (Builder $owner) => $owner->where(
                            'organization_id',
                            $this->user->organization_id,
                        )->where('owner_id', '<>', $this->user->id));
                    });
                });
            });
    });
}
```

**公开歌曲判定**：`is_public = true` **且** 所有者与当前用户同组织 **且** 不是自己的歌。

| 检查项 | 是否涉及 |
|--------|---------|
| 组织隔离（`organization_id`） | ✅ 严格限制同组织 |
| 用户偏好（`includePublicMedia`） | ✅ 关闭则完全不显示公开歌曲 |
| 歌曲所有者（`ownedBy`） | 单独的 OR 分支 |
| 播客订阅 | ✅ 播客剧集单独检查 |
| "排除自己" | ✅ 公开歌曲特指"他人的公开歌曲" |

**结论**：`accessible()` 对公开歌曲是**同组织内可见**，且受用户偏好开关控制。

---

### 1.3 规则差异对照表

| 维度 | `Song::accessibleBy()`（模型方法） | `SongBuilder::accessible()`（查询作用域） |
|------|-------------------------------------|-------------------------------------------|
| 用途 | Policy 授权（单曲） | 列表查询（批量） |
| 组织隔离 | ❌ 无 | ✅ 同组织内可见 |
| `includePublicMedia` 偏好 | ❌ 不受影响 | ✅ 关闭则不显示公开歌 |
| 自己的公开歌 | 算 `is_public` 为 true（但也匹配 ownedBy） | 不算在"公开歌曲"分支，走 `ownedBy` 分支 |
| 跨组织公开歌曲 | ✅ 可访问 | ❌ 不可见 |
| 调用方 | SongPolicy 的 access/edit/download | SongRepository 所有查询方法 |

---

## 二、三条关键访问路径的拦截层分析

### 2.1 路径一：单曲播放（PlayController）

**文件**：[PlayController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Http/Controllers/PlayController.php)

**完整拦截链**：

```
GET /play/{song}/{transcode?}
  ↓
① audio.auth 中间件
  [AudioAuthenticate.php]
  检查：令牌有 `audio` 能力
  失败 → 401
  ↓
② 路由模型绑定
  直接通过 id 加载 Song，无权限过滤
  ↓
③ $this->authorize('access', $song)
  → SongPolicy@access
    → Song::accessibleBy($user)
      → is_public || ownedBy
  检查：无组织隔离，全局公开即可
  失败 → 403
  ↓
④ Streamer 流式输出
  [Streamer.php]
  无额外权限检查
```

**关键拦截点**：第③步，使用 `Song::accessibleBy()`，**无组织隔离**。

**实际影响**：
- 如果知道其他组织的公开歌曲 ID，直接访问播放 URL 就可以播放
- 播放接口不检查 `includePublicMedia` 用户偏好
- Streamer 层纯技术实现，没有任何权限校验

---

### 2.2 路径二：歌曲列表查询（SongController@index 等）

**文件**：[SongController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Http/Controllers/API/SongController.php)

**完整拦截链**（以歌曲列表为例）：

```
GET /api/songs
  ↓
① auth 中间件
  [Authenticate.php]
  检查：令牌有 `*` 能力
  失败 → 401
  ↓
② SongController@index
  → SongRepository::paginate(scopedUser: $user)
    → Song::query(user: $user)->withUserContext()
      → SongBuilder::accessible()
        检查：
        - includePublicMedia 关闭 → 仅自己的歌
        - includePublicMedia 开启 → 自己的歌 + 同组织他人的公开歌
      → SQL 层面 WHERE 过滤
  结果：数据被静默过滤，不会有额外错误
```

**关键拦截点**：第②步，使用 `SongBuilder::accessible()`，**有严格的组织隔离**。

**影响的查询方法**（全部走 `withUserContext()` → `accessible()`）：

| Repository 方法 | 用途 |
|-----------------|------|
| `paginate()` | 歌曲列表分页 |
| `getOne()` | 单曲详情 |
| `getMany()` | 批量获取（按 ID） |
| `getByAlbum()` | 专辑下的歌曲 |
| `getByArtist()` | 艺术家下的歌曲 |
| `getByPlaylist()` | 播放列表中的歌曲 |
| `getFavorites()` | 收藏的歌曲 |
| `getRecentlyPlayed()` / `getMostPlayed()` | 播放历史 |
| `getByGenre()` | 流派下的歌曲 |
| `search()` | 搜索结果 |

---

### 2.3 路径三：播放列表歌曲加载（PlaylistSongController@index）

这是最复杂的一条路径，有**两层拦截**：播放列表权限层 + 歌曲可见性层。

**文件**：
- [PlaylistSongController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Http/Controllers/API/PlaylistSongController.php)
- [SongRepository.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Repositories/SongRepository.php#L199-L258)

**完整拦截链**：

```
GET /api/playlists/{playlist}/songs
  ↓
① auth 中间件 → 401
  ↓
② 路由模型绑定 → 加载 Playlist
  ↓
③ 播放列表权限检查（第一层）
  → $this->authorize('collaborate', $playlist)
    → PlaylistPolicy@collaborate
      → own() || hasCollaborator()
  检查：用户是所有者或协作者
  失败 → 403
  ↓
④ 歌曲可见性过滤（第二层）
  → SongRepository::getByPlaylist($playlist, $user)
    → Song::query(user: $user)->withUserContext()
      → SongBuilder::accessible()
        检查：组织隔离 + includePublicMedia 偏好
  结果：不可见歌曲被静默过滤
  ↓
⑤ 返回歌曲列表
```

**两层拦截的分工**：
- **第一层（PlaylistPolicy）**：确保用户能"碰"这个播放列表
- **第二层（SongBuilder::accessible()）**：确保列表里的每首歌用户都能听到

**关键问题：歌曲在播放列表里，但用户听不到？**

可能的原因：
1. 这首歌是私有的，且用户不是所有者
2. 这首歌是公开的，但所有者与用户不在同一个组织
3. 用户关闭了 `includePublicMedia` 偏好（即使同组织公开歌也不显示）
4. 这是播客剧集，用户未订阅该播客

**协作播放列表的特殊处理**：

当播放列表有协作者时，系统会自动将列表中的歌曲设为公开：

**文件**：[PlaylistService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Services/Playlist/PlaylistService.php#L146-L149)

```php
public function makePlaylistContentPublic(Playlist $playlist): void
{
    $playlist->playables()->where('is_public', false)->update(['is_public' => true]);
}
```

**触发时机**：
1. 已有协作者的播放列表加入新歌曲时
2. 新协作者加入播放列表时（通过 `NewPlaylistCollaboratorJoined` 事件）

但注意：`makePlaylistContentPublic` 只设置 `is_public = true`，并**不能保证跨组织可见**。如果协作者来自不同组织，即使歌曲设为公开，在列表查询时仍会被 `SongBuilder::accessible()` 的组织过滤排除。

---

## 三、不一致性与边界情况

### 3.1 跨组织公开歌曲：能播但搜不到

这是最显著的不一致：

| 操作 | 结果 | 原因 |
|------|------|------|
| 直接访问播放 URL `/play/{id}` | ✅ 可以播放 | 走 `accessibleBy()`，无组织过滤 |
| 在歌曲列表中查看 | ❌ 看不到 | 走 `accessible()`，有组织过滤 |
| 通过搜索查找 | ❌ 搜不到 | 走 `SongRepository::search()` → `accessible()` |
| 在他人播放列表中看到 | ❌ 看不到 | 走 `getByPlaylist()` → `accessible()` |

**安全提示**：如果已知歌曲 ID，跨组织用户可以直接播放公开歌曲。列表过滤只是"不可见"，不是"不可访问"。

### 3.2 `includePublicMedia` 偏好只影响列表，不影响播放

用户关闭"包含公开媒体"偏好后：
- 歌曲列表里看不到别人的公开歌 ✅
- 但直接访问播放 URL 仍然可以播放 ❌（因为 `accessibleBy()` 不检查这个偏好）

### 3.3 歌曲详情接口的"双重检查"

**文件**：[SongController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Http/Controllers/API/SongController.php#L40-L45)

```php
public function show(Song $song)
{
    $this->authorize('access', $song);       // 第一次检查：accessibleBy()，无组织过滤
    return SongResource::make($this->songRepository->getOne($song->id, $this->user)); // 第二次：accessible()，有组织过滤
}
```

流程：
1. 路由模型绑定加载 Song
2. `authorize('access')` → `accessibleBy()` → 如果是跨组织公开歌，这里会通过
3. `songRepository->getOne()` → `accessible()` → 组织过滤 → 如果跨组织，这里会 404

**结果**：跨组织公开歌曲访问详情接口时返回 404（不是 403），因为第二次查询被过滤掉了。用户体验上可能困惑："我明明能播放，为什么详情页 404？"

### 3.4 编辑/删除权限的跨组织问题

SongPolicy 的 `edit` 和 `delete` 方法：

- `edit`：高级版下走 `accessibleBy()` → 公开歌就能编辑？
- `delete`：高级版下走 `ownedBy()` → 只有自己的歌能删

等等，让我再确认一下 `edit` 的逻辑：

```php
public function edit(User $user, Song $song): bool
{
    return License::isCommunity() ? $user->hasPermissionTo(Permission::MANAGE_SONGS) : $song->accessibleBy($user);
}
```

高级版下，`edit` 权限 = `accessibleBy()` = `is_public || ownedBy`。

这意味着：**任何公开歌曲，任何人都可以编辑**？这似乎是一个设计问题。但实际执行时，编辑操作是对歌曲元数据的修改，通常只有所有者才有意义。需要结合业务逻辑进一步验证。

---

## 四、权限拦截汇总图

```
                        ┌─────────────────────┐
                        │   请求进入路由       │
                        └─────────┬───────────┘
                                  │
                  ┌───────────────┴───────────────┐
                  │                               │
          ┌───────┴───────┐               ┌───────┴───────┐
          │  API 路由      │               │  WEB 音频路由 │
          │  auth 中间件   │               │ audio.auth 中 │
          │  令牌 * 能力   │               │   间 令牌audio│
          └───────┬───────┘               └───────┬───────┘
                  │                               │
          ┌───────┴───────┐               ┌───────┴───────┐
          │ 控制器方法     │               │ PlayController│
          │               │               │               │
   单曲操作│ $this->authorize│        播放 │ authorize(     │
          │ → SongPolicy   │               │   'access',   │
          │ → accessibleBy │               │   $song)      │
          │ (无组织过滤)   │               │ → accessibleBy│
          └───────┬───────┘               │ (无组织过滤)   │
                  │                        └───────┬───────┘
          ┌───────┴───────┐                       │
          │ 列表查询       │                       │
          │ → Repository  │                       │
          │ → withUser-   │                       │
          │   Context()   │                       │
          │ → accessible()│                       │
          │ (有组织过滤)   │                       │
          └───────────────┘                       │
                                                  │
                                          ┌───────┴───────┐
                                          │   Streamer    │
                                          │ (无权限检查)  │
                                          └───────────────┘
```

---

## 五、关键文件速查

| 文件 | 作用 | 关键方法 |
|------|------|---------|
| [Song.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Models/Song.php) | 歌曲模型 | `accessibleBy()`, `ownedBy()` |
| [SongBuilder.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Builders/SongBuilder.php) | 歌曲查询构造器 | `accessible()`, `withUserContext()` |
| [SongPolicy.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Policies/SongPolicy.php) | 歌曲授权策略 | `access()`, `edit()`, `delete()`, `download()` |
| [SongRepository.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Repositories/SongRepository.php) | 歌曲仓储 | `paginate()`, `getOne()`, `getByPlaylist()` 等 |
| [PlaylistPolicy.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Policies/PlaylistPolicy.php) | 播放列表授权策略 | `access()`, `collaborate()`, `own()` |
| [PlayController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Http/Controllers/PlayController.php) | 播放控制器 | `__invoke()` → 唯一的歌曲播放入口 |
| [Streamer.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Services/Streamer/Streamer.php) | 流播放器 | 纯技术实现，无权限检查 |
| [PlaylistService.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Services/Playlist/PlaylistService.php) | 播放列表服务 | `makePlaylistContentPublic()` |
| [Authenticate.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Http/Middleware/Authenticate.php) | API 认证中间件 | 检查 `*` 令牌能力 |
| [AudioAuthenticate.php](file:///d:/fz/0508-2/solo-dogfeeding/code/114-koel/app/Http/Middleware/AudioAuthenticate.php) | 音频认证中间件 | 检查 `audio` 令牌能力 |

---

## 六、排查指南

**问题：用户说他听不到某首歌**

按以下顺序排查：

1. **先确认许可证**：社区版所有歌都可见，如果是社区版听不到，问题不在权限层
2. **检查播放列表权限**：用户是不是播放列表的所有者或协作者？
3. **检查歌曲属性**：
   - 这首歌是公开的还是私有的？
   - 所有者是谁？和用户同组织吗？
4. **检查用户偏好**：`includePublicMedia` 是否关闭了？
5. **区分路径**：
   - 直接播放 URL 能不能播？（走 `accessibleBy()`，无组织过滤）
   - 列表里能不能看到？（走 `accessible()`，有组织过滤）
6. **播客剧集特殊处理**：用户有没有订阅该播客？

**常见"听不到"原因**：
- 歌曲是私有歌曲，用户不是所有者 → 完全听不到
- 歌曲是其他组织的公开歌 → 列表里看不到，但直接访问播放 URL 可以播（不一致性）
- 用户关闭了"包含公开媒体" → 所有他人的公开歌都不出现在列表里
- 协作播放列表里有跨组织协作者 → 歌曲虽然设为公开，但不同组织的协作者在列表里看不到
