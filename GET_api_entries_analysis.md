# GET /api/entries 查询参数转换分析

## 1. 整体流程概述

`GET /api/entries` 接口的请求处理流程分为三个主要阶段：

1. **参数接收与转换**：在 [EntryRestController::getEntriesAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L314-L379) 中接收并转换查询参数
2. **查询构建**：在 [EntryRepository::findEntries](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L282-L372) 中构建 Doctrine 查询
3. **分页响应**：使用 Hateoas Pagerfanta 生成分页响应

---

## 2. 查询参数到 Doctrine 查询的映射

### 2.1 控制器参数处理

在 [EntryRestController.php 第316-329行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L316-L329)，每个参数都经过特定转换：

```php
$isArchived = (null === $request->query->get('archive')) ? null : (bool) $request->query->get('archive');
$isStarred = (null === $request->query->get('starred')) ? null : (bool) $request->query->get('starred');
$isPublic = (null === $request->query->get('public')) ? null : (bool) $request->query->get('public');
$isNotParsed = (null === $request->query->get('notParsed')) ? null : (bool) $request->query->get('notParsed');
$sort = strtolower($request->query->get('sort', 'created'));
$order = strtolower($request->query->get('order', 'desc'));
$page = $request->query->getInt('page', 1);
$perPage = $request->query->getInt('perPage', 30);
$tags = \is_array($request->query->all()['tags'] ?? '') ? '' : (string) $request->query->get('tags', '');
$since = $request->query->getInt('since');
$detail = strtolower($request->query->get('detail', 'full'));
$domainName = (null === $request->query->get('domain_name')) ? '' : (string) $request->query->get('domain_name');
$httpStatus = (!\array_key_exists((int) $request->query->get('http_status'), Response::$statusTexts)) ? null : (int) $request->query->get('http_status');
$hasAnnotations = (null === $request->query->get('annotations')) ? null : (bool) $request->query->get('annotations');
```

### 2.2 参数映射详解

| 参数名 | 类型转换 | 传递到 findEntries | Doctrine 查询构建位置 |
|--------|----------|-------------------|----------------------|
| `archive` | `null` 或 `bool` | `$isArchived` | [EntryRepository.php 第298-300行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L298-L300) |
| `starred` | `null` 或 `bool` | `$isStarred` | [EntryRepository.php 第302-304行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L302-L304) |
| `public` | `null` 或 `bool` | `$isPublic` | [EntryRepository.php 第306-308行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L306-L308) |
| `notParsed` | `null` 或 `bool` | `$isNotParsed` | [EntryRepository.php 第310-312行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L310-L312) |
| `sort` | `strtolower`, 默认 `'created'` | `$sort` | [EntryRepository.php 第361-367行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L361-L367) |
| `order` | `strtolower`, 默认 `'desc'` | `$order` | [EntryRepository.php 第357-359行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L357-L359) |
| `since` | `int`, 默认 `0` | `$since` | [EntryRepository.php 第314-316行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L314-L316) |
| `tags` | 数组转空字符串，否则原样 | `$tags` | [EntryRepository.php 第318-337行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L318-L337) |
| `detail` | `strtolower`, 默认 `'full'` | `$detail` | [EntryRepository.php 第284-296行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L284-L296) |
| `domain_name` | `''` 或 `string` | `$domainName` | [EntryRepository.php 第343-345行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L343-L345) |
| `http_status` | 验证 `Response::$statusTexts`，失败为 `null` | `$httpStatus` | [EntryRepository.php 第339-341行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L339-L341) |
| `annotations` | `null` 或 `bool` | `$hasAnnotations` | [EntryRepository.php 第347-355行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L347-L355) |

### 2.3 各参数的查询构建

#### archive / starred / notParsed
```php
if (null !== $isArchived) {
    $qb->andWhere('e.isArchived = :isArchived')->setParameter('isArchived', (bool) $isArchived);
}
```
使用标准的 `andWhere` 等值比较。

#### public
```php
if (null !== $isPublic) {
    $qb->andWhere('e.uid IS ' . (true === $isPublic ? 'NOT' : '') . ' NULL');
}
```
通过检查 `uid` 字段是否为 `NULL` 来判断是否公开（公开条目有 uid）。

#### since
```php
if ($since > 0) {
    $qb->andWhere('e.updatedAt > :since')->setParameter('since', new \DateTime(date('Y-m-d H:i:s', $since)));
}
```
时间戳转换为 `DateTime` 对象，比较 `updatedAt` 字段。

#### domain_name
```php
if (\is_string($domainName) && '' !== $domainName) {
    $qb->andWhere('e.domainName = :domainName')->setParameter('domainName', $domainName);
}
```
等值比较 `domainName` 字段。

#### http_status
```php
if (\is_int($httpStatus)) {
    $qb->andWhere('e.httpStatus = :httpStatus')->setParameter('httpStatus', $httpStatus);
}
```
等值比较 `httpStatus` 字段。**注意**：在控制器层已通过 `Response::$statusTexts` 验证，无效值会被设为 `null`，从而被跳过。

#### annotations
```php
if (null !== $hasAnnotations) {
    if ($hasAnnotations) {
        $qb->leftJoin('e.annotations', 'a')
           ->andWhere('a.id IS NOT NULL');
    } else {
        $qb->leftJoin('e.annotations', 'a')
           ->andWhere('a.id IS NULL');
    }
}
```
通过 `LEFT JOIN` 关联 `annotations`，然后检查关联是否存在。

#### sort / order
```php
if ('created' === $sort) {
    $qb->orderBy('e.id', $order);
} elseif ('updated' === $sort) {
    $qb->orderBy('e.updatedAt', $order);
} elseif ('archived' === $sort) {
    $qb->orderBy('e.archivedAt', $order);
}
```
根据 `sort` 值选择不同的排序字段，`order` 控制升序/降序。

---

## 3. detail=metadata 构造 partial 并排除 content

### 3.1 实现机制

在 [EntryRepository.php 第292-296行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L292-L296)：

```php
if ('metadata' === $detail) {
    $fieldNames = $this->getClassMetadata()->getFieldNames();
    $fields = array_filter($fieldNames, static fn ($k) => 'content' !== $k);
    $qb->select(\sprintf('partial e.{%s}', implode(',', $fields)));
}
```

### 3.2 关键步骤

1. **获取所有字段名**：通过 `$this->getClassMetadata()->getFieldNames()` 获取 `Entry` 实体的所有数据库字段名
2. **过滤 content 字段**：使用 `array_filter` 排除 `'content'` 字段
3. **构造 partial select**：使用 Doctrine 的 `partial` 语法 `partial e.{field1,field2,...}`

### 3.3 为什么使用 partial

- **性能优化**：`content` 字段通常存储大量 HTML 内容，不查询可以显著减少内存使用和传输数据量
- **Doctrine 特性**：`partial` 语法允许查询实体的部分字段，同时保持返回对象是 `Entry` 实体实例（而非数组）
- **未选中字段为 null**：`content` 字段在返回的实体中为 `null`，序列化时不会出现在响应中

---

## 4. 多标签过滤使用子查询的原因

### 4.1 实现代码

在 [EntryRepository.php 第318-337行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L318-L337)：

```php
if (\is_string($tags) && '' !== $tags) {
    foreach (explode(',', $tags) as $i => $tag) {
        $entryAlias = 'e' . $i;
        $tagAlias = 't' . $i;

        // Complex queries to ensure multiple tags are associated to an entry
        // https://stackoverflow.com/a/6638146/569101
        $qb->andWhere($qb->expr()->in(
            'e.id',
            $this->createQueryBuilder($entryAlias)
                ->select($entryAlias . '.id')
                ->leftJoin($entryAlias . '.tags', $tagAlias)
                ->where($tagAlias . '.label = :label' . $i)
                ->getDQL()
        ));

        $qb->setParameter('label' . $i, $tag);
    }
}
```

### 4.2 为什么不用 JOIN

如果使用简单的 `JOIN + IN` 方式：
```sql
SELECT e FROM Entry e JOIN e.tags t WHERE t.label IN ('foo', 'bar')
```
这会返回**包含任一标签**的条目（OR 逻辑），而不是**包含所有标签**的条目（AND 逻辑）。

### 4.3 子查询方案的原理

对于每个标签，创建一个独立的子查询：
- 标签 1：`e.id IN (SELECT e1.id FROM Entry e1 JOIN e1.tags t1 WHERE t1.label = 'foo')`
- 标签 2：`e.id IN (SELECT e2.id FROM Entry e2 JOIN e2.tags t2 WHERE t2.label = 'bar')`

多个 `AND` 连接的 `IN` 子查询确保条目必须包含**所有**指定标签。

### 4.4 生成的 SQL 示例

查询 `tags=foo,bar` 会生成类似：
```sql
SELECT e FROM Wallabag\Entity\Entry e 
LEFT JOIN e.tags t 
WHERE e.user = :userId 
AND e.id IN (SELECT e0.id FROM Wallabag\Entity\Entry e0 LEFT JOIN e0.tags t0 WHERE t0.label = :label0)
AND e.id IN (SELECT e1.id FROM Wallabag\Entity\Entry e1 LEFT JOIN e1.tags t1 WHERE t1.label = :label1)
```

---

## 5. 无效 detail/order 参数转换为 BadRequest

### 5.1 参数验证位置

在 [EntryRepository.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php) 中有两处验证：

**detail 参数验证（第284-286行）**：
```php
if (!\in_array(strtolower($detail), ['full', 'metadata'], true)) {
    throw new \Exception('Detail "' . $detail . '" parameter is wrong, allowed: full or metadata');
}
```

**order 参数验证（第357-359行）**：
```php
if (!\in_array(strtolower($order), ['asc', 'desc'], true)) {
    throw new \Exception('Order "' . $order . '" parameter is wrong, allowed: asc or desc');
}
```

### 5.2 异常转换为 BadRequest

在 [EntryRestController.php 第331-350行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L331-L350)：

```php
try {
    $pager = $entryRepository->findEntries(...);
} catch (\Exception $e) {
    throw new BadRequestHttpException($e->getMessage());
}
```

控制器捕获 `findEntries` 抛出的普通 `\Exception`，将其包装为 `BadRequestHttpException`，Symfony 会将其转换为 HTTP 400 响应。

### 5.3 为什么不直接在控制器验证

- **关注点分离**：参数验证逻辑放在 Repository 中，使控制器保持简洁
- **复用性**：如果其他地方调用 `findEntries`，也能获得同样的验证
- **集中管理**：所有查询相关的逻辑都在 Repository 中

---

## 6. Hateoas 分页响应构建

### 6.1 实现代码

在 [EntryRestController.php 第352-376行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L352-L376)：

```php
$pager->setMaxPerPage($perPage);
$pager->setCurrentPage($page);

$pagerfantaFactory = new PagerfantaFactory('page', 'perPage');
$paginatedCollection = $pagerfantaFactory->createRepresentation(
    $pager,
    new HateoasRoute(
        'api_get_entries',
        [
            'archive' => $isArchived,
            'starred' => $isStarred,
            'public' => $isPublic,
            'notParsed' => $isNotParsed,
            'sort' => $sort,
            'order' => $order,
            'page' => $page,
            'perPage' => $perPage,
            'tags' => $tags,
            'since' => $since,
            'detail' => $detail,
            'annotations' => $hasAnnotations,
        ],
        true
    )
);
```

### 6.2 关键组件

1. **Pagerfanta**：Doctrine 查询结果被包装为 `Pagerfanta` 对象，处理分页逻辑
2. **PagerfantaFactory**：Hateoas 提供的工厂类，将 Pagerfanta 转换为 HAL 格式的分页表示
3. **HateoasRoute**：定义分页链接的路由信息，包含所有查询参数以确保翻页时保留过滤条件
4. **HAL 格式响应**：包含 `_links`（self/first/last/prev/next）、`_embedded.items`、`total`、`page`、`pages`、`limit` 等字段

---

## 7. 测试验证

### 7.1 metadata 不返回 content

测试方法：[testGetEntriesDetailMetadata](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/functional/Controller/Api/EntryRestControllerTest.php#L134-L151)

```php
public function testGetEntriesDetailMetadata(): void
{
    $this->client->request('GET', '/api/entries?detail=metadata');
    $this->assertSame(200, $this->client->getResponse()->getStatusCode());
    
    $content = json_decode($this->client->getResponse()->getContent(), true);
    $this->assertNull($content['_embedded']['items'][0]['content']);
}
```

**验证点**：`$content['_embedded']['items'][0]['content']` 为 `null`

### 7.2 http_status 过滤

**匹配测试**：[testGetEntriesByHttpStatusWithMatching](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/functional/Controller/Api/EntryRestControllerTest.php#L172-L190)

```php
public function testGetEntriesByHttpStatusWithMatching(): void
{
    $this->client->request('GET', '/api/entries?http_status=302');
    $content = json_decode($this->client->getResponse()->getContent(), true);
    
    $this->assertSame(1, $content['total']);
    $this->assertSame('test title entry7', $content['_embedded']['items'][0]['title']);
    $this->assertSame('302', $content['_embedded']['items'][0]['http_status']);
}
```

**不匹配测试**：[testGetEntriesByHttpStatusNoMatching](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/functional/Controller/Api/EntryRestControllerTest.php#L192-L207)

```php
public function testGetEntriesByHttpStatusNoMatching(): void
{
    $this->client->request('GET', '/api/entries?http_status=404');
    $content = json_decode($this->client->getResponse()->getContent(), true);
    
    $this->assertEmpty($content['_embedded']['items']);
    $this->assertSame(0, $content['total']);
}
```

### 7.3 annotations 过滤

**有注解过滤**：[testGetEntriesWithAnnotationsFilter](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/functional/Controller/Api/EntryRestControllerTest.php#L310-L348)

```php
public function testGetEntriesWithAnnotationsFilter(): void
{
    $this->client->request('GET', '/api/entries', ['annotations' => 1]);
    $content = json_decode($this->client->getResponse()->getContent(), true);
    
    $entriesWithAnnotations = ['http://0.0.0.0/entry1', 'http://0.0.0.0/entry2'];
    $entriesWithoutAnnotations = ['http://0.0.0.0/entry4', 'http://0.0.0.0/entry5', ...];
    
    foreach ($content['_embedded']['items'] as $item) {
        $this->assertNotContains($item['url'], $entriesWithoutAnnotations);
    }
    
    $foundUrls = array_column($content['_embedded']['items'], 'url');
    $this->assertContains('http://0.0.0.0/entry1', $foundUrls);
    $this->assertContains('http://0.0.0.0/entry2', $foundUrls);
}
```

**无注解过滤**：[testGetEntriesWithoutAnnotationsFilter](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/functional/Controller/Api/EntryRestControllerTest.php#L350-L388)

```php
public function testGetEntriesWithoutAnnotationsFilter(): void
{
    $this->client->request('GET', '/api/entries', ['annotations' => 0]);
    $content = json_decode($this->client->getResponse()->getContent(), true);
    
    $entriesWithAnnotations = ['http://0.0.0.0/entry1', 'http://0.0.0.0/entry2'];
    
    foreach ($content['_embedded']['items'] as $item) {
        $this->assertNotContains($item['url'], $entriesWithAnnotations);
    }
}
```

### 7.4 无效 order 参数返回 BadRequest

测试方法：[testGetStarredEntriesWithBadSort](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/functional/Controller/Api/EntryRestControllerTest.php#L406-L413)

```php
public function testGetStarredEntriesWithBadSort(): void
{
    $this->client->request('GET', '/api/entries', ['starred' => 1, 'sort' => 'updated', 'order' => 'unknown']);
    $this->assertSame(400, $this->client->getResponse()->getStatusCode());
}
```

---

## 8. 总结流程图

```
请求 GET /api/entries?archive=1&tags=foo,bar&detail=metadata
    ↓
[EntryRestController::getEntriesAction]
    ├─ 参数转换 (第316-329行)
    │   ├─ archive → (bool) true
    │   ├─ tags → "foo,bar"
    │   └─ detail → "metadata"
    │
    ├─ 调用 EntryRepository::findEntries (第333-347行)
    │   ├─ 验证 detail/order 参数 (第284-286, 357-359行)
    │   ├─ 构造 partial select 排除 content (第292-296行)
    │   ├─ 构建 archive 条件 (第298-300行)
    │   ├─ 构建多标签子查询 (第318-337行)
    │   └─ 返回 Pagerfanta 对象
    │
    ├─ 异常捕获 → BadRequestHttpException (第348-350行)
    │
    └─ 构建 Hateoas 分页响应 (第355-376行)
        ├─ PagerfantaFactory::createRepresentation
        ├─ HateoasRoute 包含所有查询参数
        └─ 返回 HAL 格式 JSON
    ↓
响应 (HTTP 200)
```

---

## 9. 关键文件索引

| 文件 | 说明 |
|------|------|
| [EntryRestController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php) | API 控制器，参数处理和响应构建 |
| [EntryRepository.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php) | 数据访问层，findEntries 方法实现所有查询逻辑 |
| [EntryRestControllerTest.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/functional/Controller/Api/EntryRestControllerTest.php) | 功能测试，验证各种参数组合的行为 |
