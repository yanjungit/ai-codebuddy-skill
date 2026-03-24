# H6-OpenAPI2 Create Generator - 使用说明

## 文档说明

本技能包含以下文档：

- **INSTRUCTIONS.md** - 本文件，快速开始指南
- **SKILL.md** - 完整的技能文档和代码模板（AI执行时参考）
- **validation-rules.yaml** - 验证规则配置文件
- **validation-rules-guide.md** - 验证规则使用和配置指南

## 快速开始

通过以下命令使用此技能生成h6-openapi2项目文件：

```
genType=create, moduleId={moduleId}, package={package}, cls={cls}
```

## 参数说明

- `genType`: 固定为 "create"
- `moduleId`: 模块ID，小驼峰命名（如：vdrPay, proOrder）
- `package`: 包名（如：bill.account, bill.pro）
- `cls`: 单据类型枚举值（如：交款单, 自营进货定单）

## 自动生成内容

### 1. Java文件（共5个）
- Persistence实体类: `P{template}.java`
- 明细Persistence实体类: `P{template}Detail.java`
- Service接口: `{template}Service.java`
- Service实现类: `{template}ServiceImpl.java`
- Repository接口: `{template}Repository.java`

### 2. Oracle脚本文件（3个更新/新增）
- `PPS_OpenApi2_Pkg.oracle.sql` - 添加Process和CheckBeforeUpload路由
- `PPS_OpenApi2_Account_Pkg.oracle.sql` - 添加Check和ProcessSingle函数
- `PPS_OpenApi2_Account_Hdr.oracle.sql` - 添加函数声明

### 3. 更新的文件（2个）
- `BillCls.java` - 添加单据类型枚举
- `DataProcessHandler.java` - 添加事件处理逻辑

## 智能字段检测

技能会自动检测 `{template}Detail` 的字段类型，生成对应的校验逻辑：

### 商品型明细（包含goodsCode字段）
- 校验商品代码存在性
- 校验商品输入码匹配
- 校验包装规格（qpcStr）
- 校验计量单位（munit）
- 校验价格类型和价格

### 通用明细
- 校证明细记录不为空

## Git操作

所有生成的文件会自动执行 `git add` 操作，暂存到git仓库。

## 文件位置

### Java文件
- Persistence: `h6-openapi2-core/src/main/java/com/hd123/h6/openapi2/repo/{package}/`
- Service: `h6-openapi2-api/src/main/java/com/hd123/h6/openapi2/service/{package}/`
- ServiceImpl: `h6-openapi2-core/src/main/java/com/hd123/h6/openapi2/core/{package}/`
- Repository: `h6-openapi2-core/src/main/java/com/hd123/h6/openapi2/repo/{package}/`

### Oracle脚本
- `h6-openapi2-rdb-setup/src/main/resources/META-INF/h6-openapi2-setup/script/oracle/sp/`

## 验证代码生成

根据 `validation-rules.yaml` 自动生成验证代码：
- 代码存在性验证
- 组织匹配验证（对于require_org_match=true的字段）

## 示例

### 生成商品型单据（如：自营进货定单）
```
genType=create, moduleId=proOrder, package=bill.pro, cls=自营进货定单
```
生成包含商品相关校验的Oracle脚本。

### 生成费用型单据（如：交款单）
```
genType=create, moduleId=vdrPay, package=bill.account, cls=交款单
```
生成包含费用相关校验的Oracle脚本。

## 注意事项

1. 确保API对象（{template}.java和{template}Detail.java）已存在
2. 单据类型枚举值必须是BillCls.java中定义的值
3. Oracle脚本中的TODO标记需要根据实际业务需求补充
4. 生成的文件会自动添加到git暂存区
