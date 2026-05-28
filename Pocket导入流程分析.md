# Pocket 导入流程分析

## 一、OAuth 认证流程

### 1. 认证入口：`/import/pocket/auth`

**文件位置**: [PocketController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Import/PocketController.php)

**流程说明：

当用户在表单提交后，`authAction` 方法被调用：

```php
#[Route(path: '/import/pocket/auth', name: 'import_pocket_auth', methods: ['POST'])]
public function authAction(Request $request, PocketImport $pocketImport)
```

#### requestToken 进入 Session 的过程：

1. **获取 Request Token**：调用 `PocketImport::getRequestToken()` 向 Pocket API 请求 request_token
   - 调用 Pocket API: `POST https://getpocket.com/v3/oauth/request`
   -参数：`consumer_key` 和 `redirect_uri`
   - 返回：`code` 字段作为 requestToken

2. **存储到 Session**：
   ```php
   $this->session->set('import.pocket.code', $requestToken);
   ```
   - `requestToken 以 `import.pocket.code` 为键存入 Session

3. **mark_as_read 进入 Session 的过程：

   ```php
   $form = $request->request->all('form');
   if (\array_key_exists('mark_as_read', $form)) {
       $this->session->set('mark_as_read', $form['mark_as_read']);
   }
   ```
   - 从 POST 请求的表单数据中获取 `mark_as_read`
   - 以 `mark_as_read` 为键存入 Session

4. **重定向到 Pocket 授权页面**：
   - 重定向 URL: `https://getpocket.com/auth/authorize`
   - 参数：`request_token` 和 `redirect_uri`（回调地址）

---

### 2. 回调处理：`/import/pocket/callback`

**文件位置**: [PocketController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Import/PocketController.php)

```php
#[Route(path: '/import/pocket/callback', name: 'import_pocket_callback', methods: ['GET'])]
public function callbackAction(PocketImport $pocketImport, TranslatorInterface $translator)
```

#### PocketImport::authorize 获取 accessToken 的过程：

1. **从 Session 取出数据**：
   ```php
   $markAsRead = $this->session->get('mark_as_read');
   $this->session->remove('mark_as_read');
   ```

2. **调用 authorize 方法**：
   ```php
   $pocket->authorize($this->session->get('import.pocket.code'))
   ```

3. **PocketImport::authorize 实现**（位于 [PocketImport.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Import/PocketImport.php#L77-L95)

   ```php
   public function authorize($code)
   {
       try {
           $response = $this->client->request(Request::METHOD_POST, 'https://getpocket.com/v3/oauth/authorize', [
               'json' => [
                   'consumer_key' => $this->user->getConfig()->getPocketConsumerKey(),
                   'code' => $code,
               ],
           ]);

           $this->accessToken = $response->toArray()['access_token'];

           return true;
       } catch (ExceptionInterface $e) {
           // ... error handling
           return false;
       }
   }
   ```

   - 调用 Pocket API: `POST https://getpocket.com/v3/oauth/authorize`
   - 参数：`consumer_key` 和 `code`（即之前存储的 requestToken
   - 返回的 `access_token` 赋值给 `$this->accessToken` 属性

4. **设置 mark_as_read 并启动导入**：
   ```php
   $pocket->setMarkAsRead($markAsRead)->import()
   ```

---

## 二、导入分页机制

### 1. 递归分页实现

**文件位置**: [PocketImport.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Import/PocketImport.php#L97-L139)

```php
public function import($offset = 0)
{
    static $run = 0;

    try {
        $response = $this->client->request(Request::METHOD_POST, 'https://getpocket.com/v3/get', [
            'json' => [
                'consumer_key' => $this->user->getConfig()->getPocketConsumerKey(),
                'access_token' => $this->accessToken,
                'detailType' => 'complete',
                'state' => 'all',
                'sort' => 'newest',
                'count' => self::NB_ELEMENTS,
                'offset' => $offset,
            ],
        ]);

        $entries = $response->toArray();

        if ($this->producer) {
            $this->parseEntriesForProducer($entries['list']);
        } else {
            $this->parseEntries($entries['list']);
        }

        // 递归分页逻辑
        if (self::NB_ELEMENTS === \count($entries['list'])) {
            ++$run;

            return $this->import(self::NB_ELEMENTS * $run);
        }

        return true;
    } catch (ExceptionInterface $e) {
        return false;
    }
}
```

#### 分页逻辑详解：

1. **NB_ELEMENTS 常量**：`const NB_ELEMENTS = 30;` - 每页30条

2. **静态变量 $run**：记录递归调用次数

3. **API 请求参数**：
   - `count` => `self::NB_ELEMENTS` - 每页数量
   - `offset` => `$offset` - 偏移量

4. **递归条件**：
   ```php
   if (self::NB_ELEMENTS === \count($entries['list'])) {
       ++$run;
       return $this->import(self::NB_ELEMENTS * $run);
   }
   ```
   - 当返回的条目数等于请求数量时，说明还有更多数据
   - `$run` 递增
   - 递归调用 `import()`，偏移量为 `NB_ELEMENTS * $run`
   - 第一次：offset = 0
   - 第二次：offset = 30
   - 第三次：offset = 60
   - 以此类推...

5. **终止条件**：当返回条目数 < NB_ELEMENTS 时停止递归

---

## 三、条目解析与处理

### 1. parseEntry 方法详解

**文件位置**: [PocketImport.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Import/PocketImport.php#L161-L217)

```php
public function parseEntry(array $importedEntry)
```

#### (1) URL 选择：resolved_url 或 given_url

```php
$url = isset($importedEntry['resolved_url']) && '' !== $importedEntry['resolved_url'] ? $importedEntry['resolved_url'] : $importedEntry['given_url'];
```

- **优先级**：优先使用 `resolved_url`（Pocket 解析后的最终 URL
- **回退**：如果 `resolved_url` 为空时使用 `given_url`（用户原始提交的 URL

#### (2) 去重处理

```php
$existingEntry = $this->em
    ->getRepository(Entry::class)
    ->findByUrlAndUserId($url, $this->user->getId());

if (false !== $existingEntry) {
    ++$this->skippedEntries;

    return null;
}
```

- 通过 URL 和用户 ID 检查是否已存在
- 已存在则跳过，增加 `skippedEntries` 计数
- 返回 null 表示不导入

#### (3) status 和 favorite 处理

```php
// 0, 1, 2 - 1 if the item is archived - 2 if the item should be deleted
$entry->updateArchived(1 === (int) $importedEntry['status'] || $this->markAsRead);

// 0 or 1 - 1 if the item is starred
$entry->setStarred(1 === (int) $importedEntry['favorite']);
```

**status 字段含义**：
- `0` - 未归档（未读
- `1` - 已归档（已读）
- `2` - 应删除

**标记已读逻辑**：
- Pocket 状态为 1（已归档）时标记已读
- **OR** 用户选择了 `mark_as_read 选项时也标记已读

**favorite 字段含义**：
- `0` - 未收藏
- `1` - 已收藏（starred）

#### (4) 标签创建

```php
if (isset($importedEntry['tags']) && !empty($importedEntry['tags'])) {
    $this->tagsAssigner->assignTagsToEntry(
        $entry,
        array_keys($importedEntry['tags']),
        $this->em->getUnitOfWork()->getScheduledEntityInsertions()
    );
}
```

- Pocket 的 tags 是一个关联数组结构：`['tag_name' => ['tag' => 'tag_name', ...]]
- 使用 `array_keys()` 提取标签名称数组
- 调用 `TagsAssigner::assignTagsToEntry()` 为条目分配标签

#### (5) 预览图处理

```php
// 0, 1, or 2 - 1 if the item has images in it - 2 if the item is an image
if (isset($importedEntry['has_image']) && $importedEntry['has_image'] > 0 && isset($importedEntry['images'][1])) {
    $entry->setPreviewPicture($importedEntry['images'][1]['src']);
}
```

**has_image 字段含义**：
- `0` - 无图片
- `1` - 包含图片
- `2` - 本身是图片

**预览图选择逻辑**：
- 当 `has_image > 0` 且存在 `images'][1'] 时
- 使用第一张图片（索引 1 的 src 作为预览图

---

## 四、队列模式处理

### 1. 队列启用条件

**文件位置**: [PocketController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Import/PocketController.php#L116-L127)

```php
private function getPocketImportService(PocketImport $pocketImport): PocketImport
{
    $pocketImport->setUser($this->getUser());

    if ($this->craueConfig->get('import_with_rabbitmq')) {
        $pocketImport->setProducer($this->rabbitMqProducer);
    } elseif ($this->craueConfig->get('import_with_redis')) {
        $pocketImport->setProducer($this->redisProducer);
    }

    return $pocketImport;
}
```

- 配置 `import_with_rabbitmq` 或 `import_with_redis` 时启用队列

### 2. parseEntriesForProducer 转交过程

**文件位置**: [AbstractImport.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Import/AbstractImport.php#L197-L211)

```php
protected function parseEntriesForProducer(array $entries): void
{
    foreach ($entries as $importedEntry) {
        // set userId for the producer (it won't know which user is connected)
        $importedEntry['userId'] = $this->user->getId();

        if ($this->markAsRead) {
            $importedEntry = $this->setEntryAsRead($importedEntry);
        }

        ++$this->queuedEntries;

        $this->producer->publish(json_encode($importedEntry));
    }
}
```

#### 队列模式特点**：

1. **不做数据库检查**：不进行去重检查，去重等操作推迟到消费者端执行
2. **附加用户 ID**：在条目中添加 `userId` 字段，供消费者识别用户
3. **标记已读处理**：在发送前先调用 `setEntryAsRead()` 处理
4. **JSON 序列化**：将条目 JSON 编码后发布到队列
5. **计数统计**：递增 `queuedEntries` 统计入队数量

### 3. 队列 vs 非队列对比**：

| 特性 | 队列模式 | 非队列模式 |
|------|---------|------------|
| 去重检查 | 消费者端 | 导入时 |
| 数据库操作 | 无 | 实时 |
| 速度 | 快 | 慢 |
| 内存占用 | 低 | 高 |

---

## 五、完整流程图

```
用户提交表单
    ↓
/import/pocket/auth
    ├─→ getRequestToken() → Pocket API
    │      ↓
    │   requestToken
    │      ↓
    ├─→ session.set('import.pocket.code', requestToken)
    ├─→ session.set('mark_as_read', form_value)
    │
    └─→ 重定向到 Pocket 授权页面
                  ↓
            用户授权
                  ↓
/import/pocket/callback
    ├─→ session.get('mark_as_read')
    ├─→ session.remove('mark_as_read')
    ├─→ authorize(session.get('import.pocket.code')
    │      ↓
    │   Pocket API → accessToken
    │
    └─→ setMarkAsRead()->import()
                  ↓
            import(offset=0)
                ├─→ Pocket API v3/get (count=30, offset=0)
                ├─→ 有 producer?
                │   ├─ 是 → parseEntriesForProducer()
                │   └─ 否 → parseEntries()
                │        └─→ parseEntry()
                │             ├─ URL 选择
                │             ├─ 去重检查
                │             ├─ status/favorite 处理
                │             ├─ 标签创建
                │             └─ 预览图处理
                │
                └─→ 返回条目数 == 30?
                     ├─ 是 → import(offset=30)
                     └─ 否 → 结束
```
