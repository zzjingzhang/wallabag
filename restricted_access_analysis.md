# Wallabag restricted_access 付费墙/需登录网站抓取机制分析

## 概述

当 `restricted_access` 配置开启后，Wallabag 具备为需要登录或付费墙网站抓取内容的能力。整个流程涉及凭据管理、加密存储、自动登录、会话保持等多个环节。本文档详细分析从凭据创建到内容抓取的完整技术实现。

---

## 1. restricted_access 配置说明

### 1.1 配置定义

配置位于 [wallabag.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/wallabag.yml#L90-L92)，默认值为 0（禁用）：

```yaml
-
    name: restricted_access
    value: 0
    section: entry
```

### 1.2 服务注入

在 [services.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/services.yml#L36-L36) 中，通过表达式语言动态获取配置值：

```yaml
$restrictedAccess: '@=service(''craue_config'').get(''restricted_access'')'
```

### 1.3 全局检查点

- **控制器层面**：所有 `SiteCredentialController` 操作前都会检查，详见 [SiteCredentialController::isSiteCredentialsEnabled](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/SiteCredentialController.php#L154-L159)
- **HTTP 客户端层面**：每次请求前检查，详见 [WallabagClient::request](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/HttpClient/WallabagClient.php#L31-L33)
- **模板层面**：在 [layout.html.twig](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/templates/layout.html.twig#L115-L120) 中控制菜单显示

---

## 2. SiteCredentialController - 凭据创建与编辑流程

### 2.1 restricted_access 检查

所有操作入口都会调用 `isSiteCredentialsEnabled()` 方法，位于 [SiteCredentialController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/SiteCredentialController.php#L154-L159)：

```php
private function isSiteCredentialsEnabled(): void
{
    if (!$this->craueConfig->get('restricted_access')) {
        throw $this->createNotFoundException('Feature "restricted_access" is disabled, controllers too.');
    }
}
```

### 2.2 创建新凭据 - newAction

位于 [SiteCredentialController::newAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/SiteCredentialController.php#L57-L85)，核心流程：

1. 检查 `restricted_access` 配置
2. 创建 `SiteCredential` 实体并关联当前用户
3. 处理表单提交
4. **加密用户名和密码**（关键步骤）：
   ```php
   $credential->setUsername($this->cryptoProxy->crypt($credential->getUsername()));
   $credential->setPassword($this->cryptoProxy->crypt($credential->getPassword()));
   ```
5. 持久化到数据库

### 2.3 编辑凭据 - editAction

位于 [SiteCredentialController::editAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/SiteCredentialController.php#L94-L122)，与创建流程类似，表单提交后同样会重新加密：

```php
$siteCredential->setUsername($this->cryptoProxy->crypt($siteCredential->getUsername()));
$siteCredential->setPassword($this->cryptoProxy->crypt($siteCredential->getPassword()));
```

### 2.4 数据实体

凭据数据存储在 [SiteCredential.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/SiteCredential.php) 实体中，核心字段：
- `host`：站点域名（如 `example.com`）
- `username`：加密后的用户名
- `password`：加密后的密码
- `user`：关联的 Wallabag 用户

---

## 3. CryptoProxy - 加密与密钥管理

### 3.1 密钥文件路径配置

在 [wallabag.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/wallabag.yml#L39-L39) 中定义：

```yaml
wallabag.site_credentials.encryption_key_path: "%kernel.project_dir%/data/site-credentials-secret-key.txt"
```

在 [services.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/services.yml#L23-L23) 中注入：

```yaml
$encryptionKeyPath: "%wallabag.site_credentials.encryption_key_path%"
```

### 3.2 密钥自动创建

位于 [CryptoProxy::__construct](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/CryptoProxy.php#L18-L30)：

```php
public function __construct(
    $encryptionKeyPath,
    private readonly LoggerInterface $logger,
) {
    if (!file_exists($encryptionKeyPath)) {
        $key = Key::createNewRandomKey();
        file_put_contents($encryptionKeyPath, $key->saveToAsciiSafeString());
        chmod($encryptionKeyPath, 0600);
    }
    $this->encryptionKey = file_get_contents($encryptionKeyPath);
}
```

**关键点**：
- 密钥文件不存在时自动创建
- 使用 `Defuse\Crypto\Key` 生成安全随机密钥
- 密钥以 ASCII 安全字符串格式存储
- 文件权限设置为 `0600`（仅所有者可读写）

### 3.3 加密实现 - crypt 方法

位于 [CryptoProxy::crypt](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/CryptoProxy.php#L39-L44)：

```php
public function crypt($secretValue)
{
    $this->logger->debug('Crypto: crypting value: ' . $this->mask($secretValue));
    return Crypto::encrypt($secretValue, $this->loadKey());
}
```

使用 `Defuse\Crypto\Crypto::encrypt()` 进行加密。

### 3.4 解密实现 - decrypt 方法

位于 [CryptoProxy::decrypt](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/CryptoProxy.php#L53-L62)：

```php
public function decrypt($cryptedValue)
{
    $this->logger->debug('Crypto: decrypting value: ' . $this->mask($cryptedValue));
    try {
        return Crypto::decrypt($cryptedValue, $this->loadKey());
    } catch (WrongKeyOrModifiedCiphertextException $e) {
        throw new \RuntimeException('Decrypt fail: ' . $e->getMessage());
    }
}
```

### 3.5 密钥加载 - loadKey 方法

位于 [CryptoProxy::loadKey](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/CryptoProxy.php#L69-L72)：

```php
private function loadKey()
{
    return Key::loadFromAsciiSafeString($this->encryptionKey);
}
```

### 3.6 日志脱敏 - mask 方法

位于 [CryptoProxy::mask](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/CryptoProxy.php#L81-L84)：

```php
private function mask($value)
{
    return \strlen($value) > 0 ? $value[0] . '*****' . $value[\strlen($value) - 1] : 'Empty value';
}
```

---

## 4. SiteCredentialRepository - 凭据查询与解密

### 4.1 findOneByHostsAndUser 方法

位于 [SiteCredentialRepository::findOneByHostsAndUser](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Repository/SiteCredentialRepository.php#L33-L52)，完整实现：

```php
public function findOneByHostsAndUser($hosts, $userId)
{
    $res = $this->createQueryBuilder('s')
        ->select('s.username', 's.password')
        ->where('s.host IN (:hosts)')->setParameter('hosts', $hosts)
        ->andWhere('s.user = :userId')->setParameter('userId', $userId)
        ->setMaxResults(1)
        ->getQuery()
        ->getOneOrNullResult();

    if (null === $res) {
        return null;
    }

    // decrypt user & password before returning them
    $res['username'] = $this->cryptoProxy->decrypt($res['username']);
    $res['password'] = $this->cryptoProxy->decrypt($res['password']);

    return $res;
}
```

**流程说明**：
1. 根据主机列表和用户ID查询数据库
2. 仅选取 `username` 和 `password` 字段
3. 查询结果为空时返回 `null`
4. **关键步骤**：返回前通过 `CryptoProxy::decrypt()` 解密用户名和密码
5. 返回解密后的明文凭据

---

## 5. GrabySiteConfigBuilder - SiteConfig 构造流程

### 5.1 服务配置

在 [services.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/services.yml#L232-L238) 中定义：

```yaml
Wallabag\SiteConfig\GrabySiteConfigBuilder:
    tags:
        - { name: monolog.logger, channel: graby }

Wallabag\SiteConfig\SiteConfigBuilder:
    alias: Wallabag\SiteConfig\GrabySiteConfigBuilder
```

### 5.2 buildForHost 主方法

位于 [GrabySiteConfigBuilder::buildForHost](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/SiteConfig/GrabySiteConfigBuilder.php#L23-L80)，完整流程：

#### 步骤 1：获取当前登录用户

```php
$user = $this->getUser();
```

`getUser()` 方法通过 TokenStorage 获取，详见 [GrabySiteConfigBuilder::getUser](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/SiteConfig/GrabySiteConfigBuilder.php#L109-L116)：

```php
private function getUser()
{
    if ($this->token->getToken() && null !== $this->token->getToken()->getUser()) {
        return $this->token->getToken()->getUser();
    }
    return null;
}
```

#### 步骤 2：Host 规范化处理

```php
$host = strtolower($host);
if (str_starts_with($host, 'www.')) {
    $host = substr($host, 4);
}
```

#### 步骤 3：生成候选主机列表

```php
$hosts = [$host];
// will try to see for a host without the first subdomain (fr.example.org & .example.org)
$split = explode('.', $host);

if (\count($split) > 1) {
    // remove first subdomain
    array_shift($split);
    $hosts[] = '.' . implode('.', $split);
}
```

**示例**：对于 `fr.example.org`，生成的 `$hosts` 为 `['example.org', '.example.org']`

#### 步骤 4：查询并解密凭据

```php
$credentials = $this->credentialRepository->findOneByHostsAndUser($hosts, $user->getId());
```

#### 步骤 5：构建 Graby SiteConfig

```php
$config = $this->grabyConfigBuilder->buildForHost($host);
```

#### 步骤 6：合并参数构造 SiteConfig

```php
$parameters = [
    'host' => $host,
    'requiresLogin' => $config->requires_login ?: false,
    'loginUri' => $config->login_uri ?: null,
    'usernameField' => $config->login_username_field ?: null,
    'passwordField' => $config->login_password_field ?: null,
    'extraFields' => $this->processExtraFields($config->login_extra_fields),
    'notLoggedInXpath' => $config->not_logged_in_xpath ?: null,
    'username' => $credentials['username'],
    'password' => $credentials['password'],
    'httpHeaders' => $config->http_header,
];

$config = new SiteConfig($parameters);
```

#### 步骤 7：日志脱敏

```php
$parameters['username'] = '**masked**';
$parameters['password'] = '**masked**';
```

### 5.3 processExtraFields 方法

位于 [GrabySiteConfigBuilder::processExtraFields](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/SiteConfig/GrabySiteConfigBuilder.php#L90-L107)，处理额外登录字段：

```php
protected function processExtraFields($extraFieldsStrings)
{
    if (!\is_array($extraFieldsStrings)) {
        return [];
    }

    $extraFields = [];
    foreach ($extraFieldsStrings as $extraField) {
        if (!str_contains((string) $extraField, '=')) {
            continue;
        }
        [$fieldName, $fieldValue] = explode('=', (string) $extraField, 2);
        $extraFields[$fieldName] = $fieldValue;
    }

    return $extraFields;
}
```

### 5.4 SiteConfig 数据结构

位于 [SiteConfig.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/SiteConfig/SiteConfig.php)，核心属性：
- `host`：站点主机
- `requiresLogin`：是否需要登录
- `loginUri`：登录表单提交 URI
- `usernameField` / `passwordField`：表单字段名
- `extraFields`：额外提交字段
- `notLoggedInXpath`：检测未登录状态的 XPath
- `username` / `password`：明文凭据
- `httpHeaders`：HTTP 请求头

---

## 6. Authenticator - 登录认证流程

### 6.1 loginIfRequired - 主动登录检查

位于 [Authenticator::loginIfRequired](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/HttpClient/Authenticator.php#L32-L50)：

```php
public function loginIfRequired(string $url): bool
{
    $config = $this->buildSiteConfig(new Uri($url));
    if (false === $config || !$config->requiresLogin()) {
        $this->logger->debug('loginIfRequired> will not require login');
        return false;
    }

    if ($this->authenticator->isLoggedIn($config)) {
        return false;
    }

    $this->logger->debug('loginIfRequired> user is not logged in, attach authenticator');
    $this->authenticator->login($config);

    return true;
}
```

**流程**：
1. 解析 URL 构建 `SiteConfig`
2. 检查站点是否需要登录
3. 检查是否已有有效会话（Cookie）
4. 未登录则执行登录

### 6.2 loginIfRequested - 响应式登录检查

位于 [Authenticator::loginIfRequested](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/HttpClient/Authenticator.php#L52-L80)：

```php
public function loginIfRequested(ResponseInterface $response): bool
{
    $config = $this->buildSiteConfig(new Uri($response->getInfo('url')));
    if (false === $config || !$config->requiresLogin()) {
        return false;
    }

    $body = $response->getContent();
    if ('' === $body) {
        return false;
    }

    $isLoginRequired = $this->authenticator->isLoginRequired($config, $body);

    if (!$isLoginRequired) {
        return false;
    }

    $this->authenticator->login($config);
    return true;
}
```

**流程**：
1. 从响应中获取 URL 构建 `SiteConfig`
2. 检查响应内容是否为空
3. 通过 XPath 检测页面是否显示未登录状态
4. 需要登录则执行登录后重试

---

## 7. LoginFormAuthenticator - 登录操作实现

### 7.1 login - 执行登录请求

位于 [LoginFormAuthenticator::login](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/SiteConfig/LoginFormAuthenticator.php#L27-L37)：

```php
public function login(SiteConfig $siteConfig)
{
    $postFields = [
        $siteConfig->getUsernameField() => $siteConfig->getUsername(),
        $siteConfig->getPasswordField() => $siteConfig->getPassword(),
    ] + $this->getExtraFields($siteConfig);

    $this->browser->request('POST', $siteConfig->getLoginUri(), $postFields, [], $this->getHttpHeaders($siteConfig));

    return $this;
}
```

**关键点**：
- 合并用户名、密码和额外字段
- 使用 `HttpBrowser` 发送 POST 请求
- Cookie 自动由 `HttpBrowser` 的 CookieJar 管理

### 7.2 isLoggedIn - 检查登录状态

位于 [LoginFormAuthenticator::isLoggedIn](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/SiteConfig/LoginFormAuthenticator.php#L44-L54)：

```php
public function isLoggedIn(SiteConfig $siteConfig)
{
    foreach ($this->browser->getCookieJar()->all() as $cookie) {
        if ($cookie->getDomain() === $siteConfig->getHost()) {
            return true;
        }
    }
    return false;
}
```

通过检查 CookieJar 中是否存在对应域名的 Cookie 来判断。

### 7.3 isLoginRequired - 检测未登录状态

位于 [LoginFormAuthenticator::isLoginRequired](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/SiteConfig/LoginFormAuthenticator.php#L63-L75)：

```php
public function isLoginRequired(SiteConfig $siteConfig, $html)
{
    try {
        $crawler = new Crawler((string) $html);
        $loggedIn = $crawler->evaluate((string) $siteConfig->getNotLoggedInXpath());
    } catch (\Throwable) {
        return false;
    }
    return \count($loggedIn) > 0;
}
```

通过 XPath 查询页面中是否存在未登录的特征元素。

### 7.4 getExtraFields - 处理额外字段

位于 [LoginFormAuthenticator::getExtraFields](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/SiteConfig/LoginFormAuthenticator.php#L101-L119)，支持表达式语言（`@=` 前缀）：

```php
private function getExtraFields(SiteConfig $siteConfig)
{
    $extraFields = [];
    foreach ($siteConfig->getExtraFields() as $fieldName => $fieldValue) {
        if (str_starts_with((string) $fieldValue, '@=')) {
            $fieldValue = $this->expressionLanguage->evaluate(
                substr((string) $fieldValue, 2),
                ['config' => $siteConfig]
            );
        }
        $extraFields[$fieldName] = $fieldValue;
    }
    return $extraFields;
}
```

---

## 8. WallabagClient - HTTP 请求与重试流程

### 8.1 request 主方法

位于 [WallabagClient::request](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/HttpClient/WallabagClient.php#L27-L58)，完整流程：

```php
public function request(string $method, string $url, array $options = []): ResponseInterface
{
    $this->logger->log('debug', 'Restricted access config enabled?', ['enabled' => (int) $this->restrictedAccess]);

    // 步骤 1: 检查 restricted_access 是否开启
    if (0 === (int) $this->restrictedAccess) {
        return $this->httpClient->request($method, $url, $options);
    }

    // 步骤 2: 主动检查是否需要预先登录
    $login = $this->authenticator->loginIfRequired($url);

    if (!$login) {
        return $this->httpClient->request($method, $url, $options);
    }

    // 步骤 3: 携带 Cookie 发送请求
    if (null !== $cookieHeader = $this->getCookieHeader($url)) {
        $options['headers']['cookie'] = $cookieHeader;
    }

    $response = $this->httpClient->request($method, $url, $options);

    // 步骤 4: 检查响应是否需要二次登录（首次登录失效场景）
    $login = $this->authenticator->loginIfRequested($response);

    if (!$login) {
        return $response;
    }

    // 步骤 5: 重新获取 Cookie 并重试请求
    if (null !== $cookieHeader = $this->getCookieHeader($url)) {
        $options['headers']['cookie'] = $cookieHeader;
    }

    return $this->httpClient->request($method, $url, $options);
}
```

### 8.2 getCookieHeader - 构建 Cookie 头

位于 [WallabagClient::getCookieHeader](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/HttpClient/WallabagClient.php#L70-L83)：

```php
private function getCookieHeader(string $url): ?string
{
    $cookies = [];
    foreach ($this->browser->getCookieJar()->allRawValues($url) as $name => $value) {
        $cookies[] = $name . '=' . $value;
    }

    if ([] === $cookies) {
        return null;
    }

    return implode('; ', $cookies);
}
```

---

## 9. 完整流程时序图

```
用户创建/编辑凭据
    ↓
[SiteCredentialController]
    → isSiteCredentialsEnabled() 检查 restricted_access
    → CryptoProxy::crypt() 加密 username/password
    → 持久化到数据库
    ↓
用户发起内容抓取请求
    ↓
[WallabagClient::request]
    ├─ 检查 restricted_access 配置
    │  └─ 未开启 → 直接请求 → 返回响应
    └─ 已开启
        ├─ [Authenticator::loginIfRequired]
        │   ├─ 构建 SiteConfig (GrabySiteConfigBuilder)
        │   │   ├─ TokenStorage 获取当前用户
        │   │   ├─ Host 规范化（去 www、小写）
        │   │   ├─ 生成候选 Host 列表
        │   │   ├─ SiteCredentialRepository::findOneByHostsAndUser
        │   │   │   └─ CryptoProxy::decrypt() 解密凭据
        │   │   └─ 构造 SiteConfig 对象
        │   ├─ 检查 requiresLogin
        │   ├─ LoginFormAuthenticator::isLoggedIn() 检查 Cookie
        │   └─ 未登录 → LoginFormAuthenticator::login() POST 登录
        ├─ getCookieHeader() 从 CookieJar 获取 Cookie
        ├─ 携带 Cookie 发送请求
        ├─ [Authenticator::loginIfRequested]
        │   ├─ 检查响应内容
        │   ├─ LoginFormAuthenticator::isLoginRequired() XPath 检测
        │   └─ 需要登录 → 重新执行 login()
        └─ 二次请求（如需要）→ 返回响应
```

---

## 10. 关键技术要点总结

| 环节 | 技术实现 | 关键文件 |
|------|----------|----------|
| **配置开关** | CraueConfigBundle 动态配置 | [wallabag.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/wallabag.yml#L90-L92) |
| **加密算法** | Defuse\Crypto 库（Authenticated Encryption） | [CryptoProxy.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/CryptoProxy.php) |
| **密钥管理** | 文件存储 + 0600 权限 + 自动创建 | [CryptoProxy.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/CryptoProxy.php#L18-L30) |
| **Host 匹配** | 精确匹配 + 通配子域名匹配 | [GrabySiteConfigBuilder.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/SiteConfig/GrabySiteConfigBuilder.php#L39-L47) |
| **登录检测** | XPath 页面元素检测 | [LoginFormAuthenticator.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/SiteConfig/LoginFormAuthenticator.php#L63-L75) |
| **会话保持** | HttpBrowser CookieJar 管理 | [WallabagClient.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/HttpClient/WallabagClient.php#L70-L83) |
| **重试机制** | 预检登录 + 响应检测二次登录 | [WallabagClient.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/HttpClient/WallabagClient.php#L27-L58) |
| **安全日志** | 敏感信息脱敏（首尾字符 + 星号） | [CryptoProxy.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Helper/CryptoProxy.php#L81-L84) |

---

## 11. 安全设计考量

1. **加密存储**：用户名密码不明文存储，使用 Defuse 库提供认证加密
2. **密钥安全**：密钥文件权限 0600，独立于数据库存储
3. **日志脱敏**：日志中不输出完整敏感信息
4. **用户隔离**：凭据按用户隔离，查询时必须匹配 `userId`
5. **按需登录**：仅在需要时执行登录，避免不必要的认证请求
6. **动态检测**：通过 XPath 动态检测登录状态，应对会话过期场景
