# Wallabag 文章公开分享完整路径追踪

## 目录

- [1. 整体流程概览](#1-整体流程概览)
- [2. 开启分享：EntryController::shareAction](#2-开启分享entrycontrollershareaction)
- [3. 关闭分享：EntryController::deleteShareAction](#3-关闭分享entrycontrollerdeleteshareaction)
- [4. Entry 实体的 uid 语义三角：generateUid / cleanUid / isPublic](#4-entry-实体的-uid-语义三角generateuid--cleanuid--ispublic)
- [5. 匿名访问分享页：shareEntryAction](#5-匿名访问分享页shareentryaction)
- [6. security.yml 为何允许 /share 匿名访问](#6-securityyml-为何允许-share-匿名访问)
- [7. API 层的 public 参数同步：postEntriesAction 与 patchEntriesAction](#7-api-层的-public-参数同步postentriesaction-与-patchentriesaction)
- [8. 完整调用链时序图](#8-完整调用链时序图)

---

## 1. 整体流程概览

文章公开分享涉及三个核心阶段：

| 阶段 | 触发方式 | 路由 | 权限 | 核心操作 |
|------|---------|------|------|---------|
| 开启分享 | Web POST / API POST/PATCH | `/share/{id}` 或 API `public=1` | `SHARE` + CSRF / API OAuth | `entry->generateUid()` |
| 访问分享 | 匿名 GET | `/share/{uid}` | `PUBLIC_ACCESS` + `share_public` 开关 | Doctrine ParamConverter 按 `uid` 查找 Entry |
| 关闭分享 | Web POST / API PATCH | `/share/delete/{id}` 或 API `public=0` | `UNSHARE` + CSRF / API OAuth | `entry->cleanUid()` |

---

## 2. 开启分享：EntryController::shareAction

**源码位置**：[shareAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/EntryController.php#L538-L556)

```php
#[Route(path: '/share/{id}', name: 'share', methods: ['POST'], requirements: ['id' => '\d+'])]
#[IsGranted('SHARE', subject: 'entry')]
public function shareAction(Request $request, Entry $entry)
{
    if (!$this->isCsrfTokenValid('share-entry', $request->request->get('token'))) {
        throw new BadRequestHttpException('Bad CSRF token.');
    }

    if (null === $entry->getUid()) {
        $entry->generateUid();
        $this->entityManager->persist($entry);
        $this->entityManager->flush();
    }

    return $this->redirect($this->generateUrl('share_entry', [
        'uid' => $entry->getUid(),
    ]));
}
```

### CSRF 保护

- 使用 `isCsrfTokenValid('share-entry', ...)` 验证表单提交的 `token` 字段
- 模板中对应生成：`{{ csrf_token('share-entry') }}`（见 [entry.html.twig](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/templates/Entry/entry.html.twig#L177)）
- CSRF Token ID 为 `'share-entry'`，与 `isCsrfTokenValid` 的第一个参数严格匹配
- 若 CSRF 验证失败，抛出 `BadRequestHttpException('Bad CSRF token.')`

### 对象级权限（EntryVoter）

- `#[IsGranted('SHARE', subject: 'entry')]` 声明当前用户必须对 `$entry` 拥有 `SHARE` 权限
- 由 [EntryVoter](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Security/Voter/EntryVoter.php#L17) 处理，逻辑如下：

```php
// EntryVoter::voteOnAttribute (第 50-51 行)
self::VIEW, self::EDIT, self::RELOAD, self::STAR, self::ARCHIVE,
self::SHARE, self::UNSHARE, self::EXPORT, self::DELETE, ... =>
    $user === $subject->getUser(),
```

即：**只有 Entry 的所有者（`$user === $entry->getUser()`）才能执行 SHARE 操作**。非所有者直接被拒绝。

### 幂等性设计

- 检查 `null === $entry->getUid()` 后才调用 `generateUid()`
- 若 Entry 已有 uid，则跳过生成，直接重定向到分享页
- 这意味着重复点击"分享"不会覆盖已有的 uid

---

## 3. 关闭分享：EntryController::deleteShareAction

**源码位置**：[deleteShareAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/EntryController.php#L563-L579)

```php
#[Route(path: '/share/delete/{id}', name: 'delete_share', methods: ['POST'], requirements: ['id' => '\d+'])]
#[IsGranted('UNSHARE', subject: 'entry')]
public function deleteShareAction(Request $request, Entry $entry)
{
    if (!$this->isCsrfTokenValid('delete-share', $request->request->get('token'))) {
        throw new BadRequestHttpException('Bad CSRF token.');
    }

    $entry->cleanUid();
    $this->entityManager->persist($entry);
    $this->entityManager->flush();

    return $this->redirect($this->generateUrl('view', [
        'id' => $entry->getId(),
    ]));
}
```

### CSRF 保护

- CSRF Token ID 为 `'delete-share'`
- 模板中对应：`{{ csrf_token('delete-share') }}`（见 [entry.html.twig](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/templates/Entry/entry.html.twig#L188)）
- 与 shareAction 的 Token ID **不同**，各自独立

### 对象级权限（EntryVoter）

- `#[IsGranted('UNSHARE', subject: 'entry')]` 要求 `UNSHARE` 权限
- EntryVoter 中 `UNSHARE` 同样只允许 Entry 所有者操作
- `SHARE` 和 `UNSHARE` 虽然权限名不同，但在 EntryVoter 中的判定逻辑完全一致：都是 `$user === $subject->getUser()`

### 模板层的双重门控

在 [entry.html.twig](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/templates/Entry/entry.html.twig#L173-L195) 中，分享按钮的展示有两层条件：

```
{% if craue_setting('share_public') %}        ← 第一层：全局开关
    {% if is_granted('SHARE', entry) %}       ← 第二层：对象权限
        ... 分享按钮 ...
    {% endif %}
    {% if is_granted('UNSHARE', entry) %}     ← 第二层：对象权限
        ... 取消分享按钮 ...
    {% endif %}
{% endif %}
```

只有 `share_public` 配置开启 **且** 用户拥有对应权限时，按钮才会渲染。

---

## 4. Entry 实体的 uid 语义三角：generateUid / cleanUid / isPublic

**源码位置**：[Entry.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/Entry.php#L750-L792)

### 数据库映射

```php
#[ORM\Column(name: 'uid', type: 'string', length: 23, nullable: true)]
#[ORM\Index(columns: ['uid'])]  // 表级索引，加速按 uid 查找
private $uid;
```

- `uid` 字段类型为 `string(23)`，**nullable**
- `null` 表示未分享，非 null 表示已分享
- 数据库层为 `uid` 建立了索引，用于 `/share/{uid}` 路由的快速查找

### generateUid()

```php
public function generateUid(): void
{
    if (null === $this->uid) {
        $this->uid = uniqid('', true);
    }
}
```

- **语义**：为 Entry 生成唯一的公开标识符，使其可通过 `/share/{uid}` 被匿名访问
- **幂等保护**：仅在 `uid === null` 时生成，不会覆盖已有值
- **生成算法**：`uniqid('', true)` — 基于 microtime 生成 23 字符的十六进制字符串（如 `6651a3b2d4e7f1.12345678`），`true` 参数增加额外熵使其更难预测
- **不返回值**：调用后需通过 `getUid()` 获取生成的 uid

### cleanUid()

```php
public function cleanUid(): void
{
    $this->uid = null;
}
```

- **语义**：清除 uid，关闭该 Entry 的公开分享
- **无幂等保护**：无论当前 uid 是否为 null，直接置 null
- **效果**：之后任何对 `/share/{旧uid}` 的访问将因 Doctrine ParamConverter 找不到 Entry 而返回 404

### isPublic()

```php
#[VirtualProperty]
#[SerializedName('is_public')]
#[Groups(['entries_for_user'])]
public function isPublic()
{
    return null !== $this->uid;
}
```

- **语义**：判断 Entry 是否处于公开分享状态
- **推导逻辑**：`uid !== null` ⟹ `isPublic === true`
- **不是独立字段**：`isPublic` 是 `uid` 的派生状态，不持久化，仅作为虚拟属性暴露
- **API 序列化**：在 `entries_for_user` 序列化组中以 `is_public` 名称输出
- **前端过滤**：在 [EntryFilterType](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Form/Type/EntryFilterType.php#L188-L203) 中，`isPublic` 过滤器实际转换为 `uid IS NOT NULL` 查询条件：

```php
->add('isPublic', CheckboxFilterType::class, [
    'apply_filter' => static function (QueryInterface $filterQuery, $field, $values) {
        // is_public isn't a real field
        // we should use the "uid" field to determine if the entry has been made public
        $expression = $filterQuery->getExpr()->isNotNull($values['alias'] . '.uid');
        return $filterQuery->createCondition($expression);
    },
])
```

### 三者关系

```
uid = null    ⟹  isPublic() = false  （未分享）
uid = "xxx"   ⟹  isPublic() = true   （已分享）

generateUid() : null → "xxx"  （开启分享）
cleanUid()    : "xxx" → null  （关闭分享）
```

---

## 5. 匿名访问分享页：shareEntryAction

**源码位置**：[shareEntryAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/EntryController.php#L586-L599)

```php
#[Route(path: '/share/{uid}', name: 'share_entry', methods: ['GET'], requirements: ['uid' => '.+'])]
#[Cache(maxage: 25200, smaxage: 25200, public: true)]
#[IsGranted('PUBLIC_ACCESS')]
public function shareEntryAction(Entry $entry, Config $craueConfig)
{
    if (!$craueConfig->get('share_public')) {
        throw $this->createAccessDeniedException('Sharing an entry is disabled for this user.');
    }

    return $this->render(
        'Entry/share.html.twig',
        ['entry' => $entry]
    );
}
```

### 为何是 PUBLIC_ACCESS 仍受 share_public 限制

这是一个**双层安全架构**的设计：

| 层级 | 机制 | 作用 |
|------|------|------|
| 第一层：框架层 | `#[IsGranted('PUBLIC_ACCESS')]` + security.yml | 允许匿名请求到达此 Action |
| 第二层：业务层 | `$craueConfig->get('share_public')` | 即使匿名请求到达，也检查管理员是否全局启用了分享功能 |

**为什么需要两层？**

1. **`PUBLIC_ACCESS` 是 Symfony Security 的概念**：它仅告诉安全组件"这个路由不需要认证"，是**访问控制**层面的放行。不设 `PUBLIC_ACCESS`，匿名用户根本无法到达此 Action。

2. **`share_public` 是业务配置开关**：管理员可能需要在**不修改代码和路由**的情况下，全局关闭分享功能。例如：
   - 系统维护期间临时禁止分享
   - 合规要求关闭公开访问
   - 单实例多用户场景下统一管控

3. **两者职责不同**：
   - `PUBLIC_ACCESS` 解答"匿名用户能否到达此路由？" → **是**
   - `share_public` 解答"系统是否允许公开展示文章？" → **视配置而定**

4. **不矛盾的设计**：`PUBLIC_ACCESS` 允许请求进来，`share_public` 在进来的请求中做二次判断。关闭 `share_public` 后，已存在的 uid 不会从数据库中删除，但访问 `/share/{uid}` 会被拒绝（返回 403），实现了**软开关**效果。

### Doctrine ParamConverter 如何按 uid 查找 Entry

路由参数名为 `uid`，方法签名为 `shareEntryAction(Entry $entry, ...)`。SensioFrameworkExtraBundle 的 `DoctrineParamConverter` 自动执行以下逻辑：

1. 检测到 `$entry` 类型为 `Entry` 实体
2. 路由参数 `uid` 匹配 `Entry` 实体的 `uid` 属性
3. 调用 `EntityManager->getRepository(Entry::class)->findBy(['uid' => $uid])`
4. 找不到时抛出 `NotFoundHttpException`（404）

这依赖于 Entry 实体中 `uid` 字段上的数据库索引（`#[ORM\Index(columns: ['uid'])]`）来保证查询效率。

### HTTP 缓存

```php
#[Cache(maxage: 25200, smaxage: 25200, public: true)]
```

- `maxage: 25200`：浏览器缓存 7 小时（25200 秒）
- `smaxage: 25200`：CDN/代理缓存 7 小时
- `public: true`：允许中间代理缓存（因为内容本身是公开的）

---

## 6. security.yml 为何允许 /share 匿名访问

**源码位置**：[security.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/security.yml#L75)

```yaml
access_control:
    - { path: ^/share, roles: IS_AUTHENTICATED_ANONYMOUSLY }
```

### 设计原因

1. **分享的本质是匿名访问**：`/share/{uid}` 的设计目标就是让**任何未经身份验证的人**通过链接查看文章。如果要求登录才能访问，分享就失去了意义。

2. **防火墙层面的放行**：`main` 防火墙虽然启用了 `form_login` 和 `anonymous: true`，但 `access_control` 中默认要求 `ROLE_USER`（最后一行 `path: ^/, roles: ROLE_USER`）。如果没有显式放行 `/share`，匿名用户会在 `access_control` 层被拦截，永远到达不了 `shareEntryAction`。

3. **路径匹配优先级**：Symfony 按顺序匹配 `access_control` 规则。`^/share` 在 `^/` 之前，确保 `/share/*` 路径优先匹配匿名放行规则。

4. **仅放行 GET 访问**：`/share/{uid}` 路由只接受 GET 方法（`methods: ['GET']`）。开启分享的 POST 请求（`/share/{id}`）路径虽然也匹配 `^/share`，但该路由有 `#[IsGranted('SHARE', subject: 'entry')]` 保护，匿名用户无法通过 EntryVoter 的权限检查。

5. **IS_AUTHENTICATED_ANONYMOUSLY vs PUBLIC_ACCESS**：
   - `IS_AUTHENTICATED_ANONYMOUSLY`：允许匿名用户和已认证用户
   - `PUBLIC_ACCESS`（Symfony 5.3+）：语义相同但更明确
   - 此处 `security.yml` 使用旧语法 `IS_AUTHENTICATED_ANONYMOUSLY`，Controller 注解使用新语法 `PUBLIC_ACCESS`，两者等效

### 安全保障

虽然 `/share` 路径允许匿名访问，但安全性由以下机制保障：

- **uid 不可预测**：`uniqid('', true)` 生成的 23 字符标识符具有一定不可预测性
- **share_public 开关**：管理员可全局关闭
- **仅限 GET**：不能通过匿名请求修改数据
- **无 uid 即 404**：ParamConverter 找不到 Entry 时直接 404

---

## 7. API 层的 public 参数同步：postEntriesAction 与 patchEntriesAction

### postEntriesAction（创建文章）

**源码位置**：[postEntriesAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L715-L811)

```php
#[Route(path: '/api/entries.{_format}', name: 'api_post_entries', methods: ['POST'])]
#[IsGranted('CREATE_ENTRIES')]
public function postEntriesAction(Request $request, ...)
{
    // ... 创建或查找 Entry ...

    $data = $this->retrieveValueFromRequest($request);

    // ... 处理 archive, starred, tags, origin_url ...

    if (null !== $data['isPublic']) {
        if (true === (bool) $data['isPublic'] && null === $entry->getUid()) {
            $entry->generateUid();
        } elseif (false === (bool) $data['isPublic']) {
            $entry->cleanUid();
        }
    }

    // ... persist & flush ...
}
```

### patchEntriesAction（修改文章）

**源码位置**：[patchEntriesAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L938-L1025)

```php
#[Route(path: '/api/entries/{entry}.{_format}', name: 'api_patch_entries', methods: ['PATCH'])]
#[IsGranted('EDIT', subject: 'entry')]
public function patchEntriesAction(Entry $entry, Request $request, ...)
{
    $data = $this->retrieveValueFromRequest($request);

    // ... 处理 content, title, language, archive, starred, tags ...

    if (null !== $data['isPublic']) {
        if (true === (bool) $data['isPublic'] && null === $entry->getUid()) {
            $entry->generateUid();
        } elseif (false === (bool) $data['isPublic']) {
            $entry->cleanUid();
        }
    }

    // ... persist & flush ...
}
```

### retrieveValueFromRequest 统一参数解析

**源码位置**：[retrieveValueFromRequest](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L1404-L1419)

```php
private function retrieveValueFromRequest(Request $request)
{
    return [
        // ...
        'isPublic' => $request->request->get('public'),
        // ...
    ];
}
```

API 请求体中的 `public` 参数被映射为内部 `isPublic` 键。

### public 参数同步 uid 的完整逻辑

```
请求体: public=1 (isPublic = true)
  ├─ entry.uid === null  →  entry.generateUid()   （首次开启分享）
  └─ entry.uid !== null  →  不操作                  （已分享，保持幂等）

请求体: public=0 (isPublic = false)
  └─ 无论 entry.uid 状态  →  entry.cleanUid()       （关闭分享）

请求体: 未传 public (isPublic = null)
  └─ null !== $data['isPublic'] 为 false  →  不操作  （不影响分享状态）
```

### Web 与 API 的对比

| 维度 | Web (EntryController) | API (EntryRestController) |
|------|----------------------|--------------------------|
| 开启分享 | `shareAction` + CSRF + `SHARE` 权限 | `public=1` 参数 + OAuth + `CREATE_ENTRIES`/`EDIT` 权限 |
| 关闭分享 | `deleteShareAction` + CSRF + `UNSHARE` 权限 | `public=0` 参数 + OAuth + `EDIT` 权限 |
| uid 生成 | `$entry->generateUid()` | `$entry->generateUid()`（同一方法） |
| uid 清除 | `$entry->cleanUid()` | `$entry->cleanUid()`（同一方法） |
| 权限粒度 | 区分 SHARE / UNSHARE | 不区分，统一使用 CREATE_ENTRIES 或 EDIT |
| CSRF 保护 | 必须 | 不需要（API 使用 OAuth token 认证） |

### API 中 POST 和 PATCH 的权限差异

- **POST（创建）**：`#[IsGranted('CREATE_ENTRIES')]` — 由 [MainVoter](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Security/Voter/MainVoter.php#L11) 判定，仅需 `ROLE_USER`
- **PATCH（修改）**：`#[IsGranted('EDIT', subject: 'entry')]` — 由 [EntryVoter](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Security/Voter/EntryVoter.php#L13) 判定，必须是 Entry 所有者

这意味着通过 API POST 创建新文章时即可同时设置 `public=1`，而 PATCH 修改已有文章的公开状态时，API 会验证操作者是 Entry 所有者。

---

## 8. 完整调用链时序图

```
┌────────┐     ┌──────────────────┐     ┌──────────────┐     ┌──────────┐     ┌─────────────┐
│  User  │     │ EntryController  │     │  EntryVoter  │     │   Entry  │     │ CraueConfig │
└───┬────┘     └────────┬─────────┘     └──────┬───────┘     └────┬─────┘     └──────┬──────┘
    │                   │                      │                  │                  │
    │  POST /share/{id} │                      │                  │                  │
    │  token=csrf_token │                      │                  │                  │
    ├──────────────────►│                      │                  │                  │
    │                   │                      │                  │                  │
    │                   │  isGranted(SHARE)    │                  │                  │
    │                   ├─────────────────────►│                  │                  │
    │                   │  user === owner?     │                  │                  │
    │                   │◄─────────────────────┤ true             │                  │
    │                   │                      │                  │                  │
    │                   │  isCsrfTokenValid()  │                  │                  │
    │                   │  ('share-entry')     │                  │                  │
    │                   │────────── ✓ ─────────│                  │                  │
    │                   │                      │                  │                  │
    │                   │  getUid()            │                  │                  │
    │                   ├──────────────────────┼─────────────────►│                  │
    │                   │  null                │                  │                  │
    │                   │◄─────────────────────┼──────────────────┤                  │
    │                   │                      │                  │                  │
    │                   │  generateUid()       │                  │                  │
    │                   ├──────────────────────┼─────────────────►│ uid = uniqid()   │
    │                   │                      │                  │                  │
    │                   │  persist + flush     │                  │                  │
    │                   ├──────────────────────┼─────────────────►│ SAVE uid to DB   │
    │                   │                      │                  │                  │
    │  302 → /share/{uid}                     │                  │                  │
    │◄──────────────────┤                      │                  │                  │
    │                   │                      │                  │                  │
    │  GET /share/{uid} │                      │                  │                  │
    ├──────────────────►│                      │                  │                  │
    │                   │                      │                  │                  │
    │                   │  ParamConverter      │                  │                  │
    │                   │  findBy(['uid'=>uid])│                  │                  │
    │                   ├──────────────────────┼─────────────────►│                  │
    │                   │  Entry $entry        │                  │                  │
    │                   │◄─────────────────────┼──────────────────┤                  │
    │                   │                      │                  │                  │
    │                   │  PUBLIC_ACCESS ✓     │                  │                  │
    │                   │                      │                  │                  │
    │                   │  craueConfig->get('share_public')       │                  │
    │                   ├──────────────────────┼──────────────────┼─────────────────►│
    │                   │  true                │                  │                  │
    │                   │◄─────────────────────┼──────────────────┼──────────────────┤
    │                   │                      │                  │                  │
    │  200 share.html.twig                     │                  │                  │
    │◄──────────────────┤                      │                  │                  │
    │                   │                      │                  │                  │
    │  POST /share/delete/{id}                 │                  │                  │
    │  token=csrf_token │                      │                  │                  │
    ├──────────────────►│                      │                  │                  │
    │                   │  isGranted(UNSHARE)  │                  │                  │
    │                   ├─────────────────────►│                  │                  │
    │                   │  user === owner?     │                  │                  │
    │                   │◄─────────────────────┤ true             │                  │
    │                   │                      │                  │                  │
    │                   │  isCsrfTokenValid()  │                  │                  │
    │                   │  ('delete-share')    │                  │                  │
    │                   │────────── ✓ ─────────│                  │                  │
    │                   │                      │                  │                  │
    │                   │  cleanUid()          │                  │                  │
    │                   ├──────────────────────┼─────────────────►│ uid = null       │
    │                   │                      │                  │                  │
    │                   │  persist + flush     │                  │                  │
    │                   ├──────────────────────┼─────────────────►│ SAVE null to DB  │
    │                   │                      │                  │                  │
    │  302 → /view/{id} │                      │                  │                  │
    │◄──────────────────┤                      │                  │                  │
```

---

## 关键文件索引

| 文件 | 关键行 | 职责 |
|------|--------|------|
| [EntryController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/EntryController.php#L538-L599) | L538-599 | shareAction / deleteShareAction / shareEntryAction |
| [Entry.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/Entry.php#L750-L792) | L750-792 | uid 字段映射 / generateUid / cleanUid / isPublic |
| [EntryVoter.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Security/Voter/EntryVoter.php#L17-L18) | L17-18 | SHARE / UNSHARE 权限常量与判定逻辑 |
| [MainVoter.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Security/Voter/MainVoter.php#L11) | L11 | CREATE_ENTRIES 等角色级权限 |
| [security.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/security.yml#L75) | L75 | `/share` 路径匿名放行 |
| [wallabag.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/wallabag.yml#L42-L44) | L42-44 | share_public 默认值定义 |
| [EntryRestController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/EntryRestController.php#L778-L784) | L778-784 | API public 参数 → uid 同步逻辑 |
| [EntryFilterType.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Form/Type/EntryFilterType.php#L188-L203) | L188-203 | isPublic 过滤器 → uid IS NOT NULL |
| [entry.html.twig](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/templates/Entry/entry.html.twig#L173-L195) | L173-195 | 分享按钮模板（双重门控） |
| [EntryRepository.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/EntryRepository.php#L306-L308) | L306-308 | isPublic 过滤查询 → uid IS [NOT] NULL |
