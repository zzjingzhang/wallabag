# Wallabag 2FA 状态转换分析文档

## 目录
1. [概述](#概述)
2. [ConfigController 2FA 操作分析](#configcontroller-2fa-操作分析)
3. [User 实体 Scheb 接口实现](#user-实体-scheb-接口实现)
4. [AuthCodeMailer 邮件验证码发送](#authcodemailer-邮件验证码发送)
5. [数据库迁移分析](#数据库迁移分析)
6. [状态转换流程图](#状态转换流程图)

---

## 概述

Wallabag 支持两种双因素认证方式：
- **邮箱 2FA**：通过邮件发送验证码
- **OTP App**：使用 Google Authenticator 等 TOTP 应用

本文档详细分析用户在配置页的 2FA 操作、管理员编辑用户时的状态转换，以及相关底层实现。

---

## ConfigController 2FA 操作分析

### 1. 启用邮箱 2FA - `otpEmailAction`

**文件路径**：[ConfigController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/ConfigController.php#L279-L300)

```php
public function otpEmailAction(Request $request)
{
    // CSRF 验证
    if (!$this->isCsrfTokenValid('otp', $request->request->get('token'))) {
        throw new BadRequestHttpException('Bad CSRF token.');
    }

    $user = $this->getUser();

    // 关键状态转换
    $user->setGoogleAuthenticatorSecret(null);  // 清理 Google Secret
    $user->setBackupCodes(null);                  // 清理备份码
    $user->setEmailTwoFactor(true);               // 启用邮箱 2FA

    $this->userManager->updateUser($user);
    $this->entityManager->flush();

    // ... flash 消息与重定向
}
```

**状态转换逻辑**：

| 操作 | 字段 | 原值 | 新值 | 说明 |
|------|------|------|------|------|
| 清理 | `googleAuthenticatorSecret` | 任意 | `null` | 确保 OTP App 密钥被清除 |
| 清理 | `backupCodes` | 任意 | `null` | 确保备份码被清除 |
| 启用 | `emailTwoFactor` | `false` | `true` | 激活邮箱双因素认证 |

**设计意图**：启用邮箱 2FA 时，必须完全禁用 OTP App 相关数据，避免两种认证方式同时生效造成混乱。

---

### 2. 启用 OTP App - `otpAppAction`

**文件路径**：[ConfigController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/ConfigController.php#L335-L363)

```php
public function otpAppAction(Request $request, GoogleAuthenticatorInterface $googleAuthenticator)
{
    // CSRF 验证
    if (!$this->isCsrfTokenValid('otp', $request->request->get('token'))) {
        throw new BadRequestHttpException('Bad CSRF token.');
    }

    $user = $this->getUser();
    $secret = $googleAuthenticator->generateSecret();  // 生成新密钥

    $user->setGoogleAuthenticatorSecret($secret);      // 设置密钥
    $user->setEmailTwoFactor(false);                   // 禁用邮箱 2FA

    // 生成并哈希备份码
    $backupCodes = (new BackupCodes())->toArray();
    $backupCodesHashed = array_map(
        static fn ($backupCode) => password_hash((string) $backupCode, \PASSWORD_DEFAULT),
        $backupCodes
    );

    $user->setBackupCodes($backupCodesHashed);

    $this->userManager->updateUser($user);
    $this->entityManager->flush();

    // 返回二维码、密钥和明文备份码给用户
    return $this->render('Config/otp_app.html.twig', [
        'backupCodes' => $backupCodes,      // 明文，仅显示一次
        'qr_code' => $googleAuthenticator->getQRContent($user),
        'secret' => $secret,
    ]);
}
```

**关键注意点**：

> ⚠️ **重要**：此方法**不会**调用 `$user->setGoogleAuthenticator(true)`。

此时 `googleAuthenticator` 字段仍为 `false`，表示 OTP App 处于**待确认**状态。用户需要在下一页面输入验证码确认。

**状态转换**：

| 字段 | 操作 | 值 |
|------|------|----|
| `googleAuthenticatorSecret` | 设置 | 新生成的 TOTP 密钥 |
| `emailTwoFactor` | 设置 | `false` |
| `backupCodes` | 设置 | 哈希后的备份码数组 |
| `googleAuthenticator` | - | 保持 `false`（未激活） |

---

### 3. 确认 OTP 代码 - `otpAppCheckAction`

**文件路径**：[ConfigController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/ConfigController.php#L387-L426)

```php
public function otpAppCheckAction(Request $request, GoogleAuthenticatorInterface $googleAuthenticator)
{
    // CSRF 验证
    if (!$this->isCsrfTokenValid('otp', $request->request->get('token'))) {
        throw new BadRequestHttpException('Bad CSRF token.');
    }

    $user = $this->getUser();

    // 验证输入的验证码
    $isValid = $googleAuthenticator->checkCode(
        $user,
        $request->request->get('_auth_code')
    );

    if ($isValid) {
        // === 验证成功 ===
        $this->addFlash('notice', 'flashes.config.notice.otp_enabled');
        $user->setGoogleAuthenticator(true);           // ✅ 激活 OTP App
        $this->userManager->updateUser($user);
        $this->entityManager->flush();

        return $this->redirect($this->generateUrl('config') . '#set3');
    }

    // === 验证失败 ===
    $this->addFlash('notice', 'flashes.config.notice.otp_code_invalid');

    // 清理已设置的密钥和备份码，回滚状态
    $user->setGoogleAuthenticatorSecret(null);
    $user->setBackupCodes(null);

    $this->userManager->updateUser($user);
    $this->entityManager->flush();

    // 307 重定向保持 POST 方法，返回到启用页面
    return $this->redirect($this->generateUrl('config_otp_app'), 307);
}
```

**成功路径**（验证码正确）：
1. 设置 `googleAuthenticator = true`
2. OTP App 正式启用
3. 重定向到配置页

**失败路径**（验证码错误）：
1. 清理 `googleAuthenticatorSecret` → `null`
2. 清理 `backupCodes` → `null`
3. **不设置** `googleAuthenticator`（保持 false）
4. 状态回滚到启用前
5. 重定向回启用页面重试

---

### 4. 禁用两类 2FA

#### 4.1 禁用邮箱 2FA - `disableOtpEmailAction`

**文件路径**：[ConfigController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/ConfigController.php#L254-L272)

```php
public function disableOtpEmailAction(Request $request)
{
    // CSRF 验证
    $user = $this->getUser();
    $user->setEmailTwoFactor(false);  // 仅设置此标志位

    $this->userManager->updateUser($user);
    $this->entityManager->flush();

    // ... flash 消息与重定向
}
```

**状态转换**：`emailTwoFactor` → `false`

---

#### 4.2 禁用 OTP App - `disableOtpAppAction`

**文件路径**：[ConfigController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/ConfigController.php#L307-L328)

```php
public function disableOtpAppAction(Request $request)
{
    // CSRF 验证
    $user = $this->getUser();

    $user->setGoogleAuthenticatorSecret('');   // 清空密钥
    $user->setGoogleAuthenticator(false);      // 禁用标志位
    $user->setBackupCodes(null);               // 清空备份码

    $this->userManager->updateUser($user);
    $this->entityManager->flush();

    // ... flash 消息与重定向
}
```

**状态转换**：

| 字段 | 新值 |
|------|------|
| `googleAuthenticatorSecret` | `''`（空字符串） |
| `googleAuthenticator` | `false` |
| `backupCodes` | `null` |

> 💡 **注意**：密钥被设置为空字符串 `''` 而非 `null`，这与 `otpEmailAction` 中的清理方式略有不同。

---

### 5. 管理员编辑用户 - `UserController::editAction`

**文件路径**：[UserController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/UserController.php#L79-L114)

```php
public function editAction(Request $request, User $user, UserManagerInterface $userManager, GoogleAuthenticatorInterface $googleAuthenticator)
{
    $deleteForm = $this->createDeleteForm($user);
    $form = $this->createForm(UserType::class, $user);
    $form->handleRequest($request);

    // 初始化表单数据：如果用户已启用 OTP App，勾选复选框
    if (true === $user->isGoogleAuthenticatorEnabled() && false === $form->isSubmitted()) {
        $form->get('googleTwoFactor')->setData(true);
    }

    if ($form->isSubmitted() && $form->isValid()) {
        // 根据复选框状态变化处理 OTP
        if (true === $form->get('googleTwoFactor')->getData() && false === $user->isGoogleAuthenticatorEnabled()) {
            // 勾选：启用 OTP App（生成新密钥）
            $user->setGoogleAuthenticatorSecret($googleAuthenticator->generateSecret());
            $user->setEmailTwoFactor(false);
            // ⚠️ 注意：这里不设置 googleAuthenticator = true
            // 管理员需要另外的方式确认或告知用户自行激活
        } elseif (false === $form->get('googleTwoFactor')->getData() && true === $user->isGoogleAuthenticatorEnabled()) {
            // 取消勾选：禁用 OTP App
            $user->setGoogleAuthenticatorSecret(null);
            // ⚠️ 注意：这里没有设置 googleAuthenticator = false
            // 也没有清理 backupCodes
        }

        $userManager->updateUser($user);

        // ... flash 消息与重定向
    }

    // ... 渲染模板
}
```

**管理员编辑的状态转换**：

| 场景 | 操作 |
|------|------|
| **从未启用 → 启用** | 1. 生成新的 `googleAuthenticatorSecret`<br>2. 设置 `emailTwoFactor = false`<br>3. **不设置** `googleAuthenticator = true` |
| **已启用 → 禁用** | 1. 设置 `googleAuthenticatorSecret = null`<br>2. **不设置** `googleAuthenticator = false`<br>3. **不清理** `backupCodes` |

> ⚠️ **潜在问题**：管理员禁用 OTP App 时只清空了密钥，但没有重置 `googleAuthenticator` 标志位和 `backupCodes`。这可能导致状态不一致。

---

## User 实体 Scheb 接口实现

**文件路径**：[User.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/User.php#L31-L32)

### 接口实现概览

```php
class User extends BaseUser implements 
    EmailTwoFactorInterface,    // Scheb 邮箱 2FA 接口
    GoogleTwoFactorInterface,   // Scheb Google 2FA 接口
    BackupCodeInterface         // Scheb 备份码接口
{
```

### 核心字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `authCode` | `integer` | 临时存储邮箱验证码 |
| `googleAuthenticatorSecret` | `string` | OTP App 密钥 |
| `googleAuthenticator` | `boolean` | OTP App 是否启用标志 |
| `backupCodes` | `json` | 哈希后的备份码数组 |
| `emailTwoFactor` | `boolean` | 邮箱 2FA 是否启用标志 |

---

### EmailTwoFactorInterface 实现

**文件路径**：[User.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/User.php#L285-L303)

```php
// 检查是否启用邮箱 2FA
public function isEmailAuthEnabled(): bool
{
    return $this->emailTwoFactor;
}

// 获取邮箱验证码
public function getEmailAuthCode(): string
{
    return $this->authCode;
}

// 设置邮箱验证码
public function setEmailAuthCode(string $authCode): void
{
    $this->authCode = $authCode;
}

// 获取收件人邮箱
public function getEmailAuthRecipient(): string
{
    return $this->email;
}
```

---

### GoogleTwoFactorInterface 实现

**文件路径**：[User.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/User.php#L305-L323)

```php
// 检查是否启用 Google 2FA
public function isGoogleAuthenticatorEnabled(): bool
{
    return $this->googleAuthenticator;
}

// 获取用户名（用于二维码）
public function getGoogleAuthenticatorUsername(): string
{
    return $this->username;
}

// 获取密钥
public function getGoogleAuthenticatorSecret(): string
{
    return $this->googleAuthenticatorSecret;
}

// 设置密钥
public function setGoogleAuthenticatorSecret(?string $googleAuthenticatorSecret): void
{
    $this->googleAuthenticatorSecret = $googleAuthenticatorSecret;
}
```

---

### BackupCodeInterface 实现

**文件路径**：[User.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/User.php#L325-L347)

```php
// 设置备份码
public function setBackupCodes(?array $codes = null): void
{
    $this->backupCodes = $codes;
}

// 获取备份码
public function getBackupCodes()
{
    return $this->backupCodes;
}

// 检查是否为有效备份码
public function isBackupCode(string $code): bool
{
    return false === $this->findBackupCode($code) ? false : true;
}

// 使备份码失效（使用后删除）
public function invalidateBackupCode(string $code): void
{
    $key = $this->findBackupCode($code);
    if (false !== $key) {
        unset($this->backupCodes[$key]);
    }
}

// 内部方法：验证并查找备份码
private function findBackupCode(string $code)
{
    foreach ($this->backupCodes as $key => $backupCode) {
        // 使用 password_verify 验证哈希值
        if (password_verify($code, (string) $backupCode)) {
            return $key;
        }
    }
    return false;
}
```

**安全设计**：
- 备份码使用 `password_hash()` 哈希存储
- 验证时使用 `password_verify()` 进行比对
- 使用后立即从数组中移除（单次有效）

---

## AuthCodeMailer 邮件验证码发送

**文件路径**：[AuthCodeMailer.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Mailer/AuthCodeMailer.php#L17-L63)

### 类结构

```php
class AuthCodeMailer implements AuthCodeMailerInterface
{
    public function __construct(
        private readonly MailerInterface $mailer,   // Symfony Mailer
        private readonly Environment $twig,         // Twig 模板引擎
        private $senderEmail,                       // 发件人邮箱
        private $senderName,                        // 发件人名称
        private $supportUrl,                        // 支持链接
    ) {
    }
```

### sendAuthCode 方法实现

```php
public function sendAuthCode(TwoFactorInterface $user): void
{
    // 类型断言确保是 Wallabag User 实体
    \assert($user instanceof User);

    // 加载 Twig 模板
    $template = $this->twig->load('TwoFactor/email_auth_code.html.twig');

    // 分别渲染三个 block
    $subject = $template->renderBlock('subject', []);
    
    $bodyHtml = $template->renderBlock('body_html', [
        'user' => $user->getName(),
        'code' => $user->getEmailAuthCode(),
        'support_url' => $this->supportUrl,
    ]);
    
    $bodyText = $template->renderBlock('body_text', [
        'user' => $user->getName(),
        'code' => $user->getEmailAuthCode(),
        'support_url' => $this->supportUrl,
    ]);

    // 构建并发送邮件
    $email = (new Email())
        ->from(new Address($this->senderEmail, $this->senderName ?: $this->senderEmail))
        ->to($user->getEmailAuthRecipient())  // 获取用户邮箱
        ->subject($subject)
        ->text($bodyText)
        ->html($bodyHtml);

    $this->mailer->send($email);
}
```

### 渲染流程

```
Twig 模板 (email_auth_code.html.twig)
    ├── subject block       → 邮件主题
    ├── body_html block     → HTML 格式正文
    └── body_text block     → 纯文本格式正文
```

**数据传递**：
- `user`：用户显示名称
- `code`：生成的验证码（从 `$user->getEmailAuthCode()` 获取）
- `support_url`：支持页面链接

---

## 数据库迁移分析

### Version20250413133131 迁移

**文件路径**：[Version20250413133131.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/migrations/Version20250413133131.php#L11-L45)

### 迁移背景

在旧版本中，仅通过 `googleAuthenticatorSecret` 是否为 `null` 来判断 OTP App 是否启用。这种设计存在问题：当用户进入 OTP App 启用流程但未完成验证时，密钥已存在但实际上并未激活。

新增 `google_authenticator` 布尔列用于明确标记启用状态。

---

### up 方法 - 添加列

```php
public function up(Schema $schema): void
{
    $userTable = $schema->getTable($this->getTable('user'));

    // 幂等性检查：列已存在则跳过
    $this->skipIf($userTable->hasColumn('google_authenticator'), 'It seems that you already played this migration.');

    // 添加布尔列，默认 false
    $userTable->addColumn('google_authenticator', 'boolean', [
        'default' => false,
        'notnull' => true,
    ]);
}
```

---

### postUp 方法 - 数据迁移

```php
public function postUp(Schema $schema): void
{
    $this->skipIf(!$schema->getTable($this->getTable('user'))->hasColumn('google_authenticator'), 'Unable to update google_authenticator column');
    
    // 历史数据映射：有密钥的用户视为已启用
    $this->connection->executeQuery(
        'UPDATE ' . $this->getTable('user') . ' SET google_authenticator = :googleAuthenticator WHERE googleAuthenticatorSecret IS NOT NULL AND googleAuthenticatorSecret <> :emptyString',
        [
            'googleAuthenticator' => true,
            'emptyString' => '',
        ]
    );
}
```

**映射逻辑**：

```
旧版本判断条件：googleAuthenticatorSecret IS NOT NULL AND <> ''
                    ↓
新版本设置：google_authenticator = true
```

**兼容说明**：
- 迁移前有密钥（非 `null` 且非空字符串）的所有用户，自动标记为启用
- 这是一个**最佳努力**的兼容方案，因为旧设计无法区分"待确认"和"已启用"状态
- 迁移后新的两步验证流程（生成密钥 → 代码确认 → 标记启用）将正常工作

---

### down 方法 - 回滚

```php
public function down(Schema $schema): void
{
    $userTable = $schema->getTable($this->getTable('user'));
    $userTable->dropColumn('google_authenticator');
}
```

---

## 状态转换流程图

### 用户配置页 2FA 状态机

```
初始状态
  │
  ├─→ 启用邮箱 2FA
  │     ├─ 清除 googleAuthenticatorSecret
  │     ├─ 清除 backupCodes
  │     └─ 设置 emailTwoFactor = true
  │
  ├─→ 启用 OTP App
  │     ├─ 生成 secret
  │     ├─ 设置 googleAuthenticatorSecret = secret
  │     ├─ 生成并哈希 backupCodes
  │     ├─ 设置 emailTwoFactor = false
  │     ├─ （googleAuthenticator 保持 false）
  │     │
  │     └─→ 用户输入验证码
  │           ├─ 验证成功
  │           │   └─ 设置 googleAuthenticator = true
  │           └─ 验证失败
  │               ├─ 清除 googleAuthenticatorSecret
  │               └─ 清除 backupCodes
  │
  ├─→ 禁用邮箱 2FA
  │     └─ 设置 emailTwoFactor = false
  │
  └─→ 禁用 OTP App
        ├─ 设置 googleAuthenticatorSecret = ''
        ├─ 设置 googleAuthenticator = false
        └─ 设置 backupCodes = null
```

---

### 管理员编辑用户状态转换

```
管理员编辑表单
  │
  ├─ 勾选 Google 2FA（从未启用 → 启用）
  │     ├─ 生成新的 googleAuthenticatorSecret
  │     ├─ 设置 emailTwoFactor = false
  │     └─ ⚠️ googleAuthenticator 保持 false（需要用户确认）
  │
  └─ 取消勾选 Google 2FA（已启用 → 禁用）
        ├─ 设置 googleAuthenticatorSecret = null
        └─ ⚠️ googleAuthenticator 和 backupCodes 未清理
```

---

## 关键发现与注意事项

1. **两阶段启用**：OTP App 启用采用"生成密钥 → 验证代码 → 激活"的三步骤流程，确保用户正确配置了应用。

2. **互斥设计**：邮箱 2FA 和 OTP App 是互斥的，启用其中一个会自动禁用另一个。

3. **哈希存储**：备份码使用 `password_hash()` 哈希存储，符合安全最佳实践。

4. **迁移兼容**：数据库迁移通过"有密钥即启用"的假设实现向后兼容。

5. **管理员编辑局限**：管理员编辑用户时的 2FA 处理存在不一致性，可能需要进一步完善。
