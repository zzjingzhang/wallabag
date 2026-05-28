# 网页导出与 API 单篇导出如何复用 EntriesExport

## 1. 两条调用入口概览

| 维度 | 网页批量导出 | API 单篇导出 |
|------|-------------|-------------|
| 控制器 | [ExportController](src/Controller/ExportController.php) | [EntryRestController](src/Controller/Api/EntryRestController.php) |
| 路由 | `/export/{category}.{format}` | `/api/entries/{entry}/export.{_format}` |
| 方法 | `downloadEntriesAction` | `getEntryExportAction` |
| 数据来源 | 根据 category 动态查询 EntryRepository | 直接通过 `{entry}` 参数由 ParamConverter 注入单个 Entry 实体 |
| 核心调用链 | `setEntries($entries)` → `updateTitle($title)` → `updateAuthor($method)` → `exportAs($format)` | `setEntries($entry)` → `updateTitle('entry')` → `updateAuthor('entry')` → `exportAs($format)` |

两条路径最终都汇聚到同一个 [EntriesExport](src/Helper/EntriesExport.php) 服务，通过一致的 Fluent API 完成导出。

---

## 2. ExportController::downloadEntriesAction 的 category 动态分发

在 [downloadEntriesAction](src/Controller/ExportController.php#L48-L106) 中，`category` 参数受路由约束限制为 `all|unread|starred|archive|tag_entries|untagged|search|annotated|same_domain`。

### 2.1 约定优于配置：动态方法名拼接

```php
$method = ucfirst($category);               // 例: 'all' → 'All'
$methodBuilder = 'getBuilderFor' . $method . 'ByUser'; // → 'getBuilderForAllByUser'
```

对于 `all`、`unread`、`starred`、`archive`、`untagged` 这五种 category，[EntryRepository](src/Repository/EntryRepository.php) 中都存在对应的 `getBuilderFor{Category}ByUser` 方法：

| category | 拼接后的方法 | EntryRepository 实现 |
|----------|-------------|---------------------|
| `all` | `getBuilderForAllByUser` | [L34-L39](src/Repository/EntryRepository.php#L34-L39) |
| `unread` | `getBuilderForUnreadByUser` | [L62-L68](src/Repository/EntryRepository.php#L62-L68) |
| `starred` | `getBuilderForStarredByUser` | [L149-L155](src/Repository/EntryRepository.php#L149-L155) |
| `archive` | `getBuilderForArchiveByUser` | [L119-L125](src/Repository/EntryRepository.php#L119-L125) |
| `untagged` | `getBuilderForUntaggedByUser` | [L212-L215](src/Repository/EntryRepository.php#L212-L215) |

这些方法返回 `QueryBuilder`，控制器再调用 `->getQuery()->getResult()` 获取 Entry 数组。

### 2.2 四种特殊 category 的单独处理

以下四种 category **无法**通过简单的 `getBuilderFor{X}ByUser` 拼接来调用，需要额外参数或不同的查询方式：

#### 2.2.1 `same_domain`

```php
$entries = $entryRepository->getBuilderForSameDomainByUser(
    $this->getUser()->getId(),
    $request->query->getInt('entry')  // 需要额外 entry ID 参数
)->getQuery()->getResult();
$title = 'Same domain';
```

[getBuilderForSameDomainByUser](src/Repository/EntryRepository.php#L93-L110) 需要第二个参数 `$entryId`，通过子查询找到与指定条目同域名的所有文章。简单拼接的 `getBuilderForSameDomainByUser($userId)` 缺少必要参数，因此不能走通用路径。

#### 2.2.2 `tag_entries`

```php
$tag = $tagRepository->findOneBySlug($request->query->get('tag'));
$entries = $entryRepository->findAllByTagId(
    $this->getUser()->getId(),
    $tag->getId()  // 需要 Tag 实体 ID
);
$title = 'Tag ' . $tag->getLabel();
```

[findAllByTagId](src/Repository/EntryRepository.php#L484-L491) 需要 `$tagId` 参数，且需先通过 [TagRepository](src/Repository/TagRepository.php) 的 `findOneBySlug` 获取 Tag 实体。方法签名也不符合 `getBuilderFor{X}ByUser` 的命名约定。

#### 2.2.3 `search`

```php
$searchTerm = $request->query->all('search_entry')['term'] ?? '';
$currentRoute = $request->query->get('currentRoute') ?? '';
$entries = $entryRepository->getBuilderForSearchByUser(
    $this->getUser()->getId(),
    $searchTerm,     // 搜索关键词
    $currentRoute    // 当前路由 (starred/unread/archive 等)
)->getQuery()->getResult();
$title = 'Search ' . $searchTerm;
```

[getBuilderForSearchByUser](src/Repository/EntryRepository.php#L181-L203) 需要 `$term` 和 `$currentRoute` 两个额外参数，并在查询中对 `content`、`title`、`url`、`annotation.text` 进行 `LIKE` 搜索，还会根据 `$currentRoute` 叠加状态过滤。

#### 2.2.4 `annotated`

```php
$entries = $entryRepository->getBuilderForAnnotationsByUser(
    $this->getUser()->getId()
)->getQuery()->getResult();
$title = 'With annotations';
```

[getBuilderForAnnotationsByUser](src/Repository/EntryRepository.php#L224-L230) 方法名符合拼接规则 `getBuilderForAnnotatedByUser`，但实际上 Repository 中的方法命名为 `getBuilderForAnnotationsByUser`（复数 Annotations），而 category 值为 `annotated`（单数），拼接会得到 `getBuilderForAnnotatedByUser`，与实际方法名不匹配。因此必须单独处理。

### 2.3 通用 fallback 路径

```php
} else {
    $entries = $entryRepository
        ->$methodBuilder($this->getUser()->getId())
        ->getQuery()
        ->getResult();
}
```

对于 `all`、`unread`、`starred`、`archive`、`untagged`，控制器使用 PHP 的可变方法调用 `$entryRepository->$methodBuilder(...)`，这是典型的约定优于配置模式。

---

## 3. API 单篇导出：EntryRestController::getEntryExportAction

在 [getEntryExportAction](src/Controller/Api/EntryRestController.php#L446-L455) 中：

```php
#[Route(path: '/api/entries/{entry}/export.{_format}', name: 'api_get_entry_export', methods: ['GET'], defaults: ['_format' => 'json'])]
public function getEntryExportAction(Entry $entry, Request $request, EntriesExport $entriesExport)
{
    return $entriesExport
        ->setEntries($entry)
        ->updateTitle('entry')
        ->updateAuthor('entry')
        ->exportAs($request->attributes->get('_format'));
}
```

关键差异：
- **数据获取**：通过 Symfony ParamConverter 自动将 `{entry}` 转换为 `Entry` 实体，无需查询 Repository
- **format 来源**：从 FOSRest 的 `_format` 属性获取（由格式监听器解析），而非路由参数
- **title/author**：固定传入 `'entry'` 字符串，触发 EntriesExport 中单篇条目的特殊处理逻辑

---

## 4. EntriesExport 的 Fluent API 内部机制

### 4.1 setEntries — 统一单篇/批量为数组

[setEntries](src/Helper/EntriesExport.php#L48-L58) 接受 `array|Entry` 类型参数：

```php
public function setEntries($entries)
{
    if (!\is_array($entries)) {
        $this->language = $entries->getLanguage();  // 仅单篇时设置语言
        $entries = [$entries];                       // 包装为数组
    }
    $this->entries = $entries;
    return $this;
}
```

- **单篇**（API 导出 / 网页单篇导出）：传入 Entry 对象 → 提取 `language` 后包装为 `[Entry]`
- **批量**（网页分类导出）：传入 `Entry[]` → 直接赋值，`language` 保持空字符串

`language` 字段在 `produceEpub` 中用于设置电子书的 BCP47 语言标识。

### 4.2 updateTitle — 区分单篇标题与分类标题

[updateTitle](src/Helper/EntriesExport.php#L67-L76)：

```php
public function updateTitle($method)
{
    $this->title = $method . ' articles';     // 默认: "All articles", "Unread articles" 等
    if ('entry' === $method) {
        $this->title = $this->entries[0]->getTitle();  // 单篇: 使用文章标题
    }
    return $this;
}
```

| 调用场景 | `$method` 值 | 最终 title |
|---------|-------------|-----------|
| API 单篇导出 | `'entry'` | 文章原始标题 |
| 网页单篇导出 | `'entry'` | 文章原始标题 |
| 批量 `all` | `'All'` | `"All articles"` |
| 批量 `same_domain` | `'Same domain'` | `"Same domain articles"` |
| 批量 `tag_entries` | `'Tag PHP'` | `"Tag PHP articles"` |
| 批量 `search` | `'Search keyword'` | `"Search keyword articles"` |
| 批量 `annotated` | `'With annotations'` | `"With annotations articles"` |

`title` 最终用于：
- EPub/PDF 的文档元数据
- 导出文件名（经 [getSanitizedFilename](src/Helper/EntriesExport.php#L494-L499) 转换）

### 4.3 updateAuthor — 区分单篇作者与合集作者

[updateAuthor](src/Helper/EntriesExport.php#L87-L103)：

```php
public function updateAuthor($method)
{
    if ('entry' !== $method) {
        $this->author = 'Various authors';    // 批量: 固定值
        return $this;
    }
    $this->author = $this->entries[0]->getDomainName(); // 单篇默认: 域名
    $publishedBy = $this->entries[0]->getPublishedBy();
    if (!empty($publishedBy)) {
        $this->author = implode(', ', $publishedBy);    // 有作者信息: 逗号拼接
    }
    return $this;
}
```

| 场景 | author 值 |
|------|----------|
| 单篇且 `publishedBy` 非空 | 如 `"John Doe, Jane Smith"` |
| 单篇且 `publishedBy` 为空 | 域名如 `"example.com"` |
| 任何批量导出 | `"Various authors"` |

### 4.4 exportAs — 策略模式格式分发

[exportAs](src/Helper/EntriesExport.php#L112-L120)：

```php
public function exportAs($format)
{
    $functionName = 'produce' . ucfirst($format);
    if (method_exists($this, $functionName)) {
        return $this->$functionName();
    }
    throw new \InvalidArgumentException(...);
}
```

通过 `"produce" + ucfirst($format)` 的命名约定，将格式字符串映射到私有方法：

| format | 方法 | 实现方式 | Content-Type |
|--------|------|---------|-------------|
| `epub` | [produceEpub](src/Helper/EntriesExport.php#L130-L246) | PHPePub 库生成 EPUB3 | `application/epub+zip` |
| `pdf` | [producePdf](src/Helper/EntriesExport.php#L251-L324) | TCPDF 生成 PDF | `application/pdf` |
| `csv` | [produceCsv](src/Helper/EntriesExport.php#L329-L369) | PHP `fputcsv` 手工写行 | `application/csv` |
| `json` | [produceJson](src/Helper/EntriesExport.php#L374-L385) | JMS Serializer + `entries_for_user` group | `application/json` |
| `xml` | [produceXml](src/Helper/EntriesExport.php#L390-L401) | JMS Serializer + `entries_for_user` group | `application/xml` |
| `txt` | [produceTxt](src/Helper/EntriesExport.php#L406-L425) | Html2Text 纯文本转换 | `text/plain` |
| `md` | [produceMd](src/Helper/EntriesExport.php#L430-L448) | HTMLToMarkdown 转换 | `text/markdown` |

**关键区别**：
- `produceJson` / `produceXml` 内部调用 [prepareSerializingContent](src/Helper/EntriesExport.php#L457-L466)，使用 JMS Serializer 并指定序列化 group
- `produceEpub` / `producePdf` / `produceCsv` / `produceTxt` / `produceMd` 则是手工从 Entry 对象提取字段，**不经过 JMS 序列化**

---

## 5. JMS 序列化 Groups 如何影响 JSON/XML 导出字段

### 5.1 EntriesExport 中的序列化上下文

[prepareSerializingContent](src/Helper/EntriesExport.php#L457-L466)：

```php
private function prepareSerializingContent($format)
{
    $serializer = SerializerBuilder::create()->build();
    return $serializer->serialize(
        $this->entries,
        $format,
        SerializationContext::create()->setGroups(['entries_for_user'])
    );
}
```

**所有通过 `produceJson` 和 `produceXml` 的导出**，无论是单篇还是批量，都使用 `entries_for_user` 这一个序列化 Group。

### 5.2 Entry 实体中 entries_for_user Group 覆盖的字段

在 [Entry](src/Entity/Entry.php) 实体中，标注了 `#[Groups(['entries_for_user', 'export_all'])]` 的字段如下：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | int | 条目 ID |
| `uid` | string | 公共链接唯一标识 |
| `title` | string | 文章标题 |
| `url` | string | 文章 URL |
| `origin_url` | string | 来源 URL |
| `given_url` | string | 用户输入的原始 URL |
| `is_archived` | int (VirtualProperty) | 归档状态，通过 [is_Archived()](src/Entity/Entry.php#L376-L382) 虚拟属性输出为整数 |
| `is_starred` | int (VirtualProperty) | 收藏状态，通过 [is_Starred()](src/Entity/Entry.php#L415-L421) 虚拟属性输出为整数 |
| `archived_at` | DateTime | 归档时间 |
| `starred_at` | DateTime | 收藏时间 |
| `content` | text | 文章内容 |
| `created_at` | DateTime | 创建时间 |
| `updated_at` | DateTime | 更新时间 |
| `published_at` | DateTime | 发布时间 |
| `published_by` | array | 作者列表 |
| `annotations` | Collection | 注解集合 |
| `mimetype` | string | MIME 类型 |
| `language` | string | 语言 |
| `reading_time` | int | 阅读时间 |
| `domain_name` | string | 域名 |
| `preview_picture` | string | 预览图 |
| `http_status` | string | HTTP 状态码 |
| `headers` | array | HTTP 头信息 |
| `is_not_parsed` | bool (VirtualProperty via Exclude+Groups) | 未解析状态 |
| `is_public` | bool (VirtualProperty) | 仅在 `entries_for_user` group 中，不在 `export_all` 中 |

**被排除的字段**：
- `user`：标注 `#[Exclude]` + `#[Groups(['export_all'])]`，在 `entries_for_user` 下不输出
- `hashedUrl` / `hashedGivenUrl`：无 Groups 注解，不参与序列化

### 5.3 Annotation 实体

[Annotation](src/Entity/Annotation.php) 使用 `#[ExclusionPolicy('none')]`（默认全暴露），但其中 `user` 和 `entry` 字段标注了 `#[Exclude]`。在 `entries_for_user` group 中会输出的注解字段：

| 字段 | Groups |
|------|--------|
| `text` | `entries_for_user`, `export_all` |
| `quote` | `entries_for_user`, `export_all` |
| `ranges` | `entries_for_user`, `export_all` |

`id`、`created_at`、`updated_at` 无 Groups 限制，因 ExclusionPolicy 为 none 也会被序列化。

### 5.4 Tag 实体

[Tag](src/Entity/Tag.php) 使用 `#[ExclusionPolicy('all')]` + `#[Expose]`，仅暴露 `id`、`label`、`slug`。Tag 不依赖 Groups 机制，只要被序列化就输出这三个字段。

但在 Entry 中，tags 的序列化是通过 VirtualProperty [getSerializedTags](src/Entity/Entry.php#L632-L643) 实现的：

```php
#[VirtualProperty]
#[SerializedName('tags')]
#[Groups(['entries_for_user', 'export_all'])]
public function getSerializedTags()
{
    $data = [];
    foreach ($this->tags as $tag) {
        $data[] = $tag->getLabel();  // 仅返回标签文本数组
    }
    return $data;
}
```

因此在 JSON/XML 导出中，`tags` 字段输出的是字符串数组 `["tag1", "tag2"]`，而非完整的 Tag 对象。

### 5.5 entries_for_user 与 export_all 的差异

| 字段 | entries_for_user | export_all | 说明 |
|------|:---:|:---:|------|
| `is_public` | ✅ | ❌ | 仅 entries_for_user 暴露 |
| `user` (完整对象) | ❌ | ✅ | 仅 export_all 暴露 |

`export_all` group 在当前代码中**未在导出流程中使用**，它可能是预留给全量数据迁移/备份场景的。

---

## 6. FOSRest 对 /api/entries/{id}/export 格式监听规则的特殊配置

### 6.1 配置内容

在 [config.yml](app/config/config.yml#L107-L140) 中：

```yaml
fos_rest:
    format_listener:
        enabled: true
        rules:
            - { path: "^/api/entries/([0-9]+)/export.(.*)", priorities: ['epub', 'pdf', 'txt', 'csv'], fallback_format: json, prefer_extension: false }
            - { path: "^/api", priorities: ['json', 'xml'], fallback_format: json, prefer_extension: false }
            - { path: "^/annotations", priorities: ['json', 'xml'], fallback_format: json, prefer_extension: false }
            - { path: '^/', priorities: ['text/html', '*/*'], fallback_format: html, prefer_extension: false }
```

### 6.2 为什么需要特殊配置

#### 问题一：通用 API 规则不支持二进制格式

第二条规则 `^/api` 的 priorities 为 `['json', 'xml']`，这意味着所有 `/api` 路径的请求默认只协商 JSON 和 XML 格式。如果 `/api/entries/123/export.epub` 走这条规则，EPub/PDF/TXT/CSV 格式将无法被识别，FOSRest 会回退到 `fallback_format: json`，导致 `exportAs('json')` 被调用而非 `exportAs('epub')`。

#### 问题二：FOSRest FormatListener 的全局拦截

配置注释中明确指出：

> for an unknown reason, EACH REQUEST goes to FOS\RestBundle\EventListener\FormatListener
> so we need to add custom rule for custom api export but also for all other routes of the application...

FOSRest 的 FormatListener 会拦截**所有**请求（不仅仅是 API 请求），因此必须为非 API 路径添加第四条规则 `'/' → html`，否则普通网页请求也会被强制按 JSON 处理。

#### 问题三：导出端点的格式需求与常规 API 完全不同

| 维度 | 常规 API (`/api/entries`, `/api/tags` 等) | 导出 API (`/api/entries/{id}/export.{_format}`) |
|------|------------------------------------------|-----------------------------------------------|
| 优先格式 | `json`, `xml` | `epub`, `pdf`, `txt`, `csv` |
| 响应类型 | 结构化数据 | 二进制文件/附件 |
| fallback | `json` | `json`（兜底） |
| 扩展名优先 | `prefer_extension: false` | `prefer_extension: false` |

### 6.3 规则匹配顺序的关键性

FOSRest 的 FormatListener 按声明顺序匹配，**第一条匹配的规则生效**。因此导出规则必须放在通用 API 规则之前：

1. `^/api/entries/([0-9]+)/export.(.*)` — 导出端点（epub/pdf/txt/csv 优先）
2. `^/api` — 常规 API（json/xml 优先）
3. `^/annotations` — 注解 API（json/xml 优先）
4. `^/` — 其余所有路径（html 优先）

如果顺序颠倒，`^/api` 会先匹配到 `/api/entries/123/export.epub`，导致格式协商失败。

### 6.4 `prefer_extension: false` 的含义

设置 `prefer_extension: false` 意味着不会仅凭 URL 中的扩展名来决定格式，而是结合 `Accept` 请求头和 priorities 列表综合判断。但实际使用中，客户端通常通过 URL 扩展名（如 `.epub`）指定格式，FOSRest 仍然会从 `_format` 路由参数中读取。这个 `false` 值主要是防止扩展名覆盖 `Accept` 头的协商结果。

### 6.5 路由定义与 FOSRest 的协作

[getEntryExportAction](src/Controller/Api/EntryRestController.php#L446) 的路由定义为：

```php
#[Route(path: '/api/entries/{entry}/export.{_format}', defaults: ['_format' => 'json'])]
```

FOSRest 的 FormatListener 解析请求后，会将协商得出的格式写入 `$request->attributes->get('_format')`，控制器再将其传给 `exportAs()`。配置中的正则 `^/api/entries/([0-9]+)/export.(.*)` 确保只有单篇导出路径命中此规则，而不会误匹配 `/api/entries/{entry}` 的 GET/PATCH/DELETE 等常规操作。

---

## 7. 复用模式总结

```
┌─────────────────────────────────┐     ┌──────────────────────────────────┐
│  ExportController (Web)         │     │  EntryRestController (API)       │
│                                 │     │                                  │
│  downloadEntryAction            │     │  getEntryExportAction            │
│    ↓ Entry (单实体)              │     │    ↓ Entry (ParamConverter)      │
│                                 │     │                                  │
│  downloadEntriesAction          │     │                                  │
│    ↓ 根据 category 查询         │     │                                  │
│      same_domain → 额外参数     │     │                                  │
│      tag_entries → TagRepository│     │                                  │
│      search → 搜索词+路由       │     │                                  │
│      annotated → 方法名不匹配   │     │                                  │
│      其他 → 动态方法拼接        │     │                                  │
└──────────────┬──────────────────┘     └──────────────┬───────────────────┘
               │                                        │
               │    setEntries() / updateTitle()        │
               │    updateAuthor() / exportAs()         │
               └────────────────┬───────────────────────┘
                                │
                    ┌───────────▼───────────┐
                    │    EntriesExport       │
                    │                       │
                    │  produceEpub  (PHPePub)│
                    │  producePdf   (TCPDF)  │
                    │  produceCsv   (fputcsv)│
                    │  produceJson  (JMS)   │ ← entries_for_user group
                    │  produceXml   (JMS)   │ ← entries_for_user group
                    │  produceTxt   (Html2Text)│
                    │  produceMd    (HtmlConverter)│
                    └───────────────────────┘
```

**核心复用要点**：
1. **EntriesExport 是无状态服务**（每次调用通过 Fluent API 设置临时状态），可安全地在多个控制器间共享
2. **setEntries 的多态接受**（`array|Entry`）是复用的关键——单篇/批量在入口处统一为数组
3. **`'entry'` 作为魔法字符串**，在 `updateTitle` 和 `updateAuthor` 中触发单篇特殊逻辑
4. **exportAs 的策略模式**让格式选择与数据准备完全解耦，新增格式只需添加 `produceXxx` 方法
5. **JMS Groups 的统一**确保 JSON/XML 导出无论来源如何，字段集一致且可控
