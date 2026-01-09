---
name: backend-feature
description: 配合 constitution.md 使用的 Java 21 + Spring Boot 3 后端核心开发技能。包含全套基础设施（Result/Page/I18n/Exception）、MapStruct 转换策略及分层架构模版。Use this skill when implementing Java backend features. This skill enforces strict coding standards`No @Data on DOs, Jakarta EE imports, MapStruct with unmapped policy, Internationalized Exceptions, and PageResult responses.`
---

# 后端功能开发

本技能为 Java 后端开发提供标准模版并强制执行编码规范。**本技能必须与系统上下文中的 `constitution.md` 配合使用。**

## 核心规则（红线 - 必须遵守）

### 红线 1：环境与注解
- **禁止**：使用 `javax.*` 包（必须使用 `jakarta.*`）。
- **禁止**：在 `*DO` 实体类上使用 `@Data`（防止 HashCode 死循环及懒加载性能问题）。
- **必须**：`*DO` 类使用 `@Getter`, `@Setter`, `@ToString`。

### 红线 2：对象转换
- **禁止**：使用 BeanUtils、反射或在 Service 层手写 Setter 链。
- **必须**：使用 MapStruct，接口必须配置 `unmappedTargetPolicy = ReportingPolicy.IGNORE`。

### 红线 3：异常与国际化
- **禁止**：抛出 `RuntimeException` 或直接硬编码中文错误提示（如 "用户不存在"）。
- **必须**：抛出 `BusinessException` 配合 `GlobalErrorCode`（枚举值存放 i18n Key，如 `user.not.found`）。

### 红线 4：响应结构
- **必须**：所有 API 返回 `CommonResult<T>`。
- **必须**：列表查询接口必须返回 `PageResult<ItemDTO>`（含 total, list, pageNum, pageSize）。

## 标准模版

### 模版 1：基础设施 (Result, Page, Exception, I18n)

**必须先行确立的基础类**：

```java
package com.example.common.result;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import java.util.List;

/**
 * 统一分页结果
 */
@Data
@NoArgsConstructor
@AllArgsConstructor
public class PageResult<T> {
    private List<T> list;
    private Long total;
    private Long pageNum;
    private Long pageSize;
}
```

```java
package com.example.common.exception;

import lombok.Getter;

public interface IErrorCode {
    Integer getCode();
    String getMsg(); // 返回 i18n key
}
```

```java
package com.example.common.enums;

import com.example.common.exception.IErrorCode;
import lombok.AllArgsConstructor;
import lombok.Getter;

@Getter
@AllArgsConstructor
public enum GlobalErrorCode implements IErrorCode {
    SUCCESS(0, "success"),
    SYSTEM_ERROR(500, "error.system"),
    PARAM_ERROR(400, "error.param"),
    // 业务示例：Msg 必须是 messages.properties 中的 key
    USER_NOT_FOUND(1001, "user.not.found");

    private final Integer code;
    private final String msg;
}
```

```java
package com.example.common.exception;

import lombok.Getter;

@Getter
public class BusinessException extends RuntimeException {
    private final Integer code;
    private final Object[] args; // i18n 参数

    public BusinessException(IErrorCode errorCode, Object... args) {
        super(errorCode.getMsg());
        this.code = errorCode.getCode();
        this.args = args;
    }
}
```

```java
package com.example.common.handler;

import com.example.common.enums.GlobalErrorCode;
import com.example.common.exception.BusinessException;
import com.example.common.result.CommonResult;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.MessageSource;
import org.springframework.context.i18n.LocaleContextHolder;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import java.util.Objects;

/**
 * 全局异常处理器 (含 I18n 解析)
 */
@Slf4j
@RestControllerAdvice
@RequiredArgsConstructor
public class GlobalExceptionHandler {

    private final MessageSource messageSource;

    @ExceptionHandler(BusinessException.class)
    public CommonResult<Object> handleBusinessException(BusinessException e) {
        // 解析国际化消息
        String message = messageSource.getMessage(e.getMessage(), e.getArgs(), e.getMessage(), LocaleContextHolder.getLocale());
        log.warn("Business Exception: code={}, msg={}", e.getCode(), message);
        return CommonResult.error(e.getCode(), message);
    }
    
    // ... 其他异常处理 (MethodArgumentNotValidException, Exception) 略，逻辑同上
}
```

### 模版 2：数据层 (DO & Mapper)

```java
package com.example.dal.dataobject;

import com.baomidou.mybatisplus.annotation.*;
import lombok.Getter;
import lombok.Setter;
import lombok.ToString;
import java.time.LocalDateTime;

@Getter
@Setter
@ToString
@TableName("user")
public class UserDO {
    @TableId(type = IdType.AUTO)
    private Long id; // JSON序列化需全局配置转String
    
    private String username;
    private Integer status;
    
    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;
    
    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;
    
    @TableLogic
    private Integer deleted;
}
```

### 模版 3：转换层 (MapStruct)

```java
package com.example.service.adapter;

import com.example.controller.request.UserCreateRequest;
import com.example.controller.request.UserUpdateRequest;
import com.example.controller.vo.UserVO;
import com.example.dal.dataobject.UserDO;
import org.mapstruct.*;
import java.util.List;

/**
 * 必须配置：componentModel = "spring", unmappedTargetPolicy = IGNORE
 */
@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface UserAdapter {
    
    UserDO toUserDO(UserCreateRequest request);
    
    UserVO toUserVO(UserDO userDO);
    
    List<UserVO> toUserVOList(List<UserDO> list);
    
    // 更新时忽略 null 值
    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    void updateUserDO(UserUpdateRequest request, @MappingTarget UserDO userDO);
}
```

### 模版 4：服务层 (Service Impl)

```java
package com.example.service.impl;

import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import com.example.common.enums.GlobalErrorCode;
import com.example.common.exception.BusinessException;
import com.example.common.result.PageResult;
import com.example.controller.vo.UserVO;
import com.example.dal.dataobject.UserDO;
import com.example.service.UserService;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
public class UserServiceImpl implements UserService {
    
    private final UserMapper userMapper;
    private final UserAdapter userAdapter; // 依赖接口
    
    @Override
    @Transactional(rollbackFor = Exception.class)
    public void updateUser(Long id, UserUpdateRequest request) {
        UserDO userDO = userMapper.selectById(id);
        if (userDO == null) {
            // ✅ 抛出错误码，而非中文
            throw new BusinessException(GlobalErrorCode.USER_NOT_FOUND);
        }
        userAdapter.updateUserDO(request, userDO);
        userMapper.updateById(userDO);
    }

    @Override
    public PageResult<UserVO> queryUsers(UserQueryRequest request) {
        Page<UserDO> page = new Page<>(request.getPageNum(), request.getPageSize());
        userMapper.selectPage(page, new LambdaQueryWrapper<UserDO>()
                .like(request.getUsername() != null, UserDO::getUsername, request.getUsername()));
        
        // ✅ 封装标准分页结果
        return new PageResult<>(
            userAdapter.toUserVOList(page.getRecords()),
            page.getTotal(),
            page.getCurrent(),
            page.getSize()
        );
    }
}
```

### 模版 5：接口层 (Controller)

```java
package com.example.controller;

import com.example.common.result.CommonResult;
import com.example.common.result.PageResult;
import com.example.controller.vo.UserVO;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.RequiredArgsConstructor;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;
import jakarta.validation.Valid; // ✅ Spring Boot 3 使用 Jakarta

@Tag(name = "用户管理")
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
@Validated
public class UserController {
    
    private final UserService userService;
    
    @Operation(summary = "分页查询")
    @GetMapping
    public CommonResult<PageResult<UserVO>> query(@Valid UserQueryRequest request) {
        return CommonResult.success(userService.queryUsers(request));
    }
}
```

## 配置说明

在使用本 Skill 前，请确保 `src/main/resources` 下包含国际化配置文件：

1.  **文件位置**：
    - `i18n/messages.properties` (默认/英文)
    - `i18n/messages_zh_CN.properties` (中文)
2.  **配置内容 (application.yml)**：
    ```yaml
    spring:
      messages:
        basename: i18n/messages
        encoding: UTF-8
    ```
```