# h6-openapi2 验证规则使用指南

## 概述

`validation-rules.yaml` 定义了字段名到验证代码的映射关系，用于根据 API 对象字段自动生成 ServiceImpl 中的验证逻辑。

## 规则文件结构

### 1. header_fields（单头字段验证规则）

定义单据主表的字段验证规则：

```yaml
header_fields:
  storeCode:
    table: Store          # 验证时查询的表名
    column: Code          # 验证时查询的列名
    description: 门店代码  # 错误提示中的中文名称
    require_org_match: true  # 是否需要匹配组织关系
    org_match_field: orgCode # 组织关系匹配时使用的字段名
```

**字段说明：**
- `table`: 数据库表名（用于验证时查询）
- `column`: 表中的列名
- `description`: 错误消息中显示的中文名称
- `require_org_match`: 是否需要验证与 orgCode 的组织关系
- `org_match_field`: 组织匹配时使用的字段名（通常是 orgCode）

### 2. detail_fields（明细字段验证规则）

定义明细表的字段验证规则，格式与 header_fields 类似，但用于 `getDetails()` 集合。

```yaml
detail_fields:
  goodsCode:
    table: Goods
    column: Code
    description: 明细商品代码
    require_org_match: false
```

### 3. table_descriptions（表名中文映射）

表名到中文名称的映射，用于错误提示。

```yaml
table_descriptions:
  Store: 门店
  Vendor: 供应商
  WareHouse: 仓位
```

### 4. field_aliases（字段别名映射）

处理不同命名风格的字段，统一映射到标准字段名。

```yaml
field_aliases:
  store_code: storeCode    # 下划线命名转驼峰
  vendor: vendorCode      # 简写别名
```

### 5. default_config（默认配置）

全局配置项。

```yaml
default_config:
  include_todo: false              # 是否包含 TODO 注释
  auto_generate_org_match: true    # 是否自动生成组织匹配验证
  detail_prefix: 明细              # 明细验证的前缀
```

## 使用示例

### 示例 1: 基本字段验证

**API 对象字段：**
```java
private String orgCode;
private String storeCode;
private String vendorCode;
```

**生成的验证代码：**
```java
sb.append({moduleId}Repository.validateCodeExists(Collections.singletonList({moduleId}), "Store",
    "Code", "orgCode", "组织代码"));
sb.append({moduleId}Repository.validateCodeExists(Collections.singletonList({moduleId}), "Store",
    "Code", "storeCode", "门店代码"));
sb.append({moduleId}Repository.validateCodeExists(Collections.singletonList({moduleId}), "Vendor",
    "Code", "vendorCode", "供应商代码"));
```

### 示例 2: 带组织匹配的验证

**API 对象字段：**
```java
private String orgCode;
private String storeCode;
```

**生成的验证代码（代码存在性验证 + 组织匹配验证）：**
```java
// 代码存在性验证
sb.append({moduleId}Repository.validateCodeExists(Collections.singletonList({moduleId}), "Store",
    "Code", "orgCode", "组织代码"));
sb.append({moduleId}Repository.validateCodeExists(Collections.singletonList({moduleId}), "Store",
    "Code", "storeCode", "门店代码"));

// 组织匹配验证
sb.append({moduleId}Repository.validateCodeMatchOrg({moduleId}.getStoreCode(),
    {moduleId}.getOrgCode(), "Store", "Code", "门店代码"));
```

### 示例 3: 明细字段验证

**Detail API 对象字段：**
```java
private String goodsCode;
private String inputCode;
```

**生成的验证代码：**
```java
sb.append({moduleId}Repository.validateCodeExists({moduleId}.getDetails(), "Goods", "Code",
    "goodsCode", "明细商品代码"));
sb.append({moduleId}Repository.validateCodeExists({moduleId}.getDetails(), "GdInput", "Code",
    "inputCode", "明细商品输入码"));
```

## 字段别名处理

如果 API 对象使用了别名，会自动映射到标准字段：

```java
// API 对象中使用别名
private String vendor;  // 映射到 vendorCode

// 生成的验证代码
sb.append({moduleId}Repository.validateCodeExists(Collections.singletonList({moduleId}), "Vendor",
    "Code", "vendorCode", "供应商代码"));
```

## 自定义规则

### 添加新的字段验证规则

在 `header_fields` 或 `detail_fields` 中添加新字段：

```yaml
header_fields:
  customField:
    table: CustomTable
    column: CustomColumn
    description: 自定义字段
    require_org_match: true
    org_match_field: orgCode
```

### 添加新的表名映射

在 `table_descriptions` 中添加：

```yaml
table_descriptions:
  CustomTable: 自定义表
```

### 添加字段别名

在 `field_aliases` 中添加：

```yaml
field_aliases:
  custom_field: customField
```

## 验证代码生成逻辑

### 1. 单头字段验证

对于每个 header_fields 中定义的字段，生成：

```java
sb.append({moduleId}Repository.validateCodeExists(Collections.singletonList({moduleId}), "{table}",
    "{column}", "{fieldName}", "{description}"));
```

如果 `require_org_match: true`，还会生成：

```java
sb.append({moduleId}Repository.validateCodeMatchOrg({moduleId}.get{FieldName}(),
    {moduleId}.get{OrgMatchField}(), "{table}", "{column}", "{description}"));
```

### 2. 明细字段验证

对于每个 detail_fields 中定义的字段，生成：

```java
sb.append({moduleId}Repository.validateCodeExists({moduleId}.getDetails(), "{table}",
    "{column}", "{fieldName}", "{detail_prefix}{description}"));
```

## 注意事项

1. **字段名匹配**: 规则匹配是区分大小写的
2. **优先级**: 直接字段匹配 > 别名映射
3. **必填字段**: orgCode 通常作为基础字段，应优先验证
4. **明细字段**: 必须是 API 对象中的 `details` 列表项的属性
5. **组织匹配**: 只有当 API 对象同时包含目标字段和 orgCode 时才会生成组织匹配验证

## 与 h6-openapi2-create skill 的集成

验证规则已经完全集成到 `h6-openapi2-create` skill 中。生成 ServiceImpl 时会自动：

1. 读取 API 对象（{template}.java）的所有字段
2. 读取 Detail API 对象（{template}Detail.java）的所有字段
3. 检查字段名在规则文件中的定义（包括别名映射）
4. 根据规则生成对应的验证代码：
   - 为匹配的 header_fields 生成 `validateCodeExists` 调用
   - 为需要组织匹配的字段生成 `validateCodeMatchOrg` 调用
   - 为匹配的 detail_fields 生成针对明细集合的验证代码
5. 按照字段顺序组织代码（通常 orgCode 在最前）
6. 将生成的验证代码插入到 ServiceImpl 的 create 方法中

**重要提示**：
- 只有在 `validation-rules.yaml` 中定义的字段才会生成验证代码
- 字段名匹配区分大小写
- 可以通过 `field_aliases` 支持字段名别名
- 组织匹配验证会自动检查 API 对象是否同时包含目标字段和 orgCode 字段

详细的生成逻辑和代码模板请参考 `SKILL.md` 中的完整说明。

## 版本控制

规则文件使用版本号管理，当前版本为 `1.0`。如有重大变更，请更新版本号。

## 扩展建议

未来可以扩展以下功能：
1. 支持自定义验证方法（不仅仅是 validateCodeExists）
2. 支持条件验证（某些字段在其他字段满足特定值时才验证）
3. 支持字段间的依赖关系验证
4. 支持自定义错误消息模板
