# Wallabag 双哈希字段分析：hashed_url 与 hashed_given_url

## 1. 为什么同时维护 hashed_url 和 hashed_given_url

### 1.1 字段语义差异

在 [Entry.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/Entry.php#L64-L97) 实体中，存在两对 URL/哈希字段：

| 字段 | 含义 | 来源 |
|------|------|------|
| `url` / `hashed_url` | wallabag 实际抓取的最终 URL（可能经过重定向） | Graby 抓取后由 `ContentProxy::updateOriginUrl` 设置 |
| `given_url` / `hashed_given_url` | 用户最初提交的原始 URL（未经重定向） | API 调用时由 `ContentProxy::updateEntry` 直接设置 |

`url` 的注释明确说明："Define the url fetched by wallabag (the final url after potential redirections)"，而 `given_url` 的注释则是："Define the url entered by the user (without redirections)"。

### 1.2 典型场景

用户提交 `https://t.co/abc123`（短链接），经过 HTTP 重定向后，实际内容在 `https://example.com/article`。此时：

- `given_url` = `https://t.co/abc123`（用户输入）
- `url` = `https://example.com/article`（重定向后）

当另一个用户直接提交 `https://example.com/article` 时，仅凭 `hashed_url` 匹配即可发现该条目已存在；而当用户提交 `https://t.co/abc123` 时，需要 `hashed_given_url` 才能匹配到该条目。双哈希字段确保了无论用哪种 URL 查询，都能找到对应条目。

### 1.3 数据库索引

迁移 [Version20190401105353](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/migrations/Version20190401105353.php) 添加了 `hashed_url` 列及联合索引 `(user_id, hashed_url)`；迁移 [Version20190601125843](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/migrations/Version20190601125843.php) 添加了 `given_url`、`hashed_given_url` 列及联合索引 `(user_id, hashed_given_url)`。两个索引为不同查询路径提供了性能保障。

### 1.4 自动维护机制

在 [Entry::setUrl](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/Entry.php#L296-L302) 中：

```php
public function setUrl($url)
{
    $this->url = $url;
    $this->hashedUrl = UrlHasher::hashUrl($url);
    return $this;
}
```

在 [Entry::setGivenUrl](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/Entry.php#L903-L909) 中：

```php
public function setGivenUrl($givenUrl)
{
    $this->givenUrl = $givenUrl;
    $this->hashedGivenUrl = UrlHasher::hashUrl($givenUrl);
    return $this;
}
```

两个 setter 均在赋值原始 URL 的同时自动计算并填充对应的哈希字段，保证了数据一致性。

---

## 2. UrlHasher::hashUrl 为何对 urldecode 后的 URL 做 hash

### 2.1 代码

[UrlHasher::hashUrl](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/UrlHasher.php#L16-L21)：

```php
public static function hashUrl(string $url, $algorithm = 'sha1')
{
    return hash($algorithm, urldecode($url));
}
```

### 2.2 原因分析

核心问题是 **URL 编码的不确定性**。同一个 URL 在不同来源中可能以不同编码形式出现：

- 编码形式：`https://example.com/hello%20world`
- 解码形式：`https://example.com/hello world`

如果直接对编码后的 URL 做 hash，`hash('sha1', 'https://example.com/hello%20world')` 和 `hash('sha1', 'https://example.com/hello world')` 将产生完全不同的结果，导致同一条目无法匹配。

通过先 `urldecode` 再 hash，wallabag 确保无论 URL 以编码还是解码形式传入，都能得到相同的哈希值，从而实现正确匹配。

### 2.3 具体匹配场景

在 [ContentProxy::updateOriginUrl](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/ContentProxy.php#L374-L378) 的 `case ['path']` 分支中：

```php
case ['path']:
    if (($parsed_entry_url['path'] . '/' === $parsed_content_url['path'])
        || ($url === urldecode($entry->getUrl()))) {
        $entry->setUrl($url);
    }
    break;
```

注释明确写了："we update entry url if new url is a decoded version of it, see EntryRepository#findByUrlAndUserId"。这说明 wallabag 确实会存储解码后的 URL，而 `hashUrl` 中的 `urldecode` 与此策略一致——无论 URL 是否编码，解码后 hash 值相同，查找结果也相同。

### 2.4 历史兼容

在 [EntryRepository::findAllByUrlAndUserId](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L682-L697) 中也可以看到同样的模式：

```php
->where('e.url = :url')->setParameter('url', urldecode($url))
```

整个代码库统一使用 `urldecode` 来规范化 URL，消除编码差异。

---

## 3. EntryRepository::findByHashedUrlAndUserId 为何先查 hashedUrl 再查 hashedGivenUrl

### 3.1 代码

[EntryRepository::findByHashedUrlAndUserId](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L526-L560)：

```php
public function findByHashedUrlAndUserId($hashedUrl, $userId)
{
    // try first using hashed_url (to use the database index)
    $res = $this->createQueryBuilder('e')
        ->where('e.hashedUrl = :hashed_url')->setParameter('hashed_url', $hashedUrl)
        ->andWhere('e.user = :user_id')->setParameter('user_id', $userId)
        ->getQuery()
        ->getResult();

    if (\count($res)) {
        return current($res);
    }

    // then try using hashed_given_url (to use the database index)
    $res = $this->createQueryBuilder('e')
        ->where('e.hashedGivenUrl = :hashed_given_url')->setParameter('hashed_given_url', $hashedUrl)
        ->andWhere('e.user = :user_id')->setParameter('user_id', $userId)
        ->getQuery()
        ->getResult();

    if (\count($res)) {
        return current($res);
    }

    return false;
}
```

### 3.2 先查 hashed_url 的原因

**（1）概率优先**：大多数场景下，用户查询的 URL 与条目的最终 URL（`url`/`hashed_url`）一致。用户通常看到的是网页的真实地址，而非短链接。

**（2）数据库索引优化**：代码注释明确说明 "to use the database index"。两个联合索引 `(user_id, hashed_url)` 和 `(user_id, hashed_given_url)` 各自独立。先查 `hashed_url` 命中时无需执行第二次查询，减少数据库负载。

**（3）语义精确性**：`hashed_url` 代表的是条目的"真实地址"——内容实际所在的 URL。如果用最终 URL 查询且命中 `hashed_url`，说明这是一个精确匹配，语义上最准确。

### 3.3 再查 hashed_given_url 的原因

**（1）覆盖短链接场景**：当用户用原始短链接（如 `https://t.co/abc123`）查询时，该 URL 存储在 `given_url` 中而非 `url` 中（`url` 已被重定向更新）。只有查询 `hashed_given_url` 才能匹配。

**（2）避免误判为不存在**：如果只查 `hashed_url`，短链接查询将返回 `false`，导致同一网页被重复创建。查 `hashed_given_url` 作为兜底确保了幂等性。

### 3.4 调用链

此方法被 [EntryRepository::findByUrlAndUserId](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L493-L508) 调用：

```php
public function findByUrlAndUserId($url, $userId)
{
    return $this->findByHashedUrlAndUserId(
        UrlHasher::hashUrl($url),
        $userId
    );
}
```

而 `findByUrlAndUserId` 被 [EntryRestController::postEntriesAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L728-L736) 等方法使用，在创建新条目前先检查是否已存在。

---

## 4. /api/entries/exists 全流程跟踪

### 4.1 入口

[EntryRestController::getEntriesExistsAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L90-L146)，路由 `GET /api/entries/exists.{_format}`。

### 4.2 阶段一：输入收集与归一化

```php
$hashedUrls = $request->query->all('hashed_urls');  // 数组：hashed_urls[]=xxx&hashed_urls[]=yyy
$hashedUrl = $request->query->get('hashed_url', '');  // 单个：hashed_url=xxx
if (!empty($hashedUrl)) {
    $hashedUrls[] = $hashedUrl;
}

$urls = $request->query->all('urls');  // 数组：urls[]=http...&urls[]=http...
$url = $request->query->get('url', '');  // 单个：url=http...
if (!empty($url)) {
    $urls[] = $url;
}
```

四种输入参数被合并为两个数组：
- `$hashedUrls`：来自 `hashed_url` 和 `hashed_urls` 的已哈希值
- `$urls`：来自 `url` 和 `urls` 的原始 URL

### 4.3 阶段二：URL 哈希化与映射表构建

```php
$urlHashMap = [];
foreach ($urls as $urlToHash) {
    $urlHash = UrlHasher::hashUrl($urlToHash);
    $hashedUrls[] = $urlHash;
    $urlHashMap[$urlHash] = $urlToHash;
}
```

对每个原始 URL：
1. 调用 `UrlHasher::hashUrl` 计算哈希值（内部对 URL 先 `urldecode` 再 `sha1`）
2. 将哈希值加入 `$hashedUrls` 统一集合
3. 在 `$urlHashMap` 中记录 `哈希值 → 原始URL` 的映射，用于后续将结果键从哈希值还原为原始 URL

### 4.4 阶段三：批量数据库查询

```php
$results = array_fill_keys($hashedUrls, null);
$res = $entryRepository->findByUserIdAndBatchHashedUrls($this->getUser()->getId(), $hashedUrls);
```

[EntryRepository::findByUserIdAndBatchHashedUrls](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L562-L576)：

```php
public function findByUserIdAndBatchHashedUrls($userId, $hashedUrls)
{
    $qb = $this->createQueryBuilder('e')->select(['e.id', 'e.hashedUrl', 'e.hashedGivenUrl']);
    $res = $qb->where('e.user = :user_id')->setParameter('user_id', $userId)
                ->andWhere(
                    $qb->expr()->orX(
                        $qb->expr()->in('e.hashedUrl', $hashedUrls),
                        $qb->expr()->in('e.hashedGivenUrl', $hashedUrls)
                    )
                )
                ->getQuery()
                ->getResult();
    return $res;
}
```

该查询用一条 SQL 同时在 `hashedUrl` 和 `hashedGivenUrl` 中搜索，返回匹配条目的 `id`、`hashedUrl`、`hashedGivenUrl`。

`$results` 初始化为所有哈希值作为键、`null` 作为值的关联数组。

### 4.5 阶段四：结果匹配与键替换

```php
foreach ($res as $e) {
    $_hashedUrl = array_keys($hashedUrls, 'blah', true);
    if ([] !== array_keys($hashedUrls, $e['hashedUrl'], true)) {
        $_hashedUrl = $e['hashedUrl'];
    } elseif ([] !== array_keys($hashedUrls, $e['hashedGivenUrl'], true)) {
        $_hashedUrl = $e['hashedGivenUrl'];
    } else {
        continue;
    }
    $results[$_hashedUrl] = $e['id'];
}
```

对每条数据库结果：
1. 优先检查 `$e['hashedUrl']` 是否在请求的 `$hashedUrls` 集合中——若是，用 `hashedUrl` 作为结果键
2. 否则检查 `$e['hashedGivenUrl']` 是否在集合中——若是，用 `hashedGivenUrl` 作为结果键
3. 都不匹配则跳过（理论上不会出现）
4. 将匹配到的条目 `id` 写入 `$results` 对应键

**注意**：`$_hashedUrl = array_keys($hashedUrls, 'blah', true);` 这行看起来是一个残留代码/占位代码，因为 `'blah'` 不可能出现在 `$hashedUrls` 中，其结果总是空数组。但它会被立即覆盖，不影响逻辑。

### 4.6 阶段五：返回值转换

```php
if (false === $returnId) {
    $results = array_map(static fn ($v) => null !== $v, $results);
}
```

- 如果未传 `return_id=1`，则将 `id` 值转为布尔值（`true`/`false`）
- 如果传了 `return_id=1`，则保留实际 `id` 值

### 4.7 阶段六：哈希键还原为原始 URL

```php
$results = $this->replaceUrlHashes($results, $urlHashMap);
```

[EntryRestController::replaceUrlHashes](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L1380-L1396)：

```php
private function replaceUrlHashes(array $results, array $urlHashMap)
{
    $newResults = [];
    foreach ($results as $hash => $res) {
        if (isset($urlHashMap[$hash])) {
            $newResults[$urlHashMap[$hash]] = $res;
        } else {
            $newResults[$hash] = $res;
        }
    }
    return $newResults;
}
```

遍历结果数组：
- 如果键（哈希值）在 `$urlHashMap` 中有对应原始 URL，则用原始 URL 替换键
- 如果没有映射（说明该哈希值来自 `hashed_url`/`hashed_urls` 参数而非原始 URL 参数），则保留哈希值作为键

### 4.8 阶段七：单值/多值返回格式

```php
if (!empty($url) || !empty($hashedUrl)) {
    $hu = array_keys($results)[0];
    return $this->sendResponse(['exists' => $results[$hu]]);
}
return $this->sendResponse($results);
```

- 如果使用了单值参数（`url` 或 `hashed_url`），返回 `{"exists": true/false}` 简洁格式
- 如果使用了数组参数（`urls` 或 `hashed_urls`），返回完整关联数组

### 4.9 完整流程图

```
请求参数                归一化                 哈希化                数据库查询
─────────────────────────────────────────────────────────────────────────────────
hashed_url    ──┐
hashed_urls[] ──┼──→ $hashedUrls[]
                │
url           ──┼──→ $urls[]  ──→ UrlHasher::hashUrl() ──→ $hashedUrls[]
urls[]        ──┘                  ↓                        ↓
                              $urlHashMap               findByUserIdAndBatchHashedUrls
                              {hash→rawUrl}             WHERE hashedUrl IN (...) 
                                                                   OR hashedGivenUrl IN (...)

数据库结果              结果匹配                 键还原                  格式化输出
─────────────────────────────────────────────────────────────────────────────────
{id, hashedUrl,     优先 hashedUrl         replaceUrlHashes        单值: {exists: bool}
 hashedGivenUrl}    兜底 hashedGivenUrl     hash→rawUrl             多值: {url: bool, ...}
                     → $results[hash]=id     (有映射时替换)
```

---

## 5. GenerateUrlHashesCommand 如何补齐历史空 hash

### 5.1 背景

`hashed_url` 字段在 [Version20190401105353](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/migrations/Version20190401105353.php) 迁移中新增，但迁移只添加了列和索引，**不回填数据**。因此升级前已存在的条目其 `hashed_url` 为 `NULL` 或空字符串。

同样，`hashed_given_url` 在 [Version20190601125843](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/migrations/Version20190601125843.php) 中添加，同样未回填。

### 5.2 命令代码

[GenerateUrlHashesCommand](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Command/GenerateUrlHashesCommand.php#L68-L83)：

```php
private function generateHashedUrls(User $user): void
{
    $entries = $this->entryRepository->findByEmptyHashedUrlAndUserId($user->getId());

    $i = 1;
    foreach ($entries as $entry) {
        $entry->setHashedUrl(UrlHasher::hashUrl($entry->getUrl()));
        $this->entityManager->persist($entry);

        if (0 === ($i % 20)) {
            $this->entityManager->flush();
        }
        ++$i;
    }

    $this->entityManager->flush();
}
```

### 5.3 工作流程

1. **查找空哈希条目**：[EntryRepository::findByEmptyHashedUrlAndUserId](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L510-L524) 查询 `hashed_url = ''` 或 `hashed_url IS NULL` 且 `url IS NOT NULL` 的条目
2. **逐条计算哈希**：对每个条目调用 `UrlHasher::hashUrl($entry->getUrl())` 计算哈希值
3. **直接设置哈希**：调用 `$entry->setHashedUrl()` 而非 `setUrl()`，避免覆盖原始 URL（仅更新哈希字段）
4. **分批刷入**：每 20 条执行一次 `flush()`，避免内存溢出
5. **最终刷入**：循环结束后再执行一次 `flush()`，确保剩余条目也被持久化

### 5.4 局限性

**仅回填 `hashed_url`，不回填 `hashed_given_url`**。这是因为命令只查找 `hashedUrl` 为空的条目，且只调用 `setHashedUrl` 而不处理 `hashedGivenUrl`。对于 `hashed_given_url` 的历史空值，目前没有对应的回填命令——由于 `Entry::setGivenUrl` 中自动计算 `hashedGivenUrl`，新条目不会有此问题，但迁移前的旧条目如果从未重新保存，其 `hashed_given_url` 可能仍为空。

### 5.5 使用方式

```bash
# 处理所有用户
php bin/console wallabag:generate-hashed-urls

# 处理指定用户
php bin/console wallabag:generate-hashed-urls admin
```

---

## 6. ContentProxy::updateOriginUrl 对重定向 URL 和原始 givenUrl 的影响

### 6.1 调用上下文

[ContentProxy::updateEntry](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/ContentProxy.php#L43-L79) 的关键流程：

```php
public function updateEntry(Entry $entry, $url, array $content = [], $disableContentUpdate = false): void
{
    // ... 获取内容 ...

    // 设置 given_url 为用户提交的原始 URL
    $entry->setGivenUrl($url);

    // stockEntry 内部调用 updateOriginUrl
    $this->stockEntry($entry, $content);
}
```

[ContentProxy::stockEntry](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/ContentProxy.php#L236-L240) 的第一行：

```php
private function stockEntry(Entry $entry, array $content): void
{
    $this->updateOriginUrl($entry, $content['url']);
    // ...
}
```

### 6.2 givenUrl 的设置时机

在 `updateEntry` 中，`$entry->setGivenUrl($url)` 在 `stockEntry`（进而 `updateOriginUrl`）之前执行。`$url` 参数是用户提交的原始 URL。这意味着：

- **`given_url` 始终记录用户最初输入的 URL**，不受重定向影响
- 调用 `setGivenUrl` 时自动计算 `hashed_given_url`

### 6.3 updateOriginUrl 的完整逻辑

[ContentProxy::updateOriginUrl](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/ContentProxy.php#L328-L393)：

```php
private function updateOriginUrl(Entry $entry, $url)
{
    // 1. URL 相同或为空 → 不做任何处理
    if (empty($url) || $entry->getUrl() === $url) {
        return false;
    }

    // 2. 解析两个 URL，计算差异部分
    $parsed_entry_url = parse_url($entry->getUrl());
    $parsed_content_url = parse_url($url);
    // ... 计算 diff_keys ...

    // 3. 忽略源处理器（如 feedproxy）→ 直接替换 URL，不记录 origin
    if ($this->ignoreOriginProcessor->process($entry)) {
        $entry->setUrl($url);
        return false;
    }

    // 4. 根据差异类型分别处理
    switch ($diff_keys) {
        case ['path']:
            // 仅路径不同：尾斜杠差异或 URL 解码差异 → 仅更新 URL
            if (($parsed_entry_url['path'] . '/' === $parsed_content_url['path'])
                || ($url === urldecode($entry->getUrl()))) {
                $entry->setUrl($url);
            }
            break;
        case ['scheme']:
            // 仅协议不同（http→https）→ 仅更新 URL
            $entry->setUrl($url);
            break;
        case ['fragment']:
            // 仅锚点不同 → 不做任何处理
            break;
        default:
            // 其他情况（域名变更、路径+查询参数变更等）→ 记录 origin_url 并更新 URL
            if (empty($entry->getOriginUrl())) {
                $entry->setOriginUrl($entry->getUrl());
            }
            $entry->setUrl($url);
            break;
    }
}
```

### 6.4 各场景影响分析

| 场景 | entry_url（初始） | content_url（重定向后） | 对 url 的影响 | 对 origin_url 的影响 | 对 given_url 的影响 |
|------|-------------------|------------------------|--------------|---------------------|-------------------|
| **无重定向** | `https://a.com/1` | `https://a.com/1` | 不变 | 不变 | 保持用户输入 |
| **完全重定向** | `https://t.co/abc` | `https://b.com/article` | → `https://b.com/article` | → `https://t.co/abc`（首次） | 保持 `https://t.co/abc` |
| **尾斜杠** | `https://a.com/hello` | `https://a.com/hello/` | → `https://a.com/hello/` | 不变 | 保持用户输入 |
| **URL解码** | `https://a.com/hello%20world` | `https://a.com/hello world` | → `https://a.com/hello world` | 不变 | 保持用户输入 |
| **协议升级** | `http://a.com/1` | `https://a.com/1` | → `https://a.com/1` | 不变 | 保持用户输入 |
| **锚点差异** | `https://a.com/1` | `https://a.com/1#section` | 不变 | 不变 | 保持用户输入 |
| **查询参数变更** | `https://a.com/1` | `https://a.com/1?foo` | → `https://a.com/1?foo` | → `https://a.com/1` | 保持用户输入 |
| **feedproxy忽略** | `http://feedproxy.google.com/...` | `https://b.com/article` | → `https://b.com/article` | 不变（忽略处理器直接替换） | 保持用户输入 |

### 6.5 对 hashed_url 和 hashed_given_url 的影响

- **`hashed_url` 随 `url` 变化**：`updateOriginUrl` 中每次调用 `$entry->setUrl($url)` 都会触发 `Entry::setUrl`，自动重新计算 `hashed_url`。因此 `hashed_url` 始终反映条目的最终 URL。
- **`hashed_given_url` 始终不变**：`given_url` 在 `updateEntry` 开头设置后不再被 `updateOriginUrl` 修改。它始终记录用户提交的原始 URL，其哈希值也保持不变。
- **`origin_url` 与两者无关**：`origin_url` 没有对应的哈希字段，不参与 exists API 查询逻辑，仅作为用户参考信息展示"这篇文章最初来自哪个 URL"。

### 6.6 三者的关系总结

```
用户提交: https://t.co/abc123
           │
           ▼
    setGivenUrl("https://t.co/abc123")
    hashed_given_url = sha1(urldecode("https://t.co/abc123"))
           │
           ▼
    Graby 抓取 → 重定向到 https://example.com/article
           │
           ▼
    updateOriginUrl(entry, "https://example.com/article")
    ├── entry.getUrl() ≠ content_url
    ├── diff_keys = ['host', 'path']  (default 分支)
    ├── origin_url 为空 → setOriginUrl("https://t.co/abc123")
    └── setUrl("https://example.com/article")
        └── hashed_url = sha1(urldecode("https://example.com/article"))
```

最终数据库记录：

| 字段 | 值 |
|------|-----|
| `url` | `https://example.com/article` |
| `hashed_url` | `sha1("https://example.com/article")` |
| `given_url` | `https://t.co/abc123` |
| `hashed_given_url` | `sha1("https://t.co/abc123")` |
| `origin_url` | `https://t.co/abc123` |

---

## 7. 总结

wallabag 维护 `hashed_url` 和 `hashed_given_url` 双哈希字段的根本原因是 **URL 重定向导致同一个网页可能有两个不同的入口 URL**：

1. `url`/`hashed_url` 追踪内容的真实地址，确保通过最终 URL 能找到条目
2. `given_url`/`hashed_given_url` 保留用户的原始输入，确保通过短链接等入口也能找到条目
3. `UrlHasher::hashUrl` 先 `urldecode` 再 hash，消除了 URL 编码差异导致的匹配失败
4. `findByHashedUrlAndUserId` 先查 `hashed_url` 再查 `hashed_given_url`，兼顾了查询效率和覆盖完整性
5. `GenerateUrlHashesCommand` 为迁移前历史数据补齐 `hashed_url`，但未处理 `hashed_given_url`
6. `ContentProxy::updateOriginUrl` 根据重定向类型智能处理 `url` 变更，同时保护 `given_url` 不被覆盖，确保双哈希查找机制始终有效
