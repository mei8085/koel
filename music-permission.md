# 音乐权限分析

## 一、核心概念：两套访问判定

Koel 高级版存在两套独立的歌曲访问判定逻辑，分别服务于不同场景，规则不一致是理解权限问题的关键。

### 1.1 `Song::accessibleBy()` — 模型方法

**位置**：`App\Models\Song`

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
- 播客剧集：用户已订阅该播客
- 普通歌曲：`is_public = true` 或 `ownedBy($user)`

**特点**：无组织隔离检查，不受 `includePublicMedia` 用户偏好影响。

### 1.2 `SongBuilder::accessible()` — 查询作用域

**位置**：`App\Builders\SongBuilder`

**高级版判定规则**：
- 播客剧集：必须属于用户已订阅的播客
- 普通歌曲：
  - `includePublicMedia = false`：仅用户自己拥有的歌曲
  - `includePublicMedia = true`：用户自己的歌曲 + 同组织其他用户的公开歌曲

公开歌曲的 SQL 过滤条件：
```sql
songs.is_public = true
AND owner.organization_id = user.organization_id
AND songs.owner_id <> user.id
```

**特点**：有严格的组织隔离，受 `includePublicMedia` 用户偏好控制。

### 1.3 差异对照表

| 对比维度 | `Song::accessibleBy()` | `SongBuilder::accessible()` |
|---------|----------------------|---------------------------|
| 调用层 | Policy（单对象授权） | Repository（列表/单条查询） |
| 组织隔离 | 无 | 有（同组织内可见） |
| `includePublicMedia` 偏好 | 不影响 | 影响（关闭则不显示公开歌） |
| 跨组织公开歌曲 | 可访问 | 不可见 |
| 播客剧集检查 | 有 | 有 |
| 失败表现 | 403 Forbidden | 数据不返回（列表）或 404（单条） |

---

## 二、歌曲编辑接口完整链路分析

**入口**：`App\Http\Controllers\API\SongController@update`

编辑接口横跨三层权限检查，每一层使用的判定规则不同，是理解权限不一致的典型场景。

### 2.1 完整流程

```
PUT /api/songs
  ↓
① auth 中间件
   检查：令牌有 `*` 能力
   失败 → 401
  ↓
② 授权检查（控制器层）
   Song::query()->findMany($request->songs)
   → 直接用 Query Builder 加载，无可见性过滤
   → 每首歌调用 $this->authorize('edit', $song)
   → SongPolicy@edit
   → Song::accessibleBy()
   检查：is_public || ownedBy（无组织隔离，不受 includePublicMedia 影响）
   失败 → 403
  ↓
③ 实际修改（服务层）
   SongService::updateSongs()
   → Song::query()->with(...)->findMany($ids)
   → 直接用 Query Builder 加载，无可见性过滤
   → 修改歌曲属性，$song->push() 保存
  ↓
④ 回读（服务层，每首歌修改后）
   $this->songRepository->getOne($song->id)
   → Song::query(user: auth()->user())->withUserContext()->findOrFail()
   → SongBuilder::accessible()
   检查：组织隔离 + includePublicMedia 偏好
   失败 → findOrFail 抛出 ModelNotFoundException → 事务回滚
  ↓
⑤ 回读关联资源（控制器层）
   albumRepository->getMany(...)
   artistRepository->getMany(...)
   → AlbumBuilder / ArtistBuilder 的 withUserContext()
   → 同样有可见性过滤
  ↓
⑥ 返回结果
```

### 2.2 三个关键节点的规则对比

| 节点 | 判定方式 | 组织隔离 | `includePublicMedia` | 跨组织公开歌 |
|------|---------|---------|---------------------|-------------|
| ② 授权检查 | `SongPolicy@edit` → `accessibleBy()` | 无 | 不影响 | ✅ 可通过 |
| ③ 实际修改 | 直接操作模型，无权限检查 | — | — | ✅ 可修改 |
| ④ 回读歌曲 | `SongRepository::getOne` → `accessible()` | 有 | 影响 | ❌ 不可见 |

### 2.3 跨组织公开歌曲的编辑结果

用户编辑一首跨组织的公开歌曲时：

1. 授权检查通过（`accessibleBy()` 只要 `is_public = true` 就放行）
2. 歌曲属性在事务内被成功修改
3. `getOne()` 回读时因组织过滤找不到记录，`findOrFail` 抛出 `ModelNotFoundException`
4. 异常导致 `DB::transaction` 回滚，修改不生效
5. 最终响应：由 Laravel 异常处理器将 `ModelNotFoundException` 转为 404

**结论**：跨组织公开歌曲的编辑操作最终会失败，但失败原因不是 403（授权不通过），而是回读时的 404。

### 2.4 关闭 `includePublicMedia` 时的编辑结果

用户关闭"包含公开媒体"偏好后，编辑一首同组织的公开歌曲：

- 授权检查：通过（`accessibleBy()` 不检查 `includePublicMedia`）
- 实际修改：成功
- 回读：`getOne()` 走 `accessible()` → `includePublicMedia = false` → 仅自己的歌可见 → 他人的公开歌被过滤 → `findOrFail` 抛异常 → 事务回滚

**结论**：关闭 `includePublicMedia` 后，编辑他人的公开歌曲同样会因回读失败导致事务回滚。

### 2.5 编辑自己的歌曲

用户编辑自己拥有的歌曲时，无论 `includePublicMedia` 开启与否，三层检查都通过：
- 授权：`ownedBy()` 为 `true`
- 修改：正常执行
- 回读：`accessible()` 中 `whereBelongsTo($user, 'owner')` 分支匹配，能找到记录

---

## 三、SongRepository 查询作用域覆盖范围

`App\Repositories\SongRepository` 的方法按来源可分为三类：基础继承方法、显式业务方法、私有辅助方法。每类的可见性过滤策略不同。

### 3.1 方法来源分类

```
SongRepository
├── 基础继承方法（来自 Repository 抽象基类）
│   ├── 被 SongRepository 覆盖的：有过滤
│   └── 未被覆盖的：无过滤
├── 显式业务方法（SongRepository 自身定义）
│   ├── 面向终端用户的：有过滤
│   └── 内部/扫描用途的：无过滤
└── 私有辅助方法
    └── getByStandardPlaylist / getBySmartPlaylist：有过滤
```

### 3.2 基础继承方法的过滤情况

`App\Repositories\Repository` 抽象基类定义了 8 个通用方法，均直接使用 `$this->modelClass::query()`，**没有任何权限过滤**。SongRepository 覆盖了其中 3 个，使其具备可见性过滤。

| 基类方法 | 是否被覆盖 | 过滤情况 | 说明 |
|---------|-----------|---------|------|
| `getOne($id)` | ✅ 已覆盖 | 有过滤 | 覆盖后使用 `withUserContext()` |
| `findOne($id)` | ✅ 已覆盖 | 有过滤 | 覆盖后使用 `withUserContext()` |
| `getMany($ids, $preserveOrder)` | ✅ 已覆盖 | 有过滤 | 覆盖后使用 `withUserContext()` |
| `resolveOne($modelOrId)` | ❌ 未覆盖 | 有过滤* | 调用 `$this->getOne()`，因多态走 SongRepository 版本 |
| `getOneBy(array $params)` | ❌ 未覆盖 | 无过滤 | 直接 `where()->firstOrFail()` |
| `findOneBy(array $params)` | ❌ 未覆盖 | 无过滤 | 直接 `where()->first()` |
| `getAll()` | ❌ 未覆盖 | 无过滤 | 直接 `all()` |
| `findFirstWhere(...$params)` | ❌ 未覆盖 | 无过滤 | 直接 `firstWhere()` |

*\* `resolveOne()` 虽然定义在基类中，但其内部调用 `$this->getOne()`，由于 PHP 多态，实际执行的是 SongRepository 覆盖后的版本，因此具备过滤能力。*

**未被覆盖的 4 个方法（`getOneBy`、`findOneBy`、`getAll`、`findFirstWhere`）是权限盲区**：它们继承自基类，直接查询全表，不应用 `accessible()` 过滤。如果新增业务调用了这些方法，需要额外注意权限控制。

### 3.3 显式业务方法的过滤情况

SongRepository 自身定义的业务方法中，面向终端用户查询的均使用 `withUserContext()` 或直接调用 `accessible()` 应用过滤。

#### 3.3.1 有过滤的业务方法

| 方法 | 过滤方式 | 典型用途 |
|------|---------|---------|
| `paginate()` | `withUserContext()` | 歌曲列表分页 |
| `paginateByGenre()` | `withUserContext()` | 按流派分页 |
| `paginateInFolder()` | `withUserContext()` | 文件夹内分页（含根目录） |
| `getForQueue()` | `withUserContext()` | 队列播放列表 |
| `getByAlbum()` | `withUserContext()` | 专辑下的歌曲 |
| `getByArtist()` | `withUserContext()` | 艺术家下的歌曲 |
| `getByPlaylist()` | `withUserContext()` | 播放列表中的歌曲（智能+普通） |
| `getFavorites()` | `withUserContext(favoritesOnly: true)` | 收藏的歌曲 |
| `getRecentlyAdded()` | `withUserContext()` | 最近添加 |
| `getMostPlayed()` | `withUserContext()` | 播放最多 |
| `getLeastPlayed()` | `withUserContext()` | 播放最少 |
| `getRecentlyPlayed()` | `withUserContext()` | 最近播放 |
| `getRandom()` | `withUserContext()` | 随机歌曲 |
| `getByGenre()` | `withUserContext()` | 按流派获取 |
| `getEpisodesByPodcast()` | `withUserContext()` | 播客剧集列表 |
| `getUnderPaths()` | `withUserContext()` | 按文件夹路径递归获取 |
| `getInFolder()` | `withUserContext()` | 单个文件夹内歌曲（不含子文件夹） |
| `searchByLyrics()` | `withUserContext()` | 歌词搜索 |
| `getSimilarToMany()` | `withUserContext()` | 相似歌曲 |
| `search()` | 间接（通过 `getMany()`） | 搜索结果 |
| `getForEmbed()` | 间接（通过各 getByXxx） | 嵌入用歌曲 |
| `getManyInCollaborativeContext()` | `withUserContext()` | 协作用上下文的批量获取 |
| `countAccessibleByIds()` | 直接 `accessible()` | 按 ID 统计可访问数量 |
| `countSongs()` | 直接 `accessible()` | 歌曲总数统计 |
| `getTotalSongLength()` | 直接 `accessible()` | 歌曲总时长 |

**文件夹相关方法的过滤说明**：
- `paginateInFolder()` 和 `getInFolder()` 都在 `Song::query(user: $scopedUser)->withUserContext()` 的基础上追加 `folder_id` 条件，**权限过滤是完整的**
- 两者区别仅在于：`paginateInFolder()` 返回分页结果，`getInFolder()` 返回集合且有 500 条限制
- `getUnderPaths()` 也是同样模式，支持多路径递归获取

#### 3.3.2 无过滤的内部方法

以下方法用于扫描、同步等内部场景，不直接面向终端用户，因此不应用 `accessible()` 过滤：

| 方法 | 说明 | 用途 |
|------|------|------|
| `findOneByPath()` | 按文件路径查找，直接 `Song::query()` | 媒体扫描时定位文件 |
| `findByHash()` | 按哈希+所有者查找，仅过滤 `owner_id` | 去重/查重 |
| `getAllStoredOnCloud()` | 获取所有云存储歌曲，无过滤 | 云盘同步内部逻辑 |
| `getEpisodeGuidsByPodcast()` | 通过 `$podcast->episodes()` 查询，无用户上下文 | 播客同步内部逻辑 |

### 3.4 私有辅助方法

| 方法 | 过滤情况 | 调用方 |
|------|---------|-------|
| `getByStandardPlaylist()` | 有过滤（`withUserContext()`） | `getByPlaylist()` |
| `getBySmartPlaylist()` | 有过滤（`withUserContext()`） | `getByPlaylist()` |

两个私有方法均在 `withUserContext()` 基础上追加播放列表条件，权限过滤完整。

### 3.5 总结：哪些路径会自动套 `accessible()`

**会自动过滤的路径**：
- 所有通过 `withUserContext()` 入口的查询（面向用户的列表、详情、搜索、统计）
- 被 SongRepository 覆盖的基础方法：`getOne`、`findOne`、`getMany`
- 通过多态间接获得过滤的：`resolveOne`
- 私有辅助方法：`getByStandardPlaylist`、`getBySmartPlaylist`

**不会自动过滤的路径**：
- 未被覆盖的基础继承方法：`getOneBy`、`findOneBy`、`getAll`、`findFirstWhere`
- 内部扫描/同步方法：`findOneByPath`、`findByHash`、`getAllStoredOnCloud`、`getEpisodeGuidsByPodcast`
- 控制器中直接 `Song::query()->findMany()` 绕过 Repository 的调用（如编辑接口的授权检查）

---

## 四、其他关键路径的拦截流程

### 4.1 单曲播放路径

**入口**：`App\Http\Controllers\PlayController`

```
GET /play/{song}/{transcode?}
  ↓
① audio.auth 中间件
   检查：令牌有 `audio` 能力
   失败 → 401
  ↓
② 路由模型绑定加载 Song（无权限过滤）
  ↓
③ $this->authorize('access', $song)
   → SongPolicy@access
   → Song::accessibleBy()
   检查：is_public || ownedBy（无组织隔离）
   失败 → 403
  ↓
④ Streamer 流式输出（无额外权限检查）
```

**跨组织公开歌曲**：可以播放。播放接口只走 `accessibleBy()`，不检查组织。

**关闭 `includePublicMedia`**：可以播放。`accessibleBy()` 不受该偏好影响。

### 4.2 歌曲详情路径

**入口**：`App\Http\Controllers\API\SongController@show`

```
GET /api/songs/{song}
  ↓
① auth 中间件 → 401
  ↓
② 路由模型绑定加载 Song
  ↓
③ $this->authorize('access', $song)
   → SongPolicy@access → accessibleBy()
   检查：无组织隔离
   失败 → 403
  ↓
④ songRepository->getOne($song->id, $this->user)
   → SongBuilder::accessible()
   检查：组织隔离 + includePublicMedia
   失败 → 404
```

**双重检查现象**：先过 403（Policy 层，无组织过滤），再过 404（Repository 层，有组织过滤）。跨组织公开歌曲在详情接口返回 404。

### 4.3 播放列表歌曲加载路径

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
   → PlaylistPolicy 检查用户角色（owner / collaborator）
   失败 → 403
  ↓
④ 第二层：歌曲可见性过滤
   SongRepository::getByPlaylist($playlist, $user)
   → Song::query(user: $user)->withUserContext()
   → SongBuilder::accessible()
   检查：组织隔离 + includePublicMedia 偏好
   结果：不可见歌曲被静默过滤
```

**播放列表权限 ≠ 歌曲访问权**：用户能访问播放列表，不代表列表中的每首歌都可见。歌曲仍会经过 `accessible()` 的独立过滤。

**协作播放列表的特殊处理**：
- 当播放列表有协作者时，`PlaylistService::makePlaylistContentPublic()` 会将列表中的所有歌曲设为 `is_public = true`
- 触发时机：添加歌曲时（检查是否已有协作者）、新协作者加入时（通过事件异步触发）
- 注意：仅设置 `is_public`，不能绕过组织隔离。跨组织协作者添加的歌曲，即使设为公开，在对方的列表视图中仍会被 `accessible()` 过滤。

---

## 五、权限体系分层总览

| 层级 | 机制 | 作用范围 | 失败表现 |
|------|------|---------|---------|
| L1 | 认证中间件 | 路由入口 | 401 Unauthorized |
| L2 | Policy 授权 | 控制器方法（单对象） | 403 Forbidden |
| L3 | 查询作用域 | Repository 层查询 | 数据被静默过滤 / 404 |
| L4 | 协作级联 | 播放列表协作时 | 主动变更歌曲属性 |

许可证类型是核心分支条件：社区版（Community）所有歌曲全局可见，高级版（Plus）才有精细化的歌曲级访问控制。

### L1：认证中间件

- **`auth` 中间件**（`App\Http\Middleware\Authenticate`）：API 路由组使用，检查令牌 `*` 能力
- **`audio.auth` 中间件**（`App\Http\Middleware\AudioAuthenticate`）：音频流路由使用，检查令牌 `audio` 能力
- **`RestrictPlusFeatures` 中间件**：全局附加，检查 `#[RequiresPlus]` 注解

### L2：Policy 授权

- **SongPolicy**：`access` / `edit` 走 `accessibleBy()`，`delete` / `own` 走 `ownedBy()`
- **PlaylistPolicy**：`access` / `collaborate` 检查所有者或协作者，`edit` / `delete` 仅限所有者

### L3：查询作用域

- **SongBuilder**：`accessible()` + `withUserContext()` 提供标准查询入口
- **AlbumBuilder / ArtistBuilder**：同样有 `accessible()` 和 `withUserContext()`，规则类似
- **PlaylistRepository**：`accessibleByUser()` 区分社区版（仅自己的）和高级版（拥有+协作）

### L4：协作级联

- **`makePlaylistContentPublic()`**：协作播放列表中的歌曲自动设为公开
- **触发**：添加歌曲时实时执行，新协作者加入时通过事件异步执行

---

## 六、关键文件

| 文件 | 作用 |
|------|------|
| `app/Models/Song.php` | 歌曲模型，`accessibleBy()` / `ownedBy()` |
| `app/Builders/SongBuilder.php` | 歌曲查询构造器，`accessible()` / `withUserContext()` |
| `app/Policies/SongPolicy.php` | 歌曲授权策略 |
| `app/Policies/PlaylistPolicy.php` | 播放列表授权策略 |
| `app/Repositories/SongRepository.php` | 歌曲仓储 |
| `app/Repositories/PlaylistRepository.php` | 播放列表仓储 |
| `app/Http/Controllers/PlayController.php` | 歌曲播放控制器 |
| `app/Http/Controllers/API/SongController.php` | 歌曲 API 控制器（列表/详情/编辑/删除） |
| `app/Http/Controllers/API/PlaylistSongController.php` | 播放列表歌曲控制器 |
| `app/Services/SongService.php` | 歌曲服务，`updateSongs()` / `deleteSongs()` |
| `app/Services/Playlist/PlaylistService.php` | 播放列表服务，`makePlaylistContentPublic()` |
| `app/Http/Middleware/Authenticate.php` | API 认证中间件 |
| `app/Http/Middleware/AudioAuthenticate.php` | 音频认证中间件 |
| `app/Listeners/MakePlaylistSongsPublic.php` | 协作者加入事件监听器 |
