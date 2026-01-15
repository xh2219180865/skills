---
name: backend-feature
description: 配合 constitution.md 使用的 Java 21 + Spring Boot 3 后端核心开发技能。强制执行严格的编码标准：全套基础设施（CommonResult/PageResult/I18n）、MapStruct 转换（禁止 BeanUtils）、Long 类型防丢失序列化、以及分层架构模版。Use this skill when implementing Java backend features.
---

# 后端功能开发

本技能为 Java 后端开发提供标准模版并强制执行编码规范。**本技能必须与系统上下文中的 `constitution.md` 配合使用。**

## 核心规则（红线 - 必须遵守）

### 红线 1：环境与注解
- **禁止**：使用 `javax.*` 包（必须使用 `jakarta.*`）。
- **禁止**：在 `*DO` 实体类上使用 `@Data`（防止 HashCode 死循环及懒加载性能问题）。
- **必须**：`*DO` 类主键（Long 类型）必须加 `@JsonSerialize(using = ToStringSerializer.class)` 防止前端精度丢失。

### 红线 2：对象转换
- **禁止**：使用 BeanUtils、反射或在 Service 层手写 Setter 链。
- **必须**：使用 MapStruct，接口必须配置 `unmappedTargetPolicy = ReportingPolicy.IGNORE`。

### 红线 3：异常与国际化
- **禁止**：抛出 `RuntimeException` 或直接硬编码中文错误提示（如 "用户不存在"）。
- **必须**：抛出 `BusinessException` 配合 `GlobalErrorCodeConstants`（枚举值存放 i18n Key，如 `user.not.found`）。

### 红线 4：响应结构
- **必须**：所有 API 返回 `CommonResult<T>`。
- **必须**：分页查询请求继承 `PageQuery`，返回 `PageResult<ItemDTO>`。

## 标准模版

### 模版 1：基础设施 (Result, Page, Query)

**必须先行确立的基础类**：


```java
package com.example.common.req;

import io.swagger.v3.oas.annotations.media.Schema;
import jakarta.validation.constraints.Min;
import lombok.Data;

/**
 * 分页查询基类
 */
@Data
public class PageQuery {
    @Schema(description = "页码", defaultValue = "1")
    @Min(value = 1, message = "pageNum must be greater than 0")
    private Long pageNum = 1L;

    @Schema(description = "页大小", defaultValue = "10")
    @Min(value = 1, message = "pageSize must be greater than 0")
    private Long pageSize = 10L;
}
```

### 模版 2：异常与国际化 (Exception & Handler)

```java
package com.example.common.handler;

import com.example.common.exception.BusinessException;
import com.example.common.result.CommonResult;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.MessageSource;
import org.springframework.context.i18n.LocaleContextHolder;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@Slf4j
@RestControllerAdvice
@RequiredArgsConstructor
public class GlobalExceptionHandler {

    private final MessageSource messageSource;

    @ExceptionHandler(BusinessException.class)
    public CommonResult<Object> handleBusinessException(BusinessException e) {
        // 解析国际化消息，如果解析失败则返回 key 本身或默认错误
        String message;
        try {
            message = messageSource.getMessage(e.getMessage(), e.getArgs(), LocaleContextHolder.getLocale());
        } catch (Exception ex) {
            log.warn("I18n key missing: {}", e.getMessage());
            message = e.getMessage(); // 降级策略
        }
        
        log.warn("Business Exception: code={}, msg={}", e.getCode(), message);
        return CommonResult.error(e.getCode(), message);
    }
}
```

### 模版 3：数据层 (DO & Mapper)


### 模版 4：转换层 (MapStruct)

```java
package com.example.service.convert;

import com.example.controller.req.UserCreateRequest;
import com.example.controller.req.UserUpdateRequest;
import com.example.controller.vo.UserVO;
import com.example.dal.dataobject.UserDO;
import org.mapstruct.*;
import java.util.List;

/**
 * 必须配置：componentModel = "spring", unmappedTargetPolicy = IGNORE
 * 注意：Maven 编译插件中，lombok 必须在 mapstruct-processor 之前
 */
@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface UserConvert {
    
    UserDO toUserDO(UserCreateRequest request);
    
    UserVO toUserVO(UserDO userDO);
    
    List<UserVO> toUserVOList(List<UserDO> list);
    
    // 更新时忽略 null 值
    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    void updateUserDO(UserUpdateRequest request, @MappingTarget UserDO userDO);
}
```

### 模版 5：服务层 (Service Impl)

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
    private final UserConvert userConvert; 
    
    @Override
    @Transactional(rollbackFor = Exception.class)
    public void updateUser(Long id, UserUpdateRequest request) {
        UserDO userDO = userMapper.selectById(id);
        if (userDO == null) {
            // ✅ 抛出错误码枚举
            throw new BusinessException(GlobalErrorCode.USER_NOT_FOUND);
        }
        userConvert.updateUserDO(request, userDO);
        userMapper.updateById(userDO);
    }

    @Override
    public PageResult<UserVO> queryUsers(UserQueryRequest request) {
        // ✅ 使用基类的 pageNum/pageSize
        Page<UserDO> page = new Page<>(request.getPageNum(), request.getPageSize());
        
        userMapper.selectPage(page, new LambdaQueryWrapper<UserDO>()
                .like(request.getUsername() != null, UserDO::getUsername, request.getUsername()));
        
        return new PageResult<>(
            userConvert.toUserVOList(page.getRecords()),
            page.getTotal(),
            page.getCurrent(),
            page.getSize()
        );
    }
}
```

### 模版 6：接口层 (Controller)

```java
package com.example.controller;

import com.example.common.result.CommonResult;
import com.example.common.result.PageResult;
import com.example.controller.req.UserQueryRequest;
import com.example.controller.vo.UserVO;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.RequiredArgsConstructor;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;
import jakarta.validation.Valid;

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
        // UserQueryRequest 必须继承 PageQuery
        return CommonResult.success(userService.queryUsers(request));
    }
}
```

## 配置检查清单

1.  **I18n 文件**：`src/main/resources/i18n/messages.properties` 是否存在。
2.  **Maven/Gradle**：确保 `lombok` 在 `mapstruct-processor` 之前加载。
3.  **Application.yml**：
    ```yaml
    spring:
      messages:
        basename: i18n/messages
        encoding: UTF-8
      jackson:
        default-property-inclusion: non_null # 可选：不返回 null 字段
    ```
```