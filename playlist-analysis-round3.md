# Koel 播放列表实现事实核对（第三轮）

## 1. playlists.description 字段约束核对

### 1.1 数据库层约束

**迁移文件**：`database/migrations/2025_09_04_055648_add_description_to_playlists_table.php`

```php
Schema::table('playlists', static function (Blueprint $table): void {
    $table->text('description')->default('')->after('name');
});
```

**实际约束**：
| 属性 | 值 |
|------|----|
| 字段类型 | `text` |
| 是否可空 | 隐含 NOT NULL（因设置了 `default('')`） |
| 默认值 | 空字符串 `''` |
| 位置 | `name` 字段之后 |

### 1.2 请求验证层约束

**PlaylistStoreRequest**（`app/Http/Requests/API/Playlist/PlaylistStoreRequest.php:31`）：
```php
'description' => 'string|sometimes|nullable', // backward compatibility for mobile apps
```

**PlaylistUpdateRequest**（`app/Http/Requests/API/Playlist/PlaylistUpdateRequest.php:28`）：
```php
'description' => 'string|sometimes|nullable',
```

### 1.3 DTO 层处理

**PlaylistCreateData**（`app/Values/Playlist/PlaylistCreateData.php:16`）：
```php
public string $description,  // 声明为非空 string
```

在 `toDto()` 中进行强制转换：
```php
// PlaylistStoreRequest.php:48
description: (string) $this->description,

// PlaylistUpdateRequest.php:45
description: (string) $this->description,
```

### 1.4 模型层配置

**Playlist 模型**（`app/Models/Playlist.php:58`）：
```php
protected $guarded = [];  // 所有字段可批量赋值
```

### 1.5 约束不一致性分析

| 层级 | 约束 | 说明 |
|------|------|------|
| 数据库 | `text NOT NULL DEFAULT ''` | 不允许 NULL，默认空字符串 |
| 请求验证 | `string\|sometimes\|nullable` | 允许 NULL（为了移动端兼容） |
| DTO | `string` 类型声明 + `(string)` 强制转换 | NULL 被转换为空字符串 |

**结论**：虽然请求层允许 `nullable`，但通过 `(string)` 强制转换，最终写入数据库的始终是字符串（NULL → `''`），与数据库约束一致。

---

## 2. playlist_song.song_id 字段长度核对

### 2.1 初始创建（2015-11-23）

**迁移文件**：`database/migrations/2015_11_23_082854_create_playlist_song_table.php`

```php
Schema::create('playlist_song', static function (Blueprint $table): void {
    $table->increments('id');
    $table->integer('playlist_id')->unsigned();
    $table->string('song_id', 32);  // 初始长度 32
});
```

对应 `songs.id` 当时的定义：
```php
// database/migrations/2015_11_23_074713_create_songs_table.php:12
$table->string('id', 32)->primary();
```

### 2.2 UUID 迁移（2022-08-01）

**迁移文件**：`database/migrations/2022_08_01_093952_use_uuids_for_song_ids.php`

```php
// songs 表
Schema::table('songs', static function (Blueprint $table): void {
    $table->string('id', 36)->change();  // 32 → 36
});

// playlist_song 表同步修改
Schema::table('playlist_song', static function (Blueprint $table): void {
    $table->string('song_id', 36)->change();  // 32 → 36
    $table->foreign('song_id')->references('id')->on('songs')
        ->cascadeOnDelete()->cascadeOnUpdate();
});

// 同步修改 interactions 表
Schema::table('interactions', static function (Blueprint $table): void {
    $table->string('song_id', 36)->change();
});
```

### 2.3 现状

| 表 | 字段 | 当前长度 |
|----|------|---------|
| `songs` | `id` | `string(36)` |
| `playlist_song` | `song_id` | `string(36)` |
| `interactions` | `song_id` | `string(36)` |

**结论**：三者保持一致，均为 36 字符的 UUID 格式。

---

## 3. own_songs_only 字段完整生命周期追踪

### 3.1 字段新增（2024-01-12）

**迁移文件**：`database/migrations/2024_01_12_101606_add_own_songs_only_into_playlists_table.php`

```php
Schema::table('playlists', static function (Blueprint $table): void {
    $table->boolean('own_songs_only')->default(false);
});
```

**字段设计意图推测**：
- 布尔型字段，默认 `false`
- 可能用于标记播放列表是否仅包含用户自己上传的歌曲
- 与协作功能相关（非所有者添加的歌曲可能被过滤）

### 3.2 字段移除（2025-07-11）

**迁移文件**：`database/migrations/2025_07_11_100738_remove_own_songs_only_setting.php`

```php
Schema::table('playlists', static function (Blueprint $table): void {
    $table->dropColumn('own_songs_only');
});
```

### 3.3 代码使用情况核查

通过全代码库搜索 `own_songs_only` 或 `ownSongsOnly`：

| 位置 | 结果 |
|------|------|
| `app/` 目录 | 无任何引用 |
| `resources/` 目录 | 无任何引用 |
| `tests/` 目录 | 无任何引用 |
| 仅存在于 | 两个迁移文件 + 文档历史记录 |

### 3.4 生命周期时间线

```
2024-01-12: 新增字段 (add_own_songs_only_into_playlists_table)
        ↓
        [间隔约 18 个月]
        ↓
2025-07-11: 移除字段 (remove_own_songs_only_setting)
```

### 3.5 结论

1. **功能未实现**：字段仅存在于数据库迁移中，从未在业务代码中使用
2. **可能原因**：
   - 功能规划后被放弃
   - 实现方式改变（改用其他机制实现相同功能）
   - 实验性功能被回滚
3. **对当前代码无影响**：字段已完全移除，不影响任何查询判断

---

## 4. 字段设计对查询判断的影响分析

### 4.1 description 字段的影响

**对查询判断的影响：无直接影响**

- **用途**：仅作为展示性字段，用于描述播放列表
- **不参与**：
  - 播放列表类型判断（`is_smart` 由 `rules` 字段决定）
  - 权限校验
  - 歌曲查询条件
  - 任何业务逻辑分支判断

### 4.2 song_id 字段长度的影响

**对查询判断的影响：无逻辑影响，仅为数据一致性保障**

- **一致性保证**：`songs.id`、`playlist_song.song_id`、`interactions.song_id` 三者长度一致，确保外键关联和 JOIN 操作正常
- **查询性能**：2026-04-02 新增联合索引 `(playlist_id, song_id)` 优化查询性能
- **不参与**：业务逻辑判断

### 4.3 own_songs_only 字段的影响

**对查询判断的影响：已完全移除，无任何影响**

- 从未实际使用，因此从未影响查询逻辑
- 如果该功能被实现，可能会影响：
  ```php
  // 假设的实现逻辑（实际不存在）
  if ($playlist->own_songs_only) {
      $query->where('songs.owner_id', $user->id);
  }
  ```
- 当前查询逻辑中无此类过滤

### 4.4 实际影响查询判断的核心字段

| 字段 | 影响的查询判断 |
|------|---------------|
| `rules` | 决定 `is_smart`，智能 vs 手动播放列表的查询分支 |
| `playlist_user.role` | 决定用户权限（owner vs collaborator） |
| `playlist_song.position` | 手动播放列表的排序顺序 |
| `playlist_song.user_id` | 协作模式下显示添加者信息 |
| `interactions.user_id` | 用户播放记录关联 |

---

## 5. 核对结论汇总表

| 核查项 | 迁移文件 | 事实结论 | 对查询的影响 |
|--------|---------|---------|-------------|
| `playlists.description` 约束 | `2025_09_04_055648` | `text NOT NULL DEFAULT ''` | 无，仅展示用 |
| `playlist_song.song_id` 初始长度 | `2015_11_23_082854` | `string(32)` | 无，历史状态 |
| `playlist_song.song_id` 当前长度 | `2022_08_01_093952` | `string(36)` | 无，仅数据一致性 |
| `own_songs_only` 新增 | `2024_01_12_101606` | `boolean DEFAULT false` | 无，从未使用 |
| `own_songs_only` 移除 | `2025_07_11_100738` | 已删除 | 无，完全移除 |

---

## 6. 相关文件索引

### 迁移文件
- `database/migrations/2025_09_04_055648_add_description_to_playlists_table.php` - description 字段
- `database/migrations/2015_11_23_082854_create_playlist_song_table.php` - 初始 song_id(32)
- `database/migrations/2022_08_01_093952_use_uuids_for_song_ids.php` - song_id 改为 36
- `database/migrations/2024_01_12_101606_add_own_songs_only_into_playlists_table.php` - 新增 own_songs_only
- `database/migrations/2025_07_11_100738_remove_own_songs_only_setting.php` - 移除 own_songs_only

### 验证与 DTO
- `app/Http/Requests/API/Playlist/PlaylistStoreRequest.php` - 创建请求验证
- `app/Http/Requests/API/Playlist/PlaylistUpdateRequest.php` - 更新请求验证
- `app/Values/Playlist/PlaylistCreateData.php` - 创建 DTO
- `app/Values/Playlist/PlaylistUpdateData.php` - 更新 DTO

### 模型
- `app/Models/Playlist.php` - 播放列表模型
