# Wallabag 批量接口创建与删除流程分析

## 1. 接口路由总览

| 接口 | HTTP 方法 | 路由名称 | 控制器方法 | 权限注解 |
|------|----------|---------|-----------|---------|
| `/api/entries/lists.{_format}` | POST | `api_post_entries_list` | [postEntriesListAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L538) | `CREATE_ENTRIES` |
| `/api/entries/list.{_format}` | DELETE | `api_delete_entries_list` | [deleteEntriesListAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L480) | `DELETE_ENTRIES` |

两个接口路径仅差一个 `s`（`lists` vs `list`），但语义完全不同：`lists` 用于批量创建，`list` 用于批量删除。

---

## 2. `api_limit_mass_actions` 如何限制创建数量

### 2.1 配置定义

在 [wallabag.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/wallabag.yml#L38) 中定义了默认值：

```yaml
wallabag.api_limit_mass_actions: 10
```

### 2.2 注入方式

该值通过 [services.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/services.yml#L30) 注入到控制器构造函数：

```yaml
$apiLimitMassActions: "%wallabag.api_limit_mass_actions%"
```

在 [WallabagRestController](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/WallabagRestController.php#L32) 中作为 `int $apiLimitMassActions` 属性接收。

### 2.3 限制逻辑

在 [postEntriesListAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L542-L544) 中，**在遍历 URL 之前**进行一次性校验：

```php
if (\count($urls) > $this->apiLimitMassActions) {
    throw new BadRequestHttpException('API limit reached');
}
```

**关键特征：**

- 校验是**前置的**——先检查数组长度，再处理任何 URL，超过限制直接抛出 `BadRequestHttpException`，不会处理任何条目
- 限制值为硬编码的参数默认 `10`，意味着单次批量请求最多处理 10 个 URL
- **仅对创建接口生效**——`deleteEntriesListAction` 没有对应的数量限制检查
- 校验使用 `>`（严格大于），即恰好 10 个 URL 是允许的

### 2.4 设计意图

防止客户端一次性提交大量 URL 导致：
- 服务端长时间阻塞在内容抓取（Graby HTTP 请求）上
- 数据库连接和事务长时间占用
- 图片下载等副作用操作成倍放大

---

## 3. `deleteEntriesListAction` 为什么逐个查找并检查 DELETE 权限

### 3.1 完整流程

[deleteEntriesListAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L480-L511) 的核心逻辑：

```php
foreach ($urls as $key => $url) {
    $entry = $entryRepository->findByUrlAndUserId($url, $this->getUser()->getId());

    $results[$key]['url'] = $url;

    if (false !== $entry && $this->authorizationChecker->isGranted('DELETE', $entry)) {
        $eventDispatcher->dispatch(new EntryDeletedEvent($entry), EntryDeletedEvent::NAME);
        $this->entityManager->remove($entry);
        $this->entityManager->flush();
    }

    $results[$key]['entry'] = $entry instanceof Entry ? true : false;
}
```

### 3.2 逐个查找的原因

[findByUrlAndUserId](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L502-L508) 内部调用了 [findByHashedUrlAndUserId](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L535-L560)，该方法执行两次数据库查询：

1. 先查 `hashedUrl` 索引（利用数据库索引 `user_id + hashed_url`）
2. 未找到再查 `hashedGivenUrl` 索引（利用数据库索引 `user_id + hashed_given_url`）

**为什么不批量查询？** 虽然项目中存在 [findByUserIdAndBatchHashedUrls](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L562-L576) 批量查询方法（在 `getEntriesExistsAction` 中使用），但 `deleteEntriesListAction` 没有采用它。原因是：

- 批量查询返回的是轻量级 `SELECT e.id, e.hashedUrl, e.hashedGivenUrl`，不包含完整的 `Entry` 实体
- 删除操作需要完整的 `Entry` 实体对象来执行 `$entityManager->remove()` 和派发 `EntryDeletedEvent`
- 权限检查需要 `Entry` 实体的 `getUser()` 方法来判断归属

### 3.3 逐个权限检查的原因

虽然接口级 `#[IsGranted('DELETE_ENTRIES')]` 确保了用户具有删除条目的**类级别**权限（只需 `ROLE_USER`，见 [MainVoter](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Security/Voter/MainVoter.php#L44-L45)），但**对象级别**的 DELETE 权限由 [EntryVoter](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Security/Voter/EntryVoter.php#L50-L53) 控制：

```php
self::DELETE => $user === $subject->getUser(),
```

即只有条目的**所有者**才能删除。逐个检查是因为：

1. **安全防护**：即使在同一用户的 API 请求中，也可能存在 URL 对应的条目属于其他用户（理论上不可能，但作为纵深防御）
2. **空结果处理**：`findByUrlAndUserId` 可能返回 `false`（URL 对应条目不存在），此时不应执行删除
3. **单条失败不影响其他**：某个 URL 查找失败或权限不足，不会中断整个批量操作

### 3.4 逐条 flush 的问题

循环内每次删除都执行 `$entityManager->flush()`，这意味着 N 个 URL 就产生 N 次 `DELETE` SQL 语句执行。对比 [EntryController](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/EntryController.php#L124-L129) 中批量删除的做法——先循环 `remove`，最后统一 `flush`：

```php
// EntryController 的做法
foreach ($entries as $entry) {
    $this->eventDispatcher->dispatch(new EntryDeletedEvent($entry), EntryDeletedEvent::NAME);
    $this->entityManager->remove($entry);
}
$this->entityManager->flush();
```

批量接口中逐条 flush 是一个**潜在的性能问题**，但存在一个隐含原因：`EntryDeletedEvent` 的订阅者 `DownloadImagesSubscriber` 会执行文件系统操作（删除图片），逐条 flush 确保数据库删除与文件系统删除的原子性——如果某条删除失败，之前的已经完成。

---

## 4. `postEntriesListAction` 对已存在 Entry 和新 Entry 的不同处理

### 4.1 完整流程

[postEntriesListAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L538-L576)：

```php
foreach ($urls as $key => $url) {
    $entry = $entryRepository->findByUrlAndUserId($url, $this->getUser()->getId());

    $results[$key]['url'] = $url;

    if (false === $entry) {
        // 新 Entry 路径
        $entry = new Entry($this->getUser());
        $contentProxy->updateEntry($entry, $url);
    }
    // 已存在 Entry: 不执行任何内容更新

    $this->entityManager->persist($entry);
    $this->entityManager->flush();

    $results[$key]['entry'] = $entry->getId();

    $eventDispatcher->dispatch(new EntrySavedEvent($entry), EntrySavedEvent::NAME);
}
```

### 4.2 两种路径对比

| 维度 | 新 Entry（`$entry === false`） | 已存在 Entry（`$entry !== false`） |
|------|------|------|
| 实体创建 | `new Entry($this->getUser())` | 使用已有实体 |
| 内容抓取 | `ContentProxy::updateEntry()` 触发 Graby 抓取 | **不调用** `ContentProxy`，内容不更新 |
| 持久化 | `persist()` + `flush()` | `persist()` + `flush()`（实际是 update） |
| 事件派发 | `EntrySavedEvent` | `EntrySavedEvent`（**同样派发**） |
| 返回值 | 返回新 Entry 的 ID | 返回已有 Entry 的 ID |

### 4.3 关键差异：已存在 Entry 不更新内容

这是与单个条目创建接口 [postEntriesAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L717-L811) 的重大区别。在 `postEntriesAction` 中：

```php
if (false === $entry) {
    $entry = new Entry($this->getUser());
    $entry->setUrl($url);
}
// 无论新旧，都会执行 contentProxy->updateEntry()
$contentProxy->updateEntry($entry, $entry->getUrl(), [...]);
```

单个接口中，**已存在的条目也会被重新抓取内容**。但批量接口中，已存在的条目只是 `persist` + `flush`（相当于无变更的 UPDATE），不会重新抓取。

### 4.4 设计考量

- 批量接口定位为"快速收藏"，而非"内容刷新"
- 内容抓取是耗时操作（Graby 需要发 HTTP 请求获取远程页面），批量场景下对已存在的条目再抓取会导致请求超时
- 已存在条目仍然派发 `EntrySavedEvent`，这会触发 `DownloadImagesSubscriber`（见第 6 节），但此时 entry 的内容未变，`DownloadImages` 的 `processHtml` 会重新处理 HTML 中的图片——**这是一次无意义的重复操作**

---

## 5. `EntrySavedEvent` 和 `EntryDeletedEvent` 的派发时机

### 5.1 事件定义

| 事件 | 常量名 | 值 | 定义位置 |
|------|--------|-----|---------|
| `EntrySavedEvent` | `NAME` | `entry.saved` | [EntrySavedEvent.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Event/EntrySavedEvent.php#L13) |
| `EntryDeletedEvent` | `NAME` | `entry.deleted` | [EntryDeletedEvent.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Event/EntryDeletedEvent.php#L13) |

两个事件都携带一个 `Entry` 实体，通过 `getEntry()` 获取。

### 5.2 在批量创建接口中的派发

[postEntriesListAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L566-L572)：

```
foreach 每个 URL:
  1. 查找/创建 Entry
  2. 新 Entry: ContentProxy->updateEntry()
  3. persist + flush          ← Entry 已有 ID
  4. dispatch EntrySavedEvent ← 在 flush 之后
```

**关键点：** 事件在 `flush()` 之后派发。这确保了 `Entry` 已有数据库 ID，`DownloadImagesSubscriber` 可以用此 ID 创建图片存储路径。对比 [AbstractImport](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Import/AbstractImport.php#L160-L162) 中的注释：

> entry.saved needs the entry to be persisted in db because it needs its id to generate images (at least)

### 5.3 在批量删除接口中的派发

[deleteEntriesListAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L499-L504)：

```
foreach 每个 URL:
  1. 查找 Entry
  2. 权限检查
  3. dispatch EntryDeletedEvent ← 在 remove + flush 之前
  4. entityManager->remove($entry)
  5. entityManager->flush()
```

**关键点：** 事件在 `remove()` + `flush()` **之前**派发。这确保了 `DownloadImagesSubscriber` 在删除文件系统图片时，数据库中条目记录仍然存在（虽然实际上只需要 entry ID）。

### 5.4 所有派发场景汇总

| 场景 | 事件 | 派发时机 | 代码位置 |
|------|------|---------|---------|
| 批量创建 POST `/api/entries/lists` | `EntrySavedEvent` | `persist + flush` 之后 | [L572](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L572) |
| 批量删除 DELETE `/api/entries/list` | `EntryDeletedEvent` | `remove + flush` 之前 | [L501](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L501) |
| 单条创建 POST `/api/entries` | `EntrySavedEvent` | `persist + flush` 之后 | [L808](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L808) |
| 单条删除 DELETE `/api/entries/{entry}` | `EntryDeletedEvent` | `remove + flush` 之前 | [L1125](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L1125) |
| 单条修改 PATCH `/api/entries/{entry}` | `EntrySavedEvent` | `persist + flush` 之后 | [L1022](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L1022) |
| 重载 PATCH `/api/entries/{entry}/reload` | `EntrySavedEvent` | `persist + flush` 之后 | [L1076](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L1076) |
| 导入流程（AbstractImport 等） | `EntrySavedEvent` | 每 20 条 flush 后批量派发 | [AbstractImport L170](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Import/AbstractImport.php#L170) |

---

## 6. `DownloadImagesSubscriber` 在批量接口中的性能和一致性影响

### 6.1 订阅者机制

[DownloadImagesSubscriber](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Event/Subscriber/DownloadImagesSubscriber.php#L13-L108) 监听两个事件：

```php
return [
    EntrySavedEvent::NAME => 'onEntrySaved',
    EntryDeletedEvent::NAME => 'onEntryDeleted',
];
```

该功能通过 `craue_config` 中的 `download_images_enabled` 配置控制开关（默认为 `0`，即关闭）。

### 6.2 `onEntrySaved` 的性能影响

[onEntrySaved](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Event/Subscriber/DownloadImagesSubscriber.php#L34-L61) 执行：

1. **`downloadImages($entry)`**：调用 [DownloadImages::processHtml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/DownloadImages.php#L65-L93)
   - 解析 HTML 中所有 `<img>` 标签的 `src` 和 `srcset`
   - 对每张图片：HTTP 请求下载 → GD 库重新生成图片 → 写入本地文件系统
   - 返回替换了图片路径的 HTML

2. **`downloadPreviewImage($entry)`**：调用 [DownloadImages::processSingleImage](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/DownloadImages.php#L108-L222)
   - 同样的 HTTP 下载 → 图片重生成流程

3. **额外的 `persist + flush`**：更新 entry 的 `content` 和 `previewPicture` 字段

**在批量创建接口中的影响：**

对于 N 个 URL 的批量请求，会产生：
- 至少 N 次 HTTP 请求（抓取页面内容，由 `ContentProxy` 触发）
- N × M 次 HTTP 请求（下载 M 张图片，由 `DownloadImagesSubscriber` 触发）
- N × M 次 GD 图片重生成操作
- N × M 次文件系统写入
- 2N 次额外的数据库 `persist + flush`（1 次在控制器，1 次在订阅者）

假设每个条目有 5 张图片，批量 10 个 URL 就是：
- 10 次页面抓取 + 50 次图片下载 + 50 次 GD 处理 + 50 次磁盘写入 + 20 次 DB 写入

### 6.3 已存在 Entry 的无谓触发

如第 4 节所述，批量创建接口对已存在的条目也会派发 `EntrySavedEvent`，但内容未更新。此时 `DownloadImagesSubscriber` 会：

- 重新解析 HTML 中的图片 URL
- 重新下载并重生成图片（即使图片已存在本地）
- 覆盖本地已有的同名图片文件
- 执行一次额外的 `persist + flush` 更新数据库

**这是一次完全冗余的操作**，浪费 HTTP 请求、CPU 和磁盘 I/O。

### 6.4 `onEntryDeleted` 的影响

[onEntryDeleted](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Event/Subscriber/DownloadImagesSubscriber.php#L66-L75) 执行 [DownloadImages::removeImages](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/DownloadImages.php#L229-L245)：

- 使用 `Finder` 遍历条目对应的图片目录
- 逐个 `unlink` 文件
- `rmdir` 删除目录

在批量删除中，这是 N 次文件系统清理操作，相对轻量，但仍然有 I/O 开销。

### 6.5 一致性问题

#### 6.5.1 图片路径替换与数据库写入的非原子性

[onEntrySaved](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Event/Subscriber/DownloadImagesSubscriber.php#L44-L60) 中：

```php
$html = $this->downloadImages($entry);  // 步骤1: 下载图片到磁盘
if (false !== $html) {
    $entry->setContent($html);           // 步骤2: 更新内容引用本地路径
}
$previewPicture = $this->downloadPreviewImage($entry);  // 步骤3: 下载预览图
if (false !== $previewPicture) {
    $entry->setPreviewPicture($previewPicture);          // 步骤4: 更新预览图路径
}
$this->em->persist($entry);
$this->em->flush();                     // 步骤5: 写入数据库
```

如果步骤 5 失败（数据库异常），磁盘上的图片文件已经写入但数据库中没有记录指向它们——**产生孤儿图片文件**。

#### 6.5.2 批量接口中的部分失败

批量接口逐条处理，没有事务包裹。如果第 3 个 URL 处理时 `DownloadImagesSubscriber` 中的图片下载超时：

- 前 2 个 URL 已成功保存（包括图片）
- 第 3 个 URL 可能处于不一致状态（部分图片已下载，但 entry 内容未更新）
- 后续 URL 不会被处理（因为异常中断了循环，除非被 catch）

控制器中 `ContentProxy::updateEntry()` 有 try-catch，但 `DownloadImagesSubscriber` 中的操作**没有异常保护**，任何异常都会传播到控制器导致整个请求失败。

#### 6.5.3 `processHtml` 中图片 URL 替换的覆盖问题

[processHtml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/DownloadImages.php#L65-L93) 使用 `str_replace` 逐一替换图片 URL：

```php
foreach ($imagesUrls as $image) {
    $newImage = $this->processSingleImage($entryId, $image, $url, $relativePath);
    if (false === $newImage) {
        continue;
    }
    $html = str_replace($image, $newImage, $html);
}
```

如果同一张图片 URL 在 HTML 中出现多次，`str_replace` 会替换所有出现——这本身是正确的。但如果两张图片 URL 存在包含关系（如 `http://a.com/1.jpg` 和 `http://a.com/1.jpg?size=large`），`str_replace` 可能导致错误替换。代码通过 `arsort($imagesUrls)` 按长度降序排列来**部分缓解**此问题，但并非完美解决方案。

### 6.6 与 Import 流程的对比

[AbstractImport](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Import/AbstractImport.php#L160-L186) 中的事件派发策略更高效：

- 每 20 条 entry 批量 `flush` 后，再逐条派发 `EntrySavedEvent`
- 派发后执行 `$em->clear()` 清除 Doctrine 实体管理器，减少内存占用
- 批量接口没有采用这种优化——每条都 `flush` + `dispatch`，没有 `clear()`

### 6.7 改进建议

1. **已存在 Entry 跳过事件派发**：批量创建时，如果 entry 已存在且内容未变，不应派发 `EntrySavedEvent`，避免 `DownloadImagesSubscriber` 的无谓触发
2. **批量删除统一 flush**：参照 `EntryController` 的做法，先循环 `remove` + `dispatch`，最后统一 `flush`
3. **为 `DownloadImagesSubscriber` 添加异常保护**：在 `onEntrySaved` 中 wrap 整个流程，避免图片下载失败导致批量请求中断
4. **图片去重/跳过已存在**：[processSingleImage](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/DownloadImages.php#L108-L222) 当前总是重新下载并覆盖，应检查文件是否已存在
5. **删除接口也应添加数量限制**：`deleteEntriesListAction` 没有 `api_limit_mass_actions` 检查，理论上可提交无限 URL

---

## 7. 完整流程图

### 7.1 批量创建流程（POST `/api/entries/lists`）

```
客户端请求 → URLs JSON 解码
    │
    ▼
检查 count(urls) > api_limit_mass_actions (10) ─── 是 → 400 Bad Request
    │ 否
    ▼
┌─────────────── foreach url ───────────────┐
│                                           │
│  findByUrlAndUserId(url, userId)          │
│       │                                   │
│       ├── entry === false                 │
│       │     new Entry(user)               │
│       │     ContentProxy.updateEntry()    │
│       │       ├── Graby 抓取页面内容       │
│       │       ├── 设置 title/content/等    │
│       │       └── RuleBasedTagger 自动标签  │
│       │                                   │
│       └── entry !== false                 │
│             (不更新内容)                    │
│                                           │
│  persist + flush                          │
│       │                                   │
│       ▼                                   │
│  dispatch EntrySavedEvent                 │
│       │                                   │
│       ▼                                   │
│  DownloadImagesSubscriber::onEntrySaved   │
│       ├── processHtml (下载所有图片)       │
│       ├── processSingleImage (预览图)      │
│       └── persist + flush (更新图片路径)   │
│                                           │
└───────────────────────────────────────────┘
    │
    ▼
返回 JSON 结果 [{url, entry_id}, ...]
```

### 7.2 批量删除流程（DELETE `/api/entries/list`）

```
客户端请求 → URLs JSON 解码
    │
    ▼
┌─────────────── foreach url ───────────────┐
│                                           │
│  findByUrlAndUserId(url, userId)          │
│       │                                   │
│       ├── entry === false                 │
│       │     (跳过，results[key]=false)     │
│       │                                   │
│       └── entry !== false                 │
│             │                             │
│             ▼                             │
│         isGranted('DELETE', entry)?       │
│             │                             │
│             ├── 否 (跳过)                  │
│             │                             │
│             └── 是                         │
│                   │                       │
│                   ▼                       │
│               dispatch EntryDeletedEvent  │
│                   │                       │
│                   ▼                       │
│            DownloadImagesSubscriber       │
│            ::onEntryDeleted               │
│              └── removeImages(删除文件)    │
│                   │                       │
│                   ▼                       │
│               entityManager->remove()     │
│               entityManager->flush()      │
│                                           │
└───────────────────────────────────────────┘
    │
    ▼
返回 JSON 结果 [{url, entry: bool}, ...]
```

---

## 8. 关键源码索引

| 组件 | 文件路径 |
|------|---------|
| 批量接口控制器 | [EntryRestController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php) |
| 基类控制器（含 apiLimitMassActions） | [WallabagRestController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/WallabagRestController.php) |
| Entry 仓库 | [EntryRepository.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php) |
| EntrySavedEvent | [EntrySavedEvent.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Event/EntrySavedEvent.php) |
| EntryDeletedEvent | [EntryDeletedEvent.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Event/EntryDeletedEvent.php) |
| DownloadImagesSubscriber | [DownloadImagesSubscriber.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Event/Subscriber/DownloadImagesSubscriber.php) |
| DownloadImages 辅助类 | [DownloadImages.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/DownloadImages.php) |
| ContentProxy | [ContentProxy.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/ContentProxy.php) |
| EntryVoter | [EntryVoter.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Security/Voter/EntryVoter.php) |
| MainVoter | [MainVoter.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Security/Voter/MainVoter.php) |
| 配置文件 | [wallabag.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/wallabag.yml) |
| 服务定义 | [services.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/services.yml) |
| Import 基类（对比参考） | [AbstractImport.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Import/AbstractImport.php) |
