# Koel 播放列表实现事实核对（第四轮）

## 1. playlists.description 字段可空性证据链

### 1.1 证据链结构说明

本证据链从数据库定义开始，逐层向上追踪每个代码层对 `description` 字段的处理，明确每一层的可空性判定。**默认值与可空性是两个独立的数据库概念，本证据链分别说明**。

---

### 1.2 证据链 1/6：数据库迁移层（Schema 定义）

**文件**：`database/migrations/2025_09_04_055648_add_description_to_playlists_table.php`

```php
Schema::table('playlists', static function (Blueprint $table): void {
    $table->text('description')->default('')->after('name');
});
```

**直接证据**：
- 调用 `->text('description')` 未链式调用 `->nullable()`
- Laravel 迁移中，**未显式调用 `nullable()` 的字段默认为 NOT NULL**
- 链式调用 `->default('')` 设置默认值为空字符串

**数据库层结论**：
| 属性 | 值 | 判定依据 |
|------|----|---------|
| 可空性 | `NOT NULL` | 未调用 `nullable()` |
| 默认值 | `''`（空字符串） | 显式 `default('')` |

---

### 1.3 证据链 2/6：模型 PHPDoc 声明

**文件**：`app/Models/Playlist.php:31`

```php
/**
 * @property string $description
 */
```

**直接证据**：
- PHPDoc 声明为 `string` 类型
- 未使用 `?string` 表示可空

**模型层结论**：
| 属性 | 值 | 判定依据 |
|------|----|---------|
| 可空性 | 不可空 | PHPDoc 声明为 `string` 而非 `?string` |

---

### 1.4 证据链 3/6：表单请求验证层

**文件 1**：`app/Http/Requests/API/Playlist/PlaylistStoreRequest.php:31`

```php
'description' => 'string|sometimes|nullable',
```

**文件 2**：`app/Http/Requests/API/Playlist/PlaylistUpdateRequest.php:28`

```php
'description' => 'string|sometimes|nullable',
```

**直接证据**：
- 验证规则显式包含 `nullable`
- 注释说明：`// backward compatibility for mobile apps`

**请求层结论**：
| 属性 | 值 | 判定依据 |
|------|----|---------|
| 可空性 | 允许 NULL | 显式 `nullable` 规则 |

---

### 1.5 证据链 4/6：DTO 值对象层

**文件 1**：`app/Values/Playlist/PlaylistCreateData.php:16`

```php
public string $description,  // 类型声明为非空 string
```

**文件 2**：`app/Values/Playlist/PlaylistUpdateData.php:12`

```php
public string $description,  // 类型声明为非空 string
```

**DTO 构造调用**：

```php
// PlaylistStoreRequest.php:48
description: (string) $this->description,

// PlaylistUpdateRequest.php:45
description: (string) $this->description,
```

**直接证据**：
- DTO 属性类型声明为 `string`（不可空）
- 传入 DTO 前使用 `(string)` 强制转换
- PHP 中 `(string) null` 的转换结果为空字符串 `''`

**DTO 层结论**：
| 属性 | 值 | 判定依据 |
|------|----|---------|
| 可空性 | 不可空 | 类型声明 `string` + `(string)` 强制转换 |

---

### 1.6 证据链 5/6：服务层写入

**文件**：`app/Services/Playlist/PlaylistService.php:38-43`

```php
$playlist = Playlist::query()->create([
    'name' => $data->name,
    'description' => $data->description,  // 直接使用 DTO 的 string 值
    'rules' => $data->ruleGroups,
    'cover' => $cover,
]);
```

**更新操作**（同文件第 65-69 行）：
```php
$data = [
    'name' => $dto->name,
    'description' => $dto->description,  // 直接使用 DTO 的 string 值
    'rules' => $dto->ruleGroups,
];
```

**直接证据**：
- 服务层直接传递 DTO 的 `string` 类型值
- 无额外的空值检查或转换

**服务层结论**：
| 属性 | 值 | 判定依据 |
|------|----|---------|
| 可空性 | 不可空 | 直接传递 DTO 的非空 string |

---

### 1.7 证据链 6/6：模型批量赋值配置

**文件**：`app/Models/Playlist.php:58`

```php
protected $guarded = [];  // 所有字段可批量赋值
```

**直接证据**：
- `$guarded = []` 表示无字段保护，`description` 可被批量赋值
- 无 `$casts` 配置对 `description` 进行类型转换

**模型赋值结论**：
| 属性 | 值 | 判定依据 |
|------|----|---------|
| 可空性 | 取决于传入值 | 无 Cast 转换，直接写入 |

---

### 1.8 可空性跨层对照总表

| 层级 | 可空性判定 | 关键字段 | 与数据库一致性 |
|------|-----------|---------|--------------|
| 数据库 | NOT NULL | `text NOT NULL DEFAULT ''` | - |
| 模型 PHPDoc | 不可空 | `@property string $description` | 一致 |
| 请求验证 | 允许 NULL | `nullable` 规则 | 不一致（为兼容移动端） |
| DTO | 不可空 | `string` 类型 + `(string)` 强制转换 | 一致（通过转换弥合） |
| 服务层 | 不可空 | 直接传递 DTO 值 | 一致 |
| 最终写入 | 不可空 | 始终为字符串（NULL 被转为 `''`） | 一致 |

**最终结论**：尽管请求验证层允许 NULL（为了移动端兼容），但通过 `(string)` 强制转换，最终写入数据库的 `description` 字段**始终是非空字符串**，与数据库的 `NOT NULL` 约束完全一致。

---

## 2. 模型到查询链路的字段角色对照

### 2.1 参考链路：rules → rule_groups → is_smart → 查询分支

这是智能/手动播放列表的核心判定链路，作为对照基准：

```
数据库字段: playlists.rules (text, JSON)
        ↓
[模型层] SmartPlaylistRulesCast
        ↓
[模型层] $rules 属性 (SmartPlaylistRuleGroupCollection|null)
        ↓
[模型层] $rule_groups 属性 (Attribute 别名)
        ↓
[模型层] is_smart 访问器
        return (bool) $this->rule_groups?->isNotEmpty()
        ↓
[控制器层] 权限分支判断
        if ($playlist->is_smart) { authorize('own') }
        else { authorize('collaborate') }
        ↓
[仓储层] SongRepository@getByPlaylist
        if ($playlist->is_smart) { getBySmartPlaylist() }
        else { getByStandardPlaylist() }
        ↓
[SQL 层] 两个完全不同的查询语句
```

**基准链路特征**：
- 每层都参与逻辑判断
- 字段值直接决定代码分支走向
- 最终影响生成的 SQL 结构

---

### 2.2 对照分析表

将 `description`、`song_id`（长度）、`own_songs_only` 三个字段与基准链路逐层对照：

| 链路阶段 | 基准链路 (rules) | description | playlist_song.song_id (长度) | own_songs_only |
|---------|-----------------|-------------|------------------------------|----------------|
| **数据库字段** | `rules` (text/JSON) | `description` (text) | `song_id` (string) | 已移除 |
| **模型 Cast** | `SmartPlaylistRulesCast` | 无 | 无（外键关联） | 无（已移除） |
| **模型属性** | `$rules` (对象集合) | `$description` (string) | 无直接属性 | 无 |
| **访问器判断** | `is_smart` 访问器 | 无访问器 | 无访问器 | 无 |
| **控制器分支** | `if ($is_smart)` 权限分支 | 无分支判断 | 无分支判断 | 无 |
| **服务层逻辑** | 决定能否添加/移动歌曲 | 仅透传存储 | 仅数据关联 | 无 |
| **仓储层查询** | `getBySmartPlaylist()` vs `getByStandardPlaylist()` | 不参与查询条件 | 仅 JOIN 关联字段 | 无（已移除） |
| **SQL 结构影响** | 生成完全不同的 SQL | 不影响 SQL | 不影响 SQL 结构 | 无 |

---

### 2.3 字段角色分类

基于对照结果，将字段分为三类：

#### 类别 A：参与分支判断的关键字段
| 字段 | 参与的判断 | 影响范围 |
|------|-----------|---------|
| `playlists.rules` | `is_smart` 访问器 → 权限分支 → 查询分支 | 端到端全链路 |

#### 类别 B：仅影响数据一致性的字段
| 字段 | 作用 | 是否参与逻辑判断 |
|------|------|-----------------|
| `playlists.description` | 展示性描述文本 | ❌ 否 |
| `playlist_song.song_id` 长度 | 保证外键关联匹配（36 vs 36） | ❌ 否 |

#### 类别 C：已移除/未使用字段
| 字段 | 生命周期 | 当前状态 |
|------|---------|---------|
| `playlists.own_songs_only` | 2024-01-12 新增，2025-07-11 移除 | 完全不影响 |

---

### 2.4 详细角色说明

#### description 字段角色

```
数据流路径：
用户输入 → 请求验证 (允许 NULL) → DTO (string 强制转换) → 服务层 (透传) → 数据库 (存储) → API 响应 (原样返回)

逻辑判断路径：
无任何分支判断。该字段仅作为数据在各层透传，不影响任何 if/else 分支。
```

**使用位置**：
- `PlaylistResource.php:51` - API 响应中返回给前端
- `PlaylistService.php:40,67` - 创建/更新时透传
- 无其他逻辑引用

#### song_id 字段长度角色

```
数据流路径：
songs.id (string 36) ←→ playlist_song.song_id (string 36) ←→ interactions.song_id (string 36)

逻辑判断路径：
无任何分支判断。长度一致仅为保证外键关联和 JOIN 操作的技术正确性。
```

**技术影响（非逻辑影响）**：
- 外键约束有效性
- JOIN 操作匹配正确性
- 索引效率

**逻辑影响**：无。无论长度是 32 还是 36，查询逻辑完全相同，只是数据能否正确存储的区别。

#### own_songs_only 字段角色

```
生命周期：
2024-01-12 新增 → 从未在代码中使用 → 2025-07-11 移除

逻辑判断路径：
无。从未有任何 if ($playlist->own_songs_only) 判断。
```

---

### 2.5 查询判断实际依赖的字段总览

| 字段 | 参与的查询判断 | 判断位置 |
|------|---------------|---------|
| `playlists.rules` | 智能 vs 手动播放列表查询分支 | `SongRepository.php:105-109` |
| `playlist_user.role` | 权限判断（owner/collaborator） | `PlaylistPolicy.php` |
| `playlist_song.position` | 手动播放列表排序 | `SongRepository.php:141` |
| `playlist_song.user_id` | 协作信息附加（Koel Plus） | `SongRepository.php:126-139` |
| `interactions.user_id` | 用户上下文关联 | `SongBuilder` |
| **`description`** | ❌ 无 | - |
| **`song_id` 长度** | ❌ 无（仅技术约束） | - |
| **`own_songs_only`** | ❌ 无（已移除） | - |

---

## 3. 可核对结论汇总

### 3.1 description 可空性结论

✅ **可直接核对的证据链**：
1. 迁移文件：`->text('description')->default('')` 未加 `nullable()` → NOT NULL
2. 模型 PHPDoc：`@property string $description` → 不可空
3. 请求验证：`nullable` 规则 → 允许 NULL（兼容移动端）
4. DTO 类型：`string` + `(string)` 强制转换 → NULL → `''`
5. 服务层：直接传递 DTO 值 → 不可空
6. 最终写入：始终为字符串 → 与数据库一致

### 3.2 字段角色分类结论

✅ **可直接核对的分类表**：
| 字段 | 类别 | 是否参与分支判断 | 证据 |
|------|------|-----------------|------|
| `rules` | A（关键字段） | ✅ 是 | `is_smart` 访问器、权限分支、查询分支 |
| `description` | B（数据字段） | ❌ 否 | 仅透传展示，无 if/else 引用 |
| `song_id` 长度 | B（数据字段） | ❌ 否 | 仅技术约束，不影响逻辑 |
| `own_songs_only` | C（已移除） | ❌ 否 | 全代码库无引用，已 drop |

---

## 4. 相关文件索引

### description 证据链文件
- `database/migrations/2025_09_04_055648_add_description_to_playlists_table.php` - 数据库定义
- `app/Models/Playlist.php:31` - PHPDoc 声明
- `app/Http/Requests/API/Playlist/PlaylistStoreRequest.php:31` - 创建请求验证
- `app/Http/Requests/API/Playlist/PlaylistUpdateRequest.php:28` - 更新请求验证
- `app/Values/Playlist/PlaylistCreateData.php:16` - 创建 DTO
- `app/Values/Playlist/PlaylistUpdateData.php:12` - 更新 DTO
- `app/Services/Playlist/PlaylistService.php:40,67` - 服务层写入
- `app/Http/Resources/PlaylistResource.php:51` - API 响应

### 对照链路文件
- `app/Models/Playlist.php:63,108-117` - rules Cast + is_smart 访问器
- `app/Casts/SmartPlaylistRulesCast.php` - 规则 Cast 类
- `app/Http/Controllers/API/PlaylistSongController.php:29-40` - 控制器分支
- `app/Repositories/SongRepository.php:199-258` - 仓储查询分支
- `database/migrations/2024_01_12_101606_add_own_songs_only_into_playlists_table.php` - own_songs_only 新增
- `database/migrations/2025_07_11_100738_remove_own_songs_only_setting.php` - own_songs_only 移除
