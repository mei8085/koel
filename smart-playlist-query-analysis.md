# Koel 智能播放列表：完整代码路径与查询机制分析

## 一、端到端完整代码路径

从用户点击播放列表到看到歌曲列表的完整调用链如下：

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  前端 (Vue 3 + TypeScript)                                                    │
│  ┌───────────────────────────────────────────────────────────────────────┐   │
│  │ PlaylistScreen.vue                                                     │   │
│  │   Line 207: fetchDetails()                                            │   │
│  │     Line 216: playableStore.fetchForPlaylist(playlist, refresh)      │   │
│  └──────────────────────────────────────────┬────────────────────────────┘   │
│                                             │                                │
│  ┌──────────────────────────────────────────▼────────────────────────────┐   │
│  │ playableStore.ts                                                       │   │
│  │   Line 228-244: fetchForPlaylist()                                    │   │
│  │     Line 232: cache.remove() (if refresh)                             │   │
│  │     Line 236: cache.remember(['playlist.songs', id], async () => ...)│   │
│  │       Line 237: http.get<Song[]>('playlists/{id}/songs')             │   │
│  │     Line 237: syncWithVault() (注册到前端单例歌曲库)                   │   │
│  └──────────────────────────────────────────┬────────────────────────────┘   │
└─────────────────────────────────────────────┼────────────────────────────────┘
                                              │ HTTP GET /api/playlists/{id}/songs
                                              ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  后端 (Laravel 12 + PHP 8.4)                                                 │
│  ┌───────────────────────────────────────────────────────────────────────┐   │
│  │ 路由: routes/api.base.php L194-L195                                    │   │
│  │   Route::apiResource('playlists.songs', PlaylistSongController::class)│   │
│  │   → GET /api/playlists/{playlist}/songs → index()                    │   │
│  └──────────────────────────────────────────┬────────────────────────────┘   │
│                                             │                                │
│  ┌──────────────────────────────────────────▼────────────────────────────┐   │
│  │ 控制器: PlaylistSongController.php L29-L40                            │   │
│  │   index(Playlist $playlist)                                           │   │
│  │   ├─ 智能播放列表: $this->authorize('own', $playlist)                 │   │
│  │   │   (仅所有者可见，协作者不能访问)                                    │   │
│  │   │   → SongResource::collection(...)                                 │   │
│  │   └─ 普通播放列表: $this->authorize('collaborate', $playlist)         │   │
│  │       → License::isPlus()                                             │   │
│  │         ? CollaborativeSongResource::collection()                     │   │
│  │         : SongResource::collection()                                  │   │
│  │   数据源: $this->songRepository->getByPlaylist($playlist, $user)     │   │
│  └──────────────────────────────────────────┬────────────────────────────┘   │
│                                             │                                │
│  ┌──────────────────────────────────────────▼────────────────────────────┐   │
│  │ 仓储: SongRepository.php L199-L258                                    │   │
│  │   getByPlaylist($playlist, $user)                                     │   │
│  │   ├─ $playlist->is_smart == true                                      │   │
│  │   │   └─→ getBySmartPlaylist()  L237-L258  ← 本报告核心分析对象       │   │
│  │   └─ $playlist->is_smart == false                                     │   │
│  │       └─→ getByStandardPlaylist()  L210-L235                          │   │
│  └──────────────────────────────────────────┬────────────────────────────┘   │
│                                             │                                │
│  ┌──────────────────────────────────────────▼────────────────────────────┐   │
│  │ 查询修饰器: SmartPlaylistQueryModifier.php L13-L121                   │   │
│  │   applyRule($rule, $query)  ← 规则 → SQL WHERE 子句翻译核心          │   │
│  │     日期等值→区间 / 多对多whereHas / Raw参数绑定                      │   │
│  └──────────────────────────────────────────┬────────────────────────────┘   │
│                                             │                                │
│  ┌──────────────────────────────────────────▼────────────────────────────┐   │
│  │ Builder: SongBuilder.php + FavoriteableBuilder.php                    │   │
│  │   withUserContext() L110-L119                                          │   │
│  │   ├─ accessible()         L66-L108  (权限过滤 JOIN)                   │   │
│  │   ├─ withFavoriteStatus() L25-L41   (收藏状态 LEFT JOIN favorites)    │   │
│  │   └─ withPlayCount()      L57-L64   (播放次数 LEFT JOIN interactions) │   │
│  └──────────────────────────────────────────┬────────────────────────────┘   │
│                                             │                                │
│  ┌──────────────────────────────────────────▼────────────────────────────┐   │
│  │ 模型: Song.php L120-L130                                              │   │
│  │   Song::query(type: PlayableType::SONG, user: $user)                 │   │
│  │     setScopedUser() + whereNull('podcast_id') + addSelect('songs.*') │   │
│  │     + #[UseEloquentBuilder(SongBuilder::class)] (L84 自定义Builder)  │   │
│  └──────────────────────────────────────────┬────────────────────────────┘   │
│                                             │                                │
│  ┌──────────────────────────────────────────▼────────────────────────────┐   │
│  │ API资源: SongResource.php L78-L135 + SongResourceCollection.php       │   │
│  │   toArray($request)                                                    │   │
│  │   输出 JSON 结构: title, album_name, artist_name, length, year,       │   │
│  │                 play_count, favorite, genre, is_public, owner_id...   │   │
│  │   Song 模型的 $with 会 Eager Loading: album, artist, album.artist,   │   │
│  │                                 podcast, genres, owner                │   │
│  └───────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 关键代码定位表

| 阶段 | 文件 | 行号 | 作用 |
|------|------|------|------|
| 前端歌曲获取 | [PlaylistScreen.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/resources/assets/js/components/screens/PlaylistScreen.vue) | L207-L226 | `fetchDetails()` 触发拉取 |
| 前端 Store | [playableStore.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/resources/assets/js/stores/playableStore.ts) | L228-L244 | `fetchForPlaylist()` 含前端缓存 |
| 路由注册 | [api.base.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/routes/api.base.php) | L195 | `playlists.songs` 嵌套资源路由 |
| 控制器入口 | [PlaylistSongController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/app/Http/Controllers/API/PlaylistSongController.php) | L29-L40 | `index()` 智能/普通分流 + 授权 |
| 授权策略 | [PlaylistPolicy.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/app/Policies/PlaylistPolicy.php) | L16-L44 | `own` vs `collaborate` 权限 |
| 仓储分发 | [SongRepository.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/app/Repositories/SongRepository.php) | L199-L258 | `getByPlaylist()` 分发入口 |
| 核心查询构建 | [SongRepository.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/app/Repositories/SongRepository.php) | L237-L258 | `getBySmartPlaylist()` 主逻辑 |
| Builder 自定义 | [Song.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/app/Models/Song.php) | L84, L120-L130 | `UseEloquentBuilder` 注解 + `query()` 静态方法 |
| 用户上下文 | [SongBuilder.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/app/Builders/SongBuilder.php) | L66-L119 | `accessible()` / `withUserContext()` |
| 收藏状态 JOIN | [FavoriteableBuilder.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/app/Builders/FavoriteableBuilder.php) | L25-L41 | `withFavoriteStatus()` |
| 规则翻译引擎 | [SmartPlaylistQueryModifier.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/app/Values/SmartPlaylist/SmartPlaylistQueryModifier.php) | L13-L121 | `applyRule()` 规则→SQL 翻译 |
| 资源序列化 | [SongResource.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/app/Http/Resources/SongResource.php) | L78-L135 | `toArray()` JSON 字段映射 |

---

## 二、可查询字段（SmartPlaylistModel）的精确统计与分类

### 2.1 字段总量与可见性分层

| 层级 | 数量 | 说明 |
|------|------|------|
| PHP 枚举定义总数 | **11 个** | [SmartPlaylistModel.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/app/Enums/SmartPlaylistModel.php#L5-L56) L5-L18 的 11 个 case |
| 禁止外部配置（内部用） | **1 个** | `USER_ID`，通过 `assertConfig()` 的 `$allowUserIdModel` 开关屏蔽 |
| 前端暴露给用户 | **10 个** | [models.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/resources/assets/js/config/smart-playlist/models.ts) L1-L53 |

### 2.2 10 个用户可配置字段按数据类型分类

| TypeScript 类型 | 数量 | 字段列表 |
|----------------|------|---------|
| `text`（文本） | 4 个 | Title、Album、Artist、Genre |
| `number`（数字） | 3 个 | Year、Play Count、Length（秒） |
| `date`（日期） | 3 个 | Last Played、Date Added、Date Modified |

### 2.3 11 个 PHP 枚举字段按数据库实现分类

| 枚举 case | `toColumnName()` 返回值 | 数据库实现策略 | 涉及表/列 |
|-----------|------------------------|---------------|----------|
| `TITLE` | `songs.title` | 普通列，Eloquent where | songs 表 |
| `ALBUM_NAME` | `songs.album_name` | **冗余反范式列**，避免 JOIN albums | songs 表 |
| `ARTIST_NAME` | `songs.artist_name` | **冗余反范式列**，避免 JOIN artists | songs 表 |
| `YEAR` | `songs.year` | 普通整数列 | songs 表 |
| `LENGTH` | `songs.length` | 普通浮点列 | songs 表 |
| `DATE_ADDED` | `songs.created_at` | DATETIME 列（需 IS→BETWEEN 转换） | songs 表 |
| `DATE_MODIFIED` | `songs.updated_at` | DATETIME 列（需 IS→BETWEEN 转换） | songs 表 |
| `LAST_PLAYED` | `interactions.last_played_at` | 依赖 withPlayCount() LEFT JOIN | interactions 表 |
| `PLAY_COUNT` | `COALESCE(interactions.play_count, 0)` | **Raw SQL 表达式**（无法用列索引） | interactions 表 |
| `GENRE` | `genres.name` | **多对多 whereHas 子查询**（EXISTS） | genres + genre_song 表 |
| `USER_ID` | `interactions.user_id` | 内部保留，禁止用户配置 | interactions 表 |

### 2.4 模型辅助方法的判定矩阵

| 字段 | `isDate()` | `requiresRawQuery()` | `getManyToManyRelation()` |
|------|:----------:|:--------------------:|:-------------------------:|
| TITLE | ❌ | ❌ | null |
| ALBUM_NAME | ❌ | ❌ | null |
| ARTIST_NAME | ❌ | ❌ | null |
| YEAR | ❌ | ❌ | null |
| LENGTH | ❌ | ❌ | null |
| DATE_ADDED | ✅ | ❌ | null |
| DATE_MODIFIED | ✅ | ❌ | null |
| LAST_PLAYED | ✅ | ❌ | null |
| PLAY_COUNT | ❌ | ✅ | null |
| GENRE | ❌ | ❌ | `'genres'` |
| USER_ID | ❌ | ❌ | null |

> **设计洞察**：这三个辅助方法构成了翻译引擎的"三元开关"——它们的返回值直接决定了 `applyRule()` 走哪一条翻译路径。这种以模型元数据驱动分支选择的设计，比在翻译器内部维护一个巨大 switch 要干净得多。

---

## 三、用户上下文过滤（withUserContext）的完整 JOIN 链

`Song::query()->withUserContext()` 是**所有歌曲查询的基础入口**（[SongBuilder.php L110-L119](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/app/Builders/SongBuilder.php#L110-L119)），它依次执行三次叠加的 JOIN 操作。

### 3.1 三层 JOIN 叠加总览

```
1. accessible()          ← 最外层：可见性/权限过滤（最复杂）
   LEFT JOIN podcasts
   LEFT JOIN podcast_user

2. withFavoriteStatus()  ← 中间层：收藏状态（多态关联）
   LEFT JOIN favorites (favoriteable_id + favoriteable_type + user_id)

3. withPlayCount()       ← 最内层：播放统计
   LEFT JOIN interactions (song_id + user_id)
```

### 3.2 第一层：accessible() 的权限矩阵（[SongBuilder.php L66-L108](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/app/Builders/SongBuilder.php#L66-L108)）

这层逻辑区分 Community 版和 Plus 版：

```php
public function accessible(): self
{
    if (License::isCommunity()) {
        return $this;  // Community 版：无过滤，所有歌曲所有用户可见
    }

    // Plus 版：分级可见性控制
    $this
        ->leftJoin('podcasts as podcasts_a11y', ...)
        ->leftJoin('podcast_user as podcast_user_a11y', ...)
        ->whereNot(
            // 播客剧集必须是订阅了的
            songs.podcast_id IS NOT NULL AND podcast_user_a11y.podcast_id IS NULL
        )
        ->where(function (self $query): void {
            $query
                // 是播客剧集 → 上面的 whereNot 已经放行
                ->whereNotNull('songs.podcast_id')
                ->orWhere(function (self $q2) {
                    if (!$this->user->preferences->includePublicMedia) {
                        // 偏好：只看自己的
                        return $q2->whereBelongsTo($this->user, 'owner');
                    }
                    return $q2->where(function (self $q3): void {
                        $q3
                            // 要么自己是 owner
                            ->whereBelongsTo($this->user, 'owner')
                            ->orWhere(function (self $q4): void {
                                $q4
                                    // 要么是公开 + 同组织的其他人上传的
                                    ->where('songs.is_public', true)
                                    ->whereHas('owner', fn (Builder $owner) => $owner
                                        ->where('organization_id', $this->user->organization_id)
                                        ->where('owner_id', '<>', $this->user->id)
                                    );
                            });
                    });
                });
        });
}
```

Plus 版的歌曲可见性真值表：

| 内容类型 | 可见条件 |
|---------|---------|
| 播客剧集 | 用户必须订阅了该播客（通过 `podcast_user` 关联） |
| 普通歌曲（`includePublicMedia = false`） | `songs.owner_id == user.id`（仅自己上传） |
| 普通歌曲（`includePublicMedia = true`） | 自己上传 **OR**（`is_public = true` **AND** 上传者与我同组织 **AND** 不是我自己） |

> **修正说明**：此前的分析误将 `accessible()` 简化为「基础 WHERE」，实际上它是 Plus 版复杂权限模型的核心实现，Community 版直接跳过，这是产品级差异化的典型体现。此外，`songs.podcast_id IS NULL` 过滤不是由 accessible() 执行的，而是由更早的 `Song::query(type: PlayableType::SONG)` 追加的（[Song.php L120-L130](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/app/Models/Song.php#L120-L130)）。

### 3.3 第二层：withFavoriteStatus() 的多态 JOIN（[FavoriteableBuilder.php L25-L41](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/app/Builders/FavoriteableBuilder.php#L25-L41)）

这是一个**通用抽象**（被 SongBuilder、AlbumBuilder、ArtistBuilder 等复用），通过三列匹配实现多态收藏：

```sql
LEFT JOIN favorites ON
    favorites.favoriteable_id   = songs.id          -- 主键
AND favorites.favoriteable_type = 'song'            -- 多态类型
AND favorites.user_id           = ?                 -- 当前用户

SELECT
    CASE WHEN favorites.created_at IS NULL
         THEN false
         ELSE true
    END AS favorite   -- 输出布尔列
```

其中 `favoritesOnly = true` 时（获取用户收藏列表），`LEFT JOIN` 会切换为 `INNER JOIN`（通过 `$joinMethod` 变量动态切换）。

### 3.4 第三层：withPlayCount() 的用户交互 JOIN（[SongBuilder.php L57-L64](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/app/Builders/SongBuilder.php#L57-L64)）

```sql
LEFT JOIN interactions ON
    interactions.song_id = songs.id
AND interactions.user_id = ?  -- 按用户隔离

SELECT COALESCE(interactions.play_count, 0) AS play_count
```

> **关键设计**：`LEFT JOIN` 而不是 `INNER JOIN`，配合 `COALESCE(..., 0)`——确保从未播放过的歌曲（interactions 中无记录）仍然能出现在结果中，且 `play_count` 显示为 0 而非 NULL。这个选择直接决定了 `PLAY_COUNT > 0` 这样的规则必须使用 Raw SQL 表达式（列值是计算出来的，不存在于索引中）。

### 3.5 一次智能播放列表查询的 JOIN 总量（Plus 版）

当 11 个字段中的 GENRE 规则也被激活时：

| JOIN 来源 | 涉及表 | 连接类型 |
|-----------|--------|---------|
| accessible() | podcasts_a11y, podcast_user_a11y | 2 × LEFT JOIN |
| withFavoriteStatus() | favorites | LEFT JOIN |
| withPlayCount() | interactions | LEFT JOIN |
| GENRE 规则（每条规则触发） | genre_song, genres | EXISTS 子查询内部 INNER JOIN |
| Song 模型 $with Eager Loading | albums, artists, podcasts, genres, users, genre_song | 6 条独立的 `WHERE IN` 查询 |
| **合计** | | **~12 次数据库 round-trip** |

> **修正说明**：此前的分析忽略了 `Song::$with = ['album', 'artist', 'album.artist', 'podcast', 'genres', 'owner']`（[Song.php L118](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/app/Models/Song.php#L118)）触发的 Eager Loading。这些不是主查询的 JOIN，而是 Laravel ORM 在 `get()` 之后的**额外查询**（每个关联一条 `WHERE id IN (...)`）。它们不影响 WHERE 逻辑，但严重影响整体响应延迟（通常 500 行触发 6 条 Eager 子查询）。

---

## 四、规则组的括号嵌套机制与 AND/OR 语义正确性

### 4.1 代码中的组合逻辑原语（[SongRepository.php L243-L255](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/app/Repositories/SongRepository.php#L243-L255)）

```php
$playlist->rule_groups->each(static function (RuleGroup $group, int $index) use ($query): void {
    $whereClosure = static function (SongBuilder $subQuery) use ($group): void {
        // 组内循环：对 $subQuery 连续调用 applyRule
        $group->rules->each(static function (Rule $rule) use ($subQuery): void {
            QueryModifier::applyRule($rule, $subQuery);
        });
    };

    // 组间切换：第一个用 where，其余用 orWhere
    $query->when(
        $index === 0,
        static fn (SongBuilder $query) => $query->where($whereClosure),
        static fn (SongBuilder $query) => $query->orWhere($whereClosure),
    );
});
```

### 4.2 Eloquent `where(Closure)` 的括号生成机制

当闭包被传入 `where()` 或 `orWhere()` 时，Laravel 的 `Illuminate\Database\Eloquent\Builder` 会执行以下内部逻辑：

1. **进入嵌套层级**：`$this->query->forNestedWhere()` 创建子 QueryBuilder，继承主查询的连接和绑定
2. **执行闭包**：`$closure($newEloquentBuilder)` —— 此时 `applyRule()` 内部调用的所有 `where()`/`whereRaw()`/`whereHas()` 都写到这个嵌套 Builder 上
3. **合并层级**：`$this->query->addNestedWhereQuery($newQuery, $boolean)` —— 自动将嵌套 Query 的所有 WHERE 子句**包裹在一对括号内**，用 $boolean（and / or）连接到外层

等价 SQL 构建过程：

```
主 Builder:  WHERE (accessible 条件) AND (withPlayCount LEFT JOIN)
                ↓ where(Closure)
                ↳ 新建嵌套 Builder
                   applyRule(rule_1) → AND songs.title LIKE '%Love%'
                   applyRule(rule_2) → AND songs.year > 2010
                ↳ addNestedWhereQuery → 用括号包裹 + AND 连接
主 Builder:  WHERE (...) AND (songs.title LIKE '%Love%' AND songs.year > 2010)
                ↓ orWhere(Closure)
                ↳ 新建嵌套 Builder
                   applyRule(rule_3) → AND EXISTS (...)
                   applyRule(rule_4) → AND COALESCE(...) > 100
                ↳ addNestedWhereQuery → 用括号包裹 + OR 连接
主 Builder:  WHERE (...) AND (...) OR (EXISTS (...) AND COALESCE(...) > 100)
```

### 4.3 三层括号结构的完整展开

```sql
SELECT songs.*, ...
FROM songs
[3-5 LEFT JOINs from withUserContext]
WHERE
  ┌─ accessible() 的总括括号（播客 + 同组织/公开可见性）
  │  (podcast过滤) AND (owner OR 同组织公开)
  └─
  AND
  ┌─ Song::query(type: SONG) 追加的基础过滤
  │  songs.podcast_id IS NULL
  └─
  AND
  ┌────────────────── 组间 OR 开始 ──────────────────┐
  │  ┌─────── 规则组 #1 括号 (AND) ────────┐         │
  │  │ (                                    │         │
  │  │   songs.title LIKE '%Love%'          │         │
  │  │   AND songs.year > 2010              │         │
  │  │ )                                    │         │
  │  └──────────────────────────────────────┘         │
  │                  OR                              │
  │  ┌─────── 规则组 #2 括号 (AND) ────────┐         │
  │  │ (                                    │         │
  │  │   EXISTS (                           │         │
  │  │     ... genre_song INNER JOIN genres│         │
  │  │   )                                  │         │
  │  │   AND COALESCE(play_count,0) > 100  │         │
  │  │ )                                    │         │
  │  └──────────────────────────────────────┘         │
  └────────────────── 组间 OR 结束 ──────────────────┘
ORDER BY songs.title
LIMIT 500
```

### 4.4 语义正确性的三个保障

| 保障机制 | 代码位置 | 防止的问题 |
|---------|---------|-----------|
| 闭包自动括号 | Eloquent 内部 `addNestedWhereQuery` | AND/OR 优先级错误——无括号时 `A AND B OR C AND D` 会被 SQL 解析为 `(A AND B) OR (C AND D)`（恰好一致），但有 `AND songs.podcast_id IS NULL` 在外层时会出错 |
| 组内全部 AND，组间全部 OR | `SongRepository.php` L245-L254 | 不支持任意嵌套布尔表达式，但 DNF（析取范式）足以覆盖 95% 以上的播放列表筛选场景 |
| 第一个组用 `where` 非 `orWhere` | `SongRepository.php` L250-L254 | 避免 `OR (...)` 出现在 WHERE 开头，破坏与外部 accessible() 条件的 AND 关系 |

> **设计权衡分析**：Koel 选择了**规则组二层嵌套（组内 AND + 组间 OR）**而非任意布尔表达式树。这是一次典型的「能力换简洁」决策：
> - ✅ 前端 UI 极易实现（列表分组，组之间显示 "Match ANY / Match ALL"）
> - ✅ 翻译器逻辑线性可预测，测试量大幅减少
> - ❌ 无法表达 `(A OR B) AND (C OR D)`（合取范式），用户必须手动拆成多个组
> - ❌ 无法表达 A AND (B OR C)，必须写成 (A AND B) OR (A AND C)

---

## 五、性能权衡的深度审视（修正与补充）

### 5.1 此前分析的修正清单

| 此前表述 | 修正后表述 | 依据 |
|---------|-----------|------|
| 「500 行 LIMIT 体现了渐进式加载设计」 | **500 行是硬上限保护，并非渐进式加载** | 整个系统没有对智能播放列表做 `paginate()` 或 `cursor`；`LIST_SIZE_LIMIT` 是常量（[SongRepository.php L37](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/app/Repositories/SongRepository.php#L37)），超过 500 的匹配歌曲被静默丢弃 |
| 「冗余列 album_name / artist_name」 | **修正：这两个列不是传统意义的冗余，而是 songs 表的原生列** | Song 表的 INSERT/UPDATE 流程中，artist_name 和 album_name 与 artist_id / album_id 同步写入，它们是歌曲属性的一等公民，不是事后反范式。避免 JOIN 是其**结果**，不是**目的** |
| 「可查询字段共 12 种」 | **共 11 种 PHP 枚举，其中 10 种对用户开放** | 重新数了 [SmartPlaylistModel.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/app/Enums/SmartPlaylistModel.php#L5-L18) 的 case：11 个 |
| 「智能播放列表通过 PlayableStore 查询」 | **完整链路还要经过：前端 HTTP 缓存 + 后端 Policy 授权 + SongResource 序列化** | 见 §一 端到端路径 |
| 「ORDER BY songs.title 需要 title 索引」 | **title 索引也不会生效** | 前面有 `accessible()` 的大量条件和多个 `OR` 组，MySQL 优化器通常选择全表扫描后 filesort；**真正的瓶颈是 WHERE，不是 ORDER BY** |

### 5.2 每次查询的性能成本分解

| 环节 | 耗时占比（估算） | 特点 |
|------|:---------------:|------|
| 主查询（含 5 JOIN + 多 EXISTS 子查询 + ORDER BY） | ~60% | 磁盘 IO 瓶颈，随 songs 表行数线性增长 |
| 结果集物化 + 模型 Hydration | ~10% | PHP CPU 瓶颈，500 行较轻 |
| `$with` Eager Loading（6 条关联查询） | ~20% | 额外 round-trip，albums/artists 等表通常较小可缓存 |
| `SongResource::toArray()` 序列化 | ~8% | PHP CPU 瓶颈 |
| JSON 编码 + HTTP 传输 | ~2% | 网络瓶颈 |

### 5.3 现有优化策略的覆盖范围评估

| 优化措施 | 生效范围 | 覆盖到的瓶颈 |
|---------|---------|-------------|
| `withUserContext()` 合并 JOIN | 全部规则 | 避免每条 PLAY_COUNT / LAST_PLAYED 规则重复 JOIN interactions |
| Genre → `whereHas` EXISTS 子查询 | GENRE 规则 | 避免 JOIN genre_song 后放大行数 + DISTINCT 去重 |
| `whereDoesntHave` + `IS` 反转 | GENRE 否定规则 | 让子查询能用 `genres.name` 索引（`=` 优于 `!=`） |
| 参数绑定（`?` + binding 数组） | Raw SQL 规则 | 防注入 + 执行计划复用 |
| 500 行硬上限 | 全局 | 保护 Eager Loading / 序列化 / 传输 |
| 前端 `cache.remember('playlist.songs', id)` | 前端二次打开 | 不请求后端（规则变更 / 手动刷新时显式 `cache.remove`） |

### 5.4 现有设计的性能天花板与改进建议

#### 天花板 A：全表扫描的不可避免性

触发条件：使用了以下任意一种规则
- `CONTAINS '%keyword%'`（前后通配 LIKE，不能用 B-Tree 索引）
- `PLAY_COUNT` 条件（`COALESCE()` 表达式列，无函数索引）
- 多个 OR 规则组（MySQL 对 `OR EXISTS + 普通条件` 的索引合并能力弱）

**改进建议（按收益/复杂度排序）**：

1. **MySQL 表达式索引（最低成本）**：针对 Play Count
   ```sql
   CREATE INDEX idx_songs_play_count_expr 
   ON interactions (song_id, user_id, play_count);
   -- 或 MySQL 8.0+：
   CREATE INDEX idx_play_count_coalesce 
   ON interactions ((COALESCE(play_count, 0)));
   ```

2. **全文索引替换 LIKE（中等成本）**：针对 Title / Album / Artist 的 CONTAINS
   ```sql
   ALTER TABLE songs ADD FULLTEXT INDEX ft_title_album_artist (title, album_name, artist_name);
   -- 翻译层将 contains → MATCH(cols) AGAINST('word' IN BOOLEAN MODE)
   ```

3. **UNION 替换 OR 组（高收益，中等成本）**：当存在多个规则组时
   ```sql
   -- 原方案：(A AND B) OR (C AND D)  —— 无法用索引
   -- 新方案：
   SELECT * FROM (
       SELECT songs.* FROM songs ... WHERE (A AND B)
       UNION
       SELECT songs.* FROM songs ... WHERE (C AND D)
   ) AS matches ORDER BY title LIMIT 500
   -- 每个子查询能独立使用索引，合并后再排序
   ```

4. **预计算物化视图（最高成本）**：对热门智能播放列表周期化刷新到中间表，直接 SELECT，完全跳过实时计算。

#### 天花板 B：智能播放列表的「无缓存」假设

当前实现：**每次打开播放列表都重新查询并翻译所有规则**。对于大型音乐库（> 50,000 首）且规则复杂的场景，响应时间可能 > 1 秒。

**改进建议**：
- 服务端缓存：`Cache::remember("smart-playlist:{$playlistId}:songs", now()->addMinutes(5), fn () => $query->get())`
- 失效策略：规则变更时（PlaylistObserver）、交互记录变化时（播放/收藏）——但交互变化频率高，建议接受 5 分钟最终一致
- 版本戳：Playlist 加 `rules_version` 字段，缓存键中携带此版本，规则变更自动失效旧缓存

#### 天花板 C：Song 模型 Eager Loading 的冗余加载

`Song::$with = ['album', 'artist', 'album.artist', 'podcast', 'genres', 'owner']` 是全局生效的，即使 API 响应中只用到了 `album_name`（已冗余在 songs 表里）和 `genre`（字符串名，用 genres.name 而非整个模型），关联的 Album / Artist 对象仍然被完整加载并 Hydration。

**改进建议**：
- 智能播放列表使用 `->without(['album', 'artist', 'album.artist', 'podcast', 'owner'])` 去除不需要的 Eager Load
- `genres` 也可以不加载，因为 SongResource 中 `'genre'` 用的是 `$song->genre`（拼接成字符串的访问器，不需要整个 Genre 集合）

可节省约 5 条 SQL 和数千个 Eloquent Model 对象的内存。

---

## 六、翻译引擎的边界情况测试验证（来自集成测试）

| 测试场景 | 文件与行号 | 验证的翻译正确性 |
|---------|-----------|----------------|
| Title IS / IS_NOT | [SmartPlaylistServiceTest.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/tests/Integration/Services/Playlist/SmartPlaylistServiceTest.php) L31-L71 | 等值比较 |
| Title CONTAINS / NOT_CONTAIN | 同上 L73-L113 | LIKE 模式的通配符位置 |
| Title BEGINS_WITH / ENDS_WITH | 同上 L115-L155 | 单侧通配符 |
| Album / Artist IS | 同上 L157-L207 | 冗余列的使用，无 JOIN |
| Genre IS / IS_NOT | 同上 L209-L256 | EXISTS / NOT EXISTS + 否定反转 |
| Year > / < / BETWEEN | 同上 L258-L331 | 数值比较 |
| Play Count > | 同上 L333-L367 | Raw SQL + COALESCE + 参数绑定 |
| Last Played IN_LAST / NOT_IN_LAST / IS | 同上 L369-L475 | 日期转换 + 相对时间 |
| Length > / BETWEEN | 同上 L477-L525 | 浮点比较 |
| Date Added IN_LAST / NOT_IN_LAST | 同上 L527-L575 | created_at 日期比较 |
| 多规则组（组间 OR）| [ValidSmartPlaylistRulePayloadTest.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/tests/Unit/Rules/ValidSmartPlaylistRulePayloadTest.php) L146-L183 | 结构有效性验证 |

> **覆盖缺口**：集成测试中**没有针对多规则组组合（DNF）的端到端查询测试**。所有测试用例都是单组内单规则的场景。`IS_BETWEEN` / `IS_NOT_BETWEEN` 操作符在 Genre 等复杂模型上的组合也未覆盖。这是一个测试风险点——当 `WHERE (A AND B) OR (C AND D)` 这样的真实场景发生时，Eloquent Builder 是否能正确产生括号，只能依赖对 Eloquent 框架本身的信任，而非项目内的验证。

---

## 七、总结

### 7.1 架构亮点回顾

1. **翻译器的元数据驱动设计**：`SmartPlaylistModel` 的三个辅助方法（`isDate`/`requiresRawQuery`/`getManyToManyRelation`）作为翻译决策轴，比硬编码 switch 更具扩展性。
2. **DNF 布尔结构的取舍**：组内 AND、组间 OR 限制了表达力，但换来了可预测的 SQL 形态、简洁的前端 UI 和极小的测试表面积。
3. **否定语义双层反转**：Genre 否定查询从 `whereHas + !=` 改为 `whereDoesntHave + =`，让索引有用武之地——这是典型的「写代码的人多走一步，数据库少走一万步」。
4. **用户上下文三层叠加**：accessible / favorites / play_count 在同一个 Builder 生命周期内一次性配置完毕，后续规则翻译完全不需要感知用户隔离。
5. **前后端双重硬上限**：后端 500 行 LIMIT + 前端 playableStore 只注册不额外加载，确保大型曲库不会造成前端内存爆炸。

### 7.2 三个值得注意的架构假设

| 隐含假设 | 当假设不成立时 |
|---------|--------------|
| 智能播放列表的匹配结果 ≤ 500 首就是可接受的 | 用户筛选条件宽松时，匹配到的第 501 首及以后歌曲被静默丢弃，无任何提示 |
| 智能播放列表每次访问都应该实时重算 | 规则不变 + 歌曲库不变时，重复查询产生的浪费是可接受的 |
| 匹配歌曲数量的排序关键字就是 title | 用户无法按 date_added / play_count / length 等字段排序智能播放列表结果（当前 `getBySmartPlaylist()` 硬编码 `orderBy('songs.title')`） |

### 7.3 对后续扩展的启示

若要增加新的可查询字段（例如 `bitrate`、`file_size`、`disc_number`），所需改动极其收敛：

1. [SmartPlaylistModel.php](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/app/Enums/SmartPlaylistModel.php) —— 加一个 enum case + `toColumnName()` 返回列名
2. [models.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/19-koel/resources/assets/js/config/smart-playlist/models.ts) —— 加一条前端配置
3. 如果是数字类型 → 无需改翻译器，12 种 Operator 已全部支持 `>` / `<` / `BETWEEN`
4. 如果是文本类型 → 无需改翻译器，12 种 Operator 已全部支持 `LIKE` 系列

**无需改动**：QueryModifier、SongRepository、Controller、前端表单组件、API 资源——这些都是纯粹的「驱动者」，不依赖于具体字段清单。开闭原则在这里得到了忠实执行。
