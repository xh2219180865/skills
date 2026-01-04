---
name: backend-test
description: 配合 backend-feature 使用的 Java 21 + Spring Boot 3 后端自动化测试技能。包含单元测试(Service)、切片测试(Controller)及 AssertJ 断言规范。Use this skill to generate JUnit 5 tests. Enforces Mockito for isolation, WebMvcTest for APIs, and strictly validates CommonResult/PageResult structures.
---

# 后端自动化测试规范

本技能用于生成高质量的 Java 后端测试代码，**必须配合 `backend-feature` 的架构规范使用**。

## 核心原则 (Testing Manifesto)

1.  **分层测试策略**：
    - **Service 层**：必须是**纯单元测试**。使用 `Mockito` 模拟 Mapper 和 Converter，禁止启动 Spring 上下文（速度优先）。
    - **Controller 层**：使用 `@WebMvcTest` 进行切片测试，验证 URL 映射、参数校验 (`@Valid`) 和全局异常处理。
    - **Mapper 层**：仅在手写复杂 SQL 时编写，使用 H2 内存数据库进行 `@MybatisPlusTest`。

2.  **断言规范**：
    - **必须**：使用 **AssertJ** (`assertThat`)，禁止使用 JUnit 原生断言 (`assertEquals`)，以获得更好的可读性和错误提示。
    - **必须**：验证业务异常时，使用 `assertThatThrownBy(...).isInstanceOf(BusinessException.class).extracting("code").isEqualTo(...)`。

3.  **命名规范**：
    - 类名：`{TargetClass}Test`
    - 方法名：`should{ExpectedBehavior}_when{Condition}` (例如: `shouldThrowException_whenUserNotFound`)
    - 显示名：必须使用 `@DisplayName("中文描述测试意图")`

## 标准模版

### 模版 1：Service 层单元测试 (Pure Mockito)

**适用场景**：测试业务逻辑、分支判断、异常抛出。
**关键点**：不依赖 Spring 容器，运行速度快。

```java
package com.example.service;

import com.example.common.enums.GlobalErrorCode;
import com.example.common.exception.BusinessException;
import com.example.controller.request.UserUpdateRequest;
import com.example.dal.dataobject.UserDO;
import com.example.service.adapter.UserAdapter; // MapStruct
import com.example.service.impl.UserServiceImpl;
import com.example.mapper.UserMapper;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserMapper userMapper;

    @Mock
    private UserAdapter userAdapter; // Mock MapStruct 接口

    @InjectMocks
    private UserServiceImpl userService;

    @Test
    @DisplayName("更新用户-成功：当用户存在时应更新数据")
    void shouldUpdateUser_whenUserExists() {
        // Given
        Long userId = 1L;
        UserUpdateRequest request = new UserUpdateRequest();
        UserDO mockDO = new UserDO();
        mockDO.setId(userId);

        when(userMapper.selectById(userId)).thenReturn(mockDO);
        // MapStruct 返回 void，无需 mock 返回值，但可验证调用

        // When
        userService.updateUser(userId, request);

        // Then
        verify(userAdapter).updateUserDO(request, mockDO); // 验证转换器被调用
        verify(userMapper).updateById(mockDO);             // 验证持久层被调用
    }

    @Test
    @DisplayName("更新用户-失败：当用户不存在时应抛出业务异常")
    void shouldThrowException_whenUserNotFound() {
        // Given
        Long userId = 999L;
        when(userMapper.selectById(userId)).thenReturn(null);

        // When & Then
        assertThatThrownBy(() -> userService.updateUser(userId, new UserUpdateRequest()))
                .isInstanceOf(BusinessException.class)
                .extracting("code") // 验证错误码
                .isEqualTo(GlobalErrorCode.USER_NOT_FOUND.getCode());

        verify(userMapper, never()).updateById(any());
    }
}