# Koel 播放列表实现事实核对（第五轮）

## 1. SmartPlaylistRulesCast 降级路径传导链

### 1.1 降级触发条件

`SmartPlaylistRulesCast::get()` 中的降级逻辑：

```php
// app/Casts/SmartPlaylistRulesCast.php:10-15
public function get($model, string $key, $value, array $attributes): ?SmartPlaylistRuleGroupCollection
{
    return $value
        ? rescue(static fn () => SmartPlaylistRuleGroupCollection::create(json_decode($value, true)))
        : null;
}
```

**触发 rescue 降级的场景**：
1. JSON 解析失败（`json_decode` 返回 `null` 或抛出异常）
2. `SmartPlaylistRuleGroupCollection::create()` 抛出异常
3. `SmartPlaylistRuleGroup::make()` 抛出异常（如 UUID 校验失败）
4. `SmartPlaylistRule::assertConfig()` 断言失败（无效的 model/operator、value 格式错误等）
5. 任何其他未捕获的异常

**降级结果**：`rescue()` 捕获异常并返回 `null`。

---

### 1.2 传导链完整路径

```
数据库: playlists.rules 字段值 (text)
        ↓
[Cast 层] SmartPlaylistRulesCast::get()
        ├─ 值非空？
        │   ├─ 否 → 返回 null
        │   └─ 是 → 尝试解析
        │       ├─ 解析成功 → 返回 SmartPlaylistRuleGroupCollection
        │       └─ 解析失败 → rescue 捕获 → 返回 null
        ↓
[模型属性] $playlist->rules = null (降级后)
        ↓
[模型属性别名] ruleGroups() 访问器
        return $this->rules;  // 透传 null
        ↓
[模型属性] $playlist->rule_groups = null
        ↓
[is_smart 访问器]
        return (bool) $this->rule_groups?->isNotEmpty()
        → 空安全调用 ?-> 遇 null 短路 → null
        → (bool) null → false
        ↓
$playlist->is_smart = false
```

---

### 1.3 传导链各节点代码证据

#### 节点 1：Cast 层降级

**文件**：`app/Casts/SmartPlaylistRulesCast.php:12-14`

```php
return $value
    ? rescue(static fn () => SmartPlaylistRuleGroupCollection::create(json_decode($value, true)))
    : null;
```

**关键点**：
- `rescue()` 是 Laravel 辅助函数，捕获异常并返回默认值（此处为 `null`）
- 无论 JSON 解析失败还是值对象构造失败，最终都返回 `null`

#### 节点 2：rules 属性赋值

**文件**：`app/Models/Playlist.php:62-64`

```php
protected function casts(): array
{
    return [
        'rules' => SmartPlaylistRulesCast::class,
    ];
}
```

**关键点**：`$playlist->rules` 始终是 Cast 后的结果，即 `SmartPlaylistRuleGroupCollection|null`

#### 节点 3：rule_groups 别名

**文件**：`app/Models/Playlist.php:113-117`

```php
protected function ruleGroups(): Attribute
{
    // aliasing the attribute to avoid confusion
    return Attribute::get(fn () => $this->rules);
}
```

**关键点**：`$playlist->rule_groups` 是 `$this->rules` 的直接别名，透传 `null`

#### 节点 4：is_smart 访问器

**文件**：`app/Models/Playlist.php:108-111`

```php
protected function isSmart(): Attribute
{
    return Attribute::get(fn (): bool => (bool) $this->rule_groups?->isNotEmpty())->shouldCache();
}
```

**关键点**：
- 使用空安全运算符 `?->`，当 `rule_groups` 为 `null` 时短路返回 `null`
- `(bool) null` 强制转换为 `false`
- 结果被缓存（`shouldCache()`）

---

### 1.4 降级后对权限校验的影响

#### PlaylistSongController@index 分支

**文件**：`app/Http/Controllers/API/PlaylistSongController.php:29-40`

```php
public function index(Playlist $playlist)
{
    if ($playlist->is_smart) {
        // 降级后 is_smart = false，不进入此分支
        $this->authorize('own', $playlist);
        return SongResource::collection($this->songRepository->getByPlaylist($playlist, $this->user));
    }

    // 降级后进入此分支：手动播放列表权限
    $this->authorize('collaborate', $playlist);
    return self::createResourceCollection($this->songRepository->getByPlaylist($playlist, $this->user));
}
```

**降级后效果**：
- 原智能播放列表 → 被判定为手动播放列表
- 权限校验从 `own`（仅所有者）放宽到 `collaborate`（所有者 + 协作者）

#### PlaylistSongController@store 分支

**文件**：`app/Http/Controllers/API/PlaylistSongController.php:42-55`

```php
public function store(Playlist $playlist, AddSongsToPlaylistRequest $request)
{
    // 降级后 is_smart = false，不触发 abort
    abort_if($playlist->is_smart, Response::HTTP_FORBIDDEN, 'Smart playlist content is automatically generated');

    $this->authorize('collaborate', $playlist);
    // 允许添加歌曲
}
```

**降级后效果**：
- 原智能播放列表现在允许手动添加歌曲
- 用户可以向规则无效的智能播放列表中添加歌曲

---

### 1.5 降级后对 SongRepository 查询分支的影响

**文件**：`app/Repositories/SongRepository.php:199-208`

```php
public function getByPlaylist(Playlist|string $playlist, ?User $scopedUser = null): Collection
{
    $playlist = $this->playlistRepository->resolveOne($playlist);

    if ($playlist->is_smart) {
        // 降级后不进入此分支
        return $this->getBySmartPlaylist($playlist, $scopedUser);
    } else {
        // 降级后进入此分支
        return $this->getByStandardPlaylist($playlist, $scopedUser);
    }
}
```

**降级后查询行为变化**：
1. **SQL 结构变化**：从动态规则查询变为 `playlist_song` 中间表 JOIN 查询
2. **结果变化**：
   - 如果 `playlist_song` 表中有该播放列表的历史数据 → 返回这些歌曲
   - 如果没有历史数据 → 返回空集合
3. **排序变化**：从按 `songs.title` 排序变为按 `playlist_song.position` 排序
4. **协作信息变化**：Koel Plus 下会附加协作者信息

#### getByStandardPlaylist 内部断言

**文件**：`app/Repositories/SongRepository.php:212`

```php
throw_if($playlist->is_smart, new LogicException('Not a standard playlist.'));
```

**关键点**：降级后 `is_smart = false`，断言不会触发，查询正常执行。

#### getBySmartPlaylist 内部断言

**文件**：`app/Repositories/SongRepository.php:239`

```php
throw_unless($playlist->is_smart, NonSmartPlaylistException::create($playlist));
```

**关键点**：降级后不会调用此方法（外层 if 已拦截），因此不会触发异常。

---

### 1.6 降级传导链汇总表

| 层级 | 降级前（有效规则） | 降级后（规则无效） | 代码位置 |
|------|-------------------|-------------------|---------|
| Cast 层 | `SmartPlaylistRuleGroupCollection` | `null` | `SmartPlaylistRulesCast.php:12-14` |
| `$rules` | 集合对象 | `null` | `Playlist.php:63` |
| `$rule_groups` | 集合对象 | `null` | `Playlist.php:116` |
| `is_smart` | `true` | `false` | `Playlist.php:110` |
| 权限校验 | `authorize('own')` | `authorize('collaborate')` | `PlaylistSongController.php:32,37` |
| 添加歌曲 | 403 Forbidden | 允许添加 | `PlaylistSongController.php:47` |
| 查询分支 | `getBySmartPlaylist()` | `getByStandardPlaylist()` | `SongRepository.php:203-206` |
| SQL 来源 | 动态规则构建 | `playlist_song` 中间表 | `SongRepository.php:216-233, 243-255` |

---

## 2. 智能转手动时 playlist_song 历史数据处理

### 2.1 问题背景

当播放列表从智能转为手动时（规则被清空或失效），系统需要处理以下问题：
1. `playlist_song` 表中是否可能存在该播放列表的历史数据？
2. 这些数据在什么条件下会重新影响返回结果？
3. 转换过程中是否有清理机制？

---

### 2.2 历史数据存在的可能性分析

#### 场景 1：播放列表始终是智能的

```
创建时: rules = [...], is_smart = true
        ↓
生命周期中: 始终调用 getBySmartPlaylist()
        ↓
playlist_song 表: 无该播放列表的任何记录
```

**结论**：从未被手动添加过歌曲的智能播放列表，`playlist_song` 表中没有数据。

#### 场景 2：智能播放列表规则失效（降级）

```
创建时: rules = 有效 JSON, is_smart = true
        ↓
数据库中: rules 字段仍保存着原始 JSON
        ↓
某次访问时: rules JSON 损坏或版本不兼容 → Cast 降级
        ↓
is_smart = false → 走手动播放列表查询
        ↓
playlist_song 表: 可能仍无数据（除非降级后用户手动添加了）
```

**结论**：降级本身不会自动创建 `playlist_song` 记录。

#### 场景 3：用户主动更新播放列表，清空 rules

```
更新请求: rules = null 或 []
        ↓
PlaylistService@updatePlaylist
        ↓
$data = ['rules' => $dto->ruleGroups];  // $dto->ruleGroups = null
        ↓
SmartPlaylistRulesCast::set()
        return $value?->toJson() ?? null;  // 写入 null
        ↓
数据库: rules = NULL
        ↓
下次读取: is_smart = false
```

**关键问题**：更新时是否清理 `playlist_song` 数据？

---

### 2.3 更新操作代码分析

**文件**：`app/Services/Playlist/PlaylistService.php:63-98`

```php
public function updatePlaylist(Playlist $playlist, PlaylistUpdateData $dto): Playlist
{
    $data = [
        'name' => $dto->name,
        'description' => $dto->description,
        'rules' => $dto->ruleGroups,  // 可能为 null（表示清空规则）
    ];

    // ... 处理 cover

    $playlist->update($data);  // 更新 playlists 表

    // ... 处理文件夹关联

    return $playlist->refresh();
}
```

**关键发现**：
- `updatePlaylist()` 方法只更新 `playlists` 表的字段
- **没有任何代码清理 `playlist_song` 表中的关联数据**
- 没有调用 `$playlist->playables()->detach()` 或类似方法

---

### 2.4 历史数据影响返回结果的条件

#### 条件 1：播放列表曾是手动播放列表，后被转为智能

```
阶段 1: 手动播放列表
  - rules = NULL
  - 用户添加了 N 首歌曲 → playlist_song 表有 N 条记录
  - is_smart = false

阶段 2: 用户更新为智能播放列表
  - rules = [...有效规则...]
  - is_smart = true
  - 查询走 getBySmartPlaylist() → 忽略 playlist_song 数据
  - playlist_song 表的 N 条记录仍然存在！

阶段 3: 规则失效或被清空 → 转回手动
  - is_smart = false
  - 查询走 getByStandardPlaylist()
  - 结果：原有的 N 首歌曲重新出现！
```

**关键证据**：`getByStandardPlaylist()` 无条件查询 `playlist_song` 表：

```php
// app/Repositories/SongRepository.php:216-233
->leftJoin('playlist_song', 'songs.id', '=', 'playlist_song.song_id')
->leftJoin('playlists', 'playlists.id', '=', 'playlist_song.playlist_id')
// ...
->where('playlists.id', $playlist->id)
->orderBy('playlist_song.position')
```

**没有任何过滤条件**排除历史数据。

#### 条件 2：智能播放列表降级后，用户手动添加了歌曲

```
阶段 1: 纯智能播放列表
  - playlist_song: 无记录

阶段 2: 规则失效 → 降级为手动
  - is_smart = false
  - 查询返回空（playlist_song 无数据）

阶段 3: 用户添加歌曲
  - 调用 addPlayables() → 向 playlist_song 插入记录

阶段 4: 规则恢复（如修复了 JSON）
  - is_smart = true
  - 查询走 getBySmartPlaylist() → 忽略手动添加的歌曲
  - 但 playlist_song 表中的记录仍然存在

阶段 5: 规则再次失效
  - is_smart = false
  - 阶段 3 添加的歌曲重新出现
```

---

### 2.5 状态迁移完整说明

#### 状态定义

| 状态 | `playlists.rules` | `is_smart` | 说明 |
|------|-------------------|-----------|------|
| S0: 空手动 | NULL | false | 新建的手动播放列表，无歌曲 |
| S1: 有歌手动 | NULL | false | 有歌曲的手动播放列表，`playlist_song` 有数据 |
| S2: 有效智能 | 有效 JSON | true | 正常的智能播放列表 |
| S3: 无效智能 | 无效 JSON | false | 规则解析失败的智能播放列表（降级） |

#### 状态迁移路径

```
        创建 (songs=[])
     ┌──────────────────┐
     │                  ▼
     │                S0: 空手动
     │                  │
     │          添加歌曲 │
     │                  ▼
     │                S1: 有歌手动
     │                  │
     │    设置有效规则  │  设置有效规则
     │        ┌─────────┘
     │        ▼
     └────── S2: 有效智能
              │    │
              │    │ 规则失效/清空
              │    ▼
              │  S3: 无效智能
              │    │
              │    │ 规则恢复
              │    └───────┐
              │            ▼
              └────────────┘

              S3 → S1: 降级后添加歌曲（通过 playlist_song 累积数据）
              S2 → S1: 设置 rules = null，原 playlist_song 数据重新生效
              S1 → S2 → S3: 从有歌手动→智能→降级，手动歌曲数据始终保留
```

#### 各状态下的查询行为

| 状态 | 调用方法 | 数据来源 | playlist_song 数据是否影响结果 |
|------|---------|---------|-------------------------------|
| S0 | `getByStandardPlaylist()` | `playlist_song` | 是空表，不影响 |
| S1 | `getByStandardPlaylist()` | `playlist_song` | ✅ 直接返回这些数据 |
| S2 | `getBySmartPlaylist()` | 动态规则 | ❌ 完全忽略 |
| S3 | `getByStandardPlaylist()` | `playlist_song` | ✅ 若有历史数据则返回 |

---

### 2.6 前端更新逻辑分析

**文件**：`resources/assets/js/stores/playlistStore.ts`

```typescript
// 序列化规则
serializeSmartPlaylistRulesForStorage: (ruleGroups: SmartPlaylistRuleGroup[]) => {
  if (!ruleGroups || !ruleGroups.length) {
    return null  // 空规则组 → null
  }
  // ... 序列化逻辑
}
```

**前端提交时的规则值**：
- 用户在编辑表单中删除所有规则组 → `ruleGroups = []` → 序列化后为 `null`
- 前端发送 `rules: null` 到后端
- 后端 `PlaylistUpdateData::$ruleGroups = null`
- `updatePlaylist()` 将 `rules` 字段更新为 `NULL`
- 不清理 `playlist_song` 数据

---

### 2.7 可核对的结论

| 问题 | 结论 | 证据位置 |
|------|------|---------|
| 规则解析失败时 is_smart 变成什么？ | `false` | `Playlist.php:110` 空安全调用 + bool 转换 |
| 降级后权限校验有何变化？ | 从 `own` 放宽到 `collaborate`，允许添加歌曲 | `PlaylistSongController.php:29-55` |
| 降级后查询走哪个分支？ | `getByStandardPlaylist()` | `SongRepository.php:203-206` |
| 智能转手动时是否清理 playlist_song？ | ❌ 不清理 | `PlaylistService.php:63-98` 无 detach 调用 |
| 历史数据在什么条件下重新生效？ | 当 `is_smart` 从 `true` 变为 `false` 时（规则清空或失效） | `SongRepository.php:210-234` 无条件 JOIN |
| 智能播放列表期间 playlist_song 是否变化？ | 无变化，查询走智能分支忽略该表 | `SongRepository.php:237-258` |

---

## 3. 异常场景与边界条件

### 3.1 场景 A：手动播放列表设置无效规则

```
当前状态: S1 (有歌手动)
操作: 更新 rules = 无效 JSON
结果:
  - 数据库: rules = "无效 JSON"
  - Cast 降级: $rules = null
  - is_smart = false
  - 查询: 仍然返回手动歌曲（playlist_song 数据）
  - 效果: 用户以为设置了智能规则，但实际还是手动列表
```

### 3.2 场景 B：智能播放列表规则版本不兼容

```
当前状态: S2 (有效智能)
操作: 系统升级后，规则格式变化
结果:
  - 原有 JSON 无法解析 → Cast 降级
  - is_smart = false
  - 如果 playlist_song 有历史数据（如之前是手动列表），这些数据会突然出现
  - 用户可能看到意料之外的歌曲
```

### 3.3 场景 C：协作模式下的降级

```
当前状态: S2 (有效智能，所有者是用户 A)
操作: 规则失效降级
结果:
  - 权限从 own → collaborate
  - 协作者 B 现在可以向这个播放列表添加歌曲
  - 添加的歌曲存入 playlist_song
  - 如果规则恢复，这些手动添加的歌曲会被忽略但保留
```

---

## 4. 相关文件索引

### 降级传导链
- `app/Casts/SmartPlaylistRulesCast.php` - Cast 层降级逻辑
- `app/Models/Playlist.php:62-64, 108-117` - rules、rule_groups、is_smart 属性
- `app/Http/Controllers/API/PlaylistSongController.php:29-55` - 权限分支
- `app/Repositories/SongRepository.php:199-258` - 查询分支
- `app/Values/SmartPlaylist/SmartPlaylistRule.php:29-45` - 规则验证断言
- `app/Values/SmartPlaylist/SmartPlaylistRuleGroup.php:16` - UUID 验证断言

### 状态迁移与历史数据
- `app/Services/Playlist/PlaylistService.php:63-98` - 更新操作（无数据清理）
- `app/Repositories/SongRepository.php:210-234` - 手动播放列表查询（无条件 JOIN）
- `app/Values/Playlist/PlaylistUpdateData.php:16` - ruleGroups 可空声明
- `resources/assets/js/stores/playlistStore.ts` - 前端规则序列化（空数组→null）
