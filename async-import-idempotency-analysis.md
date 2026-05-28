# 异步导入消息幂等与失败语义分析

## 一、整体架构概览

Wallabag 的异步导入采用 **生产者-队列-消费者** 模式，支持 Redis 和 AMQP (RabbitMQ) 两种消息中间件。核心流程：

```
Import (生产者)  →  Queue (Redis/AMQP)  →  Consumer (消费者)
parseEntriesForProducer  →  publish JSON  →  handleMessage → parseEntry → flush
```

关键类层次：

- 生产者侧：[AbstractImport](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Import/AbstractImport.php) → 各具体 Import（PocketImport、WallabagImport 等）
- 消费者侧：[AbstractConsumer](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Consumer/AbstractConsumer.php) → [AMQPEntryConsumer](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Consumer/AMQPEntryConsumer.php) / [RedisEntryConsumer](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Consumer/RedisEntryConsumer.php)
- 命令行入口：[RedisWorkerCommand](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Command/Import/RedisWorkerCommand.php)

---

## 二、parseEntriesForProducer：为什么只补 userId 并发布 JSON

### 方法源码

[AbstractImport::parseEntriesForProducer](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Import/AbstractImport.php#L197-L211)：

```php
protected function parseEntriesForProducer(array $entries): void
{
    foreach ($entries as $importedEntry) {
        $importedEntry['userId'] = $this->user->getId();

        if ($this->markAsRead) {
            $importedEntry = $this->setEntryAsRead($importedEntry);
        }

        ++$this->queuedEntries;

        $this->producer->publish(json_encode($importedEntry));
    }
}
```

### 设计意图分析

**只补 userId 的原因：**

1. **消费者无法获取当前用户上下文**：生产者运行在 Web 请求上下文中，持有当前登录用户（`$this->user`）；消费者运行在独立的 Worker 进程中，无法访问 Session 或 Security Token。因此必须在消息中携带 `userId`，消费者才能通过 `UserRepository::find($storedEntry['userId'])` 还原用户。

2. **不做任何数据库校验以追求速度**：方法注释明确说明 *"no call to the database should be done to speedup queuing"*。生产者跳过了 `validateEntry()`、`parseEntry()`、URL 去重查询（`findByUrlAndUserId`）、`em->persist()` 等全部数据库操作，仅做两件事：
   - 注入 `userId`
   - 可选地调用 `setEntryAsRead()` 修改内存中的数组字段（如设置 `is_archived = 1` 或 `status = '1'`）

3. **所有校验延迟到消费者执行**：注释也说明 *"We don't care to make check at this time. They'll be done by the consumer."*。这意味着生产者以 **尽力而为（best-effort）** 的方式快速入队，牺牲即时校验换取高吞吐。

4. **消息格式为纯 JSON 字符串**：`json_encode($importedEntry)` 将原始导入数据原封不动序列化，消费者收到后 `json_decode` 还原为关联数组，交给对应 Import 的 `validateEntry` / `parseEntry` 处理。

### 子类覆写

[HtmlImport::parseEntriesForProducer](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Import/HtmlImport.php#L159-L177) 和 [BrowserImport::parseEntriesForProducer](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Import/BrowserImport.php#L186-L204) 覆写了此方法，增加了非数组元素的过滤：

```php
if ((array) $importedEntry !== $importedEntry) {
    continue;
}
```

这是因为 HTML/浏览器书签的解析结果中可能包含非数组元素（如空节点），需要在入队前跳过，否则消费者 `json_decode` 后无法正常处理。核心的"补 userId → 发布 JSON"逻辑不变。

---

## 三、AbstractConsumer::handleMessage：四类失败情况的处理

### 方法源码

[AbstractConsumer::handleMessage](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Consumer/AbstractConsumer.php#L33-L81)：

```php
protected function handleMessage($body)
{
    $storedEntry = json_decode($body, true);

    $user = $this->userRepository->find($storedEntry['userId']);

    // 情况1：用户不存在
    if (null === $user) {
        $this->logger->warning('Unable to retrieve user', ['entry' => $storedEntry]);
        return true;  // 跳过消息
    }

    $this->import->setUser($user);

    // 情况2：validateEntry 失败
    if (false === $this->import->validateEntry($storedEntry)) {
        $this->logger->warning('Entry is invalid', ['entry' => $storedEntry]);
        return true;  // 跳过消息
    }

    $entry = $this->import->parseEntry($storedEntry);

    // 情况3：parseEntry 返回 null
    if (null === $entry) {
        $this->logger->warning('Entry already exists', ['entry' => $storedEntry]);
        return true;  // 跳过消息
    }

    // 情况4：flush 异常
    try {
        $this->em->flush();
        $this->eventDispatcher->dispatch(new EntrySavedEvent($entry), EntrySavedEvent::NAME);
        $this->em->clear();
    } catch (\Exception $e) {
        $this->logger->warning('Unable to save entry', ['entry' => $storedEntry, 'exception' => $e]);
        return false;  // 消息处理失败，触发重试
    }

    $this->logger->info('Content with url imported! (' . $entry->getUrl() . ')');
    return true;  // 处理成功
}
```

### 四类情况详细分析

| # | 异常场景 | 触发条件 | 返回值 | 幂等/重试语义 |
|---|---------|---------|--------|--------------|
| 1 | 用户不存在 | `UserRepository::find(userId)` 返回 `null` | `true` | **幂等丢弃**：用户已被删除，重试无意义。返回 `true` 告知消息中间件"已处理完成，可删除消息"。 |
| 2 | validateEntry 失败 | Import 的 `validateEntry()` 返回 `false`（如 URL 为空、缺少必需字段） | `true` | **幂等丢弃**：数据本身不合法，重试不会改变结果。返回 `true` 丢弃消息。 |
| 3 | parseEntry 返回 null | 条目已存在（`findByUrlAndUserId` 找到重复） | `true` | **幂等跳过**：条目已存在于数据库，属于天然幂等。返回 `true` 确认消息。 |
| 4 | flush 异常 | 数据库写入失败（约束冲突、连接断开等） | `false` | **非幂等重试**：返回 `false` 告知中间件消息处理失败，需要重新入队。 |

### 返回值的中间件语义

- **AMQP（RabbitMQ）**：`AMQPEntryConsumer::execute` 返回 `true`/`false` 映射到 `ConsumerInterface` 的约定 —— `true` 表示 ACK（确认并删除消息），`false` 表示 REJECT（拒绝，RabbitMQ 根据配置决定是否重入队）。
- **Redis**：`RedisEntryConsumer::manage` 返回 `true`/`false` 映射到 `Simpleue` 的 `Job` 接口约定 —— `true` 表示处理完成可从队列移除，`false` 表示处理失败将重新入队。

### validateEntry 各导入实现示例

| Import 类 | validateEntry 逻辑 |
|-----------|-------------------|
| [WallabagImport](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Import/WallabagImport.php#L77-L84) | `empty($importedEntry['url'])` → 拒绝空 URL |
| [PocketImport](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Import/PocketImport.php#L149-L156) | `resolved_url` 和 `given_url` 均为空 → 拒绝 |

### parseEntry 返回 null 的幂等保障

以 [WallabagImport::parseEntry](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Import/WallabagImport.php#L86-L134) 为例：

```php
$existingEntry = $this->em
    ->getRepository(Entry::class)
    ->findByUrlAndUserId($importedEntry['url'], $this->user->getId());

if (false !== $existingEntry) {
    ++$this->skippedEntries;
    return null;  // 幂等：已存在则跳过
}
```

每条消息消费时先查询数据库判断是否已导入，若已存在则返回 `null`，消费者收到 `null` 后返回 `true`（ACK）。这保证了 **即使消息被重复投递，也不会产生重复条目**，是幂等性的核心机制。

---

## 四、AMQPEntryConsumer 与 RedisEntryConsumer：如何复用同一逻辑

### 适配器模式

两个消费者子类都继承自 [AbstractConsumer](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Consumer/AbstractConsumer.php)，核心处理逻辑完全封装在 `handleMessage()` 中。子类只需实现各自中间件的接口，做消息体提取后委托给父类方法：

```
                    ┌──────────────────────┐
                    │   AbstractConsumer    │
                    │   handleMessage($body)│  ← 核心逻辑
                    └──────┬───────┬────────┘
                           │       │
              ┌────────────┘       └────────────┐
              ▼                                  ▼
┌──────────────────────┐           ┌──────────────────────┐
│  AMQPEntryConsumer   │           │  RedisEntryConsumer   │
│  implements          │           │  implements           │
│  ConsumerInterface   │           │  Job                  │
│                      │           │                       │
│  execute(AMQPMessage)│           │  manage($job)         │
│    → handleMessage   │           │    → handleMessage    │
│      ($msg->getBody())│          │      ($job)           │
│                      │           │                       │
│                      │           │  isStopJob($job)→false│
│                      │           │  isMyJob($job)→true   │
└──────────────────────┘           └──────────────────────┘
```

### AMQPEntryConsumer::execute

[AMQPEntryConsumer](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Consumer/AMQPEntryConsumer.php#L10-L13)：

```php
public function execute(AMQPMessage $msg): int|bool
{
    return $this->handleMessage($msg->getBody());
}
```

- 实现 `OldSound\RabbitMqBundle\RabbitMq\ConsumerInterface`
- 从 `AMQPMessage` 中提取原始 body（即生产者 `json_encode` 的字符串）
- 返回值直接透传 `handleMessage` 的 `true`/`false`，由 RabbitMQ Bundle 转为 ACK/NACK

### RedisEntryConsumer::manage

[RedisEntryConsumer](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Consumer/RedisEntryConsumer.php#L16-L19)：

```php
public function manage($job)
{
    return $this->handleMessage($job);
}
```

- 实现 `Simpleue\Job\Job` 接口
- Redis 队列中 `$job` 本身就是字符串（由 `RedisQueue::sendJob` 直接存入）
- 返回值透传 `handleMessage` 的 `true`/`false`，由 Simpleue Worker 决定是否移除消息

此外，`RedisEntryConsumer` 还实现了 `Job` 接口的两个辅助方法：

- `isStopJob($job)` → 始终返回 `false`：不识别停止信号，Worker 持续运行
- `isMyJob($job)` → 始终返回 `true`：每个队列只对应一种 Job 类型，无需路由判断

### 关键差异总结

| 维度 | AMQP | Redis |
|------|------|-------|
| 消息来源 | `AMQPMessage::getBody()` | 直接字符串 |
| 接口 | `ConsumerInterface` | `Job` |
| 重试机制 | RabbitMQ NACK + 死信队列 | Simpleue 重新入队 |
| 并发控制 | `prefetch_count` 配置 | 单线程顺序消费 |
| Worker 启动 | `rabbitmq-consumer` 命令 | `RedisWorkerCommand` |

---

## 五、RedisWorkerCommand：按 serviceName 选择 queue 和 consumer

### 命令定义

[RedisWorkerCommand](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Command/Import/RedisWorkerCommand.php#L14-L53)：

```php
class RedisWorkerCommand extends Command
{
    protected static $defaultName = 'wallabag:import:redis-worker';

    protected function configure(): void
    {
        $this
            ->addArgument('serviceName', InputArgument::REQUIRED,
                'Service to use: wallabag_v1, wallabag_v2, pocket, readability, ...')
            ->addOption('maxIterations', '', InputOption::VALUE_OPTIONAL,
                'Number of iterations before stopping', false)
        ;
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $serviceName = $input->getArgument('serviceName');

        if (!$this->container->has('wallabag.queue.redis.' . $serviceName)
            || !$this->container->has('wallabag.consumer.redis.' . $serviceName)) {
            throw new Exception(sprintf(
                'No queue or consumer found for service name: "%s"', $serviceName));
        }

        $worker = new QueueWorker(
            $this->container->get('wallabag.queue.redis.' . $serviceName),
            $this->container->get('wallabag.consumer.redis.' . $serviceName),
            (int) $input->getOption('maxIterations')
        );

        $worker->start();
        return 0;
    }
}
```

### 服务名到队列/消费者的映射规则

命令接收 `serviceName` 参数，通过**字符串拼接**构造两个容器服务 ID：

- **Queue**：`wallabag.queue.redis.{serviceName}`
- **Consumer**：`wallabag.consumer.redis.{serviceName}`

这两个服务 ID 在 [services_redis.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/services_redis.yml) 中预定义。映射关系如下：

| serviceName | Queue 服务 | Queue 名称 | Consumer 服务 | 注入的 Import |
|-------------|-----------|-----------|--------------|-------------|
| `pocket` | `wallabag.queue.redis.pocket` | `wallabag.import.pocket` | `wallabag.consumer.redis.pocket` | `PocketImport` |
| `readability` | `wallabag.queue.redis.readability` | `wallabag.import.readability` | `wallabag.consumer.redis.readability` | `ReadabilityImport` |
| `wallabag_v1` | `wallabag.queue.redis.wallabag_v1` | `wallabag.import.wallabag_v1` | `wallabag.consumer.redis.wallabag_v1` | `WallabagV1Import` |
| `wallabag_v2` | `wallabag.queue.redis.wallabag_v2` | `wallabag.import.wallabag_v2` | `wallabag.consumer.redis.wallabag_v2` | `WallabagV2Import` |
| `firefox` | `wallabag.queue.redis.firefox` | `wallabag.import.firefox` | `wallabag.consumer.redis.firefox` | `FirefoxImport` |
| `chrome` | `wallabag.queue.redis.chrome` | `wallabag.import.chrome` | `wallabag.consumer.redis.chrome` | `ChromeImport` |
| `instapaper` | `wallabag.queue.redis.instapaper` | `wallabag.import.instapaper` | `wallabag.consumer.redis.instapaper` | `InstapaperImport` |
| `pinboard` | `wallabag.queue.redis.pinboard` | `wallabag.import.pinboard` | `wallabag.consumer.redis.pinboard` | `PinboardImport` |
| `delicious` | `wallabag.queue.redis.delicious` | `wallabag.import.delicious` | `wallabag.consumer.redis.delicious` | `DeliciousImport` |
| `omnivore` | `wallabag.queue.redis.omnivore` | `wallabag.import.omnivore` | `wallabag.consumer.redis.omnivore` | `OmnivoreImport` |

### 运行示例

```bash
# 启动 Pocket 导入的 Redis Worker（无限循环消费）
php bin/console wallabag:import:redis-worker pocket

# 启动 Chrome 导入的 Redis Worker（最多处理 1000 条后退出）
php bin/console wallabag:import:redis-worker chrome --maxIterations=1000

# 传入不存在的 serviceName 会抛出异常
php bin/console wallabag:import:redis-worker unknown
# → No queue or consumer found for service name: "unknown"
```

### 防御性校验

`execute()` 方法在构造 Worker 前先检查两个服务是否都存在于容器中：

```php
if (!$this->container->has('wallabag.queue.redis.' . $serviceName)
    || !$this->container->has('wallabag.consumer.redis.' . $serviceName)) {
    throw new Exception(...);
}
```

这防止了因 serviceName 拼写错误导致的运行时 `ServiceNotFoundException`，实现了 **fail-fast**。

### AMQP 的等价机制

AMQP 侧不需要类似命令，因为 [config.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/app/config/config.yml#L236-L411) 中每个 consumer 配置已绑定 callback：

```yaml
consumers:
    import_pocket:
        exchange_options: { name: 'wallabag.import.pocket', type: topic }
        queue_options: { name: 'wallabag.import.pocket' }
        callback: wallabag.consumer.amqp.pocket
```

RabbitMQ Bundle 通过 `rabbitmq-consumer` 命令按 consumer 名称启动，自动完成 queue → callback 的绑定。

---

## 六、端到端幂等与失败语义总结

### 消息生命周期

```
┌─────────────────────────────────────────────────────────────┐
│                    生产者 (Web 请求)                         │
│  import() → parseEntriesForProducer()                       │
│    1. 补 userId                                              │
│    2. 可选 setEntryAsRead                                    │
│    3. json_encode → producer->publish()                     │
│    ⚠ 不做任何 DB 操作，快速入队                               │
└──────────────────────┬──────────────────────────────────────┘
                       │ JSON 字符串
                       ▼
┌──────────────────────────────────────────────────────────────┐
│              消息队列 (Redis / RabbitMQ)                      │
│  Redis:  wallabag.import.{serviceName}                      │
│  AMQP:   wallabag.import.{serviceName} (exchange + queue)   │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                    消费者 (Worker 进程)                       │
│  handleMessage($body)                                        │
│    1. json_decode → 还原关联数组                              │
│    2. UserRepository::find(userId)                           │
│       → null?  ACK + 丢弃 (幂等)                             │
│    3. import->validateEntry()                                │
│       → false? ACK + 丢弃 (幂等)                             │
│    4. import->parseEntry()                                   │
│       → null?  ACK + 跳过 (幂等：URL 去重)                   │
│    5. em->flush() + dispatch event                           │
│       → 异常?  NACK + 重试 (非幂等)                          │
│       → 成功?  ACK (完成)                                    │
└──────────────────────────────────────────────────────────────┘
```

### 幂等保障机制

| 层次 | 机制 | 说明 |
|------|------|------|
| 生产者 | 无校验快速入队 | 不保证消息本身有效，追求吞吐 |
| 消费者 | validateEntry | 过滤无效数据（空 URL 等），无效消息幂等丢弃 |
| 消费者 | findByUrlAndUserId 去重 | 重复消息不会产生重复 Entry，天然幂等 |
| 消费者 | flush 异常返回 false | 触发消息重入队，由幂等去重保证重试安全 |

### 失败语义对比

| 场景 | 处理方式 | 返回值 | 消息去向 | 是否可重试 |
|------|---------|--------|---------|-----------|
| 用户不存在 | 日志 warning + 丢弃 | `true` | 从队列删除 | 否（无意义） |
| validateEntry 失败 | 日志 warning + 丢弃 | `true` | 从队列删除 | 否（数据本身不合法） |
| parseEntry 返回 null | 日志 warning + 跳过 | `true` | 从队列删除 | 否（已存在，幂等） |
| flush 抛异常 | 日志 warning | `false` | 重新入队 | 是（幂等去重保障重试安全） |
