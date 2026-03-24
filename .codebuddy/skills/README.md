# H6-OpenAPI2 代码生成工具使用说明

本工具包含两个独立的Skill，用于快速生成h6-openapi2业务模块代码。

## Skill列表

### 1. h6-openapi2-create
**功能**: 生成模块新增文件（不含Controller）

**生成的文件(7个)**:
- Model类
- Detail Model类
- Service接口
- Service实现类
- Persistence实体类
- Detail Persistence实体类
- Repository类

### 2. h6-openapi2-callback
**功能**: 生成模块回调文件

**生成的文件(5个) + 更新Repository**:
- Callback Model类
- Callback Detail Model类
- Callback Persistence实体类
- Callback Detail Persistence实体类
- Callback Job类
- 更新Repository添加getByNumForCallBack方法

## 使用方法

### h6-openapi2-create

**命令格式**:
```
genType=create, moduleId=<模块ID>, package=<包名>, cls=<单据类型>
```

**参数说明**:
- `genType`: 固定为 "create"
- `moduleId`: 模块ID，小驼峰命名（如: proOrder）
- `package`: 包名（如: bill.pro）
- `cls`: 单据类型枚举值（如: "自营进货定单新增", etc.）
  - 这是 `BillCls` 枚举类中的一个值
  - 常用值包括: 批发、自营进货定单、采购计划调整单等
  - 参考 `BillCls.java` 查看所有可用的枚举值

**使用示例**:
```
genType=create, moduleId=proOrder, package=bill.pro, cls=自营进货定单新增
```

**生成的文件路径**:
```
h6-openapi2-api/src/main/java/com/hd123/h6/openapi2/model/{package}/{template}.java
h6-openapi2-api/src/main/java/com/hd123/h6/openapi2/model/{package}/{template}Detail.java
h6-openapi2-api/src/main/java/com/hd123/h6/openapi2/service/{package}/{template}Service.java
h6-openapi2-core/src/main/java/com/hd123/h6/openapi2/core/{package}/{template}ServiceImpl.java
h6-openapi2-core/src/main/java/com/hd123/h6/openapi2/repo/{package}/P{template}.java
h6-openapi2-core/src/main/java/com/hd123/h6/openapi2/repo/{package}/P{template}Detail.java
h6-openapi2-core/src/main/java/com/hd123/h6/openapi2/repo/{package}/{template}Repository.java
```

**注意**: template 是 moduleId 的首字母大写形式（如: proOrder → ProOrder）

### h6-openapi2-callback

**命令格式**:
```
genType=callback, moduleId=<模块ID>, package=<包名>
```

**参数说明**:
- `genType`: 固定为 "callback"
- `moduleId`: 模块ID，小驼峰命名（如: proOrder）
- `package`: 包名（如: bill.pro）

**使用示例**:
```
genType=callback, moduleId=proOrder, package=bill.pro
```

**生成的文件路径**:
```
h6-openapi2-api/src/main/java/com/hd123/h6/openapi2/model/{package}/callback/C{template}.java
h6-openapi2-api/src/main/java/com/hd123/h6/openapi2/model/{package}/callback/C{template}Detail.java
h6-openapi2-core/src/main/java/com/hd123/h6/openapi2/repo/{package}/PC{template}.java
h6-openapi2-core/src/main/java/com/hd123/h6/openapi2/repo/{package}/PC{template}Detail.java
h6-openapi2-core/src/main/java/com/hd123/h6/openapi2/job/callback/{package}/C{template}CallBackJob.java
```

**更新文件**:
```
h6-openapi2-core/src/main/java/com/hd123/h6/openapi2/repo/{package}/{template}Repository.java
（添加 getByNumForCallBack 方法）
```

**配置要求**:
在 `application.properties` 或 `application.yml` 中添加：
```properties
# 启用callback功能
openapi2.job.{moduleId}.callback.enable=true

# Cron表达式（可选，默认值: 0 0/1 * * ?）
openapi2.job.{moduleId}.callback.cronExpression=0 0/1 * * ?
```

示例（proOrder）:
```properties
openapi2.job.proOrder.callback.enable=true
openapi2.job.proOrder.callback.cronExpression=0 0/1 * * ?
```

## 使用流程

### 场景1: 仅生成模块新增文件
直接使用 `h6-openapi2-create` 生成模块新增。

### 场景2: 生成模块新增文件 + 模块回调功能
1. 先使用 `h6-openapi2-create` 生成模块新增文件
2. 再使用 `h6-openapi2-callback` 生成模块回调文件

### 场景3: 仅生成模块回调文件
直接使用 `h6-openapi2-callback`，该工具会：
- 生成5个Callback相关文件
- 自动更新现有的Repository文件，添加getByNumForCallBack方法

## 变量替换规则

所有模板中使用的变量：
- `{genType}` - "create" 或 "callback"
- `{moduleId}` - 小驼峰模块ID（如: "proOrder"）
- `{package}` - 包名（如: "bill.pro"）
- `{template}` - 类名（moduleId首字母大写，如: "ProOrder"）
- `{cls}` - 单据类型枚举值（如: "自营进货定单", etc.）

## 注意事项

1. **包名路径转换**: 包名中的点会自动转换为路径分隔符（如: bill.pro → bill/pro/）
2. **命名规范**: 
   - moduleId 使用小驼峰命名（如: proOrder）
   - template 自动为首字母大写形式（如: ProOrder）
3. **配置文件**: 使用callback功能后，记得在配置文件中添加相应的配置项
