---
name: h6-openapi2-create
description: This skill should be used when users need to generate h6-openapi2 project basic files (Model, Service, ServiceImpl, Repository, Persistence) based on genType=create, moduleId, and package parameters. It does NOT include Controller or Callback functionality.
---

# H6-OpenAPI2 Create Generator

This skill provides automated generation of basic h6-openapi2 project files based on genType=create, moduleId, and package parameters. It does NOT include Controller or Callback functionality.

## Purpose

Generate basic set of Java files required for a new business module in h6-openapi2 project, including:
- Persistence entity (P{template}) - based on existing {template}
- Detail Persistence entity (P{template}Detail) - based on existing {template}Detail
- Service interface
- Service implementation
- Repository interface
- DataProcessHandler update - adds event handling for new bill type

Note: Controller is NOT included in this generator.
Note: This skill assumes {template} and {template}Detail already exist and will read their fields to generate corresponding Persistence entities.

## When to Use

Use this skill when:
- {template} and {template}Detail classes already exist
- User specifies `genType=create` parameter
- User specifies `moduleId` parameter (e.g., "proOrder")
- User specifies `package` parameter (e.g., "bill.pro")
- User specifies `cls` parameter (e.g., "自营进货定单" - 单据类型枚举值)
- User wants to create Persistence entities and Service/Repository files
- User wants to update DataProcessHandler to add event handling for new bill type
- User does NOT need Callback functionality (use h6-openapi2-callback skill for that)

## File Generation Rules

### Parameters

When generating files, extract following parameters from user request:
- `genType` - Must be "create" to use this skill
- `moduleId` - Module ID in lower camel case (e.g., "proOrder")
- `package` - Package name (e.g., "bill.pro")
- `cls` - Bill type enum value (单据类型枚举值, e.g., "自营进货定单", etc.)
  - This is an enum value from `BillCls` class
  - Common values include: 批发, 批发退, 直配出退, 自营进货定单, 采购计划调整单, etc.
  - Refer to `BillCls.java` for all available enum values
- Calculate `template` by capitalizing first letter of `moduleId` (e.g., "proOrder" → "ProOrder")

### Naming Conventions

0. **BillCls Enum Update**: Modify `h6-openapi2-api/src/main/java/com/hd123/h6/openapi2/model/bill/BillCls.java`
   - Add `{cls}` enum value at the end of the enum declaration
   - Place it before the closing brace, after the last enum value
   - Example: add `测试单据新增,` at the end (before closing brace)

1. **Persistence Entity (P{template})**: Located in `h6-openapi2-core/src/main/java/com/hd123/h6/openapi2/repo/{package}/P{template}.java`
   - **Extends PBaseOperBill** (which already implements HasCustomField)
   - Uses `@Setter`, `@Getter` annotations
   - Includes `TABLE_NAME` constant (table name: `OpenApi2_{template}`)
   - Includes `TO_P` Converter mapping to Detail list
   - **Fields copied from existing {template} class**: Read {template}.java and copy all business fields (excluding inherited fields from BaseOperBill2 and details list)
   - Includes `@Transient List<P{template}Detail> details` field

2. **Detail Persistence Entity (P{template}Detail)**: Located in `h6-openapi2-core/src/main/java/com/hd123/h6/openapi2/repo/{package}/P{template}Detail.java`
   - **Check inheritance**: Read {template}Detail.java to check if it extends BaseBillDetail
   - **If extends BaseBillDetail** (has goods fields like goodsCode, inputCode, qpcStr, munit, etc.):
     - **Extends PBaseBillDetail** (which already implements HasCustomField)
     - Copy all business fields from {template}Detail (excluding inherited fields from BaseBillDetail)
   - **If does NOT extend BaseBillDetail** (no goods fields):
     - **Implements HasCustomField and Serializable directly**
     - Include uuid, ownerUuid, and customFields fields explicitly
     - Copy all business fields from {template}Detail
   - Uses `@Setter`, `@Getter` annotations
   - Includes `TABLE_NAME` constant (table name: `OpenApi2_{template}Dtl`)
   - Includes `TO_P` and `TO_P_List` Converters

3. **Service Interface**: Located in `h6-openapi2-api/src/main/java/com/hd123/h6/openapi2/service/{package}/{template}Service.java`
   - Defines `DEFAULT_CONTEXT_ID = "h6-openapi2-api.{template}Service"`
   - Methods: `BillResponse create({template} template, String sid)` and `void process(String uuid)`
   - Both methods throw `SwallowsServiceException` and `ParseException`

4. **Service Implementation**: Located in `h6-openapi2-core/src/main/java/com/hd123/h6/openapi2/core/{package}/{template}ServiceImpl.java`
   - Implements Service interface
   - Uses `@Service({template}Service.DEFAULT_CONTEXT_ID)`
   - Constructor injection of `{template}Repository`
   - `create()` method includes idempotency check, validation, and upload logic
   - `process()` method calls `{moduleId}Repository.process(uuid, null)`

5. **Repository Interface**: Located in `h6-openapi2-core/src/main/java/com/hd123/h6/openapi2/repo/{package}/{template}Repository.java`
   - Extends `BaseBillRepository`
   - Uses `@Repository({template}Repository.DEFAULT_CONTEXT_ID)`
   - Defines `DEFAULT_CONTEXT_ID = "h6-openapi2-core.{template}Repository"`
   - Overrides `getMstTableName()`, `getPMstEntityClass()`, `getDtlTableName()`, `getPDtlEntityClass()`
   - NO `getByNumForCallBack` method (use h6-openapi2-callback skill to add it)

6. **DataProcessHandler Update**: Modify `h6-openapi2-core/src/main/java/com/hd123/h6/openapi2/event/DataProcessHandler.java`
   - Add import: `import com.hd123.h6.openapi2.service.{package}.{template}Service;`
   - Add condition in `handle()` method:
     ```java
     } else if (BillCls.{cls}.name().equals(((DataProcessEvent) event).getCls())) {
       get{template}Service().process(((DataProcessEvent) event).getUuid());
     }
     ```
   - Add getter method at the end of class:
     ```java
     private {template}Service get{template}Service() {
       return ApplicationContextUtils.getBean({template}Service.class);
     }
     ```

### Package Structure

Replace `package` dots with path separators:
- `bill.pro` → `bill/pro/`

### Copyright Headers

All files must include copyright header with current year (2026) and creation date.

## Workflow

1. **Parse Parameters**
   - Extract `genType` from user request (must be "create")
   - Extract `moduleId` from user request (e.g., "proOrder")
   - Extract `package` from user request (e.g., "bill.pro")
   - Extract `cls` from user request (e.g., "自营进货定单", etc.)
   - Calculate `template` by capitalizing first letter of `moduleId` (e.g., "proOrder" → "ProOrder")

2. **Load Validation Rules**
   - Read `.codebuddy/skills/h6-openapi2-create/validation-rules/validation-rules.yaml` file
   - Parse rules for header_fields, detail_fields, field_aliases, table_descriptions, default_config
   - If rules file not found, use fallback to manual generation (add TODO comment)

3. **Read API Object Files**
   - Read `{template}.java` file from `h6-openapi2-api/src/main/java/com/hd123/h6/openapi2/model/{package}/`
   - Extract all header fields (excluding inherited fields from BaseOperBill2 and details list)
   - Read `{template}Detail.java` file
   - Extract all detail fields (excluding inherited fields from BaseBillDetail)

4. **Generate Validation Code**
   - For each header field, search for matching rule in validation-rules.yaml (including aliases)
   - Generate `validateCodeExists` calls for matched header fields
   - For fields with `require_org_match: true`, generate `validateCodeMatchOrg` calls (if orgCode exists)
   - For each detail field, generate `validateCodeExists` calls for details list
   - Organize generated code into two sections: code existence validation and org matching validation

5. **Generate Files**
   - Replace variables in code templates
   - Insert generated validation code into ServiceImpl template (replace placeholders)
   - Replace `package` dots with path separators
   - Generate 5 files:
     1. P{template}.java (Persistence entity)
     2. P{template}Detail.java (Detail persistence entity)
     3. {template}Service.java (Service interface)
     4. {template}ServiceImpl.java (Service implementation with generated validation)
     5. {template}Repository.java (Repository interface)

6. **Update BillCls Enum**
   - Read BillCls.java file
   - Add `{cls},` at the end of the enum declaration (before the closing brace)

7. **Validate**
   - Check that all files are written to correct locations
   - Verify package declarations are correct
   - Ensure proper inheritance and annotations
   - Verify validation code is properly generated and formatted

8. **Update DataProcessHandler**
   - Read DataProcessHandler.java file
   - Add import statement for {template}Service
   - Add else-if condition in handle() method for BillCls.{cls}
   - Add getter method for {template}Service

9. **Git Add Files**
   - After successful file generation, execute `git add` command for each generated file
   - Command pattern: `git add <full-path-to-file>`
   - This ensures generated files are staged for commit

10. **Report**
   - List all generated files with full paths
   - Confirm that BillCls.java has been updated with {cls} enum value
   - Confirm that DataProcessHandler.java has been updated
   - Confirm that validation code was generated based on API fields
   - Confirm that all files have been added to git staging area

## Example Usage

User request: "genType=create, moduleId=proOrder, package=bill.pro, cls=自营进货定单"

**Step 1: Parse Parameters**
- `genType` = "create"
- `moduleId` = "proOrder"
- `package` = "bill.pro"
- `cls` = "自营进货定单"
- `template` = "ProOrder"

**Step 2: Generate Files**
Generate 5 files:
1. `h6-openapi2-core/.../repo/bill/pro/PProOrder.java` (based on ProOrder fields)
2. `h6-openapi2-core/.../repo/bill/pro/PProOrderDetail.java` (based on ProOrderDetail fields)
3. `h6-openapi2-api/.../service/bill/pro/ProOrderService.java`
4. `h6-openapi2-core/.../core/bill/pro/ProOrderServiceImpl.java`
5. `h6-openapi2-core/.../repo/bill/pro/ProOrderRepository.java`

**Step 3: Update BillCls Enum**
Modify `BillCls.java` to add new enum value:
- Add `自营进货定单,` at the end of enum declaration (before closing brace)

**Step 4: Update DataProcessHandler**
Modify `DataProcessHandler.java` to add event handling for the new bill type:
1. Add import statement for {template}Service
2. Add else-if condition in handle() method
3. Add getter method for {template}Service

## Code Templates

### Persistence Entity (P{template}) Template (based on existing {template} class)

**NOTE**: Read the existing {template}.java file and copy all business fields (excluding inherited fields from BaseOperBill2 and details list) to generate this class.

```java
/**
 * 版权所有(C)，XX有限公司，2026，所有权利保留。
 * <p>
 * 项目名： h6-openapi2
 * 文件名： P{template}.java
 * 模块说明：
 * 修改历史：
 * 2026年01月08日 - yanjun - 创建。
 */
package com.hd123.h6.openapi2.repo.{package};

import java.util.ArrayList;
import java.util.List;

import com.hd123.h6.openapi2.model.{package}.{template};
import com.hd123.h6.openapi2.repo.bill.common.PBaseOperBill;
import com.hd123.rumba.commons.util.converter.Converter;
import com.hd123.rumba.commons.util.converter.ConverterBuilder;
import com.hd123.swallows.api.entity.Transient;

import lombok.Getter;
import lombok.Setter;

/**
 * @author yanjun
 * @since 1.0
 */
@Getter
@Setter
public class P{template} extends PBaseOperBill {

  private static final long serialVersionUID = -3598170392114626840L;

  public static final String TABLE_NAME = "OpenApi2_{template}";

  public static Converter<{template}, P{template}> TO_P = ConverterBuilder
      .newBuilder({template}.class, P{template}.class).map("details", P{template}Detail.TO_P_List)
      .build();

  // ===== Fields copied from {template}.java =====
  // Example fields (read actual {template} class and copy all business fields here)
  private String orgCode;
  private String vendorCode;
  private String receiverCode;
  private String wrhCode;
  // Add all other business fields from {template}...
  // ============================================

  @Transient
  private List<P{template}Detail> details = new ArrayList<>();
}
```

### Service Interface Template
```java
/**
 * 版权所有(C)，XX有限公司，2026，所有权利保留。
 * <p>
 * 项目名： h6-openapi2
 * 文件名： {template}Service.java
 * 模块说明：
 * 修改历史：
 * 2026年01月08日 - yanjun - 创建。
 */
package com.hd123.h6.openapi2.service.{package};

import java.text.ParseException;

import com.hd123.h6.openapi2.model.bill.BillResponse;
import com.hd123.h6.openapi2.model.{package}.{template};
import com.hd123.swallows.exception.SwallowsServiceException;

/**
 * @author yanjun
 *
 */
public interface {template}Service {
  String DEFAULT_CONTEXT_ID = "h6-openapi2-api.{template}Service";

  BillResponse create({template} {moduleId}, String sid)
      throws SwallowsServiceException, ParseException;

  void process(String uuid) throws SwallowsServiceException;

}
```

### Service Implementation Template

**IMPORTANT**: The validation code in this template is automatically generated based on `validation-rules.yaml`. The `{GENERATED_CODE_EXISTS_VALIDATION}` and `{GENERATED_ORG_MATCH_VALIDATION}` placeholders below will be replaced with actual validation code during generation.

**Validation Generation Process**:
1. Read {template}.java and {template}Detail.java to extract all fields
2. Match fields against rules in `.codebuddy/skills/h6-openapi2-create/validation-rules/validation-rules.yaml`
3. Generate validateCodeExists calls for matched fields
4. Generate validateCodeMatchOrg calls for fields requiring org matching
5. If rules file is not found, generate with TODO comment for manual adjustment

**For detailed configuration and customization of validation rules**, refer to `validation-rules-guide.md` in the `validation-rules/` subdirectory.

```java
/**
 * 版权所有(C)，XX有限公司，2026，所有权利保留。
 * <p>
 * 项目名： h6-openapi2
 * 文件名： {template}ServiceImpl.java
 * 模块说明：
 * 修改历史：
 * 2026年01月08日 - yanjun - 创建。
 */
package com.hd123.h6.openapi2.core.{package};

import java.text.ParseException;
import java.util.Collections;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import com.hd123.h6.openapi2.model.bill.BillResponse;
import com.hd123.h6.openapi2.model.{package}.{template};
import com.hd123.h6.openapi2.repo.{package}.{template}Repository;
import com.hd123.h6.openapi2.service.{package}.{template}Service;
import com.hd123.rumba.commons.lang.StringUtil;
import com.hd123.swallows.exception.SwallowsServiceException;

/**
 * @author yanjun
 * @since 1.0
 */
@Service({template}Service.DEFAULT_CONTEXT_ID)
public class {template}ServiceImpl implements {template}Service {

  @Autowired
  private {template}Repository {moduleId}Repository;

  @Override
  public BillResponse create({template} {moduleId}, String sid)
      throws SwallowsServiceException, ParseException {
    // 幂等校验，如果已发送过请求，返回H6单号给调用方。
    String genNum = {moduleId}Repository.getGenNum({moduleId}.getPartnerNum(), sid);
    if (!StringUtil.isNullOrBlank(genNum)) {
      BillResponse billResponse = new BillResponse();
      billResponse.setNum(genNum.equals("NA") ? null : genNum);
      return billResponse;
    }

    // ===== 代码存在性验证 =====
    // 此部分代码根据 validation-rules.yaml 自动生成
    // 读取 {template}.java 和 {template}Detail.java 的字段，匹配规则后生成验证代码
    StringBuffer sb = new StringBuffer();
    {GENERATED_CODE_EXISTS_VALIDATION}
    if (sb.length() != 0) {
      throw new SwallowsServiceException(sb.toString());
    }

    // ===== 组织匹配验证 =====
    // 对于 require_org_match=true 的字段，自动生成组织匹配验证
    {GENERATED_ORG_MATCH_VALIDATION}
    if (sb.length() != 0) {
      throw new SwallowsServiceException(sb.toString());
    }
    // =====================

    return {moduleId}Repository.upload({moduleId}, sid);
  }

  @Override
  public void process(String uuid) throws SwallowsServiceException {
    {moduleId}Repository.process(uuid, BillCls.{cls}.name());
  }
}
```

**Placeholder Explanation**:
- `{GENERATED_CODE_EXISTS_VALIDATION}`: Will be replaced with `validateCodeExists` calls for header and detail fields
- `{GENERATED_ORG_MATCH_VALIDATION}`: Will be replaced with `validateCodeMatchOrg` calls for fields requiring org matching

**Generated Code Example** (if API has orgCode, storeCode, vendorCode, wrhCode):

```java
// 代码存在性验证
StringBuffer sb = new StringBuffer();
sb.append({moduleId}Repository.validateCodeExists(Collections.singletonList({moduleId}), "Store",
    "Code", "orgCode", "组织代码"));
sb.append({moduleId}Repository.validateCodeExists(Collections.singletonList({moduleId}), "Store",
    "Code", "storeCode", "门店代码"));
sb.append({moduleId}Repository.validateCodeExists(Collections.singletonList({moduleId}), "Vendor",
    "Code", "vendorCode", "供应商代码"));
sb.append({moduleId}Repository.validateCodeExists(Collections.singletonList({moduleId}), "WareHouse",
    "Code", "wrhCode", "仓位代码"));
if (sb.length() != 0) {
  throw new SwallowsServiceException(sb.toString());
}

// 组织匹配验证
sb.append({moduleId}Repository.validateCodeMatchOrg({moduleId}.getStoreCode(),
    {moduleId}.getOrgCode(), "Store", "Code", "门店代码"));
sb.append({moduleId}Repository.validateCodeMatchOrg({moduleId}.getVendorCode(),
    {moduleId}.getOrgCode(), "Vendor", "Code", "供应商代码"));
sb.append({moduleId}Repository.validateCodeMatchOrg({moduleId}.getWrhCode(),
    {moduleId}.getOrgCode(), "WareHouse", "Code", "仓位代码"));
if (sb.length() != 0) {
  throw new SwallowsServiceException(sb.toString());
}
```

### Persistence Entity Template
```java
/**
 * 版权所有(C)，XX有限公司，2026，所有权利保留。
 * <p>
 * 项目名： h6-openapi2
 * 文件名： P{template}.java
 * 模块说明：
 * 修改历史：
 * 2026年01月08日 - yanjun - 创建。
 */
package com.hd123.h6.openapi2.repo.{package};

import java.util.ArrayList;
import java.util.List;

import com.hd123.h6.openapi2.model.{package}.{template};
import com.hd123.h6.openapi2.repo.bill.common.PBaseOperBill;
import com.hd123.rumba.commons.util.converter.Converter;
import com.hd123.rumba.commons.util.converter.ConverterBuilder;
import com.hd123.swallows.api.entity.Transient;

import lombok.Getter;
import lombok.Setter;

/**
 * @author yanjun
 * @since 1.0
 */
@Getter
@Setter
public class P{template} extends PBaseOperBill {

  private static final long serialVersionUID = -3598170392114626840L;
  public static final String TABLE_NAME = "OpenApi2_{template}";
  public static Converter<{template}, P{template}> TO_P = ConverterBuilder
      .newBuilder({template}.class, P{template}.class).map("details", P{template}Detail.TO_P_List)
      .build();

  private String orgCode;
  private String storeCode;
  private String wrhCode;
  private String vendorCode;

  @Transient
  private List<P{template}Detail> details = new ArrayList<>();
}
```

### Detail Persistence Entity (P{template}Detail) Template (based on existing {template}Detail class)

**IMPORTANT**: Check whether {template}Detail extends BaseBillDetail (has goods fields) or not.

**If {template}Detail EXTENDS BaseBillDetail** (has goodsCode, inputCode, qpcStr, munit, price, priceType, etc.):

```java
/**
 * 版权所有(C)，XX有限公司，2026，所有权利保留。
 * <p>
 * 项目名： h6-openapi2
 * 文件名： P{template}Detail.java
 * 模块说明：
 * 修改历史：
 * 2026年01月08日 - yanjun - 创建。
 */
package com.hd123.h6.openapi2.repo.{package};

import com.hd123.h6.openapi2.model.{package}.{template}Detail;
import com.hd123.h6.openapi2.repo.bill.common.PBaseBillDetail;
import com.hd123.rumba.commons.util.converter.ArrayListConverter;
import com.hd123.rumba.commons.util.converter.Converter;
import com.hd123.rumba.commons.util.converter.ConverterBuilder;

import lombok.Getter;
import lombok.Setter;

/**
 * @author yanjun
 * @since 1.0
 */
@Getter
@Setter
public class P{template}Detail extends PBaseBillDetail {

  private static final long serialVersionUID = 5506892245780880222L;

  public static final String TABLE_NAME = "OpenApi2_{template}Dtl";

  public static Converter<{template}Detail, P{template}Detail> TO_P = ConverterBuilder
      .newBuilder({template}Detail.class, P{template}Detail.class).map("uuid", "srcLineUuid").build();
  public static ArrayListConverter<{template}Detail, P{template}Detail> TO_P_List = ArrayListConverter
      .newConverter(TO_P);

  // ===== Fields copied from {template}Detail.java =====
  // Example fields (read actual {template}Detail class and copy all business fields here)
  private Integer goodsType;
  private String qpcStr;
  private String munit;
  private BigDecimal qty;
  private BigDecimal price;
  private Integer priceType;
  // Add all other business fields from {template}Detail...
  // ============================================
}
```

**If {template}Detail DOES NOT EXTEND BaseBillDetail** (no goods fields):

```java
/**
 * 版权所有(C)，XX有限公司，2026，所有权利保留。
 * <p>
 * 项目名： h6-openapi2
 * 文件名： P{template}Detail.java
 * 模块说明：
 * 修改历史：
 * 2026年01月08日 - yanjun - 创建。
 */
package com.hd123.h6.openapi2.repo.{package};

import java.io.Serializable;
import java.math.BigDecimal;
import java.util.Map;

import com.hd123.h6.openapi2.customfield.HasCustomField;
import com.hd123.h6.openapi2.model.{package}.{template}Detail;
import com.hd123.rumba.commons.util.converter.ArrayListConverter;
import com.hd123.rumba.commons.util.converter.Converter;
import com.hd123.rumba.commons.util.converter.ConverterBuilder;

import lombok.Getter;
import lombok.Setter;

/**
 * @author yanjun
 * @since 1.0
 */
@Getter
@Setter
public class P{template}Detail implements HasCustomField, Serializable {

  private static final long serialVersionUID = 5506892245780880222L;

  public static final String TABLE_NAME = "OpenApi2_{template}Dtl";

  public static Converter<{template}Detail, P{template}Detail> TO_P = ConverterBuilder
      .newBuilder({template}Detail.class, P{template}Detail.class).map("uuid", "srcLineUuid").build();
  public static ArrayListConverter<{template}Detail, P{template}Detail> TO_P_List = ArrayListConverter
      .newConverter(TO_P);

  private String uuid;
  private String ownerUuid;
  // ===== Fields copied from {template}Detail.java =====
  // Example fields (read actual {template}Detail class and copy all business fields here)
  private Integer line;
  private String chgNum;
  private BigDecimal payTotal;
  private String note;
  // Add all other business fields from {template}Detail...
  // ============================================
  private Map<String, Object> customFields;
}
```

### Repository Template
```java
/**
 * 版权所有(C)，XX有限公司，2026，所有权利保留。
 * <p>
 * 项目名： h6-openapi2
 * 文件名： {template}Repository.java
 * 模块说明：
 * 修改历史：
 * 2026年01月08日 - yanjun - 创建。
 */
package com.hd123.h6.openapi2.repo.{package};

import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Repository;

import com.hd123.h6.openapi2.event.DataProcessEvent;
import com.hd123.h6.openapi2.model.bill.BillResponse;
import com.hd123.h6.openapi2.model.{package}.{template};
import com.hd123.h6.openapi2.repo.BaseBillRepository;
import com.hd123.h6.openapi2.util.customfield.CustomFieldUtil;
import com.hd123.swallows.cqrs.event.SwallowsEventBus;
import com.hd123.swallows.dao.SwallowsTX;
import com.hd123.swallows.exception.SwallowsServiceException;
import com.hd123.swallows.util.UUIDGenerator;

/**
 * @author yanjun
 * @since 1.0
 */
@Repository({template}Repository.DEFAULT_CONTEXT_ID)
public class {template}Repository extends BaseBillRepository {
  public static final String DEFAULT_CONTEXT_ID = "h6-openapi2-core.{template}Repository";

  @Autowired
  private SwallowsEventBus eventBus;

  @SwallowsTX
  public BillResponse upload({template} {moduleId}, String sid) throws SwallowsServiceException {
    P{template} p{moduleId} = P{template}.TO_P.convert({moduleId});
    p{moduleId}.setSid(sid);
    buildBaseEntity(p{moduleId});
    // 插入单头
    insert(P{template}.TABLE_NAME, CustomFieldUtil.unWrap(p{moduleId}, getMstCustomCols()));
    // 插入明细
    List<String> dtlCustomCols = getDtlCustomCols();
    List<Map<String, Object>> dtlMapLst = p{moduleId}.getDetails().stream().map(dtl -> {
      dtl.setUuid(UUIDGenerator.getUUID());
      dtl.setOwnerUuid(p{moduleId}.getUuid());
      return CustomFieldUtil.unWrap(dtl, dtlCustomCols);
    }).collect(Collectors.toList());

    batchInsert(P{template}Detail.TABLE_NAME, dtlMapLst);
    // 校验
    checkBeforeUpload(p{moduleId}.getUuid());
    BillResponse billResponse = new BillResponse();
    // 发布事件
    if (asynProcess("{moduleId}", sid)) {
      eventBus.publish(new DataProcessEvent(p{moduleId}.getUuid(), BillCls.{cls}.name()));
    } else {
      process(p{moduleId}.getUuid(), BillCls.{cls}.name());
      billResponse.setNum(getGenNum(p{moduleId}.getPartnerNum(), sid));
    }
    return billResponse;
  }

  @Override
  protected String getMstTableName() {
    return P{template}.TABLE_NAME;
  }

  @Override
  protected Class getPMstEntityClass() {
    return P{template}.class;
  }

  @Override
  protected String getDtlTableName() {
    return P{template}Detail.TABLE_NAME;
  }

  @Override
  protected Class getPDtlEntityClass() {
    return P{template}Detail.class;
  }

}
```

### Template Variable Substitution

When generating files, replace following variables:
- `{genType}` - Always "create"
- `{moduleId}` - The module ID in lower camel case (e.g., "proOrder")
- `{package}` - The package name with dots (e.g., "bill.pro")
- `{template}` - The template class name (e.g., "ProOrder" - capitalized first letter of moduleId)
- `{cls}` - The bill type enum value (单据类型枚举值, e.g., "自营进货定单", etc.)

### DataProcessHandler Update Template

**Note**: This is NOT a new file to create, but an update to the existing `DataProcessHandler.java` file.

#### 1. Add Import Statement

Find the import section (after package declaration) and add:
```java
import com.hd123.h6.openapi2.service.{package}.{template}Service;
```

Place this import alongside other service imports, typically in alphabetical order or by category.

#### 2. Add Condition in handle() Method

In the `handle()` method, find an appropriate location (preferably before the last else-if block) and add:
```java
} else if (BillCls.{cls}.name().equals(((DataProcessEvent) event).getCls())) {
  get{template}Service().process(((DataProcessEvent) event).getUuid());
}
```

#### 3. Add Getter Method

At the end of the class (before the closing brace), add:
```java
private {template}Service get{template}Service() {
  return ApplicationContextUtils.getBean({template}Service.class);
}
```

#### Example for TestBill:

**Import (add to import section):**
```java
import com.hd123.h6.openapi2.service.bill.pro.TestBillService;
```

**Condition in handle() method:**
```java
} else if (BillCls.测试单据新增.name().equals(((DataProcessEvent) event).getCls())) {
  getTestBillService().process(((DataProcessEvent) event).getUuid());
}
```

**Getter method:**
```java
private TestBillService getTestBillService() {
  return ApplicationContextUtils.getBean(TestBillService.class);
}
```

### BillCls Enum Update Template

**Note**: This is NOT a new file to create, but an update to the existing `BillCls.java` file.

#### 1. Add Enum Value at End

Find the end of the enum declaration (before the closing brace and semicolon) and add `{cls},`:

```java
  // ... existing enum values ...
  委外加工发货, {cls};
```

**Example for TestBill:**
```java
  // ... existing enum values ...
  委外加工发货, 测试单据新增;
```

**Important Notes:**
- Add the new enum value before the last closing brace
- Ensure there is a comma after the new enum value (before the closing brace)
- The `{cls}` parameter will be replaced with the actual enum value provided by the user

## Validation Rules System

### Overview

The validation rules system enables automatic generation of validation code in ServiceImpl based on API object fields. This eliminates the need for manual TODO comments and hard-coded validation logic.

### Rule File Location

Validation rules are defined in: `.codebuddy/skills/h6-openapi2-create/validation-rules/validation-rules.yaml`

### Rule File Structure

```yaml
version: "1.0"

# Header field validation rules
header_fields:
  orgCode:
    table: Store
    column: Code
    description: 组织代码
    require_org_match: false
  storeCode:
    table: Store
    column: Code
    description: 门店代码
    require_org_match: true
    org_match_field: orgCode

# Detail field validation rules
detail_fields:
  goodsCode:
    table: Goods
    column: Code
    description: 商品代码
    require_org_match: false

# Field name aliases
field_aliases:
  vendor: vendorCode
  store_code: storeCode

# Default configuration
default_config:
  include_todo: false
  auto_generate_org_match: true
  detail_prefix: 明细
```

### Field Matching Logic

1. **Direct Match**: Check if field name exists in header_fields or detail_fields
2. **Alias Match**: If not found, check field_aliases mapping
3. **Case Sensitive**: Field names are case-sensitive

### Validation Code Generation

For each matched field, generate one or more of the following:

#### 1. Code Existence Validation

```java
sb.append({moduleId}Repository.validateCodeExists(Collections.singletonList({moduleId}), "{table}",
    "{column}", "{fieldName}", "{description}"));
```

#### 2. Organization Matching Validation

Generated when `require_org_match: true` and orgCode field exists:

```java
sb.append({moduleId}Repository.validateCodeMatchOrg({moduleId}.get{FieldName}(),
    {moduleId}.getOrgCode(), "{table}", "{column}", "{description}"));
```

#### 3. Detail Validation

For detail fields, the target is `getDetails()` collection:

```java
sb.append({moduleId}Repository.validateCodeExists({moduleId}.getDetails(), "{table}",
    "{column}", "{fieldName}", "{detail_prefix}{description}"));
```

### Complete Generation Example

**Input API Object (TestBill.java):**
```java
private String orgCode;
private String vendorCode;
private String senderCode;
private String wrhCode;
private List<TestBillDetail> details;
```

**Input Detail Object (TestBillDetail.java):**
```java
private String qpcStr;
private String munit;
private BigDecimal qty;
private BigDecimal price;
private Integer priceType;
```

**Matched Rules:**
- orgCode → Store.Code, require_org_match=false
- vendorCode → Vendor.Code, require_org_match=true (needs orgCode)
- senderCode → Store.Code, require_org_match=false
- wrhCode → WareHouse.Code, require_org_match=true (needs orgCode)
- No detail fields match (qpcStr, munit, qty, price, priceType not in rules)

**Generated Validation Code:**
```java
// 代码存在性验证
StringBuffer sb = new StringBuffer();
sb.append(testBillRepository.validateCodeExists(Collections.singletonList(testBill), "Store",
    "Code", "orgCode", "组织代码"));
sb.append(testBillRepository.validateCodeExists(Collections.singletonList(testBill), "Vendor",
    "Code", "vendorCode", "供应商代码"));
sb.append(testBillRepository.validateCodeExists(Collections.singletonList(testBill), "Store",
    "Code", "senderCode", "出库单位代码"));
sb.append(testBillRepository.validateCodeExists(Collections.singletonList(testBill), "WareHouse",
    "Code", "wrhCode", "仓位代码"));
if (sb.length() != 0) {
  throw new SwallowsServiceException(sb.toString());
}

// 组织匹配验证
sb.append(testBillRepository.validateCodeMatchOrg(testBill.getVendorCode(),
    testBill.getOrgCode(), "Vendor", "Code", "供应商代码"));
sb.append(testBillRepository.validateCodeMatchOrg(testBill.getWrhCode(),
    testBill.getOrgCode(), "WareHouse", "Code", "仓位代码"));
if (sb.length() != 0) {
  throw new SwallowsServiceException(sb.toString());
}
```

### Commonly Used Fields

| Field Name | Table | Column | Description | Require Org Match |
|------------|-------|--------|-------------|------------------|
| orgCode | Store | Code | 组织代码 | No |
| storeCode | Store | Code | 门店代码 | Yes |
| vendorCode | Vendor | Code | 供应商代码 | Yes |
| wrhCode | WareHouse | Code | 仓位代码 | Yes |
| senderCode | Store | Code | 出库单位代码 | No |
| receiverCode | Store | Code | 收货单位代码 | No |
| purOrgCode | Store | Code | 采购组织代码 | No |
| saleOrgCode | Store | Code | 销售组织代码 | No |
| goodsCode (detail) | Goods | Code | 商品代码 | No |
| inputCode (detail) | GdInput | Code | 商品输入码 | No |

### Adding Custom Validation Rules

To add a new field validation rule:

1. Edit `.codebuddy/skills/h6-openapi2-create/validation-rules/validation-rules.yaml`
2. Add field to `header_fields` or `detail_fields` section:

```yaml
header_fields:
  customField:
    table: CustomTable
    column: CustomColumn
    description: 自定义字段
    require_org_match: true
    org_match_field: orgCode
```

3. Add field alias if needed:

```yaml
field_aliases:
  custom_field: customField
```

### Fallback Behavior

If `.codebuddy/skills/h6-openapi2-create/validation-rules/validation-rules.yaml` is not found or cannot be read, the system will:
1. Generate basic validation code with TODO comment
2. Include placeholder for all common fields (orgCode, storeCode, vendorCode, wrhCode)
3. Output warning message in the report

### Troubleshooting

**Issue**: Validation code not generated
- Check that `.codebuddy/skills/h6-openapi2-create/validation-rules/validation-rules.yaml` exists
- Verify YAML syntax is correct
- Check that field names match exactly (case-sensitive)

**Issue**: Some fields missing validation
- Add field to rules file or create alias mapping
- Verify field name spelling in API object

**Issue**: Org match validation missing
- Ensure `require_org_match: true` is set in rule
- Verify orgCode field exists in API object
- Check that `org_match_field` points to correct field name

### Related Documentation

- See `.codebuddy/skills/h6-openapi2-create/validation-rules/validation-rules-guide.md` for detailed rule file format
- See `.codebuddy/skills/h6-openapi2-create/validation-rules/validation-rules-integration.md` for integration details

## Oracle Script Generation

### Overview

The skill can also generate Oracle stored procedure scripts to support database-level processing of the new business module. This includes:
- Updating `PPS_OpenApi2_Pkg.oracle.sql` to add process routing
- Updating `PPS_OpenApi2_Pkg.oracle.sql` to add before-upload validation
- Generating `PPS_OpenApi2_*.sql` file with ProcessSingle and Check functions

### Oracle Package Mapping

Based on the package parameter, the Oracle package name is determined as follows:

| Package | Oracle Package | Target File |
|---------|----------------|-------------|
| `bill.pro` | `PPS_OpenApi2_Pro` | `PPS_OpenApi2_Pro_Pkg.oracle.sql` |
| `bill.sale` | `PPS_OpenApi2_Sale` | `PPS_OpenApi2_Sale_Pkg.oracle.sql` |
| `bill.account` | `PPS_OpenApi2_Account` | `PPS_OpenApi2_Account_Pkg.oracle.sql` |
| `bill.wrh` | `PPS_OpenApi2_Wrh` | `PPS_OpenApi2_Wrh_Pkg.oracle.sql` |
| `bill.dist` | `PPS_OpenApi2_Dist` | `PPS_OpenApi2_Dist_Pkg.oracle.sql` |
| `bill.wms` | `PPS_OpenApi2_Wms` | `PPS_OpenApi2_Wms_Pkg.oracle.sql` |
| `mdata` | `PPS_OpenApi2_MData` | `PPS_OpenApi2_MData_Pkg.oracle.sql` |
| `bill.wms` | `PPS_OpenApi2_Wms` | `PPS_OpenApi2_Wms_Pkg.oracle.sql` |

### Step 1: Update PPS_OpenApi2_Pkg.oracle.sql

**File Location**: `h6-openapi2-rdb-setup/src/main/resources/META-INF/h6-openapi2-setup/script/oracle/sp/PPS_OpenApi2_Pkg.oracle.sql`

**Target**: Process function

**Add the following code block** (before the closing End if of Process function):

```sql
      ElsIf piCls = {cls} Then
        v_TableName := 'OpenApi2_{template}';
        v_Ret := {oracle_package}.ProcessSingle{template}(piUuid, poErr_Msg);
        If v_Ret <> 0 Then
          Rollback;
          AddLog(v_TableName, piUuid, 9, poErr_Msg);
        Return v_Ret;
        End If;
      End If;
```

**Example** (for TestBill with package=bill.pro):
```sql
      ElsIf piCls = 测试单据新增 Then
        v_TableName := 'OpenApi2_TestBill';
        v_Ret := PPS_OpenApi2_Pro.ProcessSingleTestBill(piUuid, poErr_Msg);
        If v_Ret <> 0 Then
          Rollback;
          AddLog(v_TableName, piUuid, 9, poErr_Msg);
        Return v_Ret;
        End If;
      End If;
```

### Step 2: Update PPS_OpenApi2_Pkg.oracle.sql (CheckBeforeUpload)

**File Location**: `h6-openapi2-rdb-setup/src/main/resources/META-INF/h6-openapi2-setup/script/oracle/sp/PPS_OpenApi2_Pkg.oracle.sql`

**Target**: CheckBeforeUpload function

**Add the following code block** (before the Return v_Ret statement):

```sql
    ElsIf piTable = 'OpenApi2_{template}' Then
      v_Ret := {oracle_package}.Check{template}(piUuid, poErr_Msg);
      If v_Ret <> 0 Then Return(v_Ret); End If;
    End If;
```

**Example** (for TestBill):
```sql
    ElsIf piTable = 'OpenApi2_TestBill' Then
      v_Ret := PPS_OpenApi2_Pro.CheckTestBill(piUuid, poErr_Msg);
      If v_Ret <> 0 Then Return(v_Ret); End If;
    End If;
```

### Step 3: Generate PPS_OpenApi2_*_Pkg.oracle.sql

**File Location**: `h6-openapi2-rdb-setup/src/main/resources/META-INF/h6-openapi2-setup/script/oracle/sp/{oracle_package}_Pkg.oracle.sql`

**Generate two functions**:

#### 3.1 Check Function Template

**IMPORTANT**: Check whether {template}Detail has goods fields (goodsCode, inputCode, qpcStr, munit, price, priceType, etc.) or not.

**If {template}Detail HAS goods fields** (extends BaseBillDetail):

```sql
  Function Check{template}
  (
    piUuid         In   Varchar2, --主键
    poErr_Msg      Out  Varchar2  --错误消息
  ) Return Number
    Is
    v_Ret Int;
    v_Count Int;
    Cursor CDtl Is Select * From OpenApi2_{template}Dtl Where OwnerUuid = piUuid Order By Line;
  Begin
    v_Ret := 0;
    Select Count(1) Into v_Count From OpenApi2_{template}Dtl Where OwnerUuid = piUuid;
    If v_Count = 0 Then
      poErr_Msg := '待接收的单据没有明细记录，请检查数据';
      Return (1);
    End If;

    For RDtl In CDtl Loop
      Select Gid Into v_GdGid From GdInput Where Code = RDtl.GoodsCode;
      --校验代码和输入码是否匹配
      If Is_Not_Null(RDtl.InputCode) = 1 Then
        Select Count(1) into v_Count From GDInput Where Code = RDtl.InputCode and Gid = v_GdGid;
        If v_Count = 0 then
          poErr_Msg := '商品输入码' || RDtl.InputCode || '与商品代码'|| RDtl.GOODSCODE ||'不匹配';
          Return (1);
        End If;
      End If;
      --校验包装规格、单位
      If Is_Not_Null(RDtl.QpcStr) = 1 Then
        Select Count(1) Into v_Count From GdQpc Where Gid = v_GdGid And QpcStr = RDtl.QpcStr;
        If v_Count = 0 Then
          poErr_Msg := '第' || RDtl.Line || '行，商品代码' || RDtl.GoodsCode || '不存在此规格' || RDtl.QpcStr || '请检查!';
          Return (1);
        End If;
      End If;
      If Is_Not_Null(RDtl.Munit) = 1 Then
        Select Count(1) Into v_Count From GdQpc Where Gid = v_GdGid And Munit = RDtl.Munit;
        If v_Count = 0 Then
          poErr_Msg := '第' || RDtl.Line || '行，商品代码' || RDtl.GoodsCode || '不存在此单位' || RDtl.Munit || '请检查!';
          Return (1);
        End If;
      End If;
      If Is_Not_Null(RDtl.QpcStr) = 1 And Is_Not_Null(RDtl.Munit) = 1 Then
        Select Count(1) Into v_Count From GdQpc Where Gid = v_GdGid And QpcStr = RDtl.QpcStr And Munit = RDtl.Munit;
        If v_Count <= 0 Then
          poErr_Msg := '第' || RDtl.Line || '行，商品代码' || RDtl.GoodsCode || '规格' || RDtl.QpcStr || '和计量单位' || RDtl.Munit || '不匹配，请检查!';
          Return (1);
        End If;
      End If;
      --校验价格和价格类型
      If RDtl.priceType = 1 And (Is_Null(RDtl.qpcStr) = 1 Or Is_Null(RDtl.Munit) = 1) Then
        poErr_Msg := '当价格类型为规格价时，包装规格描述和计量单位必传';
        Return (1);
      End if;
      If Is_Null(RDtl.Price) = 1 And Is_Not_Null(RDtl.priceType) = 1 Then
        poErr_Msg := '指定了价格类型就必须指定价格';
        Return (1);
      End if;
      If Is_Null(RDtl.priceType) = 1 And Is_Not_Null(RDtl.Price) = 1 Then
        poErr_Msg := '指定了价格就必须指定价格类型';
        Return (1);
      End if;
    End Loop;
    Return (v_Ret);
  End;
```

**If {template}Detail DOES NOT HAVE goods fields** (does NOT extend BaseBillDetail):

```sql
  Function Check{template}
  (
    piUuid         In   Varchar2, --主键
    poErr_Msg      Out  Varchar2  --错误消息
  ) Return Number
    Is
    v_Ret Int;
    v_Count Int;
  Begin
    v_Ret := 0;
    Select Count(1) Into v_Count From OpenApi2_{template}Dtl Where OwnerUuid = piUuid;
    If v_Count = 0 Then
      poErr_Msg := '待接收的单据没有明细记录，请检查数据';
      Return (1);
    End If;

    -- TODO: 根据实际业务需求添加校验逻辑

    Return (v_Ret);
  End;
```

#### 3.2 ProcessSingle Function Template

**IMPORTANT: The following sections MUST be generated for ALL bill types (with or without goods fields):**
- Lines 1169-1170: Variable declarations for v_BillInfo and v_Details
- Lines 1210-1220: Core business logic (GenBill, configuration, state update, update GenNum, log success)

**Template**:

```sql
  Function ProcessSingle{template}
  (
    piUuid In Varchar2,
    poErr_Msg Out Varchar2
  ) Return Number
    Is
    v_Ret Int;
    v_Line Int;
    v_Oper Varchar2(30);
    v_BillInfo PCU_{template}.BillInfo;
    v_Details PCU_{template}.Details;
    v_Num Typedef.TBillNum%Type;
    C_TABLENAME CONSTANT VARCHAR2(255) := 'OpenApi2_{template}';
    v_Sid OpenApi2_{template}.Sid%Type;
    Cursor CDtl Is Select * From OpenApi2_{template}Dtl Where OwnerUuid = piUuid Order By Line Asc;
  Begin
    v_Ret := 0;
    -- 数据校验
    v_Ret := Check{template}(piUuid, poErr_Msg);
    If v_Ret <> 0 Then Return v_Ret; End If;

    -- 状态锁检查
    v_Ret := PPS_OpenApi2.CheckStatLock(C_TABLENAME, piUuid);
    If v_Ret <> 0 Then Return(0); End If;

    -- 读取单头数据
    Select Sid, (Case When Is_Null(o.OperCode) = 1 And Is_Null(o.OperName) = 1 Then o.Filler Else o.OperName || '[' || o.OperCode || ']' End)
      Into v_Sid, v_Oper
      From OpenApi2_{template} o Where Uuid = piUuid;

    -- TODO: 根据实际业务需求设置v_BillInfo和v_Details
    -- If {template}Detail has goods fields (extends BaseBillDetail):
    --   v_Line := 0;
    --   v_Details := PCU_{template}.Details();
    --   For RDtl In CDtl Loop
    --     v_Line := v_Line + 1;
    --     v_Details.Extend;
    --     v_Details(v_Line).Line := v_Line;
    --     Select Gid Into v_Details(v_Line).GDGid From GDInput Where Code = RDtl.GoodsCode;
    --     v_Details(v_Line).GdCode := Nvl(RDtl.InputCode, RDtl.GoodsCode);
    --     v_Details(v_Line).Qty := RDtl.Qty;
    --     v_Details(v_Line).QpcStr := RDtl.QpcStr;
    --     v_Details(v_Line).Munit := RDtl.Munit;
    --     If Is_Not_Null(RDtl.QpcStr) = 1 Then
    --       Select g.QpcStr, g.Qpc, g.Munit Into v_Details(v_Line).QpcStr, v_Details(v_Line).Qpc, v_Details(v_Line).Munit
    --         From GdQpc g Where g.Gid = v_Details(v_Line).GDGid And g.QpcStr = RDtl.QpcStr;
    --     ElsIf Is_Not_Null(RDtl.Munit) = 1 Then
    --       Select g.QpcStr, g.Qpc, g.Munit Into v_Details(v_Line).QpcStr, v_Details(v_Line).Qpc, v_Details(v_Line).Munit
    --         From GdQpc g Where g.Gid = v_Details(v_Line).GDGid And g.Munit = RDtl.Munit And RowNum < 2;
    --     End If;
    --     v_Details(v_Line).Note := RDtl.Note;
    --   End Loop;
    -- If {template}Detail does NOT have goods fields:
    --   v_Line := 0;
    --   v_Details := PCU_{template}.Details();
    --   For RDtl In CDtl Loop
    --     v_Line := v_Line + 1;
    --     v_Details.Extend;
    --     v_Details(v_Line).Line := v_Line;
    --     v_Details(v_Line).{field1} := RDtl.{field1};
    --     v_Details(v_Line).{field2} := RDtl.{field2};
    --     -- Add other business fields from {template}Detail...
    --     v_Details(v_Line).Note := RDtl.Note;
    --   End Loop;
    -- v_BillInfo.Details := v_Details;

    -- 核心业务逻辑（固定生成部分，不可省略）
    v_Ret := PCU_{template}.GenBill(v_BillInfo, v_Oper, v_Num, poErr_Msg);
    If v_Ret <> 0 Then Return v_Ret; End If;
    v_DefStat := To_Number(PPS_OpenApi2_Util.ReadConfig({cls}, v_Sid, 'defTargetStat', '100'));
    If v_DefStat <> 0 Then
      v_Ret := PPS_{template}.On_Modify(v_Num, v_Oper, v_DefStat, poErr_Msg);
      If v_Ret <> 0 Then Return v_Ret; End If;
    End If;
    Update OpenApi2_{template} Set GenNum = v_Num Where Uuid = piUuid;
    PPS_OpenApi2.RecordLog(C_TABLENAME, piUuid, C_PROCESSFLAG_SUCCESS, Null);
    Return v_Ret;
  End;
```

### Step 4: Update PPS_OpenApi2_*_Hdr.oracle.sql

**File Location**: `h6-openapi2-rdb-setup/src/main/resources/META-INF/h6-openapi2-setup/script/oracle/sp/{oracle_package}_Hdr.oracle.sql`

**Add function declarations**:

```sql
  Function Check{template}
  (
    piUuid         In   Varchar2,
    poErr_Msg      Out  Varchar2
  ) Return Number;

  Function ProcessSingle{template}
  (
    piUuid In Varchar2,
    poErr_Msg Out Varchar2
  ) Return Number;
```

### Oracle Script Workflow

1. **Determine Oracle Package Name**
   - Map package parameter to Oracle package name using the mapping table above

2. **Read Existing Oracle Package File**
   - Read the corresponding PPS_OpenApi2_*_Pkg.oracle.sql file
   - Read the corresponding PPS_OpenApi2_*_Hdr.oracle.sql file

3. **Generate Check Function**
   - Based on template, generate Check{template} function
   - Include TODO comment for validation logic implementation
   - Add detail cursor if detail table exists

4. **Generate ProcessSingle Function**
   - Based on template, generate ProcessSingle{template} function
   - Include Check{template} call
   - Include TODO comment for business logic implementation

5. **Update PPS_OpenApi2_Pkg.oracle.sql**
   - Find Process function
   - Add else-if condition for new bill type

6. **Update PPS_OpenApi2_Pkg.oracle.sql**
   - Find CheckBeforeUpload function
   - Add else-if condition for new table

7. **Update PPS_OpenApi2_*_Hdr.oracle.sql**
   - Add function declarations for Check{template} and ProcessSingle{template}

8. **Write Updated Files**
   - Write all updated Oracle script files

9. **Git Add Oracle Files**
   - Add all generated/modified Oracle files to git

### Oracle Script Example

For `genType=create, moduleId=vdrPay, package=bill.account, cls=交款单新增`:

**Parameters**:
- `moduleId` = "vdrPay"
- `template` = "VdrPay"
- `package` = "bill.account"
- `cls` = "交款单新增"
- `oracle_package` = "PPS_OpenApi2_Account"

**Generated Oracle Code**:

1. **PPS_OpenApi2_Pkg.oracle.sql Update**:
```sql
      ElsIf piCls = 交款单新增 Then
        v_TableName := 'OpenApi2_VdrPay';
        v_Ret := PPS_OpenApi2_Account.ProcessSingleVdrPay(piUuid, poErr_Msg);
        If v_Ret <> 0 Then
          Rollback;
          AddLog(v_TableName, piUuid, 9, poErr_Msg);
        Return v_Ret;
        End If;
      End If;
```

2. **PPS_OpenApi2_Def_Pkg.oracle.sql Update**:
```sql
    ElsIf piTable = 'OpenApi2_VdrPay' Then
      v_Ret := PPS_OpenApi2_Account.CheckVdrPay(piUuid, poErr_Msg);
      If v_Ret <> 0 Then Return(v_Ret); End If;
    End If;
```

3. **PPS_OpenApi2_Account_Pkg.oracle.sql Additions**:
```sql
  Function CheckVdrPay
  (
    piUuid         In   Varchar2, --主键
    poErr_Msg      Out  Varchar2  --错误消息
  ) Return Number
    Is
    v_Ret Int;
    v_Count Int;
  Begin
    v_Ret := 0;
    -- TODO: 添加校验逻辑
    Return (v_Ret);
  End;

  Function ProcessSingleVdrPay
  (
    piUuid In Varchar2,
    poErr_Msg Out Varchar2
  ) Return Number
    Is
    v_Ret Int;
    v_Oper Varchar2(30);
    C_TABLENAME CONSTANT VARCHAR2(255) := 'OpenApi2_VdrPay';
    v_Sid OpenApi2_VdrPay.Sid%Type;
  Begin
    v_Ret := 0;
    -- 数据校验
    v_Ret := CheckVdrPay(piUuid, poErr_Msg);
    If v_Ret <> 0 Then Return v_Ret; End If;
    
    -- 状态锁检查
    v_Ret := PPS_OpenApi2.CheckStatLock(C_TABLENAME, piUuid);
    If v_Ret <> 0 Then Return(0); End If;
    
    -- 读取单头数据
    Select Sid, Filler Into v_Sid, v_Oper From OpenApi2_VdrPay o Where Uuid = piUuid;
    
    -- TODO: 添加业务逻辑
    -- 记录成功日志
    PPS_OpenApi2.RecordLog(C_TABLENAME, piUuid, C_PROCESSFLAG_SUCCESS, Null);
    Return v_Ret;
  End;
```

4. **PPS_OpenApi2_Account_Hdr.oracle.sql Additions**:
```sql
  Function CheckVdrPay
  (
    piUuid         In   Varchar2,
    poErr_Msg      Out  Varchar2
  ) Return Number;

  Function ProcessSingleVdrPay
  (
    piUuid In Varchar2,
    poErr_Msg Out Varchar2
  ) Return Number;
```

### Important Notes

1. **TODO Comments**: All generated Oracle functions include TODO comments for business logic implementation
2. **Manual Review**: Always review generated Oracle scripts before applying to database
3. **Testing**: Test Oracle functions thoroughly in development environment first
4. **Backup**: Always backup existing Oracle scripts before modification
5. **Validation**: The Check function should contain appropriate validation logic based on business requirements
6. **Business Logic**: The ProcessSingle function should implement the actual H6 bill generation logic

### Oracle Script Generation Workflow Update

The main workflow should be updated to include these steps:

```markdown
7. **Generate Oracle Scripts**
   - Determine oracle_package name based on package
   - Read existing PPS_OpenApi2_*_Pkg.oracle.sql file
   - Generate Check{template} function
   - Generate ProcessSingle{template} function
   - Update PPS_OpenApi2_Pkg.oracle.sql (Process function)
   - Update PPS_OpenApi2_Pkg.oracle.sql (CheckBeforeUpload function)
   - Update PPS_OpenApi2_*_Hdr.oracle.sql (function declarations)
   - Write all updated Oracle files

8. **Validate**
   - Check that all files are written to correct locations
   - Verify package declarations are correct
   - Ensure proper inheritance and annotations
   - Verify validation code is properly generated and formatted
   - Verify Oracle scripts are syntactically correct

9. **Update DataProcessHandler**
   - Read DataProcessHandler.java file
   - Add import statement for {template}Service
   - Add else-if condition in handle() method for BillCls.{cls}
   - Add getter method for {template}Service

10. **Git Add Files**
    - Add all generated Java files
    - Add all generated/updated Oracle script files
    - Execute `git add` command for each file
```

### Related Files for Oracle Scripts

| File Type                     | Location | Purpose |
|-------------------------------|----------|---------|
| PPS_OpenApi2_Pkg.oracle.sql   | `h6-openapi2-rdb-setup/.../script/oracle/sp/` | Main routing package |
| PPS_OpenApi2_Pkg.oracle.sql   | `h6-openapi2-rdb-setup/.../script/oracle/sp/` | Default validation package |
| PPS_OpenApi2_*_Pkg.oracle.sql | `h6-openapi2-rdb-setup/.../script/oracle/sp/` | Package-specific implementation |
| PPS_OpenApi2_*_Hdr.oracle.sql | `h6-openapi2-rdb-setup/.../script/oracle/sp/` | Package header declarations |
| PPS_OpenApi2_Hdr.oracle.sql   | `h6-openapi2-rdb-setup/.../script/oracle/sp/` | Main header declarations |
