# Wallabag 多数据库支持与表前缀机制分析

## 1. InstallCommand::checkRequirements - PDO驱动、连接与版本检查

### 1.1 PDO驱动检查

位置: [InstallCommand.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Command/InstallCommand.php#L110-L207)

```php
private function checkRequirements()
{
    $databaseDriver = $this->getDatabaseDriver();
    
    // 测试数据库驱动是否存在
    if (!\extension_loaded($databaseDriver)) {
        $fulfilled = false;
        $status = '<error>ERROR!</error>';
        $help = 'Database driver "' . $databaseDriver . '" is not installed.';
    }
    // ...
}
```

**驱动映射逻辑** ([getDatabaseDriver](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Command/InstallCommand.php#L209-L219)):
- `pdo_mysql` / `mysqli` → `pdo_mysql`
- `pdo_pgsql` / `pgsql` → `pdo_pgsql`
- `pdo_sqlite` / `sqlite3` → `pdo_sqlite`

### 1.2 数据库连接检查

```php
$conn = $this->entityManager->getConnection();

try {
    $conn->connect();
} catch (\Exception $e) {
    // 忽略"数据库不存在"的错误（允许后续创建数据库）
    if (!str_contains($e->getMessage(), 'Unknown database')
        && !str_contains($e->getMessage(), 'database "' . $conn->getDatabase() . '" does not exist')) {
        $fulfilled = false;
        $status = '<error>ERROR!</error>';
        $help = 'Can\'t connect to the database: ' . $e->getMessage();
    }
}
```

**关键设计**: 
- 允许"数据库不存在"的情况，因为安装程序会自动创建
- 只有真正的连接错误（如密码错误、服务器不可达）才会终止安装

### 1.3 MySQL/PostgreSQL版本检查

**MySQL版本检查** (≥ 5.5.4):
```php
if ($conn->isConnected() && $conn->getDatabasePlatform() instanceof MySQLPlatform) {
    $version = $conn->executeQuery('select version()')->fetchOne();
    $minimalVersion = '5.5.4';

    if (false === version_compare($version, $minimalVersion, '>')) {
        $fulfilled = false;
        // 报错: MySQL版本过旧，需要支持utf8mb4
    }
}
```

**PostgreSQL版本检查** (≥ 9.2.0):
```php
if ($conn->isConnected() && $conn->getDatabasePlatform() instanceof PostgreSQLPlatform) {
    $version = $conn->executeQuery('SELECT version();')->fetchOne();
    preg_match('/PostgreSQL ([0-9\.]+)/i', (string) $version, $matches);

    if (isset($matches[1]) & version_compare($matches[1], '9.2.0', '<')) {
        $fulfilled = false;
        // 报错: PostgreSQL需要大于9.1
    }
}
```

---

## 2. setupDatabase - 四种场景下的Doctrine命令执行

位置: [InstallCommand.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Command/InstallCommand.php#L221-L298)

### 2.1 场景一: Reset模式 (--reset选项)

```php
if (true === $this->defaultInput->getOption('reset')) {
    $this->runCommand('doctrine:schema:drop', ['--force' => true, '--full-database' => true]);

    if (!$databasePlatform instanceof PostgreSQLPlatform) {
        $this->runCommand('doctrine:database:drop', ['--force' => true]);
        $this->runCommand('doctrine:database:create');
    }

    $this
        ->runCommand('doctrine:migrations:migrate', ['--no-interaction' => true])
        ->runCommand('cache:clear');
}
```

**执行命令**:
1. `doctrine:schema:drop --force --full-database` - 删除所有表
2. (非PostgreSQL) `doctrine:database:drop --force` - 删除数据库
3. (非PostgreSQL) `doctrine:database:create` - 创建数据库
4. `doctrine:migrations:migrate` - 执行迁移
5. `cache:clear` - 清除缓存

**注意**: PostgreSQL不执行drop/create database，因为通常需要特殊权限

### 2.2 场景二: 新数据库 (数据库不存在)

```php
if (!$this->isDatabasePresent()) {
    $this
        ->runCommand('doctrine:database:create')
        ->runCommand('doctrine:migrations:migrate', ['--no-interaction' => true])
        ->runCommand('cache:clear');
}
```

**执行命令**:
1. `doctrine:database:create` - 创建数据库
2. `doctrine:migrations:migrate` - 执行迁移
3. `cache:clear` - 清除缓存

### 2.3 场景三: 已有数据库（用户确认重置）

```php
if ($this->io->confirm('It appears that your database already exists. Would you like to reset it?', false)) {
    $this->runCommand('doctrine:schema:drop', ['--force' => true, '--full-database' => true]);

    if (!$databasePlatform instanceof PostgreSQLPlatform) {
        $this->runCommand('doctrine:database:drop', ['--force' => true]);
        $this->runCommand('doctrine:database:create');
    }

    $this->runCommand('doctrine:migrations:migrate', ['--no-interaction' => true]);
}
```

**执行命令** (与Reset模式类似，但需用户确认):
1. `doctrine:schema:drop --force --full-database` - 删除所有表
2. (非PostgreSQL) drop + create database
3. `doctrine:migrations:migrate` - 执行迁移

### 2.4 场景四: 已有Schema（四种子情况）

```php
} elseif ($this->isSchemaPresent()) {
    if ($this->io->confirm('Seems like your database contains schema. Do you want to reset it?', false)) {
        // 子情况A: 用户确认重置schema
        $this->dropWallabagSchemaOnly();  // 删除schema + 删除migrations表
        $this->runCommand('doctrine:migrations:migrate', ['--no-interaction' => true]);
    } else {
        // 子情况B: 用户保留现有schema，跳过管理员创建
        $this->schemaIsNew = false;
    }
} else {
    // 子情况C: 数据库存在但无schema，创建schema
    $this->runCommand('doctrine:migrations:migrate', ['--no-interaction' => true]);
}
```

**数据库存在性检测** ([isDatabasePresent](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Command/InstallCommand.php#L415-L458)):
- SQLite: 检查数据库文件是否存在
- MySQL/PostgreSQL: 检查数据库名是否在`listDatabases()`结果中

**Schema存在性检测** ([isSchemaPresent](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Command/InstallCommand.php#L464-L469)):
- 通过检查Doctrine Migrations表是否存在来判断

---

## 3. setupConfig - 写入InternalSetting和IgnoreOriginInstanceRule默认数据

位置: [InstallCommand.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Command/InstallCommand.php#L340-L369)

```php
private function setupConfig()
{
    // 先清空现有数据
    $this->entityManager->createQuery('DELETE FROM Wallabag\Entity\InternalSetting')->execute();
    $this->entityManager->createQuery('DELETE FROM Wallabag\Entity\IgnoreOriginInstanceRule')->execute();

    // 写入InternalSetting默认配置
    foreach ($this->defaultSettings as $setting) {
        $newSetting = new InternalSetting();
        $newSetting->setName($setting['name']);
        $newSetting->setValue($setting['value']);
        $newSetting->setSection($setting['section']);
        $this->entityManager->persist($newSetting);
    }

    // 写入IgnoreOriginInstanceRule默认规则
    foreach ($this->defaultIgnoreOriginInstanceRules as $ignore_origin_instance_rule) {
        $newIgnoreOriginInstanceRule = new IgnoreOriginInstanceRule();
        $newIgnoreOriginInstanceRule->setRule($ignore_origin_instance_rule['rule']);
        $this->entityManager->persist($newIgnoreOriginInstanceRule);
    }

    $this->entityManager->flush();
}
```

**关键设计**:
1. **先清空后写入**: 确保安装时配置是全新的
2. **依赖注入默认值**: 默认数据通过构造函数注入
   - `$defaultSettings` → `%wallabag.default_internal_settings%`
   - `$defaultIgnoreOriginInstanceRules` → `%wallabag.default_ignore_origin_instance_rules%`

**服务配置** ([services.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/services.yml#L275-L278)):
```yaml
Wallabag\Command\InstallCommand:
    arguments:
        $defaultSettings: '%wallabag.default_internal_settings%'
        $defaultIgnoreOriginInstanceRules: '%wallabag.default_ignore_origin_instance_rules%'
```

---

## 4. TablePrefixSubscriber - loadClassMetadata时修改表名

位置: [TablePrefixSubscriber.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Event/Subscriber/TablePrefixSubscriber.php)

### 4.1 核心实现

```php
class TablePrefixSubscriber implements EventSubscriber
{
    protected $tablePrefix = '';

    public function __construct($tablePrefix)
    {
        $this->tablePrefix = (string) $tablePrefix;
    }

    public function getSubscribedEvents(): array
    {
        return ['loadClassMetadata'];
    }

    public function loadClassMetadata(LoadClassMetadataEventArgs $args): void
    {
        $classMetadata = $args->getClassMetadata();

        // 继承层次结构中只对根实体应用一次
        if ($classMetadata->isInheritanceTypeSingleTable() && !$classMetadata->isRootEntity()) {
            return;
        }

        // 1. 修改主表名
        $classMetadata->setPrimaryTable(['name' => $this->tablePrefix . $classMetadata->getTableName()]);

        // 2. 修改多对多关联的joinTable名
        foreach ($classMetadata->getAssociationMappings() as $fieldName => $mapping) {
            if (ClassMetadataInfo::MANY_TO_MANY === $mapping['type'] 
                && isset($classMetadata->associationMappings[$fieldName]['joinTable']['name'])) {
                $mappedTableName = $classMetadata->associationMappings[$fieldName]['joinTable']['name'];
                $classMetadata->associationMappings[$fieldName]['joinTable']['name'] = $this->tablePrefix . $mappedTableName;
            }
        }
    }
}
```

### 4.2 技术要点

**事件监听**:
- 订阅Doctrine的`loadClassMetadata`事件
- 在实体元数据加载时动态修改表名

**表前缀应用范围**:
1. **主表**: `$classMetadata->setPrimaryTable()` - 修改实体对应的主表名
2. **多对多关联表**: 修改`joinTable.name` - 确保中间表也应用前缀

**继承处理**:
- 单表继承(`SINGLE_TABLE`)时，只对根实体应用一次
- 避免子类重复修改表名

**服务配置** ([services.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/services.yml#L200-L202)):
```yaml
Wallabag\Event\Subscriber\TablePrefixSubscriber:
    tags:
        - { name: doctrine.event_subscriber }
```

---

## 5. SchemaAdapterSubscriber - MySQL特殊调整

位置: [SchemaAdapterSubscriber.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Event/Subscriber/SchemaAdapterSubscriber.php)

### 5.1 核心实现

```php
class SchemaAdapterSubscriber implements EventSubscriber
{
    public function __construct(
        private readonly string $databaseTablePrefix,
    ) {
    }

    public function getSubscribedEvents(): array
    {
        return ['postGenerateSchema'];
    }

    public function postGenerateSchema(GenerateSchemaEventArgs $eventArgs): void
    {
        $platform = $eventArgs->getEntityManager()->getConnection()->getDatabasePlatform();

        // 只在MySQL上执行调整
        if (!$platform instanceof MySQLPlatform) {
            return;
        }

        $schema = $eventArgs->getSchema();

        // 1. 调整entry表字符集
        $entryTable = $schema->getTable($this->databaseTablePrefix . 'entry');
        $entryTable->addOption('collate', 'utf8mb4_unicode_ci');
        $entryTable->addOption('charset', 'utf8mb4');

        // 2. 调整tag表字符集和索引长度
        $tagTable = $schema->getTable($this->databaseTablePrefix . 'tag');
        $tagTable->addOption('collate', 'utf8mb4_bin');
        $tagTable->addOption('charset', 'utf8mb4');

        // 调整tag.label索引长度（utf8mb4下索引长度限制）
        foreach ($tagTable->getIndexes() as $index) {
            if ($index->getColumns() === ['label']) {
                $tagTable->dropIndex($index->getName());
                $tagTable->addIndex($index->getColumns(), $index->getName(), $index->getFlags(), array_merge(
                    $index->getOptions(),
                    ['lengths' => [255]]
                ));
            }
        }

        // 3. 调整OAuth相关表字段长度
        $oauth2AccessTokenTable = $schema->getTable($this->databaseTablePrefix . 'oauth2_access_tokens');
        $oauth2AccessTokenTable->modifyColumn('token', ['length' => 191]);
        $oauth2AccessTokenTable->modifyColumn('scope', ['length' => 191]);

        $oauth2AuthCodeTable = $schema->getTable($this->databaseTablePrefix . 'oauth2_auth_codes');
        $oauth2AuthCodeTable->modifyColumn('token', ['length' => 191]);
        $oauth2AuthCodeTable->modifyColumn('scope', ['length' => 191]);

        $oauth2RefreshTokenTable = $schema->getTable($this->databaseTablePrefix . 'oauth2_refresh_tokens');
        $oauth2RefreshTokenTable->modifyColumn('token', ['length' => 191]);
        $oauth2RefreshTokenTable->modifyColumn('scope', ['length' => 191]);

        // 4. 调整internal_setting表字段长度
        $internalSettingTable = $schema->getTable($this->databaseTablePrefix . 'internal_setting');
        $internalSettingTable->modifyColumn('name', ['length' => 191]);
        $internalSettingTable->modifyColumn('section', ['length' => 191]);
        $internalSettingTable->modifyColumn('value', ['length' => 191]);
    }
}
```

### 5.2 技术背景

**为什么只在MySQL上调整?**

1. **utf8mb4字符集支持**:
   - MySQL 5.5.3+ 引入utf8mb4以支持完整的Unicode（包括emoji）
   - 旧的utf8只支持3字节字符，无法存储emoji等4字节字符

2. **InnoDB索引长度限制**:
   - utf8mb4下，每个字符占4字节
   - InnoDB默认索引前缀限制为767字节
   - `varchar(255)` 在utf8mb4下 = 255 * 4 = 1020字节 > 767
   - 解决方案: 限制索引长度为191 (191 * 4 = 764 < 767)

**调整的字段列表**:
| 表名 | 字段 | 调整后长度 |
|------|------|-----------|
| oauth2_access_tokens | token, scope | 191 |
| oauth2_auth_codes | token, scope | 191 |
| oauth2_refresh_tokens | token, scope | 191 |
| internal_setting | name, section, value | 191 |
| tag | label (索引长度) | 255 |

---

## 6. MigrationFactoryDecorator - 迁移依赖注入

位置: [MigrationFactoryDecorator.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Doctrine/MigrationFactoryDecorator.php)

### 6.1 装饰器模式实现

```php
class MigrationFactoryDecorator implements MigrationFactory
{
    public function __construct(
        private readonly MigrationFactory $migrationFactory,
        private readonly string $tablePrefix,
        private readonly array $defaultIgnoreOriginInstanceRules,
        private readonly string $fetchingErrorMessage,
    ) {
    }

    public function createVersion(string $migrationClassName): AbstractMigration
    {
        $instance = $this->migrationFactory->createVersion($migrationClassName);

        // 只对WallabagMigration子类注入依赖
        if ($instance instanceof WallabagMigration) {
            $instance->setTablePrefix($this->tablePrefix);
            $instance->setDefaultIgnoreOriginInstanceRules($this->defaultIgnoreOriginInstanceRules);
            $instance->setFetchingErrorMessage($this->fetchingErrorMessage);
        }

        return $instance;
    }
}
```

### 6.2 WallabagMigration基类

位置: [WallabagMigration.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Doctrine/WallabagMigration.php)

```php
abstract class WallabagMigration extends AbstractMigration
{
    protected string $tablePrefix;
    protected array $defaultIgnoreOriginInstanceRules;
    protected string $fetchingErrorMessage;

    // setter注入
    public function setTablePrefix(string $tablePrefix): void
    {
        $this->tablePrefix = $tablePrefix;
    }

    public function setDefaultIgnoreOriginInstanceRules(array $defaultIgnoreOriginInstanceRules): void
    {
        $this->defaultIgnoreOriginInstanceRules = $defaultIgnoreOriginInstanceRules;
    }

    public function setFetchingErrorMessage(string $fetchingErrorMessage): void
    {
        $this->fetchingErrorMessage = $fetchingErrorMessage;
    }

    // 辅助方法：获取带前缀的表名
    protected function getTable($tableName, $unEscaped = false)
    {
        $table = $this->tablePrefix . $tableName;

        if (self::UN_ESCAPED_TABLE === $unEscaped) {
            return $table;
        }

        // PostgreSQL使用双引号转义
        if ($this->connection->getDatabasePlatform() instanceof PostgreSQLPlatform) {
            return '"' . $table . '"';
        }

        // MySQL使用反引号转义
        return '`' . $table . '`';
    }
}
```

### 6.3 服务配置

位置: [services.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/services.yml#L127-L128)

```yaml
Wallabag\Doctrine\MigrationFactoryDecorator:
    decorates: doctrine.migrations.migrations_factory
```

**装饰器模式优势**:
1. 不修改Doctrine MigrationsBundle的核心代码
2. 通过setter注入将配置传递给迁移类
3. 迁移类可以在SQL中正确使用表前缀

**注入的三个依赖**:
1. `$tablePrefix` - 数据库表前缀 (`%env(WALLABAG_TABLE_PREFIX)%`)
2. `$defaultIgnoreOriginInstanceRules` - 默认忽略规则
3. `$fetchingErrorMessage` - 获取失败时的错误消息

---

## 总结

Wallabag通过以下机制实现多数据库支持和表前缀功能：

| 组件 | 职责 | 关键技术 |
|------|------|---------|
| InstallCommand::checkRequirements | 驱动、连接、版本检查 | PDO扩展检测、版本对比 |
| InstallCommand::setupDatabase | 四种场景的数据库初始化 | Doctrine命令组合 |
| InstallCommand::setupConfig | 初始化配置数据 | DQL清空 + ORM持久化 |
| TablePrefixSubscriber | ORM层表前缀 | loadClassMetadata事件 |
| SchemaAdapterSubscriber | MySQL字符集和索引调整 | postGenerateSchema事件 |
| MigrationFactoryDecorator | 迁移类依赖注入 | 装饰器模式 + setter注入 |

整个架构确保了:
- ✅ MySQL/PostgreSQL/SQLite三种数据库的兼容性
- ✅ 表前缀在ORM层和迁移层的一致应用
- ✅ MySQL utf8mb4的正确支持
- ✅ 安装流程的灵活场景处理
