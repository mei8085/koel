# Koel 播放列表实现分析文档

## 1. 概述

Koel 支持两种类型的播放列表：

- **手动播放列表（Standard Playlist）**：用户手动选择并添加歌曲，歌曲与播放列表的关联关系存储在中间表中
- **智能播放列表（Smart Playlist）**：基于用户定义的规则动态筛选歌曲，规则以 JSON 格式存储，每次访问时动态计算歌曲列表

## 2. 数据存储方式

### 2.1 数据库表结构

#### `playlists` 表

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | UUID | 主键 |
| `name` | string | 播放列表名称 |
| `description` | text | 播放列表描述 |
| `rules` | text | 智能播放列表规则（JSON 格式），手动播放列表为 NULL |
| `cover` | string | 封面图片文件名 |
| `created_at` | timestamp | 创建时间 |
| `updated_at` | timestamp | 更新时间 |

#### `playlist_song` 中间表（仅手动播放列表使用）

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | integer | 主键 |
| `playlist_id` | integer | 播放列表 ID |
| `song_id` | string | 歌曲 ID |
| `position` | integer | 歌曲在播放列表中的位置 |
| `user_id` | integer | 添加该歌曲的用户 ID（协作功能使用） |
| `created_at` | timestamp | 添加时间 |
| `updated_at` | timestamp | 更新时间 |

### 2.2 存储方式对比

| 特性 | 手动播放列表 | 智能播放列表 |
|------|-------------|-------------|
| 歌曲关联 | `playlist_song` 中间表 | 无静态关联，动态计算 |
| 规则存储 | 无 | `playlists.rules` 字段（JSON） |
| 类型判断 | `rules` 字段为 NULL | `rules` 字段非空且包含有效规则组 |
| 歌曲顺序 | `position` 字段 | 查询时按 `title` 排序 |

### 2.3 类型判断逻辑

在 `Playlist` 模型中，通过 `is_smart` 访问器判断类型：

```php
// app/Models/Playlist.php:108-111
protected function isSmart(): Attribute
{
    return Attribute::get(fn (): bool => (bool) $this->rule_groups?->isNotEmpty())->shouldCache();
}
```

## 3. 智能播放列表规则建模

### 3.1 规则层级结构

```
SmartPlaylistRuleGroupCollection (集合)
└── SmartPlaylistRuleGroup (规则组 - OR 关系)
    ├── id: string
    └── rules: SmartPlaylistRule[] (规则 - AND 关系)
        ├── id: string
        ├── model: SmartPlaylistModel (字段模型)
        ├── operator: SmartPlaylistOperator (操作符)
        └── value: array (操作值，1-2个元素)
```

**逻辑关系说明**：
- 规则组之间是 **OR** 关系（满足任意一组即可）
- 同一规则组内的规则之间是 **AND** 关系（必须同时满足）

### 3.2 后端值对象设计

#### `SmartPlaylistRule`（规则）

```php
// app/Values/SmartPlaylist/SmartPlaylistRule.php
final class SmartPlaylistRule implements Arrayable
{
    public string $id;
    public SmartPlaylistModel $model;
    public SmartPlaylistOperator $operator;
    public array $value;
    
    // 工厂方法
    public static function make(array $config): self
    
    // 验证配置
    public static function assertConfig(array $config, bool $allowUserIdModel = true): void
    
    // 转换为数组（用于序列化）
    public function toArray(): array
}
```

#### `SmartPlaylistRuleGroup`（规则组）

```php
// app/Values/SmartPlaylist/SmartPlaylistRuleGroup.php
final class SmartPlaylistRuleGroup implements Arrayable
{
    public function __construct(
        public string $id,
        public Collection $rules, // SmartPlaylistRule 集合
    ) {}
}
```

#### `SmartPlaylistRuleGroupCollection`（规则组集合）

```php
// app/Values/SmartPlaylist/SmartPlaylistRuleGroupCollection.php
final class SmartPlaylistRuleGroupCollection extends Collection
{
    public static function create(array $array): self
}
```

### 3.3 字段模型（SmartPlaylistModel）

定义了可用于规则的歌曲字段：

```php
// app/Enums/SmartPlaylistModel.php
enum SmartPlaylistModel: string
{
    case ALBUM_NAME = 'album.name';
    case ARTIST_NAME = 'artist.name';
    case DATE_ADDED = 'created_at';
    case DATE_MODIFIED = 'updated_at';
    case GENRE = 'genre';
    case LAST_PLAYED = 'interactions.last_played_at';
    case LENGTH = 'length';
    case PLAY_COUNT = 'interactions.play_count';
    case TITLE = 'title';
    case USER_ID = 'interactions.user_id';
    case YEAR = 'year';
}
```

每个模型提供以下方法：
- `toColumnName()`: 转换为数据库列名
- `isDate()`: 是否为日期类型
- `requiresRawQuery()`: 是否需要原始 SQL 查询
- `getManyToManyRelation()`: 是否为多对多关系（如 genres）

### 3.4 操作符（SmartPlaylistOperator）

```php
// app/Enums/SmartPlaylistOperator.php
enum SmartPlaylistOperator: string
{
    case IS = 'is';
    case IS_NOT = 'isNot';
    case CONTAINS = 'contains';
    case NOT_CONTAIN = 'notContain';
    case IS_BETWEEN = 'isBetween';
    case IS_GREATER_THAN = 'isGreaterThan';
    case IS_LESS_THAN = 'isLessThan';
    case BEGINS_WITH = 'beginsWith';
    case ENDS_WITH = 'endsWith';
    case IN_LAST = 'inLast';
    case NOT_IN_LAST = 'notInLast';
    case IS_NOT_BETWEEN = 'isNotBetween';
}
```

### 3.5 前端类型定义

```typescript
// resources/assets/js/types.d.ts:250-305
interface SmartPlaylistRuleGroup {
  id: string
  rules: SmartPlaylistRule[]
}

interface SmartPlaylistRule {
  id: string
  model: SmartPlaylistModel
  operator: SmartPlaylistOperator['operator']
  value: any[]
}

interface SmartPlaylistModel {
  name: 'title' | 'length' | 'created_at' | 'album.name' | 'artist.name' | ...
  type: 'text' | 'number' | 'date'
  label: string
  unit?: 'seconds' | 'days'
}

interface SmartPlaylistOperator {
  operator: 'is' | 'isNot' | 'contains' | 'isBetween' | ...
  label: string
  type?: SmartPlaylistModel['type']
  unit?: SmartPlaylistModel['unit']
  inputs?: number
}
```

### 3.6 JSON 存储格式示例

```json
[
  {
    "id": "uuid-group-1",
    "rules": [
      {
        "id": "uuid-rule-1",
        "model": "artist.name",
        "operator": "is",
        "value": ["Beatles"]
      },
      {
        "id": "uuid-rule-2",
        "model": "year",
        "operator": "isGreaterThan",
        "value": [1965]
      }
    ]
  },
  {
    "id": "uuid-group-2",
    "rules": [
      {
        "id": "uuid-rule-3",
        "model": "genre",
        "operator": "is",
        "value": ["Rock"]
      }
    ]
  }
]
```

## 4. 规则条件转换为数据库查询

### 4.1 核心转换类：`SmartPlaylistQueryModifier`

```php
// app/Values/SmartPlaylist/SmartPlaylistQueryModifier.php
final class SmartPlaylistQueryModifier
{
    public static function applyRule(Rule $rule, SongBuilder $query): void
    
    private static function resolveWhereMethod(Rule $rule, Operator $operator): string
    
    private static function generateParameters(Model $model, Operator $operator, array $value): array
    
    private static function generateRawParameters(string $column, Operator $operator, array $value): array
    
    private static function generateEloquentParameters(string $column, Operator $operator, array $value): array
}
```

### 4.2 查询构建流程

在 `SongRepository::getBySmartPlaylist()` 中：

```php
// app/Repositories/SongRepository.php:237-258
private function getBySmartPlaylist(Playlist $playlist, ?User $scopedUser = null): Collection
{
    $query = Song::query(type: PlayableType::SONG, user: $scopedUser ?? $this->auth->user())
        ->withUserContext();

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

**关键点**：
- 第一个规则组使用 `where()`，后续使用 `orWhere()`（组间 OR）
- 组内规则依次调用 `QueryModifier::applyRule()`（组内 AND）
- 结果按标题排序，最多返回 500 条

### 4.3 规则应用逻辑

#### 步骤 1：日期特殊处理

对于日期类型的 `IS` 或 `IS_NOT` 操作，自动转换为范围查询：

```php
// SmartPlaylistQueryModifier.php:29-34
if ($rule->model->isDate() && in_array($operator, [Operator::IS, Operator::IS_NOT], true)) {
    $operator = $operator === Operator::IS ? Operator::IS_BETWEEN : Operator::IS_NOT_BETWEEN;
    $nextDay = Carbon::createFromFormat('Y-m-d', $value[0])->addDay()->format('Y-m-d');
    $value = [$value[0], $nextDay];
}
```

#### 步骤 2：多对多关系处理

对于多对多关系（如 genres），使用 `whereHas` 或 `whereDoesntHave`：

```php
// SmartPlaylistQueryModifier.php:36-62
if ($rule->model->getManyToManyRelation()) {
    $whereHasClause = $rule->operator->isNegative() ? 'whereDoesntHave' : 'whereHas';
    
    // 否定操作符转换为肯定形式在子查询中使用
    $operator = match ($operator) {
        Operator::IS_NOT => Operator::IS,
        Operator::NOT_CONTAIN => Operator::CONTAINS,
        Operator::IS_NOT_BETWEEN => Operator::IS_BETWEEN,
        default => $operator,
    };

    $query->{$whereHasClause}($rule->model->getManyToManyRelation(), 
        static function (Builder $subQuery) use ($rule, $operator, $value): void {
            $subQuery->{self::resolveWhereMethod($rule, $operator)}(
                ...self::generateParameters($rule->model, $operator, $value)
            );
        }
    );
}
```

#### 步骤 3：生成查询参数

根据字段类型和操作符生成对应的 Eloquent 查询参数：

```php
// 普通字段示例
private static function generateEloquentParameters(string $column, Operator $operator, array $value): array
{
    return match ($operator) {
        Operator::BEGINS_WITH => [$column, 'LIKE', "$value[0]%"],
        Operator::CONTAINS => [$column, 'LIKE', "%$value[0]%"],
        Operator::IS => [$column, '=', $value[0]],
        Operator::IS_BETWEEN => [$column, $value],
        Operator::IN_LAST => fn () => [$column, '>=', now()->subDays($value[0])],
        // ... 其他操作符
    };
}
```

### 4.4 操作符映射表

| 操作符 | 对应的查询方法 | 生成的 SQL |
|--------|---------------|-----------|
| `is` | `where` | `column = 'value'` |
| `isNot` | `where` | `column <> 'value'` |
| `contains` | `where` | `column LIKE '%value%'` |
| `notContain` | `where` | `column NOT LIKE '%value%'` |
| `isBetween` | `whereBetween` | `column BETWEEN 'v1' AND 'v2'` |
| `isNotBetween` | `whereNotBetween` | `column NOT BETWEEN 'v1' AND 'v2'` |
| `isGreaterThan` | `where` | `column > 'value'` |
| `isLessThan` | `where` | `column < 'value'` |
| `beginsWith` | `where` | `column LIKE 'value%'` |
| `endsWith` | `where` | `column LIKE '%value'` |
| `inLast` | `where` | `column >= '2024-01-01'` |
| `notInLast` | `where` | `column < '2024-01-01'` |

## 5. 前端展示与交互

### 5.1 组件结构

```
CreateSmartPlaylistForm.vue / EditSmartPlaylistForm.vue
├── useSmartPlaylistForm.ts (组合式函数)
├── SmartPlaylistRuleGroup.vue (规则组组件)
│   └── SmartPlaylistRule.vue (规则组件)
│       └── SmartPlaylistRuleInput.vue (输入控件)
└── playlistStore.ts (状态管理)
```

### 5.2 前端配置

#### 字段配置 (`models.ts`)

```typescript
// resources/assets/js/config/smart-playlist/models.ts
const models: SmartPlaylistModel[] = [
  { name: 'title', type: 'text', label: 'Title' },
  { name: 'album.name', type: 'text', label: 'Album' },
  { name: 'artist.name', type: 'text', label: 'Artist' },
  { name: 'genre', type: 'text', label: 'Genre' },
  { name: 'year', type: 'number', label: 'Year' },
  { name: 'interactions.play_count', type: 'number', label: 'Play Count' },
  { name: 'interactions.last_played_at', type: 'date', label: 'Last Played' },
  { name: 'length', type: 'number', label: 'Length', unit: 'seconds' },
  { name: 'created_at', type: 'date', label: 'Date Added' },
  { name: 'updated_at', type: 'date', label: 'Date Modified' },
]
```

#### 操作符配置 (`operators.ts`)

```typescript
// resources/assets/js/config/smart-playlist/operators.ts
export const is: SmartPlaylistOperator = { operator: 'is', label: 'is' }
export const contains: SmartPlaylistOperator = { operator: 'contains', label: 'contains' }
export const isBetween: SmartPlaylistOperator = { operator: 'isBetween', label: 'is between', inputs: 2 }
export const inLast: SmartPlaylistOperator = { 
  operator: 'inLast', 
  label: 'in the last', 
  type: 'number', 
  unit: 'days' 
}
// ... 其他操作符
```

#### 输入类型映射 (`inputTypes.ts`)

```typescript
// resources/assets/js/config/smart-playlist/inputTypes.ts
const inputTypes: SmartPlaylistInputTypes = {
  text: [is, isNot, contains, notContain, beginsWith, endsWith],
  number: [is, isNot, isGreaterThan, isLessThan, isBetween],
  date: [is, isNot, inLast, notInLast, isBetween],
}
```

### 5.3 规则序列化与反序列化

#### 序列化（提交到后端前）

```typescript
// resources/assets/js/stores/playlistStore.ts:168-182
serializeSmartPlaylistRulesForStorage: (ruleGroups: SmartPlaylistRuleGroup[]) => {
  if (!ruleGroups || !ruleGroups.length) {
    return null
  }

  const serializedGroups = JSON.parse(JSON.stringify(ruleGroups))

  serializedGroups.forEach((group: any): void => {
    group.rules.forEach((rule: any) => {
      rule.model = rule.model.name  // 将对象转换为字符串标识
    })
  })

  return serializedGroups
}
```

#### 反序列化（从后端加载后）

```typescript
// resources/assets/js/stores/playlistStore.ts:60-74
setupSmartPlaylist: (playlist: Playlist) => {
  playlist.rules.forEach(group => {
    group.rules.forEach(rule => {
      const serializedRule = rule as unknown as SerializedSmartPlaylistRule
      const model = models.find(model => model.name === serializedRule.model)

      if (!model) {
        logger.error(`Invalid model ${rule.model} found...`)
        return
      }

      rule.model = model  // 将字符串标识转换为配置对象
    })
  })
}
```

## 6. 后端服务层职责

### 6.1 服务层结构

| 服务类 | 职责范围 |
|--------|---------|
| `PlaylistService` | 手动播放列表 CRUD、歌曲添加/移除、排序、协作功能 |
| `SmartPlaylistService` | 智能播放列表歌曲查询（委托给 SongRepository） |
| `SongRepository` | 实际的查询构建和执行（`getBySmartPlaylist` 方法） |

### 6.2 创建流程

```
PlaylistController@store
  ↓
PlaylistStoreRequest (验证)
  ├─ 验证规则格式 (ValidSmartPlaylistRulePayload)
  └─ 转换为 DTO (PlaylistCreateData)
  ↓
PlaylistService@createPlaylist
  ├─ 创建 Playlist 模型 (包含 rules 字段)
  ├─ 关联所有者用户
  ├─ 处理文件夹关联
  └─ 手动播放列表: 添加歌曲到中间表
  ↓
返回 PlaylistResource
```

### 6.3 歌曲查询流程

```
访问播放列表详情页
  ↓
前端请求 /api/playlists/{id}/songs
  ↓
PlaylistSongController (或直接通过详情接口)
  ↓
SongRepository@getByPlaylist
  ├─ 智能播放列表: getBySmartPlaylist()
  │   └─ SmartPlaylistQueryModifier::applyRule()
  │       └─ 动态构建 SQL 查询
  └─ 手动播放列表: getByStandardPlaylist()
      └─ 查询 playlist_song 中间表
  ↓
返回歌曲集合
```

### 6.4 关键代码片段

#### 创建播放列表

```php
// app/Services/Playlist/PlaylistService.php:29-61
public function createPlaylist(PlaylistCreateData $data, User $user): Playlist
{
    return DB::transaction(function () use ($data, $cover, $user): Playlist {
        $playlist = Playlist::query()->create([
            'name' => $data->name,
            'description' => $data->description,
            'rules' => $data->ruleGroups,  // 自动通过 SmartPlaylistRulesCast 转换
            'cover' => $cover,
        ]);

        $user->ownedPlaylists()->attach($playlist, ['role' => 'owner']);

        if (!$playlist->is_smart && $data->playableIds) {
            $playlist->addPlayables($data->playableIds, $user);
        }

        return $playlist;
    });
}
```

#### 规则 Cast 类

```php
// app/Casts/SmartPlaylistRulesCast.php
class SmartPlaylistRulesCast implements CastsAttributes
{
    public function get($model, string $key, $value, array $attributes): ?SmartPlaylistRuleGroupCollection
    {
        return $value
            ? rescue(static fn () => SmartPlaylistRuleGroupCollection::create(json_decode($value, true)))
            : null;
    }

    public function set($model, string $key, $value, array $attributes): ?string
    {
        if (is_array($value)) {
            $value = SmartPlaylistRuleGroupCollection::create($value);
        }

        return $value?->toJson() ?? null;
    }
}
```

## 7. 前后端职责分工总览

### 7.1 前端职责

| 职责 | 实现位置 |
|------|---------|
| 规则配置定义 | `config/smart-playlist/*.ts` |
| 表单 UI 渲染 | `components/playlist/smart-playlist/*.vue` |
| 规则组/规则的增删改 | `useSmartPlaylistForm.ts` |
| 输入验证（基于 HTML5） | 各 Input 组件 |
| 规则序列化（提交前） | `playlistStore.ts` |
| 规则反序列化（加载后） | `playlistStore.ts` |
| 播放列表状态管理 | `playlistStore.ts` |
| API 调用 | `http` 服务 |

### 7.2 后端职责

| 职责 | 实现位置 |
|------|---------|
| HTTP 路由与控制器 | `PlaylistController.php` |
| 请求验证 | `PlaylistStoreRequest.php` + `ValidSmartPlaylistRulePayload.php` |
| 业务逻辑编排 | `PlaylistService.php` |
| 规则模型化（值对象） | `Values/SmartPlaylist/*.php` |
| 规则到查询的转换 | `SmartPlaylistQueryModifier.php` |
| 查询执行 | `SongRepository.php` |
| 数据持久化 | `Playlist` 模型 + `SmartPlaylistRulesCast` |
| API 资源转换 | `PlaylistResource.php` |

### 7.3 数据流动图

```
用户操作 (前端)
  ↓
表单收集规则数据 → 序列化 (model 对象 → name 字符串)
  ↓
HTTP POST /api/playlists { rules: [...] }
  ↓
后端验证 (ValidSmartPlaylistRulePayload)
  ↓
转换为值对象 (SmartPlaylistRuleGroupCollection)
  ↓
JSON 序列化存储到 playlists.rules 字段
  ↓
查询时:
  读取 rules 字段 → JSON 反序列化 → 转换为值对象
  ↓
SmartPlaylistQueryModifier 应用规则构建查询
  ↓
执行查询返回歌曲列表
  ↓
前端接收数据 → 反序列化 (name 字符串 → model 对象)
  ↓
渲染歌曲列表
```

## 8. 关键设计亮点

1. **单一职责原则**：规则建模、查询构建、服务逻辑分层清晰
2. **值对象模式**：使用不可变的值对象封装规则，提高类型安全
3. **枚举驱动**：字段和操作符使用枚举定义，避免魔法字符串
4. **Cast 自动转换**：通过 Eloquent Cast 自动处理 JSON 和值对象的转换
5. **灵活的规则组合**：支持组间 OR、组内 AND 的灵活逻辑组合
6. **前后端配置同步**：字段和操作符在前后端都有对应的配置定义
7. **统一的查询入口**：`SongRepository::getByPlaylist()` 统一处理两种播放列表类型

## 9. 相关文件索引

### 后端核心文件

| 文件路径 | 说明 |
|---------|------|
| `app/Models/Playlist.php` | 播放列表模型 |
| `app/Enums/SmartPlaylistModel.php` | 字段模型枚举 |
| `app/Enums/SmartPlaylistOperator.php` | 操作符枚举 |
| `app/Values/SmartPlaylist/SmartPlaylistRule.php` | 规则值对象 |
| `app/Values/SmartPlaylist/SmartPlaylistRuleGroup.php` | 规则组值对象 |
| `app/Values/SmartPlaylist/SmartPlaylistRuleGroupCollection.php` | 规则组集合 |
| `app/Values/SmartPlaylist/SmartPlaylistQueryModifier.php` | 规则转查询转换器 |
| `app/Casts/SmartPlaylistRulesCast.php` | 规则字段 Cast |
| `app/Services/Playlist/PlaylistService.php` | 播放列表服务 |
| `app/Services/Playlist/SmartPlaylistService.php` | 智能播放列表服务 |
| `app/Repositories/SongRepository.php` | 歌曲查询仓库 |
| `app/Http/Requests/API/Playlist/PlaylistStoreRequest.php` | 创建请求验证 |
| `app/Rules/ValidSmartPlaylistRulePayload.php` | 规则格式验证 |

### 前端核心文件

| 文件路径 | 说明 |
|---------|------|
| `resources/assets/js/types.d.ts` | TypeScript 类型定义 |
| `resources/assets/js/stores/playlistStore.ts` | 播放列表状态管理 |
| `resources/assets/js/composables/useSmartPlaylistForm.ts` | 智能表单组合式函数 |
| `resources/assets/js/config/smart-playlist/models.ts` | 字段配置 |
| `resources/assets/js/config/smart-playlist/operators.ts` | 操作符配置 |
| `resources/assets/js/config/smart-playlist/inputTypes.ts` | 输入类型映射 |
| `resources/assets/js/components/playlist/smart-playlist/*.vue` | UI 组件 |
