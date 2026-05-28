# 文章注释闭环追踪文档

本文档追踪文章注释从前端 Annotator 组件到后端控制器、表单、实体和权限校验的完整闭环。

## 整体架构图

```
前端 Annotator 组件 (annotations_controller.js)
        ↓ (HTTP 请求)
AnnotationRestController (API 层，仅转发)
        ↓ (forward)
AnnotationController (业务逻辑层)
        ↓ (表单处理)
NewAnnotationType / EditAnnotationType (表单类)
        ↓ (数据映射)
Annotation 实体 (数据持久化)
        ↓ (权限校验)
EntryVoter / AnnotationVoter (权限控制)
```

---

## 1. 前端 Stimulus 控制器：annotations_controller.js

### 1.1 控制器位置
[assets/controllers/annotations_controller.js](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/assets/controllers/annotations_controller.js)

### 1.2 URL 和 entryId 配置

控制器通过 Stimulus 的 `values` 机制接收后端传递的配置参数：

```javascript
static values = {
  entryId: Number,
  createUrl: String,
  updateUrl: String,
  destroyUrl: String,
  searchUrl: String,
};
```

### 1.3 Twig 模板中的配置来源

在 [templates/Entry/entry.html.twig](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/templates/Entry/entry.html.twig#L361-L369) 中，这些值通过 `data-*` 属性传递：

```twig
<article
    data-controller="mathjax highlight annotations"
    data-annotations-entry-id-value="{{ entry.id }}"
    data-annotations-create-url-value="{{ path('annotations_post_annotation', {'entry': entry.id}) }}"
    data-annotations-update-url-value="{{ path('annotations_put_annotation', {'annotation': 'idAnnotation'}) }}"
    data-annotations-destroy-url-value="{{ path('annotations_delete_annotation', {'annotation': 'idAnnotation'}) }}"
    data-annotations-search-url-value="{{ path('annotations_get_annotations', {'entry': entry.id}) }}"
>
```

### 1.4 Annotator 存储配置

控制器连接时，将这些 URL 配置给 Annotator 的 HTTP 存储模块 [annotations_controller.js#L25-L33](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/assets/controllers/annotations_controller.js#L25-L33)：

```javascript
this.app.include(annotator.storage.http, {
  prefix: '',
  urls: {
    create: this.createUrlValue,
    update: this.updateUrlValue,
    destroy: this.destroyUrlValue,
    search: this.searchUrlValue,
  },
  entryId: this.entryIdValue,
  // ...
});
```

### 1.5 初始加载

启动后自动加载该 entry 的所有注释 [annotations_controller.js#L49-L51](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/assets/controllers/annotations_controller.js#L49-L51)：

```javascript
this.app.start().then(() => {
  this.app.annotations.load({ entry: this.entryIdValue });
});
```

---

## 2. API 层：AnnotationRestController

### 2.1 控制器位置
[src/Controller/Api/AnnotationRestController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/AnnotationRestController.php)

### 2.2 为什么只是转发？

AnnotationRestController 继承自 `WallabagRestController`，是 REST API 的入口层，其所有方法都使用 `$this->forward()` 转发到 `AnnotationController`。

**设计原因：**
1. **关注点分离**：RestController 负责 API 路由定义、Swagger/OpenAPI 文档注解、API 格式规范（`_format` 参数）
2. **代码复用**：避免业务逻辑在 API 层和 Web 层重复实现
3. **版本兼容**：API 路由使用 `/api/` 前缀，而业务控制器使用 `/annotations/` 前缀
4. **权限注解**：在 API 层声明 `#[IsGranted]` 注解，确保 API 访问的安全性

### 2.3 四个转发方法

| 方法 | 路由 | 权限 | 转发到 |
|------|------|------|--------|
| `getAnnotationsAction` | `GET /api/annotations/{entry}` | `LIST_ANNOTATIONS` | `AnnotationController::getAnnotationsAction` |
| `postAnnotationAction` | `POST /api/annotations/{entry}` | `CREATE_ANNOTATIONS` | `AnnotationController::postAnnotationAction` |
| `putAnnotationAction` | `PUT /api/annotations/{annotation}` | `EDIT` | `AnnotationController::putAnnotationAction` |
| `deleteAnnotationAction` | `DELETE /api/annotations/{annotation}` | `DELETE` | `AnnotationController::deleteAnnotationAction` |

转发示例 [AnnotationRestController.php#L44-L46](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/Api/AnnotationRestController.php#L44-L46)：

```php
return $this->forward('Wallabag\Controller\AnnotationController::getAnnotationsAction', [
    'entry' => $entry,
]);
```

---

## 3. 业务逻辑层：AnnotationController

### 3.1 控制器位置
[src/Controller/AnnotationController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/AnnotationController.php)

### 3.2 无名表单（createNamed 空字符串）

在创建和更新注释时，使用 `createNamed('', ...)` 创建**无名表单**：

**创建注释** [AnnotationController.php#L67-L71](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/AnnotationController.php#L67-L71)：
```php
$form = $this->formFactory->createNamed('', NewAnnotationType::class, $annotation, [
    'csrf_protection' => false,
    'allow_extra_fields' => true,
]);
$form->submit($data);
```

**更新注释** [AnnotationController.php#L99-L103](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/AnnotationController.php#L99-L103)：
```php
$form = $this->formFactory->createNamed('', EditAnnotationType::class, $annotation, [
    'csrf_protection' => false,
    'allow_extra_fields' => true,
]);
$form->submit($data);
```

**为什么使用空字符串表单名？**
- Annotator.js 发送的 JSON 数据是**扁平结构**，没有包裹在表单名前缀下
- `createNamed('', ...)` 使表单直接绑定到根级 JSON 字段，而不是 `form_name[field]` 格式
- 配合 `csrf_protection' => false` 因为 API 请求不使用 CSRF token
- `allow_extra_fields' => true` 允许 JSON 中包含表单未定义的额外字段，Annotator 可能会发送多余字段

### 3.3 JSON 正文处理

**创建注释** [AnnotationController.php#L62](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/AnnotationController.php#L62)：
```php
$data = json_decode($request->getContent(), true);
```

**更新注释** [AnnotationController.php#L97](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/AnnotationController.php#L97)：
```php
$data = json_decode($request->getContent(), true, 512, \JSON_THROW_ON_ERROR);
```

关键特点：
1. 直接从 `$request->getContent()` 读取原始 JSON 字符串
2. 使用 `json_decode(..., true)` 转换为关联数组
3. 更新操作使用 `\JSON_THROW_ON_ERROR` 捕获 JSON 解析错误
4. 解析后的数据通过 `$form->submit($data)` 提交给表单进行验证和数据映射

### 3.4 四个操作方法

| 方法 | 表单类型 | 权限 | 核心逻辑 |
|------|----------|------|----------|
| `getAnnotationsAction` | 无 | `LIST_ANNOTATIONS` | 通过 Repository 查询，序列化返回 |
| `postAnnotationAction` | `NewAnnotationType` | `CREATE_ANNOTATIONS` | 创建实体，设置 user 和 entry，表单提交验证 |
| `putAnnotationAction` | `EditAnnotationType` | `EDIT` | 加载现有实体，表单提交验证 |
| `deleteAnnotationAction` | 无 | `DELETE` | 直接移除实体 |

---

## 4. 表单类：NewAnnotationType 和 EditAnnotationType

### 4.1 NewAnnotationType（创建注释）

位置：[src/Form/Type/NewAnnotationType.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Form/Type/NewAnnotationType.php)

```php
public function buildForm(FormBuilderInterface $builder, array $options): void
{
    $builder
        ->add('text', null, [
            'empty_data' => '',
        ])
        ->add('quote', null, [
            'empty_data' => '',
            'trim' => false,
        ])
        ->add('ranges', CollectionType::class, [
            'entry_type' => RangeType::class,
            'allow_add' => true,
        ])
    ;
}
```

**字段说明：**
- `text`：用户输入的注释内容
- `quote`：被注释的原文（不 trim，保留原始空格）
- `ranges`：注释在文本中的位置范围（CollectionType，嵌套 RangeType）

### 4.2 EditAnnotationType（编辑注释）

位置：[src/Form/Type/EditAnnotationType.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Form/Type/EditAnnotationType.php)

```php
public function buildForm(FormBuilderInterface $builder, array $options): void
{
    $builder
        ->add('text', null, [
            'empty_data' => '',
        ])
    ;
}
```

**为什么只有 text 字段？**
- 编辑时只允许修改注释内容（`text`）
- 不允许修改被注释的原文（`quote`）和位置范围（`ranges`）
- 这是业务约束：一旦创建，注释的位置和引用的原文不可更改

---

## 5. 实体层：Annotation 实体

### 5.1 实体位置
[src/Entity/Annotation.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/Annotation.php)

### 5.2 User 关系维护

**构造函数注入 User** [Annotation.php#L80-L83](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/Annotation.php#L80-L83)：
```php
public function __construct(User $user)
{
    $this->user = $user;
}
```

**ManyToOne 关联** [Annotation.php#L68-L70](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/Annotation.php#L68-L70)：
```php
#[ORM\ManyToOne(targetEntity: User::class)]
#[Exclude]
private $user;
```

**关键点：**
- User 关系通过**构造函数强制注入**，确保每个 Annotation 必有所属用户
- `#[Exclude]` 注解用于 JMS Serializer，序列化时不暴露 user 对象（避免泄露敏感信息）
- 通过虚拟属性 `getUserName()` 只暴露用户名 [Annotation.php#L211-L216](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/Annotation.php#L211-L216)

### 5.3 Entry 关系维护

**setEntry 方法建立双向关联** [Annotation.php#L225-L231](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/Annotation.php#L225-L231)：
```php
public function setEntry($entry)
{
    $this->entry = $entry;
    $entry->setAnnotation($this);

    return $this;
}
```

**ManyToOne 关联** [Annotation.php#L72-L75](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Entity/Annotation.php#L72-L75)：
```php
#[ORM\JoinColumn(name: 'entry_id', referencedColumnName: 'id', onDelete: 'cascade')]
#[ORM\ManyToOne(targetEntity: Entry::class, inversedBy: 'annotations')]
#[Exclude]
private $entry;
```

**关键点：**
- `inversedBy: 'annotations'` 指向 Entry 实体的 `$annotations` 属性，建立**双向关联**
- `onDelete: 'cascade'` 数据库级联删除：Entry 删除时自动删除关联的 Annotation
- `setEntry()` 同时调用 `$entry->setAnnotation($this)`，确保双向关联的一致性
- `#[Exclude]` 序列化时不暴露 entry 对象

### 5.4 Controller 中建立关系

在 `postAnnotationAction` 中 [AnnotationController.php#L64-L65](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Controller/AnnotationController.php#L64-L65)：
```php
$annotation = new Annotation($this->getUser());
$annotation->setEntry($entry);
```

- 创建时通过构造函数设置 User
- 通过 `setEntry()` 设置 Entry 并建立双向关联

---

## 6. 权限校验：EntryVoter 和 AnnotationVoter

### 6.1 EntryVoter（保护 Entry 级操作）

位置：[src/Security/Voter/EntryVoter.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Security/Voter/EntryVoter.php)

**支持的注释相关属性：**
```php
public const LIST_ANNOTATIONS = 'LIST_ANNOTATIONS';
public const CREATE_ANNOTATIONS = 'CREATE_ANNOTATIONS';
```

**权限校验逻辑** [EntryVoter.php#L50-L53](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Security/Voter/EntryVoter.php#L50-L53)：
```php
return match ($attribute) {
    self::VIEW, self::EDIT, ..., self::LIST_ANNOTATIONS, self::CREATE_ANNOTATIONS, ... 
        => $user === $subject->getUser(),
    default => false,
};
```

**校验规则：**
- 只有 **Entry 的所有者** 才能列出该 Entry 的注释（`LIST_ANNOTATIONS`）
- 只有 **Entry 的所有者** 才能为该 Entry 创建注释（`CREATE_ANNOTATIONS`）
- 校验主体是 `Entry` 实体

### 6.2 AnnotationVoter（保护 Annotation 级操作）

位置：[src/Security/Voter/AnnotationVoter.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Security/Voter/AnnotationVoter.php)

**支持的属性：**
```php
public const EDIT = 'EDIT';
public const DELETE = 'DELETE';
```

**权限校验逻辑** [AnnotationVoter.php#L38-L41](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/wallabag/src/Security/Voter/AnnotationVoter.php#L38-L41)：
```php
return match ($attribute) {
    self::EDIT, self::DELETE => $subject->getUser() === $user,
    default => false,
};
```

**校验规则：**
- 只有 **Annotation 的所有者** 才能编辑该注释（`EDIT`）
- 只有 **Annotation 的所有者** 才能删除该注释（`DELETE`）
- 校验主体是 `Annotation` 实体

### 6.3 权限应用位置汇总

| 操作 | 控制器 | 权限属性 | Voter | 校验主体 |
|------|--------|----------|-------|----------|
| 列出注释 | `getAnnotationsAction` | `LIST_ANNOTATIONS` | EntryVoter | Entry |
| 创建注释 | `postAnnotationAction` | `CREATE_ANNOTATIONS` | EntryVoter | Entry |
| 编辑注释 | `putAnnotationAction` | `EDIT` | AnnotationVoter | Annotation |
| 删除注释 | `deleteAnnotationAction` | `DELETE` | AnnotationVoter | Annotation |

### 6.4 权限注解使用

在 Controller 方法上使用 `#[IsGranted]` 注解：

```php
// EntryVoter 校验（主体是 Entry）
#[IsGranted('LIST_ANNOTATIONS', subject: 'entry')]
#[IsGranted('CREATE_ANNOTATIONS', subject: 'entry')]

// AnnotationVoter 校验（主体是 Annotation）
#[IsGranted('EDIT', subject: 'annotation')]
#[IsGranted('DELETE', subject: 'annotation')]
```

`subject` 参数指定将控制器方法的哪个参数作为权限校验的主体。

---

## 7. 完整请求流程示例

### 7.1 创建注释流程

```
1. 用户在前端选中文本，输入注释内容
2. Annotator.js 发送 POST 请求到 /annotations/{entryId}
   └─ 请求体: { "text": "注释内容", "quote": "原文", "ranges": [...] }
3. 路由匹配到 AnnotationRestController::postAnnotationAction
4. #[IsGranted('CREATE_ANNOTATIONS', subject: 'entry')] 触发 EntryVoter
   └─ 校验：当前用户 === Entry 的所有者
5. forward 到 AnnotationController::postAnnotationAction
6. json_decode 解析请求体为数组
7. new Annotation($this->getUser()) 创建实体，设置 User
8. $annotation->setEntry($entry) 设置 Entry 及双向关联
9. createNamed('', NewAnnotationType::class, ...) 创建无名表单
10. $form->submit($data) 提交数据，验证 text/quote/ranges
11. 验证通过后 persist + flush
12. 序列化 Annotation 为 JSON 返回
```

### 7.2 编辑注释流程

```
1. 用户修改已有注释内容
2. Annotator.js 发送 PUT 请求到 /annotations/{annotationId}
   └─ 请求体: { "text": "新的注释内容" }
3. 路由匹配到 AnnotationRestController::putAnnotationAction
4. #[IsGranted('EDIT', subject: 'annotation')] 触发 AnnotationVoter
   └─ 校验：当前用户 === Annotation 的所有者
5. forward 到 AnnotationController::putAnnotationAction
6. json_decode 解析请求体为数组（带 JSON_THROW_ON_ERROR）
7. createNamed('', EditAnnotationType::class, ...) 创建无名表单
8. $form->submit($data) 提交数据，只验证 text 字段
9. 验证通过后 persist + flush
10. 序列化 Annotation 为 JSON 返回
```

---

## 8. 关键设计亮点

1. **双层控制器架构**：API 层负责路由和文档，业务层负责逻辑，实现关注点分离
2. **无名表单处理**：`createNamed('', ...)` 完美适配 Annotator.js 的扁平 JSON 格式
3. **双向关联维护**：`setEntry()` 同时设置反向关联，确保数据一致性
4. **构造函数强制依赖**：Annotation 构造函数要求 User，避免无主注释
5. **分级权限控制**：
   - EntryVoter 保护 Entry 级的创建/列出操作
   - AnnotationVoter 保护 Annotation 级的编辑/删除操作
6. **序列化安全**：使用 `#[Exclude]` 隐藏敏感关联，通过虚拟属性暴露必要信息
7. **数据库级联**：`onDelete: 'cascade'` 确保 Entry 删除时注释被清理
