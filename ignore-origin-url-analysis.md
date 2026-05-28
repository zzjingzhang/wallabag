# Ignore Origin URL 机制分析

## 概述

本文档详细分析 wallabag 在抓取网页后，当 URL 发生变化时如何决定保留 `originUrl`、替换 `url` 或忽略变化的完整机制。

---

## 1. ContentProxy::updateOriginUrl - URL 变化决策逻辑

### 方法签名与位置

[ContentProxy.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/ContentProxy.php#L328-L393)

```php
private function updateOriginUrl(Entry $entry, $url)
```

### 核心流程

#### 步骤 1：前置检查
```php
if (empty($url) || $entry->getUrl() === $url) {
    return false;
}
```
- 如果新 URL 为空或与原 URL 相同，直接返回，不做任何处理

#### 步骤 2：计算 URL 差异
```php
$parsed_entry_url = parse_url($entry->getUrl());
$parsed_content_url = parse_url($url);

$diff_ec = array_diff_assoc($parsed_entry_url, $parsed_content_url);
$diff_ce = array_diff_assoc($parsed_content_url, $parsed_entry_url);

$diff = array_merge($diff_ec, $diff_ce);
$diff_keys = array_keys($diff);
sort($diff_keys);
```

**差异计算逻辑**：
- 使用 `array_diff_assoc` 双向比较 URL 组件
- 合并两个方向的差异得到所有变化的 URL 部分
- `diff_keys` 为排序后的变化部分数组（如 `['path']`, `['scheme']`, `['host', 'path']` 等）

#### 步骤 3：Ignore Origin 规则检查
```php
if ($this->ignoreOriginProcessor->process($entry)) {
    $entry->setUrl($url);
    return false;
}
```
- 先调用 `RuleBasedIgnoreOriginProcessor` 检查是否匹配忽略规则
- 如果匹配，**直接更新 URL 而不保存 originUrl**，返回结束

#### 步骤 4：根据 URL 变化类型执行不同策略

根据 `$diff_keys` 的值，进入 switch 分支：

##### 分支 1：`case ['path']` - 仅路径变化

```php
case ['path']:
    if (($parsed_entry_url['path'] . '/' === $parsed_content_url['path'])
        || ($url === urldecode($entry->getUrl()))) {
        $entry->setUrl($url);
    }
    break;
```

**行为**：
- **仅当**满足以下条件之一时才更新 URL：
  1. 差异仅是尾部斜杠（如 `/path` → `/path/`）
  2. 新 URL 是原 URL 的 URL 解码版本
- **不保存 originUrl**
- 不满足条件时：URL 保持不变

##### 分支 2：`case ['scheme']` - 仅协议变化

```php
case ['scheme']:
    $entry->setUrl($url);
    break;
```

**行为**：
- 直接更新 URL（如 http → https）
- **不保存 originUrl**

##### 分支 3：`case ['fragment']` - 仅锚点变化

```php
case ['fragment']:
    // noop
    break;
```

**行为**：
- 无操作（noop - no operation）
- URL 和 originUrl 都保持不变
- 锚点变化被认为不影响内容的唯一性

##### 分支 4：`default` - 其他变化组合

```php
default:
    if (empty($entry->getOriginUrl())) {
        $entry->setOriginUrl($entry->getUrl());
    }
    $entry->setUrl($url);
    break;
```

**适用场景**（任意组合）：
- `['host']` - 域名变化
- `['host', 'path']` - 域名和路径都变化
- `['path', 'query']` - 路径和查询参数都变化
- `['scheme', 'host']` - 协议和域名都变化
- 等等...

**行为**：
1. **保存原始 URL**：如果 `originUrl` 为空，将当前 URL 保存到 `originUrl`
2. **更新当前 URL**：设置新抓取到的 URL
3. 这是唯一会设置 `originUrl` 的分支

---

## 2. RuleBasedIgnoreOriginProcessor - 规则匹配引擎

### 类位置与功能

[RuleBasedIgnoreOriginProcessor.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/RuleBasedIgnoreOriginProcessor.php#L10-L46)

该处理器负责检查 URL 是否匹配忽略规则，匹配时将跳过 originUrl 保存逻辑。

### process 方法详解

```php
public function process(Entry $entry)
{
    $url = $entry->getUrl();
    $userRules = $entry->getUser()->getConfig()->getIgnoreOriginRules()->toArray();
    $rules = array_merge($this->ignoreOriginInstanceRuleRepository->findAll(), $userRules);

    $parsed_url = parse_url($url);
    $parsed_url['_all'] = $url;

    foreach ($rules as $rule) {
        if ($this->rulerz->satisfies($parsed_url, $rule->getRule())) {
            $this->logger->info('Origin url matching ignore rule.', [
                'rule' => $rule->getRule(),
            ]);
            return true;
        }
    }

    return false;
}
```

### 规则合并机制

**规则优先级与合并顺序**：
```
实例级规则 (IgnoreOriginInstanceRule) + 用户级规则 (IgnoreOriginUserRule)
```

1. **实例级规则**：通过 `IgnoreOriginInstanceRuleRepository::findAll()` 获取
   - 系统全局规则，对所有用户生效
   - 通常在安装时配置

2. **用户级规则**：通过 `$entry->getUser()->getConfig()->getIgnoreOriginRules()` 获取
   - 用户自定义规则
   - 仅对该用户生效

3. **合并方式**：`array_merge(实例级规则, 用户级规则)`
   - 实例级规则在前，用户级规则在后
   - 遍历检查时按顺序匹配，**第一个匹配的规则立即生效**

### RulerZ 参数传递

向 RulerZ 规则引擎传递两个变量：

| 变量名 | 说明 | 示例值 |
|--------|------|--------|
| `host` | URL 的主机部分 | `"feedproxy.google.com"` |
| `_all` | 完整的 URL 字符串 | `"https://feedproxy.google.com/~r/..."` |

**支持的操作符**（由实体类注解定义）：
- `=` - 精确匹配
- `~` - 正则表达式匹配

[IgnoreOriginUserRule.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/IgnoreOriginUserRule.php#L28-L31)

### 规则示例

```yaml
# 精确匹配主机名
rule: host = "feedproxy.google.com"

# 正则匹配完整 URL
rule: _all ~ "/amp/"
```

---

## 3. InstallCommand::setupConfig - 默认规则加载

### 方法位置

[InstallCommand.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Command/InstallCommand.php#L340-L369)

### 加载流程

```php
private function setupConfig()
{
    // 清理现有数据
    $this->entityManager->createQuery('DELETE FROM Wallabag\Entity\InternalSetting')->execute();
    $this->entityManager->createQuery('DELETE FROM Wallabag\Entity\IgnoreOriginInstanceRule')->execute();

    // ... 加载 InternalSetting ...

    // 加载默认 Ignore Origin 规则
    foreach ($this->defaultIgnoreOriginInstanceRules as $ignore_origin_instance_rule) {
        $newIgnoreOriginInstanceRule = new IgnoreOriginInstanceRule();
        $newIgnoreOriginInstanceRule->setRule($ignore_origin_instance_rule['rule']);
        $this->entityManager->persist($newIgnoreOriginInstanceRule);
    }

    $this->entityManager->flush();
}
```

### 默认规则来源

默认规则定义在 [wallabag.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/wallabag.yml#L162-L175)：

```yaml
wallabag.default_ignore_origin_instance_rules:
    -
        rule: host = "feedproxy.google.com"
    -
        rule: host = "feeds.reuters.com"
    -
        rule: host = "feeds.feedburner.com"
    -
        rule: host = "rss.cnn.com"
```

**这些规则的用途**：
- 针对常见的 RSS 代理/转发服务
- 避免因 RSS 跳转导致产生大量不必要的 originUrl
- Google FeedProxy、Reuters、FeedBurner、CNN RSS 都是常见的 URL 跳转源

### 依赖注入配置

在 [services.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/services.yml#L275-L278) 中注入：

```yaml
Wallabag\Command\InstallCommand:
    arguments:
        $defaultSettings: '%wallabag.default_internal_settings%'
        $defaultIgnoreOriginInstanceRules: '%wallabag.default_ignore_origin_instance_rules%'
```

---

## 4. ConfigController - 用户规则管理

### 控制器位置

[ConfigController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/ConfigController.php#L46-L785)

### 规则管理功能

#### 4.1 添加/编辑规则

```php
// handle ignore origin rules
$ignoreOriginUserRule = new IgnoreOriginUserRule();
$action = $this->generateUrl('config') . '#set6';

if ($request->query->has('ignore-origin-user-rule')) {
    $ignoreOriginUserRule = $ignoreOriginUserRuleRepository
        ->find($request->query->get('ignore-origin-user-rule'));

    // 权限检查：确保用户只能编辑自己的规则
    if ($this->getUser()->getId() !== $ignoreOriginUserRule->getConfig()->getUser()->getId()) {
        return $this->redirect($action);
    }

    $action = $this->generateUrl('config', [
        'ignore-origin-user-rule' => $ignoreOriginUserRule->getId(),
    ]) . '#set6';
}

$newIgnoreOriginUserRule = $this->createForm(IgnoreOriginUserRuleType::class, $ignoreOriginUserRule, ['action' => $action]);
$newIgnoreOriginUserRule->handleRequest($request);

if ($newIgnoreOriginUserRule->isSubmitted() && $newIgnoreOriginUserRule->isValid()) {
    $ignoreOriginUserRule->setConfig($config);
    $this->entityManager->persist($ignoreOriginUserRule);
    $this->entityManager->flush();

    $this->addFlash('notice', 'flashes.config.notice.ignore_origin_rules_updated');

    return $this->redirect($this->generateUrl('config') . '#set6');
}
```

**编辑流程**：
1. 通过 URL 参数 `?ignore-origin-user-rule={id}` 指定要编辑的规则
2. 从数据库加载规则实体
3. **安全检查**：验证当前用户是否为规则所有者
4. 创建表单并绑定实体
5. 提交后关联用户配置并保存

#### 4.2 删除规则

```php
#[Route(path: '/ignore-origin-user-rule/delete/{ignoreOriginUserRule}', name: 'delete_ignore_origin_rule', methods: ['POST'], requirements: ['ignoreOriginUserRule' => '\d+'])]
#[IsGranted('DELETE', subject: 'ignoreOriginUserRule')]
public function deleteIgnoreOriginRuleAction(Request $request, IgnoreOriginUserRule $ignoreOriginUserRule)
{
    if (!$this->isCsrfTokenValid('delete-ignore-origin-rule', $request->request->get('token'))) {
        throw new BadRequestHttpException('Bad CSRF token.');
    }

    $this->entityManager->remove($ignoreOriginUserRule);
    $this->entityManager->flush();

    $this->addFlash('notice', 'flashes.config.notice.ignore_origin_rules_deleted');

    return $this->redirect($this->generateUrl('config') . '#set6');
}
```

**安全机制**：
- `#[IsGranted('DELETE', subject: 'ignoreOriginUserRule')]` - 使用 Voter 进行权限检查
- CSRF Token 验证
- 参数转换器自动加载实体

#### 4.3 编辑重定向

```php
#[Route(path: '/ignore-origin-user-rule/edit/{ignoreOriginUserRule}', name: 'edit_ignore_origin_rule', methods: ['GET'], requirements: ['ignoreOriginUserRule' => '\d+'])]
#[IsGranted('EDIT', subject: 'ignoreOriginUserRule')]
public function editIgnoreOriginRuleAction(IgnoreOriginUserRule $ignoreOriginUserRule)
{
    return $this->redirect($this->generateUrl('config') . '?ignore-origin-user-rule=' . $ignoreOriginUserRule->getId() . '#set6');
}
```

---

## 5. 完整流程总结

```
抓取内容 → URL 变化检测 → 规则检查 → 行为决策
    ↓
    ├─→ 匹配忽略规则 → [仅更新 URL]
    └─→ 不匹配规则 → 分析变化类型
                    ├─→ 仅路径变化（特殊情况）→ [仅更新 URL]
                    ├─→ 仅协议变化 → [仅更新 URL]
                    ├─→ 仅锚点变化 → [无操作]
                    └─→ 其他变化 → [保存 originUrl + 更新 URL]
```

### 决策表

| URL 变化类型 | 是否检查规则 | originUrl 行为 | url 行为 |
|-------------|------------|---------------|----------|
| 无变化 | - | 不变 | 不变 |
| 匹配 Ignore 规则 | ✓ | 不变 | 更新 |
| 仅路径变化（斜杠/解码） | ✓ | 不变 | 更新 |
| 仅路径变化（其他） | ✓ | 不变 | 不变 |
| 仅协议变化 | ✓ | 不变 | 更新 |
| 仅锚点变化 | ✓ | 不变 | 不变 |
| 主机变化 | ✓ | 保存原值 | 更新 |
| 主机+路径变化 | ✓ | 保存原值 | 更新 |
| 其他组合 | ✓ | 保存原值 | 更新 |

---

## 6. 实体类关系

### IgnoreOriginInstanceRule（实例级规则）

[IgnoreOriginInstanceRule.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/IgnoreOriginInstanceRule.php#L10-L69)

- 全局规则，所有用户共享
- 无用户关联字段
- 通常在安装时初始化

### IgnoreOriginUserRule（用户级规则）

[IgnoreOriginUserRule.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/IgnoreOriginUserRule.php#L10-L95)

- 每个用户独立配置
- 通过 `config` 关联到用户配置
- 用户在设置页面自行管理

### 共同接口

两个实体都实现：
- `IgnoreOriginRuleInterface` - 提供 `getRule()` 方法
- `RuleInterface` - 标记为规则实体

---

## 7. 关键设计考量

### 7.1 originUrl 的意义

`originUrl` 字段保存用户最初输入的 URL，用途：
1. **去重检测**：防止同一篇文章因跳转产生多个条目
2. **溯源**：用户可以看到原始链接
3. **重新抓取**：必要时可以从原始 URL 重新抓取

### 7.2 忽略规则的必要性

某些网站（特别是 RSS 转发服务）会在每次访问时生成不同的跳转 URL，如果不忽略这些变化，会导致：
- 大量不必要的 originUrl 记录
- 相同内容被识别为不同文章
- 数据库冗余

### 7.3 安全性设计

- 用户只能编辑/删除自己的规则（通过 Voter 和显式检查）
- 规则语法通过 RulerZ 验证器限制（仅允许 `host`、`_all` 变量，`=`、`~` 操作符）
- CSRF 保护所有修改操作
