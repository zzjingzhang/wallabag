# 文章保存与删除时图片本地化与清理全流程追踪

## 目录

1. [整体架构概览](#1-整体架构概览)
2. [DownloadImagesSubscriber 事件订阅机制](#2-downloadimagessubscriber-事件订阅机制)
3. [download_images_enabled 开关与短路逻辑](#3-download_images_enabled-开关与短路逻辑)
4. [processHtml：提取 img/srcset 并替换 HTML](#4-processhtml提取-imgsrcset-并替换-html)
5. [processSingleImage：单图下载与安全处理全流程](#5-processsingleimage单图下载与安全处理全流程)
6. [getRelativePath：基于 entryId 的两级目录生成](#6-getrelativepath基于-entryid-的两级目录生成)
7. [onEntryDeleted 与 CleanDownloadedImagesCommand：失效目录清理](#7-onentrydeleted-与-cleandownloadedimagescommand失效目录清理)
8. [单元测试与功能测试验证的边界](#8-单元测试与功能测试验证的边界)

---

## 1. 整体架构概览

图片本地化与清理涉及以下核心类：

| 类 | 文件 | 职责 |
|---|---|---|
| `DownloadImagesSubscriber` | [DownloadImagesSubscriber.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Event/Subscriber/DownloadImagesSubscriber.php) | 事件订阅者，监听 `entry.saved` 和 `entry.deleted` |
| `DownloadImages` | [DownloadImages.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/DownloadImages.php) | 核心辅助类，负责 HTML 解析、图片下载、校验、重编码、路径生成、目录清理 |
| `CleanDownloadedImagesCommand` | [CleanDownloadedImagesCommand.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Command/CleanDownloadedImagesCommand.php) | CLI 命令，清理与已删除条目关联的孤立图片目录 |
| `EntrySavedEvent` | [EntrySavedEvent.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Event/EntrySavedEvent.php) | 条目保存事件，常量 `NAME = 'entry.saved'` |
| `EntryDeletedEvent` | [EntryDeletedEvent.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Event/EntryDeletedEvent.php) | 条目删除事件，常量 `NAME = 'entry.deleted'` |

数据流：

```
保存条目 → 派发 EntrySavedEvent
    → DownloadImagesSubscriber::onEntrySaved()
        → DownloadImages::processHtml()     ← 正文中的所有图片
        → DownloadImages::processSingleImage() ← 预览图
        → EntityManager::persist + flush

删除条目 → 派发 EntryDeletedEvent
    → DownloadImagesSubscriber::onEntryDeleted()
        → DownloadImages::removeImages()

定时/手动清理 → php bin/console wallabag:clean-downloaded-images
    → CleanDownloadedImagesCommand::execute()
```

---

## 2. DownloadImagesSubscriber 事件订阅机制

### 2.1 事件注册

[DownloadImagesSubscriber::getSubscribedEvents()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Event/Subscriber/DownloadImagesSubscriber.php#L23-L29) 返回事件映射表：

```php
public static function getSubscribedEvents(): array
{
    return [
        EntrySavedEvent::NAME => 'onEntrySaved',    // 'entry.saved' → onEntrySaved()
        EntryDeletedEvent::NAME => 'onEntryDeleted', // 'entry.deleted' → onEntryDeleted()
    ];
}
```

由于该类实现了 `EventSubscriberInterface` 并在 [services.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/services.yml#L267-L269) 中注册为服务，Symfony 事件调度器会自动注册这些监听器。

### 2.2 onEntrySaved 处理流程

[onEntrySaved()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Event/Subscriber/DownloadImagesSubscriber.php#L34-L61) 的完整执行步骤：

1. **开关检查**：若 `$this->enabled` 为 false，记录日志并直接 return
2. **获取 Entry**：从事件对象 `$event->getEntry()` 获取条目实体
3. **下载正文图片**：调用私有方法 [downloadImages($entry)](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Event/Subscriber/DownloadImagesSubscriber.php#L84-L91)，内部委托 `DownloadImages::processHtml($entryId, $content, $url)`
4. **更新正文**：若 `processHtml` 返回非 false 值，调用 `$entry->setContent($html)` 替换 HTML
5. **下载预览图**：调用私有方法 [downloadPreviewImage($entry)](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Event/Subscriber/DownloadImagesSubscriber.php#L100-L107)，内部委托 `DownloadImages::processSingleImage($entryId, $previewPicture, $url)`
6. **更新预览图**：若返回非 false 值，调用 `$entry->setPreviewPicture($previewPicture)`
7. **持久化**：`$this->em->persist($entry)` + `$this->em->flush()` 将更新写入数据库

### 2.3 onEntryDeleted 处理流程

[onEntryDeleted()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Event/Subscriber/DownloadImagesSubscriber.php#L66-L75)：

1. **开关检查**：若 `$this->enabled` 为 false，直接 return
2. **删除图片目录**：调用 `$this->downloadImages->removeImages($event->getEntry()->getId())`

### 2.4 事件派发时机

| 场景 | 事件 | 派发时机 |
|---|---|---|
| 单条创建 `POST /api/entries` | `entry.saved` | `persist + flush` 之后 |
| 批量创建 `POST /api/entries/lists` | `entry.saved` | `persist + flush` 之后 |
| 修改条目 `PATCH /api/entries/{entry}` | `entry.saved` | `persist + flush` 之后 |
| 重载条目 `PATCH /api/entries/{entry}/reload` | `entry.saved` | `persist + flush` 之后 |
| 导入流程 | `entry.saved` | 每 20 条 flush 后批量派发 |
| 单条删除 `DELETE /api/entries/{entry}` | `entry.deleted` | `remove + flush` **之前** |
| 批量删除 `DELETE /api/entries/list` | `entry.deleted` | `remove + flush` **之前** |

> **关键设计**：`entry.saved` 在 flush 之后派发，确保 Entry 已有数据库 ID，`DownloadImages` 可据此生成存储路径；`entry.deleted` 在 remove 之前派发，确保数据库记录仍存在（虽然实际只需 entry ID）。

---

## 3. download_images_enabled 开关与短路逻辑

### 3.1 配置来源

`download_images_enabled` 存储在 `craue_config_setting` 数据库表中，由 [Version20161031132655](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/migrations/Version20161031132655.php) 迁移插入，默认值为 `0`（关闭）：

```php
$this->addSql('INSERT INTO ' . $this->getTable('craue_config_setting') . 
    " (name, value, section) VALUES ('download_images_enabled', 0, 'misc')");
```

### 3.2 注入方式

在 [services.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/services.yml#L267-L269) 中，通过表达式语言在服务实例化时从 `craue_config` 读取并注入：

```yaml
Wallabag\Event\Subscriber\DownloadImagesSubscriber:
    arguments:
        $enabled: '@=service(''craue_config'').get(''download_images_enabled'')'
```

> **注意**：`$enabled` 的值在服务实例化时确定，是 **一次性的快照**，运行期间修改配置需要重建容器才能生效（功能测试中通过 `Config::set()` 修改后立即生效是因为测试环境会重新获取服务）。

### 3.3 短路逻辑

`onEntrySaved()` 和 `onEntryDeleted()` 的第一行均进行开关检查：

```php
if (!$this->enabled) {
    $this->logger->debug('DownloadImagesSubscriber: disabled.');
    return;
}
```

当 `download_images_enabled = 0` 时，两个方法在开头即 return，**不执行任何图片下载、HTML 替换或目录清理操作**。这意味着：

- 条目内容中的远程图片 URL 保持不变
- 预览图 URL 保持不变
- 删除条目时不清理图片目录

---

## 4. processHtml：提取 img/srcset 并替换 HTML

### 4.1 方法签名

[processHtml($entryId, $html, $url)](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/DownloadImages.php#L65-L93)

- `$entryId`：条目 ID，用于生成本地存储路径
- `$html`：条目的 HTML 正文内容
- `$url`：条目的原始 URL，作为相对路径解析的 base

### 4.2 图片 URL 提取

[extractImagesUrlsFromHtml($html)](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/DownloadImages.php#L46-L54) 是一个 **静态方法**，执行两步提取：

```php
public static function extractImagesUrlsFromHtml($html)
{
    $crawler = new Crawler($html);
    $imagesCrawler = $crawler->filterXpath('//img');
    $imagesUrls = $imagesCrawler->extract(['src']);          // ① 提取所有 <img src="...">
    $imagesSrcsetUrls = self::getSrcsetUrls($imagesCrawler); // ② 提取所有 srcset 中的 URL

    return array_unique(array_merge($imagesUrls, $imagesSrcsetUrls));
}
```

**srcset 提取**：[getSrcsetUrls()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/DownloadImages.php#L275-L302) 遍历所有 `<img>` 节点的 `srcset` 属性，使用正则 `/([^"'\s]+\s*(?:\d+[wx])+)/` 匹配 `URL 宽度描述符` 的模式（如 `image.jpg 2x`、`image-600w.jpg 600w`），然后取每项的 URL 部分（第一个空格前）。

### 4.3 URL 排序

```php
arsort($imagesUrls);
```

使用 `arsort` 对 URL 按值降序排列，**防止短 URL 替换时误匹配长 URL 的前缀**。例如，若存在 `http://example.com/img.jpg` 和 `http://example.com/img.jpg?w=600`，降序排列后先替换较长的 URL，避免短 URL 先替换导致长 URL 中的路径被截断。

### 4.4 逐图处理与 HTML 替换

```php
$relativePath = $this->getRelativePath($entryId);

foreach ($imagesUrls as $image) {
    $newImage = $this->processSingleImage($entryId, $image, $url, $relativePath);

    if (false === $newImage) {
        continue;  // 处理失败则保留原始 URL
    }

    $html = str_replace($image, $newImage, $html);
    
    // 处理 HTML 实体编码的 & 号
    if (false !== stripos($image, '&') && false === stripos($html, $image)) {
        $imageAmp = str_replace('&', '&amp;', $image);
        $html = str_replace($imageAmp, $newImage, $html);
        $imageUnicode = str_replace('&', '&#038;', $image);
        $html = str_replace($imageUnicode, $newImage, $html);
    }
}
```

**`&` 实体编码兼容**：HTML 中的 `&` 可能被编码为 `&amp;` 或 `&#038;`（WordPress 等常见）。当图片 URL 含 `&` 且直接替换未命中时，会尝试对这两种 HTML 实体编码形式也执行替换。

### 4.5 返回值

返回替换了所有图片 URL 的 HTML 字符串。如果所有图片都处理失败，则返回原始 HTML（不做任何修改）。

---

## 5. processSingleImage：单图下载与安全处理全流程

[processSingleImage($entryId, $imagePath, $url, $relativePath = null)](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/DownloadImages.php#L108-L222)

### 5.1 前置校验

```php
if (null === $imagePath) {
    return false;  // 预览图可能为 null
}
```

### 5.2 构造绝对 URL

[getAbsoluteLink($base, $url)](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/DownloadImages.php#L325-L342)：

1. 若 URL 已以 `http://` 或 `https://` 开头，直接返回（已是绝对路径）
2. 否则，使用 Guzzle 的 `UriResolver::resolve()` 将相对 URL 解析为绝对 URL
3. 若 base URL 缺少 scheme 或 host（如 `imgur.com/...` 无协议前缀），返回 false

```php
private function getAbsoluteLink($base, $url)
{
    if (preg_match('!^https?://!i', $url)) {
        return $url;
    }
    $base = new Uri($base);
    if ('' === $base->getAuthority() || '' === $base->getScheme()) {
        return false;
    }
    return (string) UriResolver::resolve($base, new Uri($url));
}
```

### 5.3 HTTP 请求下载

```php
try {
    $res = $this->client->request(Request::METHOD_GET, $absolutePath);
} catch (\Exception $e) {
    $this->logger->error('DownloadImages: Can not retrieve image, skipping.', ['exception' => $e]);
    return false;
}
```

### 5.4 扩展名校验（HTTP 状态 + MIME + 魔数字）

[getExtensionFromResponse($res, $imagePath)](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/DownloadImages.php#L352-L388) 执行三层校验：

**第一层：HTTP 状态码校验**

```php
if (200 !== $res->getStatusCode()) {
    return false;  // 非 200 直接拒绝
}
```

**第二层：Content-Type MIME 校验**

```php
$ext = current($this->mimeTypes->getExtensions(
    current($res->getHeaders()['content-type'] ?? [])
));
```

从 HTTP 响应头的 `Content-Type` 中提取 MIME 类型，再通过 Symfony 的 `MimeTypes` 组件反查文件扩展名。

**第三层：文件魔数字（Magic Bytes）校验**（当 Content-Type 无法确定扩展名时的后备方案）

```php
$types = [
    'jpeg' => "\xFF\xD8\xFF",           // JPEG 文件头
    'gif'  => 'GIF',                     // GIF 文件头
    'png'  => "\x89\x50\x4e\x47\x0d\x0a", // PNG 文件头（8 字节中的前 6 字节）
    'webp' => "\x52\x49\x46\x46",        // RIFF 容器头（WEBP 基于 RIFF）
];
$bytes = substr($res->getContent(), 0, 8);

foreach ($types as $type => $header) {
    if (str_starts_with($bytes, $header)) {
        $ext = $type;
        break;
    }
}
```

读取响应体前 8 字节，与已知文件签名字节序列做前缀匹配。这是在服务器返回错误或缺失 `Content-Type` 时的兜底策略。

**白名单校验**：

```php
if (!\in_array($ext, ['jpeg', 'jpg', 'gif', 'png', 'webp', 'svg'], true)) {
    return false;  // 不在允许列表中的扩展名，跳过
}
```

只有 6 种图片格式被允许：`jpeg`、`jpg`、`gif`、`png`、`webp`、`svg`。

### 5.5 本地文件路径与 URL 路径计算

```php
$hashImage = hash('crc32', $absolutePath);   // 用图片绝对 URL 的 CRC32 作为文件名
$localPath = $folderPath . '/' . $hashImage . '.' . $ext;
$urlPath   = $this->wallabagUrl . '/assets/images/' . $relativePath . '/' . $hashImage . '.' . $ext;
```

- 同一张图片（相同绝对 URL）始终映射到相同的本地文件名
- 返回的 `$urlPath` 是通过 wallabag 站点 URL 前缀构造的本地访问路径

### 5.6 SVG 特殊处理：sanitize

```php
if ('svg' === $ext) {
    $sanitizer = new Sanitizer();
    $sanitizer->minify(true);                // 压缩 SVG 输出
    $sanitizer->removeRemoteReferences(true); // 移除远程引用（防 XSS）
    $cleanSVG = $sanitizer->sanitize($res->getContent());

    if (false === $cleanSVG || !str_contains($cleanSVG, '<svg ')) {
        $this->logger->error('DownloadImages: Bad SVG given', ['path' => $imagePath]);
        return false;  // sanitize 失败或结果不含合法 SVG 标签
    }

    file_put_contents($localPath, $cleanSVG);
    return $urlPath;
}
```

SVG 使用 [enshrined/svgSanitize](https://github.com/darylldoyle/svg-sanitizer) 库进行消毒：

- `minify(true)`：压缩 SVG 输出
- `removeRemoteReferences(true)`：移除 `xlink:href` 等远程引用，防止 SVG 中的 XSS 攻击
- 额外校验：`sanitize()` 返回值必须非 false 且包含 `<svg ` 标签，双重确认是合法 SVG

> **GD 库不支持 SVG**，因此 SVG 走独立分支，不经过 `imagecreatefromstring` 重编码。

### 5.7 位图重编码（jpg/png/gif/webp）

对于非 SVG 图片，使用 GD 库进行**重新编码**（安全考虑：消除可能的图片木马）：

```php
$im = imagecreatefromstring($res->getContent());
```

若 `imagecreatefromstring` 失败（数据不是合法图片），返回 false。

按扩展名分别处理：

| 格式 | 方法 | 特殊处理 |
|---|---|---|
| **gif** | `imagegif($im, $localPath)` | 若存在 Imagick 扩展，优先用 `Imagick::readImageBlob` + `writeImages` **保留 GIF 动画**；Imagick 失败则回退到 GD（GD 不支持动画） |
| **jpeg/jpg** | `imagejpeg($im, $localPath, 80)` | 质量 = `REGENERATE_PICTURES_QUALITY = 80` |
| **png** | `imagepng($im, $localPath, 7)` | 先 `imagealphablending(false)` + `imagesavealpha(true)` **保留透明通道**；压缩级别 = `ceil(80/100*9) = 7` |
| **webp** | `imagewebp($im, $localPath, 80)` | 质量 = 80 |

最后 `imagedestroy($im)` 释放内存。

---

## 6. getRelativePath：基于 entryId 的两级目录生成

[getRelativePath($entryId, $createFolder = true)](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/DownloadImages.php#L255-L268)

### 6.1 算法

```php
$hashId = hash('crc32', (string) $entryId);
$relativePath = $hashId[0] . '/' . $hashId[1] . '/' . $hashId;
```

步骤分解：

1. 将 `$entryId` 转为字符串，计算 CRC32 哈希（8 位十六进制字符串，如 `9b0ead26`）
2. 取哈希第 1 个字符作为第一级目录（如 `9`）
3. 取哈希第 2 个字符作为第二级目录（如 `b`）
4. 完整哈希值作为第三级目录（如 `9b0ead26`）

### 6.2 生成的目录结构

```
web/assets/images/           ← baseFolder
├── 9/                       ← 第一级：CRC32 首字符
│   └── b/                   ← 第二级：CRC32 第二字符
│       └── 9b0ead26/        ← 第三级：完整 CRC32 哈希
│           ├── ebe60399.jpg ← 图片文件（URL 的 CRC32 + 扩展名）
│           └── 43cc0123.png
├── a/
│   └── f/
│       └── af12c3d4/
│           └── ...
```

### 6.3 目录自动创建

```php
if (!file_exists($folderPath) && $createFolder) {
    mkdir($folderPath, 0777, true);  // 递归创建多级目录
}
```

`$createFolder` 参数默认为 `true`；`CleanDownloadedImagesCommand` 中调用 `getRelativePath` 时也使用默认值，仅用于路径计算（目录此时已存在）。

### 6.4 设计意图

两级散列目录避免所有条目的图片目录平铺在同一个父目录下（单目录文件数过多会导致文件系统性能下降）。CRC32 首两位字符各有 16 种可能（0-9, a-f），理论上最多 256 个二级目录，每个二级目录下再分散存储条目图片目录。

---

## 7. onEntryDeleted 与 CleanDownloadedImagesCommand：失效目录清理

### 7.1 实时清理：onEntryDeleted → removeImages

当条目被删除时，[onEntryDeleted()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Event/Subscriber/DownloadImagesSubscriber.php#L66-L75) 调用 [removeImages($entryId)](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/DownloadImages.php#L229-L245)：

```php
public function removeImages($entryId): void
{
    $relativePath = $this->getRelativePath($entryId);
    $folderPath = $this->baseFolder . '/' . $relativePath;

    $finder = new Finder();
    $finder
        ->files()
        ->ignoreDotFiles(true)
        ->in($folderPath);

    foreach ($finder as $file) {
        @unlink($file->getRealPath());  // 删除所有非隐藏文件
    }

    @rmdir($folderPath);  // 删除目录本身（若目录非空则失败，@ 抑制错误）
}
```

执行步骤：

1. 根据 entryId 计算相对路径（如 `9/b/9b0ead26`）
2. 拼接完整目录路径
3. 使用 Symfony Finder 遍历目录中所有文件（忽略点文件）
4. 逐个 `unlink` 删除文件
5. 尝试 `rmdir` 删除目录（`@` 抑制错误：若目录非空或不存在，静默失败）

### 7.2 批量清理：CleanDownloadedImagesCommand

[CleanDownloadedImagesCommand](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Command/CleanDownloadedImagesCommand.php) 是一个 CLI 命令，用于清理 **孤儿图片目录**（条目已删除但图片目录未被清理的情况，例如 `download_images_enabled` 之前为关闭状态时删除的条目）。

命令名：`wallabag:clean-downloaded-images`

#### 执行流程

1. **扫描现有目录**：

```php
$finder = new Finder();
$finder
    ->directories()
    ->ignoreDotFiles(true)
    ->depth(2)       // 只查找深度为 2 的目录（即 CRC32 哈希目录）
    ->in($baseFolder);

foreach ($finder as $file) {
    $existingPaths[] = $file->getFilename();  // 收集哈希目录名
}
```

`depth(2)` 精确匹配 `baseFolder/第一级/第二级/哈希目录` 结构中的第三级。

2. **获取有效目录**：

```php
$entries = $this->entryRepository->findAllEntriesIdByUserId();

foreach ($entries as $entry) {
    $path = $this->downloadImages->getRelativePath($entry['id']);
    if (!file_exists($baseFolder . '/' . $path)) {
        continue;  // 条目存在但无图片目录，跳过
    }
    $validPaths[] = explode('/', $path)[2];  // 只取 CRC32 哈希部分
}
```

[findAllEntriesIdByUserId()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L635-L645) 查询所有条目的 ID（不区分用户），对每个 ID 重新计算路径哈希，确认目录存在后标记为有效。

3. **对比与删除**：

```php
foreach ($existingPaths as $existingPath) {
    if (!\in_array($existingPath, $validPaths, true)) {
        $fullPath = $baseFolder . '/' . $existingPath[0] . '/' . $existingPath[1] . '/' . $existingPath;
        $files = glob($fullPath . '/*.*');
        if (!$dryRun) {
            array_map('unlink', $files);  // 删除所有文件
            rmdir($fullPath);              // 删除目录
        }
        $deletedCount += \count($files);
    }
}
```

不在 `$validPaths` 中的目录即为孤儿目录，删除其中所有文件及目录本身。

#### dry-run 模式

```bash
php bin/console wallabag:clean-downloaded-images --dry-run
```

`--dry-run` 选项只输出统计信息，不执行实际删除操作。

---

## 8. 单元测试与功能测试验证的边界

### 8.1 单元测试：DownloadImagesTest

[DownloadImagesTest](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/unit/Helper/DownloadImagesTest.php) 共 10 个测试用例，覆盖以下边界：

| 测试方法 | 验证的边界 | 关键断言 |
|---|---|---|
| [testProcessHtml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/unit/Helper/DownloadImagesTest.php#L31-L44) | 正常 `<img src>` 替换 | 返回 HTML 包含本地路径前缀 |
| [testProcessHtmlWithBadImage](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/unit/Helper/DownloadImagesTest.php#L46-L57) | Content-Type 不匹配（`application/json`）时图片不被替换 | 原始 URL 保留在 HTML 中 |
| [testProcessSingleImage](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/unit/Helper/DownloadImagesTest.php#L73-L84) | 多种 MIME 类型（`pjpeg`、`jpeg`、`png`、`gif`、`webp`）的正确扩展名映射 | 返回路径包含正确扩展名 |
| [testProcessSingleImageWithBadUrl](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/unit/Helper/DownloadImagesTest.php#L86-L97) | HTTP 404 响应 | 返回 `false` |
| [testProcessSingleImageWithBadImage](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/unit/Helper/DownloadImagesTest.php#L99-L110) | Content-Type 声明为 `image/png` 但响应体为空（`imagecreatefromstring` 失败） | 返回 `false` |
| [testProcessSingleImageFailAbsolute](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/unit/Helper/DownloadImagesTest.php#L112-L123) | base URL 无协议、相对图片路径无法构造绝对 URL | 返回 `false` |
| [testProcessRealImage](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/unit/Helper/DownloadImagesTest.php#L125-L142) | 无 Content-Type 时通过魔数字识别图片类型 | 返回本地路径，日志中包含 "Checking extension (alternative)" |
| [testProcessImageWithSrcset](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/unit/Helper/DownloadImagesTest.php#L144-L159) | srcset 属性中多个 URL 的解析与替换 | HTML 中不再包含原始域名 |
| [testProcessImageWithTrickySrcset](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/unit/Helper/DownloadImagesTest.php#L161-L180) | srcset 中含复杂格式（多行 sizes、`f_auto,q_auto` 参数） | HTML 中不再包含 `f_auto,q_auto` |
| [testProcessImageWithNumericHtmlEntitySeparator](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/unit/Helper/DownloadImagesTest.php#L182-L198) | srcset 中 `&#038;` HTML 实体编码的 `&` 号替换 | 原始域名完全从 HTML 中移除 |
| [testProcessImageWithNullPath](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/unit/Helper/DownloadImagesTest.php#L200-L215) | `$imagePath` 为 `null` | 返回 `false` |
| [testEnsureOnlyFirstOccurrenceIsReplaced](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/unit/Helper/DownloadImagesTest.php#L217-L235) | src 和 srcset 引用同一图片时，两者分别被替换为不同的本地文件（因为 URL 不同） | 两个 URL 都被替换为本地路径，且 `srcset` 保留宽度描述符 `1290w` |
| [testProcessSingleImageWithSvg](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/unit/Helper/DownloadImagesTest.php#L237-L248) | SVG 文件正常 sanitize 并保存 | 返回 `.svg` 本地路径 |
| [testProcessSingleImageWithBadSvg](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/unit/Helper/DownloadImagesTest.php#L250-L261) | Content-Type 声明为 SVG 但实际内容是 PNG（sanitize 失败） | 返回 `false` |

### 8.2 功能测试：EntryControllerTest

[EntryControllerTest](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/functional/Controller/EntryControllerTest.php) 中有 2 个图片相关的功能测试：

| 测试方法 | 验证场景 |
|---|---|
| [testNewEntryWithDownloadImagesEnabled](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/functional/Controller/EntryControllerTest.php#L1271-L1309) | 开启 `download_images_enabled` 后保存条目，验证正文中的图片 URL 被替换为本地路径（`/assets/images/`） |
| [testRemoveEntryWithDownloadImagesEnabled](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/tests/functional/Controller/EntryControllerTest.php#L1314-L1349) | 依赖上一个测试，开启开关后删除条目，验证删除流程正常完成（302 重定向） |

功能测试的清理保障：

```php
public function tearDownImagesEnabled(): void
{
    if ($this->downloadImagesEnabled) {
        $client = static::createClient();
        $client->getContainer()->get(Config::class)->set('download_images_enabled', '0');
        $this->downloadImagesEnabled = false;
    }
}
```

每个测试方法执行完毕后，自动将 `download_images_enabled` 重置为 `0`，避免影响其他测试。

### 8.3 测试覆盖的边界总结

| 边界条件 | 覆盖方式 |
|---|---|
| 开关关闭时的短路 | 功能测试隐式覆盖（默认关闭状态下的正常操作） |
| `imagePath = null` | 单元测试 `testProcessImageWithNullPath` |
| HTTP 非 200 响应 | 单元测试 `testProcessSingleImageWithBadUrl`（404） |
| Content-Type 缺失/不匹配 | 单元测试 `testProcessRealImage`（魔数字回退）、`testProcessHtmlWithBadImage` |
| 图片数据损坏（`imagecreatefromstring` 失败） | 单元测试 `testProcessSingleImageWithBadImage` |
| 绝对 URL 构造失败 | 单元测试 `testProcessSingleImageFailAbsolute` |
| SVG sanitize 失败 | 单元测试 `testProcessSingleImageWithBadSvg` |
| SVG 正常处理 | 单元测试 `testProcessSingleImageWithSvg` |
| `&amp;` HTML 实体编码替换 | 单元测试 `testProcessHtml`（`&amp;` 数据提供者） |
| `&#038;` 数字字符引用替换 | 单元测试 `testProcessImageWithNumericHtmlEntitySeparator` |
| srcset 多 URL 解析 | 单元测试 `testProcessImageWithSrcset`、`testProcessImageWithTrickySrcset` |
| src 与 srcset 相同图片的不同替换 | 单元测试 `testEnsureOnlyFirstOccurrenceIsReplaced` |
| GIF 动画保留 | 仅代码层面覆盖（Imagick 可用时），无专项测试 |
| PNG 透明通道保留 | 代码层面覆盖，无专项测试 |
| `removeImages` 目录删除 | 功能测试 `testRemoveEntryWithDownloadImagesEnabled` 隐式覆盖 |
| `CleanDownloadedImagesCommand` 孤儿目录清理 | **无测试覆盖** |
