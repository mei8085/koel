# Koel 播放列表实现深度分析（第二轮）

## 1. 权限校验与协作场景下的查询分支差异

### 1.1 权限策略模型（PlaylistPolicy）

权限校验的核心位于 `app/Policies/PlaylistPolicy.php`，定义了 7 种权限能力：

| 能力 | 判定逻辑 | 适用场景 |
|------|---------|---------|
| `access` | `own() || hasCollaborator()` | 基础访问权限 |
| `own` | `ownedBy($user)` | 所有者权限 |
| `edit` | `own($user, $playlist)` | 编辑播放列表元信息 |
| `delete` | `own($user, $playlist)` | 删除播放列表 |
| `download` | `access($user, $playlist)` | 下载播放列表歌曲 |
| `inviteCollaborators` | `License::isPlus() && own() && !is_smart` | 邀请协作者（仅 Koel Plus、非智能） |
| `collaborate` | `own() || hasCollaborator()` | 协作编辑歌曲 |

**关键点**：
- 智能播放列表（`is_smart = true`）**不支持协作功能**，`inviteCollaborators` 明确排除了智能播放列表
- 权限判定与许可证级别（`License::isPlus()` / `License::isCommunity()`）深度耦合

### 1.2 控制器层的权限分支

各控制器在入口处根据播放列表类型采用不同的权限校验策略：

#### PlaylistSongController@index（获取播放列表歌曲）

```php
// app/Http/Controllers/API/PlaylistSongController.php:29-40
public function index(Playlist $playlist)
{
    if ($playlist->is_smart) {
        // 智能播放列表：仅所有者可访问
        $this->authorize('own', $playlist);
        return SongResource::collection($this->songRepository->getByPlaylist($playlist, $this->user));
    }

    // 手动播放列表：所有者或协作者均可访问
    $this->authorize('collaborate', $playlist);
    return self::createResourceCollection($this->songRepository->getByPlaylist($playlist, $this->user));
}
```

**分支差异**：
- 智能播放列表 → `own` 权限（仅所有者）
- 手动播放列表 → `collaborate` 权限（所有者 + 协作者）

#### PlaylistSongController@store（添加歌曲）

```php
// app/Http/Controllers/API/PlaylistSongController.php:42-55
public function store(Playlist $playlist, AddSongsToPlaylistRequest $request)
{
    // 智能播放列表：直接 403 拒绝
    abort_if($playlist->is_smart, Response::HTTP_FORBIDDEN, 'Smart playlist content is automatically generated');

    $this->authorize('collaborate', $playlist);
    // ... 添加歌曲逻辑
}
```

#### MovePlaylistSongsController（移动歌曲位置）

```php
// app/Http/Controllers/API/MovePlaylistSongsController.php:13-25
public function __invoke(MovePlaylistSongsRequest $request, Playlist $playlist, PlaylistService $service)
{
    $this->authorize('collaborate', $playlist);

    // PlaylistService 内部会再次检查并抛出异常
    $service->movePlayablesInPlaylist(...);
}
```

### 1.3 服务层的二次防护

即使控制器层已做校验，`PlaylistService` 仍会在核心操作中进行二次检查：

```php
// app/Services/Playlist/PlaylistService.php:152-180
public function movePlayablesInPlaylist(Playlist $playlist, array $movingIds, string $target, Placement $placement): void
{
    throw_if($playlist->is_smart, OperationNotApplicableForSmartPlaylistException::class);
    // ... 移动逻辑
}

public function addPlayablesToPlaylist(Playlist $playlist, ...): EloquentCollection
{
    // 注意：此方法没有显式检查 is_smart，但调用链上游（控制器）已保证
    // 仅手动播放列表会调用此方法
}
```

### 1.4 仓储层的查询分支

`SongRepository::getByPlaylist()` 是统一的查询入口，内部根据类型分发：

```php
// app/Repositories/SongRepository.php:199-208
public function getByPlaylist(Playlist|string $playlist, ?User $scopedUser = null): Collection
{
    $playlist = $this->playlistRepository->resolveOne($playlist);

    if ($playlist->is_smart) {
        return $this->getBySmartPlaylist($playlist, $scopedUser);
    } else {
        return $this->getByStandardPlaylist($playlist, $scopedUser);
    }
}
```

#### 手动播放列表查询（getByStandardPlaylist）

```php
// app/Repositories/SongRepository.php:210-235
private function getByStandardPlaylist(Playlist $playlist, ?User $scopedUser = null): Collection
{
    throw_if($playlist->is_smart, new LogicException('Not a standard playlist.'));

    return Song::query(user: $scopedUser ?? $this->auth->user())
        ->withUserContext()
        ->leftJoin('playlist_song', 'songs.id', '=', 'playlist_song.song_id')
        ->leftJoin('playlists', 'playlists.id', '=', 'playlist_song.playlist_id')
        // Koel Plus 专属：关联协作者信息
        ->when(License::isPlus(), static function (SongBuilder $query): SongBuilder {
            return $query->join(
                'users as collaborators',
                'playlist_song.user_id',
                '=',
                'collaborators.id',
            )->addSelect(
                'collaborators.public_id as collaborator_public_id',
                'collaborators.name as collaborator_name',
                'collaborators.email as collaborator_email',
                'collaborators.avatar as collaborator_avatar',
                'playlist_song.created_at as added_at',
            );
        })
        ->where('playlists.id', $playlist->id)
        ->orderBy('playlist_song.position')
        ->get();
}
```

**协作场景查询特点**：
- 仅 Koel Plus 许可证下才会关联 `collaborators` 表
- 查询结果包含协作者信息（添加者、添加时间）
- 按 `playlist_song.position` 排序（用户可自定义顺序）

#### 智能播放列表查询（getBySmartPlaylist）

```php
// app/Repositories/SongRepository.php:237-258
private function getBySmartPlaylist(Playlist $playlist, ?User $scopedUser = null): Collection
{
    throw_unless($playlist->is_smart, NonSmartPlaylistException::create($playlist));

    $query = Song::query(type: PlayableType::SONG, user: $scopedUser ?? $this->auth->user())
        ->withUserContext();

    // 规则组：组间 OR，组内 AND
    $playlist->rule_groups->each(static function (RuleGroup $group, int $index) use ($query): void {
        $whereClosure = static function (SongBuilder $subQuery) use ($group): void {
            $group->rules->each(static function (Rule $rule) use ($subQuery): void {
                QueryModifier::applyRule($rule, $subQuery);
            });
        };

        $query->when(
            $index === 0,
            static fn (SongBuilder $query) => $query->where($whereClosure),
            static fn (SongBuilder $query) => $query->orWhere($whereClosure),
        );
    });

    return $query->orderBy('songs.title')->limit(self::LIST_SIZE_LIMIT)->get();
}
```

**智能播放列表查询特点**：
- 无协作信息（不支持协作）
- 按 `songs.title` 固定排序（不可自定义）
- 结果限制 500 条（`LIST_SIZE_LIMIT = 500`）
- 每次访问动态计算，无静态关联表

### 1.5 PlaylistRepository 的访问范围控制

```php
// app/Repositories/PlaylistRepository.php:28-31
private function accessibleByUser(User $user): BelongsToMany
{
    // Community 版：仅返回用户拥有的播放列表
    // Plus 版：返回用户拥有的 + 协作的播放列表
    return License::isCommunity() ? $user->ownedPlaylists() : $user->playlists();
}
```

**许可证驱动的访问范围**：
- **Community**：`ownedPlaylists()` → `wherePivot('role', 'owner')`
- **Plus**：`playlists()` → 包含 `owner` 和 `collaborator` 角色

### 1.6 用户模型的关联关系

```php
// app/Models/Concerns/Users/HasUserRelationships.php:30-43
public function playlists(): BelongsToMany
{
    return $this->belongsToMany(Playlist::class)->withPivot('role', 'position')->withTimestamps();
}

public function ownedPlaylists(): BelongsToMany
{
    return $this->playlists()->wherePivot('role', 'owner');
}

public function collaboratedPlaylists(): BelongsToMany
{
    return $this->playlists()->wherePivot('role', 'collaborator');
}
```

## 2. playlists 与 playlist_song 关键字段类型及迁移历史

### 2.1 playlists 表现状

| 字段 | 类型 | 约束 | 说明 | 迁移历史 |
|------|------|------|------|---------|
| `id` | `string(36)` | PRIMARY | UUID 主键 | 2024-01-16: 从自增 INT 改为 UUID |
| `name` | `string` | NOT NULL | 播放列表名称 | 初始创建 |
| `description` | `text` | NULLABLE | 播放列表描述 | 2025-09-04: 新增 |
| `rules` | `text` | NULLABLE | 智能播放列表规则（JSON） | 2018-11-03: 新增 |
| `cover` | `string` | NULLABLE | 封面图片文件名 | 2024-02-24: 新增 |
| `created_at` | `timestamp` | NULLABLE | 创建时间 | 初始创建 |
| `updated_at` | `timestamp` | NULLABLE | 更新时间 | 初始创建 |

**已移除字段**：
- `user_id`（INT）：2025-06-03 迁移中移除，改为通过 `playlist_user` 中间表关联

### 2.2 playlist_song 表现状

| 字段 | 类型 | 约束 | 说明 | 迁移历史 |
|------|------|------|------|---------|
| `id` | `INT` | PRIMARY, AUTO_INCREMENT | 主键 | 初始创建 |
| `playlist_id` | `string(36)` | FOREIGN KEY | 播放列表 ID | 2024-01-16: 从 INT 改为 UUID |
| `song_id` | `string(32)` | FOREIGN KEY | 歌曲 ID | 初始创建 |
| `position` | `INT` | DEFAULT 0 | 歌曲在列表中的位置 | 2024-01-27: 新增 |
| `user_id` | `INT` | FOREIGN KEY | 添加该歌曲的用户 ID | 2024-01-16: 新增 |
| `created_at` | `timestamp` | NULLABLE | 添加时间 | 2024-01-16: 新增 |
| `updated_at` | `timestamp` | NULLABLE | 更新时间 | 2024-01-16: 新增 |

**索引**：
- 2026-04-02: 新增联合索引 `(playlist_id, song_id)` 优化查询性能

### 2.3 playlist_user 表（协作关系表）

该表经历了两次重大重构：

#### 第一阶段（2024-01-16）：`playlist_collaborators`

```php
// 初始结构
$table->bigIncrements('id');
$table->unsignedInteger('user_id');
$table->string('playlist_id', 36);
$table->timestamps();
```

#### 第二阶段（2025-06-03）：重命名为 `playlist_user` 并扩展

```php
// 2025_06_03_121538_modify_playlist-user_relationship.php
Schema::table('playlist_collaborators', static function (Blueprint $table): void {
    $table->string('role')->default('collaborator');  // 新增：owner / collaborator
    $table->integer('position')->default(0);           // 新增：排序位置
});

// 数据迁移：将原 playlists.user_id 转移到中间表作为 owner
DB::table('playlists')->get()->each(static function ($playlist): void {
    DB::table('playlist_collaborators')->insert([
        'user_id' => $playlist->user_id,
        'playlist_id' => $playlist->id,
        'role' => 'owner',
    ]);
});

// 重命名表
Schema::table('playlist_collaborators', static function (Blueprint $table): void {
    $table->rename('playlist_user');
});

// 移除 playlists.user_id 字段
Schema::table('playlists', static function (Blueprint $table): void {
    $table->dropForeign(['user_id']);
    $table->dropColumn('user_id');
});
```

**playlist_user 现状**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `BIGINT` | 主键 |
| `user_id` | `INT` | 用户 ID |
| `playlist_id` | `string(36)` | 播放列表 ID |
| `role` | `string` | 角色：`owner` / `collaborator` |
| `position` | `INT` | 排序位置 |
| `created_at` | `timestamp` | 创建时间 |
| `updated_at` | `timestamp` | 更新时间 |

### 2.4 关键迁移时间线

| 日期 | 迁移文件 | 变更内容 |
|------|---------|---------|
| 2015-11-23 | `create_playlists_table` | 初始创建 playlists 表（含 user_id） |
| 2015-11-23 | `create_playlist_song_table` | 初始创建 playlist_song 表 |
| 2018-11-03 | `add_rules_into_playlists` | 新增 rules 字段（智能播放列表） |
| 2022-08-01 | `use_uuids_for_song_ids` | 歌曲 ID 改为 UUID |
| 2022-08-10 | `support_playlist_folders` | 支持播放列表文件夹 |
| 2023-04-17 | `convert_playlist_rule_ids_to_uuid` | 规则 ID 改为 UUID |
| 2024-01-12 | `add_own_songs_only_into_playlists_table` | 新增 own_songs_only 字段 |
| 2024-01-16 | `use_uuids_for_playlists` | playlists.id 改为 UUID |
| 2024-01-16 | `add_playlist_collaborations_table` | 创建 playlist_collaborators 表 |
| 2024-01-16 | `add_timestamps_and_user_id_into_playlist_song_table` | playlist_song 新增 user_id 和 timestamps |
| 2024-01-24 | `add_playlist_cover` | 新增 cover 字段 |
| 2024-01-27 | `add_position_into_playlists_table` | playlist_song 新增 position 字段 |
| 2025-06-03 | `modify_playlist-user_relationship` | 重大重构：移除 playlists.user_id，playlist_collaborators → playlist_user |
| 2025-09-04 | `add_description_to_playlists_table` | 新增 description 字段 |
| 2026-04-02 | `add_missing_indexes_for_query_performance` | 优化索引 |

### 2.5 字段设计演进分析

**从 `user_id` 到 `playlist_user` 中间表的架构升级原因**：

1. **协作功能需求**：原设计中一个播放列表只能有一个所有者（`playlists.user_id`），无法支持多人协作
2. **多角色支持**：通过 `role` 字段区分 `owner` 和 `collaborator`
3. **数据完整性**：使用幂等设计（`if (Schema::hasTable('playlist_user')) return;`）确保迁移安全
4. **向后兼容**：迁移过程自动将原有所有者关系转移到新表

## 3. 从请求入口到 SQL 条件落地的完整链路

### 3.1 场景一：获取播放列表歌曲（手动播放列表 + 协作模式）

**请求链路**：

```
GET /api/playlists/{playlist}/songs
  ↓
[路由层] routes/api.base.php:195
  Route::apiResource('playlists.songs', PlaylistSongController::class)
  ↓
[中间件层] 'auth' → 'api'
  验证用户身份，注入 Authenticatable
  ↓
[控制器层] PlaylistSongController@index
  1. 隐式模型绑定：解析 {playlist} 为 Playlist 模型
  2. 权限校验：$this->authorize('collaborate', $playlist)
     ↓
     [策略层] PlaylistPolicy@collaborate
       return $this->own($user, $playlist) || $playlist->hasCollaborator($user);
  3. 分支判断：!$playlist->is_smart → 进入手动播放列表流程
  ↓
[仓储层] SongRepository@getByPlaylist
  ↓
[仓储层] SongRepository@getByStandardPlaylist
  1. 构建基础查询：Song::query(user: $user)->withUserContext()
  2. LEFT JOIN playlist_song ON songs.id = playlist_song.song_id
  3. LEFT JOIN playlists ON playlists.id = playlist_song.playlist_id
  4. when(License::isPlus()) → JOIN users AS collaborators
     附加协作者信息字段
  5. WHERE playlists.id = ?
  6. ORDER BY playlist_song.position
  ↓
[资源层] License::isPlus() 
  ? CollaborativeSongResource::collection() 
  : SongResource::collection()
  ↓
返回 JSON 响应
```

**生成的 SQL（Koel Plus 模式）**：

```sql
SELECT 
  songs.*,
  collaborators.public_id AS collaborator_public_id,
  collaborators.name AS collaborator_name,
  collaborators.email AS collaborator_email,
  collaborators.avatar AS collaborator_avatar,
  playlist_song.created_at AS added_at
FROM songs
LEFT JOIN playlist_song ON songs.id = playlist_song.song_id
LEFT JOIN playlists ON playlists.id = playlist_song.playlist_id
INNER JOIN users AS collaborators ON playlist_song.user_id = collaborators.id
WHERE playlists.id = 'uuid-playlist-123'
ORDER BY playlist_song.position ASC
```

### 3.2 场景二：获取播放列表歌曲（智能播放列表）

**请求链路**：

```
GET /api/playlists/{playlist}/songs
  ↓
[路由层] 同上
  ↓
[中间件层] 同上
  ↓
[控制器层] PlaylistSongController@index
  1. 隐式模型绑定
  2. 权限校验：$this->authorize('own', $playlist)
     ↓
     [策略层] PlaylistPolicy@own
       return $playlist->ownedBy($user);
  3. 分支判断：$playlist->is_smart → 进入智能播放列表流程
  ↓
[仓储层] SongRepository@getByPlaylist
  ↓
[仓储层] SongRepository@getBySmartPlaylist
  1. 基础查询：Song::query(type: SONG, user: $user)->withUserContext()
  2. 遍历规则组：
     - 第一组：$query->where(closure)
       - 组内遍历规则：QueryModifier::applyRule($rule, $subQuery)
     - 后续组：$query->orWhere(closure)
  3. ORDER BY songs.title
  4. LIMIT 500
  ↓
[资源层] SongResource::collection()
  ↓
返回 JSON 响应
```

**QueryModifier::applyRule 内部流程**：

```
applyRule(Rule $rule, SongBuilder $query)
  ↓
1. 日期特殊处理（IS/IS_NOT → BETWEEN/NOT_BETWEEN）
  ↓
2. 多对多关系判断（如 genre）
  ├─ 是：使用 whereHas/whereDoesntHave 子查询
  │   └─ 子查询内调用 generateParameters()
  └─ 否：直接在主查询上调用 where/whereBetween 等
  ↓
3. generateParameters(Model, Operator, value)
  ├─ 需要 Raw 查询？→ generateRawParameters()
  │   例：COALESCE(interactions.play_count, 0) > 10
  └─ 普通查询 → generateEloquentParameters()
      例：songs.title LIKE '%keyword%'
```

**生成的 SQL 示例**（规则：标题包含 "Love" AND 年份 > 1990 OR 流派是 "Rock"）：

```sql
SELECT songs.*
FROM songs
LEFT JOIN interactions ON interactions.song_id = songs.id 
  AND interactions.user_id = 'current-user-uuid'
WHERE (
  songs.title LIKE '%Love%' 
  AND songs.year > 1990
) OR EXISTS (
  SELECT 1 FROM genres
  INNER JOIN genre_song ON genres.id = genre_song.genre_id
  WHERE genre_song.song_id = songs.id
    AND genres.name = 'Rock'
)
ORDER BY songs.title ASC
LIMIT 500
```

### 3.3 场景三：添加歌曲到播放列表

**请求链路**：

```
POST /api/playlists/{playlist}/songs
  ↓
[路由层] routes/api.base.php:195
  ↓
[中间件层] 'auth'
  ↓
[控制器层] PlaylistSongController@store
  1. 智能播放列表拦截：abort_if($playlist->is_smart, 403)
  2. 权限校验：$this->authorize('collaborate', $playlist)
  3. 验证请求：AddSongsToPlaylistRequest
     - songs: array
     - 自定义规则：AllPlayablesAreAccessibleBy
  4. 获取歌曲：$this->songRepository->getMany(ids: $request->songs)
  ↓
[服务层] PlaylistService@addPlayablesToPlaylist
  ↓
[事务内]
  1. 过滤已存在歌曲
  2. 调用 $playlist->addPlayables($songs, $user)
     ↓
     [模型层] ManagesPlayables@addPlayables
       - 计算 max(position)
       - 构建 pivot 数据：[song_id => ['position' => ++pos, 'user_id' => $collaborator->id]]
       - $this->playables()->attach($data)
  3. 协同时：$this->makePlaylistContentPublic($playlist)
     - 更新歌曲 is_public = true
  ↓
[资源层] CollaborativeSongResource::collection()
  ↓
返回 201 Created
```

### 3.4 场景四：创建智能播放列表

**请求链路**：

```
POST /api/playlists
  ↓
[路由层] routes/api.base.php:194
  ↓
[中间件层] 'auth'
  ↓
[控制器层] PlaylistController@store
  1. 文件夹权限校验（如有）
  2. 验证请求：PlaylistStoreRequest
     - name: required
     - songs: array + AllPlayablesAreAccessibleBy
     - rules: array + ValidSmartPlaylistRulePayload
     - folder_id: exists + prohibits:folder_name
     - cover: ValidImageData
  3. 捕获 PlaylistBothSongsAndRulesProvidedException
  ↓
[服务层] PlaylistService@createPlaylist
  ↓
[事务内]
  1. 存储封面（可选）
  2. 创建 Playlist 模型
     - rules 字段通过 SmartPlaylistRulesCast 自动转换为 JSON
  3. 关联所有者：$user->ownedPlaylists()->attach($playlist, ['role' => 'owner'])
  4. 处理文件夹关联
  5. 非智能时：添加歌曲到中间表
  ↓
[资源层] PlaylistResource::make($playlist)
  ↓
返回 201 Created
```

**SmartPlaylistRulesCast 转换过程**：

```
set($model, $key, $value, $attributes)
  ↓
1. 数组转集合：SmartPlaylistRuleGroupCollection::create($value)
   - 递归转换：每组 → SmartPlaylistRuleGroup::make()
   - 每条规则 → SmartPlaylistRule::make()
   - 验证：model 和 operator 必须是有效枚举值
  ↓
2. 序列化为 JSON：$value->toJson()
  ↓
存储到 playlists.rules 字段
```

## 4. 异常与边界处理

### 4.1 异常类型

| 异常类 | 触发场景 |
|--------|---------|
| `NonSmartPlaylistException` | 对非智能播放列表调用智能播放列表方法 |
| `OperationNotApplicableForSmartPlaylistException` | 对智能播放列表执行仅手动播放列表支持的操作（如移动歌曲） |
| `PlaylistBothSongsAndRulesProvidedException` | 创建时同时提供 songs 和 rules |

### 4.2 校验点分布

| 层级 | 校验内容 |
|------|---------|
| 路由层 | 隐式模型绑定、中间件认证 |
| 控制器层 | 播放列表类型判断（is_smart）、权限 authorize() |
| 表单请求层 | 输入格式验证、自定义规则（ValidSmartPlaylistRulePayload、AllPlayablesAreAccessibleBy） |
| 服务层 | 业务逻辑前置检查（再次检查 is_smart）、事务边界 |
| 仓储层 | 查询前断言（throw_if / throw_unless） |
| 模型层 | Cast 转换时的验证（SmartPlaylistRule::assertConfig） |

## 5. 相关文件索引（补充）

### 权限与策略
- `app/Policies/PlaylistPolicy.php` - 播放列表权限策略
- `app/Http/Controllers/API/PlaylistSongController.php` - 歌曲 CRUD 控制器
- `app/Http/Controllers/API/MovePlaylistSongsController.php` - 移动歌曲控制器
- `app/Http/Controllers/Download/DownloadPlaylistController.php` - 下载控制器

### 数据库迁移
- `database/migrations/2015_11_23_074723_create_playlists_table.php` - 初始 playlists 表
- `database/migrations/2015_11_23_082854_create_playlist_song_table.php` - 初始 playlist_song 表
- `database/migrations/2018_11_03_182520_add_rules_into_playlists.php` - 智能播放列表规则字段
- `database/migrations/2024_01_16_215642_use_uuids_for_playlists.php` - UUID 迁移
- `database/migrations/2024_01_16_223123_add_playlist_collaborations_table.php` - 协作表初始
- `database/migrations/2025_06_03_121538_modify_playlist-user_relationship.php` - 重大重构
- `database/migrations/2026_04_02_132251_add_missing_indexes_for_query_performance.php` - 性能优化

### 模型与关联
- `app/Models/Playlist.php` - 播放列表模型
- `app/Models/Concerns/Playlists/ManagesCollaborators.php` - 协作者管理 Trait
- `app/Models/Concerns/Playlists/ManagesPlayables.php` - 歌曲管理 Trait
- `app/Models/Concerns/Users/HasUserRelationships.php` - 用户播放列表关联
- `app/Repositories/PlaylistRepository.php` - 播放列表仓储
- `app/Repositories/SongRepository.php` - 歌曲查询仓储（含智能播放列表查询构建）

### 异常类
- `app/Exceptions/NonSmartPlaylistException.php`
- `app/Exceptions/OperationNotApplicableForSmartPlaylistException.php`
- `app/Exceptions/PlaylistBothSongsAndRulesProvidedException.php`
