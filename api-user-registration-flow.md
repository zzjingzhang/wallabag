# PUT /api/user 创建用户与默认 OAuth Client 全流程追踪

## 目录

1. [总体流程概览](#1-总体流程概览)
2. [WallabagRestController::getInfoAction 如何暴露 registration_allowed](#2-wallabagrestcontrollergetinfoaction-如何暴露-registration_allowed)
3. [UserRestController::putUserAction 如何同时检查 WALLABAG_REGISTRATION_ENABLED 和 api_user_registration](#3-userrestcontrollerputuseraction-如何同时检查-wallabag_registration_enabled-和-api_user_registration)
4. [NewUserType 表单错误如何被转换为字段级 JSON](#4-newusertype-表单错误如何被转换为字段级-json)
5. [Client 实体如何生成 client_id 和 secret](#5-client-实体如何生成-client_id-和-secret)
6. [FOSUserEvents::USER_CREATED 如何触发 CreateConfigListener 创建 Config](#6-fosusereventsuser_created-如何触发-createconfiglistener-创建-config)
7. [security.yml 为什么允许 /api/user 匿名访问但其他 /api 路径走 OAuth 防火墙](#7-securityyml-为什么允许-apiuser-匿名访问但其他-api-路径走-oauth-防火墙)
8. [与网页开发者页面创建 Client 的流程对比](#8-与网页开发者页面创建-client-的流程对比)

---

## 1. 总体流程概览

`PUT /api/user` 端点的完整执行流程如下：

```
客户端发送 PUT /api/user
       │
       ▼
  security.yml 匹配 access_control 规则
  ^/api/(doc|version|info|user) → IS_AUTHENTICATED_ANONYMOUSLY
       │
       ▼
  UserRestController::putUserAction()
       │
       ├─► 检查 $this->registrationEnabled (WALLABAG_REGISTRATION_ENABLED 环境变量)
       │   AND $craueConfig->get('api_user_registration') (数据库配置)
       │   任一为 false → 返回 403
       │
       ├─► $userManager->createUser() 创建空 User 实体
       │   $user->setEnabled(false)  // 默认禁用
       │
       ├─► 创建 NewUserType 表单，关闭 CSRF
       │   $form->submit() 模拟提交
       │   │
       │   ├─► 表单无效 → 提取字段级错误 → 返回 400 JSON
       │   │
       │   └─► 表单有效 ↓
       │
       ├─► new Client($user) 创建默认 OAuth Client
       │   $client->setName($request->get('client_name', 'Default client'))
       │   $entityManager->persist($client)
       │   $user->addClient($client)
       │   $userManager->updateUser($user)
       │
       ├─► dispatch(FOSUserEvents::USER_CREATED)
       │   → CreateConfigListener::createConfig() 创建 Config 实体
       │
       └─► 返回 201，序列化 User（user_api_with_client 分组）
           包含 default_client（client_id、client_secret、name）
```

---

## 2. WallabagRestController::getInfoAction 如何暴露 registration_allowed

### 核心代码

[WallabagRestController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/WallabagRestController.php#L79-L88) 中的 `getInfoAction`：

```php
public function getInfoAction(Config $craueConfig)
{
    $info = new ApplicationInfo(
        $this->version,
        $this->registrationEnabled && $craueConfig->get('api_user_registration'),
    );

    return (new JsonResponse())->setJson($this->serializer->serialize($info, 'json'));
}
```

### 双重条件计算

`allowed_registration` 的值由两个条件的**逻辑与**决定：

1. **`$this->registrationEnabled`**：来自环境变量 `WALLABAG_REGISTRATION_ENABLED`，在 [services.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/services.yml#L35) 中绑定：
   ```yaml
   $registrationEnabled: '%env(bool:WALLABAG_REGISTRATION_ENABLED)%'
   ```

2. **`$craueConfig->get('api_user_registration')`**：来自数据库中的 CraueConfigBundle 配置项，默认值在 [wallabag.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/wallabag.yml#L150-L152) 中定义：
   ```yaml
   wallabag.default_internal_settings:
       -
           name: api_user_registration
           value: 0
           section: api
   ```

### ApplicationInfo 数据结构

[ApplicationInfo.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/Api/ApplicationInfo.php#L1-L44) 定义了返回结构：

```php
class ApplicationInfo
{
    public $appname;             // 固定为 "wallabag"
    public $version;             // 应用版本号
    public $allowed_registration; // 是否允许注册
}
```

响应示例：

```json
{
    "appname": "wallabag",
    "version": "2.7.0-dev",
    "allowed_registration": true
}
```

### 关键设计意图

此端点的目的是让**未认证**的客户端在调用 `PUT /api/user` 之前，先通过 `GET /api/info` 探测服务器是否允许注册，避免盲目发送注册请求。`allowed_registration` 字段的 OpenAPI 注解明确说明：`"Indicates whether registration is allowed. See PUT /api/user."`。

---

## 3. UserRestController::putUserAction 如何同时检查 WALLABAG_REGISTRATION_ENABLED 和 api_user_registration

### 核心代码

[UserRestController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/UserRestController.php#L105-L113)：

```php
public function putUserAction(Request $request, Config $craueConfig, UserManagerInterface $userManager, EntityManagerInterface $entityManager, EventDispatcherInterface $eventDispatcher)
{
    if (!$this->registrationEnabled || !$craueConfig->get('api_user_registration')) {
        $json = $this->serializer->serialize(['error' => "Server doesn't allow registrations"], 'json');

        return (new JsonResponse())
            ->setJson($json)
            ->setStatusCode(JsonResponse::HTTP_FORBIDDEN);
    }
    // ...
}
```

### 双层开关的设计

| 层级 | 变量 | 来源 | 作用域 | 修改方式 |
|------|------|------|--------|----------|
| 第一层 | `$this->registrationEnabled` | 环境变量 `WALLABAG_REGISTRATION_ENABLED` | 全局 | 修改 `.env` 或环境变量，需重启 |
| 第二层 | `$craueConfig->get('api_user_registration')` | 数据库 `craue_config` 表 | 仅 API 注册 | 通过管理界面或 `Config::set()` 动态修改 |

**两层的逻辑关系**：

- `$this->registrationEnabled` 是**全局总开关**：控制所有注册行为（包括 Web 注册表单和 API 注册）。[RegistrationListener](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Event/Listener/RegistrationListener.php#L29-L36) 也在 `FOSUserEvents::REGISTRATION_INITIALIZE` 中检查此值，如果为 false 则重定向到登录页。
- `api_user_registration` 是**API 专属开关**：即使全局注册开启，管理员也可以单独关闭 API 注册渠道，防止自动化脚本批量注册。

**短路求值**：当 `$this->registrationEnabled` 为 `false` 时，不会再去查询数据库中的 `api_user_registration`，直接返回 403。

### 与 getInfoAction 的一致性

`getInfoAction` 中的 `allowed_registration` 和 `putUserAction` 中的守卫条件使用完全相同的表达式：`$this->registrationEnabled && $craueConfig->get('api_user_registration')`。这保证了客户端看到的 `allowed_registration` 状态与实际注册行为一致。

---

## 4. NewUserType 表单错误如何被转换为字段级 JSON

### NewUserType 表单定义

[NewUserType.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Form/Type/NewUserType.php#L1-L60)：

```php
class NewUserType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add('username', TextType::class, ['required' => true])
            ->add('plainPassword', RepeatedType::class, [
                'type' => PasswordType::class,
                'invalid_message' => 'validator.password_must_match',
                'constraints' => [
                    new Length(['min' => 8, 'minMessage' => 'validator.password_too_short']),
                    new NotBlank(),
                ],
            ])
            ->add('email', EmailType::class)
            ->add('save', SubmitType::class);
    }
}
```

### 表单提交方式

在 `putUserAction` 中，表单通过编程方式提交而非浏览器提交：

```php
$form = $this->createForm(NewUserType::class, $user, [
    'csrf_protection' => false,  // API 无需 CSRF
]);

$form->submit([
    'username' => $request->request->get('username'),
    'plainPassword' => [
        'first' => $request->request->get('password'),
        'second' => $request->request->get('password'),
    ],
    'email' => $request->request->get('email'),
]);
```

注意 `plainPassword` 被映射为 `RepeatedType`，其内部结构为 `children.first` 和 `children.second`。

### 错误提取与转换的三步流程

[UserRestController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/UserRestController.php#L134-L158)：

**第一步**：利用 FOSRest 的 View 机制获取默认的表单错误结构

```php
$view = $this->view($form, 400);
$view->setFormat('json');
$data = json_decode($this->handleView($view)->getContent(), true)['errors']['children'];
```

此时 `$data` 的结构类似于：

```json
{
    "username": {
        "errors": ["fos_user.username.already_used"]
    },
    "email": {
        "errors": ["fos_user.email.already_used"]
    },
    "plainPassword": {
        "children": {
            "first": {
                "errors": ["validator.password_too_short"]
            },
            "second": {}
        }
    }
}
```

**第二步**：提取各字段错误并重映射键名

```php
$errors = [];

if (isset($data['username']['errors'])) {
    $errors['username'] = $this->translateErrors($data['username']['errors']);
}

if (isset($data['email']['errors'])) {
    $errors['email'] = $this->translateErrors($data['email']['errors']);
}

if (isset($data['plainPassword']['children']['first']['errors'])) {
    $errors['password'] = $this->translateErrors($data['plainPassword']['children']['first']['errors']);
}
```

关键映射：
- `plainPassword.children.first.errors` → `password`（API 使用 `password` 而非 `plainPassword`）
- 其他字段保持原名

**第三步**：翻译错误消息

[UserRestController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/UserRestController.php#L205-L213)：

```php
private function translateErrors($errors)
{
    $translatedErrors = [];
    foreach ($errors as $error) {
        $translatedErrors[] = $this->translator->trans($error);
    }
    return $translatedErrors;
}
```

FOSRest 默认返回的错误消息是翻译键（如 `validator.password_too_short`），`translateErrors` 通过 Symfony 的 Translator 将其翻译为用户可读的文本。

### 最终返回的 JSON 结构

```json
{
    "error": {
        "username": ["This value is already used."],
        "email": ["This value is already used."],
        "password": ["The password must be at least 8 characters long."]
    }
}
```

### 验证约束来源

| 字段 | 约束 | 来源 |
|------|------|------|
| `username` | `UniqueEntity('username')` | [User.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/User.php#L28) 的 `#[UniqueEntity('username')]` |
| `email` | `UniqueEntity('email')` | [User.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/User.php#L29) 的 `#[UniqueEntity('email')]` |
| `password` | `Length(min=8)` | [NewUserType.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Form/Type/NewUserType.php#L31-L34) 的 `new Length(['min' => 8])` |

---

## 5. Client 实体如何生成 client_id 和 secret

### Client 构造函数

[Client.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/Api/Client.php#L58-L62)：

```php
public function __construct(User $user)
{
    parent::__construct();
    $this->user = $user;
}
```

`Client` 继承自 `FOS\OAuthServerBundle\Entity\Client`（即 `BaseClient`），`parent::__construct()` 调用的是 FOSOAuthServerBundle 的 `BaseClient` 构造函数。

### FOSOAuthServerBundle BaseClient 的凭证生成

`FOS\OAuthServerBundle\Entity\Client` 继承自 `FOS\OAuthServerBundle\Model\Client`，其构造函数中会调用：

```php
public function __construct()
{
    $this->randomId = bin2hex(random_bytes(25));
    $this->secret = bin2hex(random_bytes(40));
    $this->redirectUris = [];
    $this->allowedGrantTypes = [];
}
```

- **`random_id`**：50 个十六进制字符（25 字节随机数），存储在 `oauth2_clients.random_id` 列
- **`secret`**：80 个十六进制字符（40 字节随机数），存储在 `oauth2_clients.secret` 列

### client_id 的合成

[Client.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/Api/Client.php#L103-L109)：

```php
#[VirtualProperty]
#[SerializedName('client_id')]
#[Groups(['user_api_with_client'])]
public function getClientId()
{
    return $this->getId() . '_' . $this->getRandomId();
}
```

`client_id` 不是独立存储的字段，而是一个**虚拟属性**，格式为 `{数据库自增ID}_{random_id}`，例如 `3_1lpybsn0od40css4w4ko8gsc8cwwskggs8kgg448ko0owo4c84`。

### client_secret 的暴露

`secret` 字段在 [Client.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/Api/Client.php#L51-L53) 中通过 JMS Serializer 注解直接序列化：

```php
#[SerializedName('client_secret')]
#[Groups(['user_api_with_client'])]
protected $secret;
```

### 在 putUserAction 中的创建流程

```php
// 1. 创建 Client，构造函数中自动生成 random_id 和 secret
$client = new Client($user);

// 2. 设置客户端名称（默认为 "Default client"）
$client->setName($request->request->get('client_name', 'Default client'));

// 3. 持久化到数据库（此时 id 由数据库自增生成）
$entityManager->persist($client);

// 4. 关联到用户
$user->addClient($client);

// 5. 保存用户（同时触发密码哈希等 FOSUserBundle 逻辑）
$userManager->updateUser($user);
```

**注意**：API 注册时 `Client` 没有显式设置 `allowedGrantTypes`，因此使用 BaseClient 构造函数的默认值（空数组）。这与网页开发者页面创建 Client 时的行为不同（后者会设置四种授权类型），详见[第8节](#8-与网页开发者页面创建-client-的流程对比)。

### 数据库表结构

`oauth2_clients` 表的核心字段：

| 列名 | 类型 | 说明 |
|------|------|------|
| `id` | INTEGER, AUTO_INCREMENT | 数据库自增主键 |
| `random_id` | VARCHAR(255) | 50 位十六进制随机字符串 |
| `secret` | VARCHAR(255) | 80 位十六进制随机字符串 |
| `name` | TEXT | 客户端名称 |
| `redirect_uris` | TEXT (array) | 回调 URL 数组 |
| `allowed_grant_types` | TEXT (array) | 允许的授权类型数组 |
| `user_id` | INTEGER | 关联的用户 ID |

---

## 6. FOSUserEvents::USER_CREATED 如何触发 CreateConfigListener 创建 Config

### 事件分发

[UserRestController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/UserRestController.php#L171-L172)：

```php
$eventDispatcher->dispatch(new UserEvent($user, $request), FOSUserEvents::USER_CREATED);
```

这里显式地手动分发了 `FOSUserEvents::USER_CREATED` 事件。在 `putUserAction` 中，由于使用了 `$userManager->updateUser()` 而非 FOSUserBundle 的 RegistrationController，FOSUserBundle 不会自动分发此事件，因此需要手动触发。

### CreateConfigListener 订阅

[CreateConfigListener.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Event/Listener/CreateConfigListener.php#L1-L67)：

```php
class CreateConfigListener implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            FOSUserEvents::REGISTRATION_COMPLETED => 'createConfig',
            FOSUserEvents::USER_CREATED => 'createConfig',
        ];
    }
}
```

`CreateConfigListener` 订阅了两个事件：

| 事件 | 触发场景 |
|------|----------|
| `REGISTRATION_COMPLETED` | 用户通过 Web 注册表单完成注册（含邮箱验证流程） |
| `USER_CREATED` | 手动创建用户：API 注册、Admin UI 创建、命令行安装 |

两个事件都指向同一个 `createConfig` 方法。

### Config 创建过程

```php
public function createConfig(UserEvent $event): void
{
    $language = $this->language;

    if ($this->requestStack->getMainRequest()) {
        $session = $this->requestStack->getMainRequest()->getSession();
        $language = $session->get('_locale', $this->language);
    }

    $user = $event->getUser();
    \assert($user instanceof User);

    $config = new Config($user);
    $config->setItemsPerPage($this->itemsOnPage);
    $config->setFeedLimit($this->feedLimit);
    $config->setLanguage($language);
    $config->setReadingSpeed($this->readingSpeed);
    $config->setActionMarkAsRead($this->actionMarkAsRead);
    $config->setListMode($this->listMode);
    $config->setDisplayThumbnails($this->displayThumbnails);

    $this->em->persist($config);
    $this->em->flush();
}
```

默认值来源于 [services.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/services.yml#L280-L288) 中的参数注入：

```yaml
Wallabag\Event\Listener\CreateConfigListener:
    arguments:
        $itemsOnPage: "%wallabag.items_on_page%"        # 12
        $feedLimit: "%wallabag.feed_limit%"              # 50
        $language: '%env(DEFAULT_LOCALE)%'               # 默认语言
        $readingSpeed: "%wallabag.reading_speed%"        # 200
        $actionMarkAsRead: "%wallabag.action_mark_as_read%" # 1
        $listMode: "%wallabag.list_mode%"                # 0
        $displayThumbnails: "%wallabag.display_thumbnails%" # 1
```

语言偏好会尝试从当前请求的 Session 中获取 `_locale`，如果不存在则使用默认语言。

### 同一事件的其他触发场景

除了 API 注册，以下场景也会分发 `FOSUserEvents::USER_CREATED`：

1. **Admin UI 创建用户**：[UserController::newAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/UserController.php#L49-L71) 中 `$eventDispatcher->dispatch($event, FOSUserEvents::USER_CREATED)`
2. **安装命令**：[InstallCommand::setupAdmin](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Command/InstallCommand.php#L330) 中 `$this->dispatcher->dispatch(new UserEvent($user), FOSUserEvents::USER_CREATED)`
3. **Fixtures**：[UserFixtures::load](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/fixtures/UserFixtures.php#L36) 中 `$this->dispatcher->dispatch(new UserEvent($user), FOSUserEvents::USER_CREATED)`

---

## 7. security.yml 为什么允许 /api/user 匿名访问但其他 /api 路径走 OAuth 防火墙

### 防火墙配置

[security.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/security.yml#L29-L34)：

```yaml
firewalls:
    api:
        pattern: /api/.*
        fos_oauth: true
        stateless: true
        anonymous: true
        provider: fos_userbundle
```

**`api` 防火墙**匹配所有 `/api/.*` 路径，使用 `fos_oauth` 作为认证提供者，同时启用 `anonymous: true`。

这意味着：
- 已携带有效 OAuth Bearer Token 的请求会被认证为具体用户
- 未携带 Token 的请求不会被拒绝，而是被赋予 `IS_AUTHENTICATED_ANONYMOUSLY` 角色

### 访问控制规则

[security.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/security.yml#L62-L78)：

```yaml
access_control:
    - { path: ^/api/(doc|version|info|user), roles: IS_AUTHENTICATED_ANONYMOUSLY }
    # ... 其他规则 ...
    - { path: ^/, roles: ROLE_USER }
```

**关键设计**：

| 路径 | 所需角色 | 说明 |
|------|----------|------|
| `/api/doc` | `IS_AUTHENTICATED_ANONYMOUSLY` | API 文档页面，公开访问 |
| `/api/version` | `IS_AUTHENTICATED_ANONYMOUSLY` | 版本信息，公开访问 |
| `/api/info` | `IS_AUTHENTICATED_ANONYMOUSLY` | 应用信息（含 registration_allowed），公开访问 |
| `/api/user` (PUT) | `IS_AUTHENTICATED_ANONYMOUSLY` | 注册新用户，必须匿名访问 |
| `/api/user` (GET) | `ROLE_USER`（由代码内部控制） | 获取当前用户信息，需认证 |
| `/api/entries` | `ROLE_USER` | 需要 OAuth 认证 |
| `/api/tags` | `ROLE_USER` | 需要 OAuth 认证 |

### /api/user 的双重访问模式

`/api/user` 同时处理 GET 和 PUT 请求，但安全需求截然不同：

- **GET /api/user**：需要认证，代码中通过 `$this->validateAuthentication()` 手动检查
- **PUT /api/user**：必须允许匿名访问，因为注册用户时还没有 OAuth 凭证

`access_control` 规则 `^/api/(doc|version|info|user)` 使用正则匹配路径，**不区分 HTTP 方法**。所以 PUT 和 GET 都被允许匿名访问。GET 的安全性由 `validateAuthentication()` 在代码层面保证：

```php
public function getUserAction()
{
    $this->validateAuthentication();
    return $this->sendUser($this->getUser());
}

protected function validateAuthentication(): void
{
    if (false === $this->authorizationChecker->isGranted('IS_AUTHENTICATED_FULLY')) {
        throw new AccessDeniedException();
    }
}
```

### 为什么不让 PUT /api/user 走 OAuth？

这是一个**鸡生蛋的问题**：用户注册时还没有 OAuth Client 和 Access Token，如果要求 OAuth 认证，就无法完成注册。因此必须允许匿名访问，并通过以下机制保护：

1. **双重开关守卫**：`WALLABAG_REGISTRATION_ENABLED` 和 `api_user_registration` 必须同时为 true
2. **用户默认禁用**：`$user->setEnabled(false)` 防止垃圾账号自动激活
3. **表单验证**：`NewUserType` 的约束（用户名唯一、邮箱唯一、密码最小长度）防止无效输入

### 其他 /api 路径的安全保障

对于不在匿名白名单中的 API 路径（如 `/api/entries`、`/api/tags`），access_control 没有显式规则，但它们会落入最后的兜底规则：

```yaml
- { path: ^/, roles: ROLE_USER }
```

加上 `api` 防火墙的 `fos_oauth: true` 和 `stateless: true`，这些路径需要有效的 OAuth Access Token 才能访问。

---

## 8. 与网页开发者页面创建 Client 的流程对比

### API 注册流程（PUT /api/user）

**入口**：[UserRestController::putUserAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/UserRestController.php#L105-L175)

```
PUT /api/user
    │
    ├─► 创建 User（默认禁用）
    ├─► new Client($user) → 构造函数自动生成 random_id 和 secret
    ├─► $client->setName('Default client')  // 或自定义 client_name
    ├─► $entityManager->persist($client)
    ├─► $user->addClient($client)
    ├─► $userManager->updateUser($user)
    └─► dispatch(USER_CREATED) → CreateConfigListener
```

### 网页开发者页面创建 Client

**入口**：[DeveloperController::createClientAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/DeveloperController.php#L39-L66)

```
GET/POST /developer/client/create
    │
    ├─► 必须已登录（main 防火墙，ROLE_USER）
    ├─► $client = new Client($this->getUser())
    ├─► $clientForm = $this->createForm(ClientType::class, $client)
    ├─► $clientForm->handleRequest($request)
    │   │
    │   ├─► 表单包含: name, redirect_uris
    │   └─► ClientType 定义见 ClientType.php
    │
    ├─► 表单有效后:
    │   $client->setAllowedGrantTypes(['token', 'authorization_code', 'password', 'refresh_token'])
    │   $entityManager->persist($client)
    │   $entityManager->flush()
    │
    └─► 渲染 client_parameters.html.twig 显示:
        - client_name
        - client_id (= getPublicId() = id + '_' + random_id)
        - client_secret (= getSecret())
```

### 关键差异对比

| 特性 | API 注册（PUT /api/user） | 网页开发者页面 |
|------|--------------------------|---------------|
| **控制器** | [UserRestController](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/UserRestController.php) | [DeveloperController](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/DeveloperController.php) |
| **表单类型** | [NewUserType](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Form/Type/NewUserType.php)（用户注册表单） | [ClientType](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Form/Type/Api/ClientType.php)（客户端创建表单） |
| **CSRF 保护** | 关闭（`csrf_protection: false`） | 开启（默认行为） |
| **认证要求** | 匿名（`IS_AUTHENTICATED_ANONYMOUSLY`） | 需登录（`ROLE_USER`，由 main 防火墙保障） |
| **创建对象** | User + Client（同时创建） | 仅 Client（用户已存在） |
| **Client 名称** | 默认 "Default client"，可通过 `client_name` 参数自定义 | 由表单字段 `name` 指定 |
| **redirect_uris** | 未设置（空数组） | 由表单字段 `redirect_uris` 指定 |
| **allowedGrantTypes** | 未设置（空数组，BaseClient 默认值） | 设置为 `['token', 'authorization_code', 'password', 'refresh_token']` |
| **用户状态** | 默认禁用（`setEnabled(false)`） | 无关（用户已存在） |
| **事件分发** | `FOSUserEvents::USER_CREATED` → 创建 Config | 无 |
| **返回方式** | JSON 响应（JMS 序列化，`user_api_with_client` 分组） | HTML 页面（[client_parameters.html.twig](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/templates/Developer/client_parameters.html.twig)） |
| **client_id 获取** | `Client::getClientId()` 虚拟属性（`id_randomId`） | `Client::getPublicId()`（FOSOAuthServer 基类方法，格式同为 `id_randomId`） |
| **后续操作** | 需管理员启用用户后才能使用 | 创建后立即可用 |

### allowedGrantTypes 差异的影响

API 注册时创建的 Client **没有设置 allowedGrantTypes**，这意味着默认为空数组。而网页开发者页面创建的 Client 设置了四种授权类型。

对于 OAuth2 协议而言，空 `allowedGrantTypes` 通常意味着该 Client 不被允许使用任何授权类型。API 注册场景下的设计意图是：注册时创建的 Client 主要用于后续通过 `password` 授权类型获取 Token。在实际使用中，FOSOAuthServerBundle 的 Token 控制器会检查 Client 的 `allowedGrantTypes` 是否包含请求的 `grant_type`。

如果需要 API 注册创建的 Client 也能正常工作，通常需要在注册后通过其他方式（如管理员操作）补充设置 `allowedGrantTypes`，或者修改 `putUserAction` 中的代码在创建 Client 时也设置授权类型。

### 流程时序图

**API 注册流程**：

```
客户端                     Server                         数据库
  │                          │                              │
  │── GET /api/info ────────►│                              │
  │◄─ {allowed_registration}─│                              │
  │                          │                              │
  │── PUT /api/user ────────►│                              │
  │   {username, password,   │── 检查双重开关 ──────────────►│
  │    email, client_name}   │◄─ 通过 ──────────────────────│
  │                          │                              │
  │                          │── createUser() ─────────────►│
  │                          │── NewUserType.submit()       │
  │                          │── new Client($user)          │
  │                          │   ├─ random_id 生成          │
  │                          │   └─ secret 生成             │
  │                          │── persist(client) ──────────►│
  │                          │── updateUser(user) ─────────►│
  │                          │── dispatch(USER_CREATED)     │
  │                          │   └─ CreateConfigListener    │
  │                          │      └─ persist(config) ────►│
  │◄─ 201 {user + client} ──│                              │
```

**网页开发者页面流程**：

```
用户                       Server                         数据库
  │                          │                              │
  │── GET /developer ───────►│（需登录）                     │
  │◄─ 客户端列表页面 ────────│                              │
  │                          │                              │
  │── GET /developer/        │                              │
  │   client/create ────────►│                              │
  │◄─ 创建表单页面 ─────────│                              │
  │                          │                              │
  │── POST /developer/       │                              │
  │   client/create ────────►│                              │
  │   {name, redirect_uris}  │── ClientType.handleRequest() │
  │                          │── setAllowedGrantTypes(...)  │
  │                          │── persist(client) ──────────►│
  │◄─ client_parameters 页面 │                              │
  │   (client_id, secret)    │                              │
```
