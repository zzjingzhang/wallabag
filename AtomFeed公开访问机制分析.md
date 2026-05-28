# AtomFeed 公开访问机制分析

## 一、概述

Wallabag 的 AtomFeed 功能通过 **feedToken 令牌机制** 实现了无需登录态即可访问用户文章列表的能力。本文档详细分析其实现原理，包括令牌管理、用户身份解析、路由选择、分页限制、排序参数和安全配置等核心机制。

---

## 二、FeedToken 令牌管理

### 2.1 令牌生成：generateTokenAction

**位置**：[ConfigController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/ConfigController.php#L431-L451)

```php
#[Route(path: '/generate-token', name: 'generate_token', methods: ['POST'])]
#[IsGranted('EDIT_CONFIG')]
public function generateTokenAction(Request $request)
{
    if (!$this->isCsrfTokenValid('generate-token', $request->request->get('token'))) {
        throw new BadRequestHttpException('Bad CSRF token.');
    }

    $config = $this->getConfig();
    $config->setFeedToken(Utils::generateToken());

    $this->entityManager->persist($config);
    $this->entityManager->flush();

    // ... flash message & redirect
}
```

**变更流程**：
1. 验证 CSRF 令牌，防止跨站请求伪造
2. 获取当前用户的 Config 实体
3. 调用 [Utils::generateToken()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Tools/Utils.php#L14-L20) 生成新的 feedToken
   - 使用 `random_bytes()` 生成安全随机字节
   - Base64 编码后截取指定长度（默认 15 字符）
   - 移除 URL 不安全字符 `+` 和 `/`
4. 将新 token 设置到 Config 实体
5. 持久化到数据库

### 2.2 令牌撤销：revokeTokenAction

**位置**：[ConfigController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/ConfigController.php#L456-L476)

```php
#[Route(path: '/revoke-token', name: 'revoke_token', methods: ['POST'])]
#[IsGranted('EDIT_CONFIG')]
public function revokeTokenAction(Request $request)
{
    if (!$this->isCsrfTokenValid('revoke-token', $request->request->get('token'))) {
        throw new BadRequestHttpException('Bad CSRF token.');
    }

    $config = $this->getConfig();
    $config->setFeedToken(null);

    $this->entityManager->persist($config);
    $this->entityManager->flush();

    // ... flash message & redirect
}
```

**变更流程**：
1. 验证 CSRF 令牌
2. 获取当前用户的 Config 实体
3. 将 feedToken 设置为 `null`，立即使所有现有 feed URL 失效
4. 持久化到数据库

### 2.3 令牌存储

**位置**：[Config.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/Config.php#L50-L54)

```php
/**
 * @var string|null
 */
#[ORM\Column(name: 'feed_token', type: 'string', nullable: true)]
#[Groups(['config_api'])]
private $feedToken;
```

- 字段类型：`string`，可空
- 数据库索引：`feed_token` 列有单独的索引，提高查询性能

---

## 三、用户身份解析：UsernameFeedTokenConverter

### 3.1 转换器工作流程

**位置**：[UsernameFeedTokenConverter.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/ParamConverter/UsernameFeedTokenConverter.php)

这是一个自定义的 ParamConverter，负责从 URL 参数中解析用户名和 token，查找对应的 User 实体。

**执行流程**：

1. **参数提取**（第 68-73 行）：
   ```php
   $username = $request->attributes->get('username');
   $feedToken = $request->attributes->get('token');

   if (!$request->attributes->has('username') || !$request->attributes->has('token')) {
       return false;
   }
   ```

2. **用户查询**（第 86 行）：
   ```php
   $user = $userRepository->findOneByUsernameAndFeedtoken($username, $feedToken);
   ```

3. **结果处理**（第 88-93 行）：
   - 如果未找到用户，抛出 `NotFoundHttpException`
   - 将找到的 User 实体设置到 Request attributes 中，供 Controller 使用

### 3.2 数据库查询逻辑

**位置**：[UserRepository.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/UserRepository.php#L28-L36)

```php
public function findOneByUsernameAndFeedtoken($username, $feedToken)
{
    return $this->createQueryBuilder('u')
        ->leftJoin('u.config', 'c')
        ->where('c.feedToken = :feed_token')->setParameter('feed_token', $feedToken)
        ->andWhere('u.username = :username')->setParameter('username', $username)
        ->getQuery()
        ->getOneOrNullResult();
}
```

**查询特点**：
- 使用 `LEFT JOIN` 关联 `config` 表
- 同时匹配 `username` 和 `config.feedToken` 两个条件
- 任一条件不满足则返回 `null`

### 3.3 转换器的使用

在 FeedController 的各个路由方法上，通过注解启用该转换器：

```php
#[ParamConverter('user', class: User::class, converter: 'username_feed_token_converter')]
```

---

## 四、Feed 路由与类型选择

### 4.1 路由定义

**位置**：[FeedController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/FeedController.php)

| 路由方法 | 路径 | 类型 | 对应查询构建器 |
|---------|------|------|---------------|
| [showUnreadFeedAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/FeedController.php#L35-L41) | `/feed/{username}/{token}/unread/{page}` | `unread` | `getBuilderForUnreadByUser` |
| [showArchiveFeedAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/FeedController.php#L48-L54) | `/feed/{username}/{token}/archive/{page}` | `archive` | `getBuilderForArchiveByUser` |
| [showStarredFeedAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/FeedController.php#L61-L67) | `/feed/{username}/{token}/starred/{page}` | `starred` | `getBuilderForStarredByUser` |
| [showAllFeedAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/FeedController.php#L74-L80) | `/feed/{username}/{token}/all/{page}` | `all` | `getBuilderForAllByUser` |
| [showTagsFeedAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/FeedController.php#L87-L151) | `/feed/{username}/{token}/tags/{slug}/{page}` | `tag` | `findAllByTagId`（独立方法） |

### 4.2 通用类型选择逻辑

**位置**：[showEntries()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/FeedController.php#L178-L219)

```php
private function showEntries(string $type, User $user, $page = 1)
{
    $qb = match ($type) {
        'starred' => $this->entryRepository->getBuilderForStarredByUser($user->getId()),
        'archive' => $this->entryRepository->getBuilderForArchiveByUser($user->getId()),
        'unread' => $this->entryRepository->getBuilderForUnreadByUser($user->getId()),
        'all' => $this->entryRepository->getBuilderForAllByUser($user->getId()),
        default => throw new \InvalidArgumentException(\sprintf('Type "%s" is not implemented.', $type)),
    };

    // ... 分页处理与响应渲染
}
```

**设计特点**：
- 使用 PHP 8 的 `match` 表达式进行类型分发
- 每个类型对应独立的查询构建器方法
- 未知类型抛出明确的异常

### 4.3 标签 Feed 的独立处理

标签 Feed 有单独的 [showTagsFeedAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/FeedController.php#L87-L151) 方法，因为：
1. 需要额外的 `Tag` 实体参数转换
2. 支持自定义排序参数（见第五节）
3. 使用不同的仓库查询方法 `findAllByTagId`

---

## 五、用户自定义 FeedLimit 覆盖机制

### 5.1 全局配置

**位置**：[wallabag.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/wallabag.yml#L29)

```yaml
wallabag.feed_limit: 50
```

全局默认的 feed 文章数量限制为 **50**。

### 5.2 用户自定义配置

**位置**：[Config.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/Config.php#L56-L63)

```php
/**
 * @var int|null
 */
#[ORM\Column(name: 'feed_limit', type: 'integer', nullable: true)]
#[Assert\GreaterThanOrEqual(value: 1, message: 'validator.feed_limit_too_low')]
#[Assert\LessThanOrEqual(value: 100000, message: 'validator.feed_limit_too_high')]
#[Groups(['config_api'])]
private $feedLimit;
```

- 字段类型：`integer`，可空
- 验证约束：1 ≤ feedLimit ≤ 100000

### 5.3 表单配置

**位置**：[FeedType.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Form/Type/FeedType.php)

用户可以在配置页面通过表单设置自定义的 feed_limit。

### 5.4 覆盖逻辑

**位置**：
- 通用 Feed：[FeedController.php#L191](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/FeedController.php#L191)
- 标签 Feed：[FeedController.php#L127](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/FeedController.php#L127)

```php
$perPage = $user->getConfig()->getFeedLimit() ?: $this->feedLimit;
$entries->setMaxPerPage($perPage);
```

**覆盖策略**：
1. `$this->feedLimit` 是构造函数注入的全局配置参数
2. 使用 `?:` 运算符（NULL 合并运算符的变体）：
   - 如果用户设置了 `feedLimit`（非 null 且非 0），则使用用户值
   - 否则使用全局默认值 50

---

## 六、TagFeed Sort 参数限制机制

**位置**：[showTagsFeedAction()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/FeedController.php#L93-L102)

```php
$sort = $request->query->get('sort', 'created');

$sorts = [
    'created' => 'createdAt',
    'updated' => 'updatedAt',
];

if (!isset($sorts[$sort])) {
    throw new BadRequestHttpException(\sprintf('Sort "%s" is not available.', $sort));
}
```

**限制机制**：
1. 默认排序方式：`created`（按创建时间）
2. 只允许两种排序方式：
   - `created` → 映射到 `createdAt` 字段
   - `updated` → 映射到 `updatedAt` 字段
3. 使用白名单数组 `$sorts` 进行验证
4. 非法值抛出 `BadRequestHttpException`，错误信息明确指出不可用的排序值
5. 后续使用 `$sorts[$sort]` 获取实际的字段名进行查询

---

## 七、安全配置与匿名访问

### 7.1 公开访问注解

所有 FeedController 的路由都标记为允许公开访问：

```php
#[IsGranted('PUBLIC_ACCESS')]
```

这意味着不需要常规的登录认证，完全依赖 feedToken 进行身份验证。

### 7.2 Security 配置

**位置**：[security.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/security.yml#L62-L78)

```yaml
access_control:
    - { path: ^/api/(doc|version|info|user), roles: IS_AUTHENTICATED_ANONYMOUSLY }
    - { path: ^/login, roles: IS_AUTHENTICATED_ANONYMOUSLY }
    - { path: ^/logout, roles: [IS_AUTHENTICATED_ANONYMOUSLY, IS_AUTHENTICATED_2FA_IN_PROGRESS] }
    - { path: ^/register, role: IS_AUTHENTICATED_ANONYMOUSLY }
    - { path: ^/resetting, role: IS_AUTHENTICATED_ANONYMOUSLY }
    - { path: /(unread|starred|archive|annotated|all).xml$, roles: IS_AUTHENTICATED_ANONYMOUSLY }   # 旧RSS格式
    - { path: ^/locale, role: IS_AUTHENTICATED_ANONYMOUSLY }
    - { path: /tags/(.*).xml$, roles: IS_AUTHENTICATED_ANONYMOUSLY }                                 # 旧标签RSS格式
    - { path: ^/feed, roles: PUBLIC_ACCESS }                                                         # 新Atom Feed格式
    - { path: /(unread|starred|archive|annotated).xml$, roles: IS_AUTHENTICATED_ANONYMOUSLY }        # 向后兼容
    - { path: ^/share, roles: IS_AUTHENTICATED_ANONYMOUSLY }
    - { path: ^/settings, roles: ROLE_SUPER_ADMIN }
    - { path: ^/2fa, role: IS_AUTHENTICATED_2FA_IN_PROGRESS }
    - { path: ^/, roles: ROLE_USER }
```

**允许匿名访问的路径**：

| 路径模式 | 说明 |
|---------|------|
| `^/feed` | 新的 Atom Feed 路径（主要路径） |
| `/(unread\|starred\|archive\|annotated\|all).xml$` | 旧的 RSS 格式 Feed（不带用户名和 token） |
| `/tags/(.*).xml$` | 旧的标签 RSS 格式 |
| `/(unread\|starred\|archive\|annotated).xml$` | 向后兼容的重复配置 |

### 7.3 旧 RSS 重定向

**位置**：[routing.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/routing.yml#L48-L87)

为了向后兼容，系统提供了 5 个永久重定向路由，将旧的 RSS URL 格式重定向到新的 Atom Feed 格式：

| 重定向路由 | 旧路径 | 新路由 |
|-----------|--------|--------|
| `rss_to_atom_unread` | `/{username}/{token}/unread.xml` | `unread_feed` |
| `rss_to_atom_archive` | `/{username}/{token}/archive.xml` | `archive_feed` |
| `rss_to_atom_starred` | `/{username}/{token}/starred.xml` | `starred_feed` |
| `rss_to_atom_all` | `/{username}/{token}/all.xml` | `all_feed` |
| `rss_to_atom_tags` | `/{username}/{token}/tags/{slug}.xml` | `tag_feed` |

所有重定向都是 **301 永久重定向**（`permanent: true`），由 Symfony 内置的 `RedirectController` 处理。

---

## 八、整体架构总结

### 8.1 公开访问流程

```
用户请求 Feed URL
       ↓
  解析 URL 参数
  (username, token, type)
       ↓
UsernameFeedTokenConverter
       ↓
  查询 User + Config
  (匹配 username + feedToken)
       ↓
  验证通过 → 获取 User 实体
       ↓
根据 type 选择查询构建器
(unread/archive/starred/all/tag)
       ↓
  应用 feedLimit（用户 → 全局）
       ↓
  应用 sort 参数（仅 tag feed）
       ↓
  返回 Atom XML 响应
```

### 8.2 关键设计要点

1. **无状态认证**：通过 URL 中的 feedToken 实现身份验证，无需 Session/Cookie
2. **令牌可控**：用户可随时生成新令牌或撤销现有令牌
3. **粒度控制**：支持按用户级别自定义 feed 数量限制
4. **向后兼容**：提供旧 RSS 格式到新 Atom 格式的永久重定向
5. **安全限制**：sort 参数使用白名单验证，防止非法字段注入
6. **索引优化**：`feed_token` 列建有数据库索引，查询性能有保障

### 8.3 安全边界

- **优势**：
  - feedToken 仅用于 Feed 访问，泄露不会影响主账户安全
  - 令牌可随时撤销，泄露后可快速失效
  - CSRF 保护令牌生成和撤销操作

- **注意事项**：
  - feedToken 出现在 URL 中，可能被服务器日志、浏览器历史记录等记录
  - 建议用户定期轮换 feedToken
  - 不要在公共场合分享包含 feedToken 的 URL
