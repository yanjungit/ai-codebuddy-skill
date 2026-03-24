---
name: h6-openapi2-callback
description: This skill should be used when users need to add Callback functionality to existing h6-openapi2 business modules. It generates Callback Model classes, Callback Persistence entities, Callback Job class, and adds getByNumForCallBack method to existing Repository.
---

# H6-OpenAPI2 Callback Generator

This skill provides automated generation of Callback files for existing h6-openapi2 business modules. It assumes basic files (Model, Service, Repository, etc.) have already been created.

## Purpose

Generate Callback-related files for an existing business module in h6-openapi2 project, including:
- Callback Persistence entity (PC{template}) - based on existing C{template}
- Callback Detail Persistence entity (PC{template}Detail) - based on existing C{template}Detail
- Callback Job class
- Update existing Repository to add `getByNumForCallBack` method

Note: This skill assumes C{template} and C{template}Detail already exist and will read their fields to generate corresponding Persistence entities.

## When to Use

Use this skill when:
- C{template} and C{template}Detail classes already exist
- User specifies `genType=callback` parameter
- User specifies `moduleId` parameter (e.g., "alcDiff")
- User specifies `package` parameter for callback files (e.g., "bill.callback.alcdiff" or "bill.alcdiff")
- User specifies `billType` parameter (e.g., "pro", "dist", "sale")
- User wants to add Persistence entities and Callback functionality

## File Generation Rules

### Parameters

When generating files, extract following parameters from user request:
- `genType` - Must be "callback" to use this skill
- `moduleId` - Module ID in lower camel case (e.g., "proOrder")
- `package` - Package name for callback files (e.g., "bill.callback.proOrder" or "bill.pro")
- `billType` - Bill type for repository location (e.g., "pro", "dist", "sale")
- Calculate `template` by capitalizing first letter of `moduleId` (e.g., "proOrder" → "ProOrder")
- **Calculate `repoPackage`**: Always use "bill.{billType}.{moduleId}" format for Repository (e.g., "bill.pro.dirwholesale")

### Naming Conventions

1. **Callback Persistence Entity (PC{template})**: Located in `h6-openapi2-core/src/main/java/com/hd123/h6/openapi2/repo/{package}/PC{template}.java`
   - **Implements `HasCustomField` and `Serializable`**
   - Uses `@Setter`, `@Getter` annotations
   - Includes `TABLE_NAME` constant (view name: `v_OpenApi2_{template}_create`)
   - Includes `TO_E` Converter mapping to Detail list
   - **Fields copied from existing C{template} class**: Read C{template}.java and copy all fields (excluding customFields-related)
   - Adds `List<PC{template}Detail> details = new ArrayList<>()` field
   - Adds `Map<String, Object> customFields` field

2. **Callback Detail Persistence Entity (PC{template}Detail)**: Located in `h6-openapi2-core/src/main/java/com/hd123/h6/openapi2/repo/{package}/PC{template}Detail.java`
   - **Extends C{template}Detail** (which already extends CBaseBillDetail that implements HasCustomField)
   - Uses `@Setter`, `@Getter` annotations
   - Includes `serialVersionUID`
   - Includes `TABLE_NAME` constant (view name: `v_OpenApi2_{template}_createDtl`)
   - Includes `TO_E` Converter
   - Includes `TO_E_List` ArrayListConverter
   - **No additional fields needed** - inherits all fields from C{template}Detail (including HasCustomField support)

3. **Callback Job Class**: Located in `h6-openapi2-core/src/main/java/com/hd123/h6/openapi2/job/callback/{package}/C{template}CallBackJob.java`
   - Extends `CallBackBaseJob<C{template}>`
   - Uses `@Component`, `@Slf4j`, `@ConditionalOnProperty` annotations
   - Property condition: `@ConditionalOnProperty(havingValue = "true", prefix = "openapi2.job.{moduleId}", name = {"callback.enable" })`
   - Autowires `{template}Repository`
   - Implements `fetchData()` method calling `repository.getByNumForCallBack(num)`
   - Implements `getDataCategorySuffix()` method
   - Implements `getCallBackType()` method
   - Implements `getDescription()` method
   - Implements `getCronExpression()` method reading from config

6. **Repository Handling**:
   - **If `{template}Repository.java` exists**: Add `getByNumForCallBack(String num)` method
   - **If `{template}Repository.java` does NOT exist**: Create the complete Repository file with `getByNumForCallBack` method
   - **Repository package path**: Uses `bill.{billType}.{moduleId}` format (e.g., "bill.pro.dirwholesale")
   - Reads from callback view tables
   - Uses `HasCustomFieldRowMapper` to handle custom fields
   - Returns `C{template}` object

### Package Structure

Replace `package` dots with path separators:
- `bill.pro` → `bill/pro/`
- Callback subpackage: `{package}/callback/`
- Callback job subpackage: `job/callback/{package}/`

### Copyright Headers

All files must include copyright header with current year (2026) and creation date.

### Configuration for Callback

After generating callback files, the following configuration is expected in `application.properties` or `application.yml`:

```properties
# Enable callback job for {moduleId}
openapi2.job.{moduleId}.callback.enable=true

# Cron expression for callback job (optional, default: "0 0/1 * * ?")
openapi2.job.{moduleId}.callback.cronExpression=0 0/1 * * ?
```

Example for proOrder:
```properties
openapi2.job.proOrder.callback.enable=true
openapi2.job.proOrder.callback.cronExpression=0 0/1 * * ?
```

## Workflow

1. **Parse Parameters**
   - Extract `genType` from user request (must be "callback")
   - Extract `moduleId` from user request (e.g., "proOrder")
   - Extract `package` from user request (e.g., "bill.pro")
   - Calculate `template` by capitalizing first letter of `moduleId` (e.g., "proOrder" → "ProOrder")

2. **Generate Files**
   - Replace variables in code templates
   - Replace `package` dots with path separators for PC files and Job
   - **Calculate `repoPackage`**: Use format `bill.{billType}.{moduleId}` (e.g., "bill.pro.dirwholesale")
   - Replace `repoPackage` dots with path separators for Repository files
   - Read existing C{template} and C{template}Detail classes to extract field information
   - Generate 3 Persistence/Job files:
     1. PC{template}.java (based on C{template} fields) - uses `package`
     2. PC{template}Detail.java (extends C{template}Detail) - uses `package`
     3. C{template}CallBackJob.java - uses `package`
   - Check if `{template}Repository.java` exists in `repoPackage` path:
     - **If exists**: Update existing Repository file to add `getByNumForCallBack` method
     - **If not exists**: Create complete Repository file with `getByNumForCallBack` method using `repoPackage`

3. **Validate**
   - Check that all files are written to correct locations
   - Verify package declarations are correct
   - Ensure proper inheritance and annotations
   - Verify Repository was updated correctly

4. **Git Add Files**
   - After successful file generation, execute `git add` command for each generated file
   - Command pattern: `git add <full-path-to-file>`
   - If Repository was created, also `git add` the Repository file
   - If Repository was updated, the modified file is already in git, so just ensure it's staged
   - This ensures all generated/modified files are staged for commit

5. **Report**
   - List all generated files with full paths
   - Indicate whether Repository was created or updated
   - Confirm that all files have been added to git staging area
   - Provide configuration example

## Example Usage

User request: "genType=callback, moduleId=proOrder, package=bill.callback.proOrder, billType=pro"

**Step 1: Parse Parameters**
- `genType` = "callback"
- `moduleId` = "proOrder"
- `package` = "bill.callback.proOrder"
- `billType` = "pro"
- `template` = "ProOrder"
- `repoPackage` = "bill.pro.proOrder"

**Step 2: Generate Files**
Generate 3 callback files and handle Repository:
1. `h6-openapi2-core/.../repo/bill/callback/proOrder/PCProOrder.java`
2. `h6-openapi2-core/.../repo/bill/callback/proOrder/PCProOrderDetail.java`
3. `h6-openapi2-core/.../job/callback/bill/callback/proOrder/CProOrderCallBackJob.java`
4. **Check Repository** in `bill.pro.proOrder`:
   - Repository location: `h6-openapi2-core/.../repo/bill/pro/proOrder/ProOrderRepository.java`
   - If exists: add getByNumForCallBack method
   - If not exists: create complete ProOrderRepository.java with getByNumForCallBack method

**Configuration Required:**
```properties
openapi2.job.proOrder.callback.enable=true
openapi2.job.proOrder.callback.cronExpression=0 0/1 * * ?
```

## Code Templates

### PC{template} Persistence Entity Template (based on existing C{template} class)

**NOTE**: Read the existing C{template}.java file and copy all fields (excluding customFields-related) to generate this class.

```java
/**
 * 版权所有(C)，XX有限公司，2026，所有权利保留。
 * <p>
 * 项目名： h6-openapi2
 * 文件名： PC{template}.java
 * 模块说明：
 * 修改历史：
 * 2026年01月23日 - yanjun - 创建。
 */
package com.hd123.h6.openapi2.repo.{package};

import java.io.Serializable;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import com.hd123.h6.openapi2.customfield.HasCustomField;
import com.hd123.h6.openapi2.model.{package}.C{template};
import com.hd123.rumba.commons.util.converter.ArrayListConverter;
import com.hd123.rumba.commons.util.converter.Converter;
import com.hd123.rumba.commons.util.converter.ConverterBuilder;

import lombok.Getter;
import lombok.Setter;

/**
 * @author yanjun
 * @since 1.0
 */
@Setter
@Getter
public class PC{template} implements HasCustomField, Serializable {

  private static final long serialVersionUID = -7753367884170523822L;

  public static final String TABLE_NAME = "v_OpenApi2_{template}_create";

  public static Converter<PC{template}, C{template}> TO_E = ConverterBuilder
      .newBuilder(PC{template}.class, C{template}.class).map("details", PC{template}Detail.TO_E_List)
      .build();

  public static ArrayListConverter<PC{template}, C{template}> TO_E_List = ArrayListConverter
      .newConverter(TO_E);

  // ===== Fields copied from C{template}.java =====
  // Example fields (read actual C{template} class and copy all fields here)
  private String num;
  private String orgCode;
  private String note;
  // Add all other fields from C{template}...
  // ============================================

  private Map<String, Object> customFields;

  private List<PC{template}Detail> details = new ArrayList<>();
}
```
  private List<C{template}Detail> details = new ArrayList<>();
  @JsonIgnore
  private Map<String, Object> customFields;
}
```

### Callback Detail Model Template
```java
/**
 * 版权所有(C)，XX有限公司，2026，所有权利保留。
 * <p>
 * 项目名： h6-openapi2
 * 文件名： C{template}.java
 * 模块说明：
 * 修改历史：
 * 2026年01月23日 - yanjun - 创建。
 */
package com.hd123.h6.openapi2.model.{package}.callback;

import java.math.BigDecimal;

import com.hd123.h6.openapi2.model.bill.callback.common.CBaseBillDetail;

import io.swagger.annotations.ApiModelProperty;
import lombok.Getter;
import lombok.Setter;

/**
 * @author yanjun
 * @since 1.0
 */
@Setter
@Getter
public class C{template}Detail extends CBaseBillDetail {
  private static final long serialVersionUID = 4459048358358611671L;
  @ApiModelProperty(value = "包装规格")
  private String qpcStr;

  @ApiModelProperty(value = "规格")
  private BigDecimal qpc;
  @ApiModelProperty(value = "单位")
  private String munit;

  @ApiModelProperty(value = "数量")
  private BigDecimal qty;
}
```

### Callback Persistence Entity Template
```java
/**
 * 版权所有(C)，XX有限公司，2026，所有权利保留。
 * <p>
 * 项目名： h6-openapi2
 * 文件名： PC{template}.java
 * 模块说明：
 * 修改历史：
 * 2026年01月23日 - yanjun - 创建。
 */
package com.hd123.h6.openapi2.repo.{package};

import java.io.Serializable;
import java.util.List;
import java.util.Map;



我注意到您发送的任务是"为选定的代码编写简要的文档注释"，但提供的代码内容是空的 markdown 文件。

同时，根据对话历史，我们正在进行的工作是：

**当前任务背景：**
- 生成类型：create
- 模块ID：aaa
- 包名：bill.pro
- 需要生成的文件：Model, Service, ServiceImpl, Repository, Persistence

**当前问题：**
技能 `h6-openapi2-create` 需要读取 `Aaa.java` 和 `AaaDetail.java` 模型文件中的字段信息来生成 Persistence 实体，但在项目中找不到这些文件。

请您明确一下：

1. **如果您想继续代码生成任务**：请提供 `Aaa.java` 和 `AaaDetail.java` 模型文件的位置，或者确认是否需要我先生成包含标准字段的基础文件。

2. **如果您想切换到编写文档注释的任务**：请提供需要编写注释的具体代码文件路径和内容。

请问您希望我继续哪个任务？
### PC{template}Detail Persistence Entity Template (extends C{template}Detail)

**NOTE**: This class extends C{template}Detail, which already extends CBaseBillDetail (which implements HasCustomField).

```java
/**
 * 版权所有(C)，XX有限公司，2026，所有权利保留。
 * <p>
 * 项目名： h6-openapi2
 * 文件名： PC{template}Detail.java
 * 模块说明：
 * 修改历史：
 * 2026年01月23日 - yanjun - 创建。
 */
package com.hd123.h6.openapi2.repo.{package};

import com.hd123.h6.openapi2.model.{package}.C{template}Detail;
import com.hd123.rumba.commons.util.converter.ArrayListConverter;
import com.hd123.rumba.commons.util.converter.Converter;
import com.hd123.rumba.commons.util.converter.ConverterBuilder;

import lombok.Getter;
import lombok.Setter;

/**
 * @author yanjun
 * @since 1.0
 */
@Setter
@Getter
public class PC{template}Detail extends C{template}Detail {

  private static final long serialVersionUID = 5252796373717089024L;

  public static final String TABLE_NAME = "v_OpenApi2_{template}_createDtl";

  public static Converter<PC{template}Detail, C{template}Detail> TO_E = ConverterBuilder
      .newBuilder(PC{template}Detail.class, C{template}Detail.class).build();

  public static ArrayListConverter<PC{template}Detail, C{template}Detail> TO_E_List = ArrayListConverter
      .newConverter(TO_E);
}
```

  public static ArrayListConverter<PC{template}Detail, C{template}Detail> TO_E_List = ArrayListConverter
      .newConverter(TO_E);



### Callback Job Template
```java
/**
 * 版权所有(C)，XX有限公司，2025，所有权利保留。
 * <p>
 * 项目名：   h6-openapi2
 * 文件名：   C{template}CallBackJob.java
 * 模块说明：
 * 修改历史：
 * 2025年05月13日 - yanjun - 创建。
 */
package com.hd123.h6.openapi2.job.callback.{package};

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.stereotype.Component;

import com.hd123.h6.openapi2.job.callback.bill.CallBackBaseJob;
import com.hd123.h6.openapi2.model.callbackmessage.CallBackType;
import com.hd123.h6.openapi2.model.notifymessage.NotifyMessage;
import com.hd123.h6.openapi2.model.{package}.C{template};
import com.hd123.h6.openapi2.repo.{package}.{template}Repository;
import com.hd123.swallows.exception.SwallowsServiceException;

import lombok.extern.slf4j.Slf4j;

/**
 * @author yanjun
 * @since 1.0
 */
@Component
@Slf4j
@ConditionalOnProperty(havingValue = "true", prefix = "openapi2.job.{moduleId}", name = {
    "callback.enable" })
public class C{template}CallBackJob extends CallBackBaseJob<C{template}> {

  @Value("${openapi2.job.{moduleId}.callback.cronExpression:0 0/1 * * ?}")
  private String cronExpression;
  @Autowired
  private {template}Repository repository;

  @Override
  protected C{template} fetchData(NotifyMessage notifyMessage) throws SwallowsServiceException {
    return repository.getByNumForCallBack(notifyMessage.getBillNumber());
  }

  @Override
  public String getDataCategorySuffix() {
    return ".{moduleId}.preaudit";
  }

  @Override
  public String getCallBackType() {
    return CallBackType.INVDIFF_CREATE;
  }

  @Override
  public String getDescription() {
    return "{template}预审推送";
  }

  @Override
  public String getCronExpression() {
    return cronExpression;
  }
}
```

### Repository Templates

**Scenario 1: Repository Already Exists** - Add method to existing `{template}Repository.java`

First, add necessary imports (if not already present):
```java
import com.hd123.h6.openapi2.model.{package}.C{template};
import com.hd123.h6.openapi2.repo.bill.common.HasCustomFieldRowMapper;
import com.hd123.h6.openapi2.repo.{repoPackage}.PC{template};
import com.hd123.h6.openapi2.repo.{repoPackage}.PC{template}Detail;
import com.hd123.rumba.commons.jdbc.sql.Predicates;
import com.hd123.rumba.commons.jdbc.sql.SelectStatement;
```

Then, add method at the end of the Repository class (before the closing brace):
```java
  public C{template} getByNumForCallBack(String num) throws SwallowsServiceException {
    SelectStatement selectStatement = selectBuilder().from(PC{template}.TABLE_NAME, "o")
        .where(Predicates.equals("o.num", num)).build();
    List<PC{template}> templates = jdbcTemplate.query(selectStatement, new HasCustomFieldRowMapper<>(
        PC{template}.class, getCustomCols(PC{template}.TABLE_NAME, PC{template}.class)));
    if (templates.isEmpty()) {
      throw new SwallowsServiceException("视图找不到该单据");
    }
    PC{template} template = templates.get(0);
    // 商品明细
    selectStatement = selectBuilder().from(PC{template}Detail.TABLE_NAME, "o")
        .where(Predicates.equals("o.num", num)).build();
    List<PC{template}Detail> details = jdbcTemplate.query(selectStatement,
        new HasCustomFieldRowMapper<>(PC{template}Detail.class, getCustomCols(PC{template}Detail.TABLE_NAME, PC{template}Detail.class)));
    template.setDetails(details);
    return PC{template}.TO_E.convert(template);
  }
```

**Scenario 2: Repository Does Not Exist** - Create complete `{template}Repository.java`

```java
/**
 * 版权所有(C)，XX有限公司，2026，所有权利保留。
 * <p>
 * 项目名： h6-openapi2
 * 文件名： {template}Repository.java
 * 模块说明：
 * 修改历史：
 * 2026年01月23日 - yanjun - 创建。
 */
package com.hd123.h6.openapi2.repo.{repoPackage};

import java.util.List;

import org.springframework.stereotype.Repository;

import com.hd123.h6.openapi2.model.{package}.callback.C{template};
import com.hd123.h6.openapi2.model.{package}.C{template};
import com.hd123.h6.openapi2.repo.BaseBillRepository;
import com.hd123.h6.openapi2.repo.bill.common.HasCustomFieldRowMapper;
import com.hd123.h6.openapi2.repo.{repoPackage}.PC{template};
import com.hd123.h6.openapi2.repo.{repoPackage}.PC{template}Detail;
import com.hd123.rumba.commons.jdbc.sql.Predicates;
import com.hd123.rumba.commons.jdbc.sql.SelectStatement;
import com.hd123.swallows.exception.SwallowsServiceException;

/**
 * @author yanjun
 * @since 1.0
 */
@Repository({template}Repository.DEFAULT_CONTEXT_ID)
public class {template}Repository extends BaseBillRepository {
  public static final String DEFAULT_CONTEXT_ID = "h6-openapi2-core.{template}Repository";

  public C{template} getByNumForCallBack(String num) throws SwallowsServiceException {
    SelectStatement selectStatement = selectBuilder().from(PC{template}.TABLE_NAME, "o")
        .where(Predicates.equals("o.num", num)).build();
    List<PC{template}> templates = jdbcTemplate.query(selectStatement, new HasCustomFieldRowMapper<>(
        PC{template}.class, getCustomCols(PC{template}.TABLE_NAME, PC{template}.class)));
    if (templates.isEmpty()) {
      throw new SwallowsServiceException("视图找不到该单据");
    }
    PC{template} template = templates.get(0);
    // 商品明细
    selectStatement = selectBuilder().from(PC{template}Detail.TABLE_NAME, "o")
        .where(Predicates.equals("o.num", num)).build();
    List<PC{template}Detail> details = jdbcTemplate.query(selectStatement,
        new HasCustomFieldRowMapper<>(PC{template}Detail.class, getCustomCols(PC{template}Detail.TABLE_NAME, PC{template}Detail.class)));
    template.setDetails(details);
    return PC{template}.TO_E.convert(template);
  }

  @Override
  protected String getMstTableName() {
    return PC{template}.TABLE_NAME;
  }

  @Override
  protected Class getPMstEntityClass() {
    return PC{template}.class;
  }

  @Override
  protected String getDtlTableName() {
    return PC{template}Detail.TABLE_NAME;
  }

  @Override
  protected Class getPDtlEntityClass() {
    return PC{template}Detail.class;
  }

}
```

### Template Variable Substitution

When generating files, replace following variables:
- `{genType}` - Always "callback"
- `{moduleId}` - The module ID in lower camel case (e.g., "alcDiff")
- `{package}` - The callback package name with dots (e.g., "bill.callback.alcdiff" or "bill.alcdiff")
- `{billType}` - The bill type for repository location (e.g., "pro", "dist", "sale")
- `{repoPackage}` - The repository package with dots, format: "bill.{billType}.{moduleId}" (e.g., "bill.dist.alcdiff")
- `{template}` - The template class name (e.g., "AlcDiff" - capitalized first letter of moduleId)
- Path conversion:
  - PC files: `{package}` → path (e.g., "bill.callback.alcdiff" → "bill/callback/alcdiff/")
  - Repository: `{repoPackage}` → path (e.g., "bill.dist.alcdiff" → "bill/dist/alcdiff/")
