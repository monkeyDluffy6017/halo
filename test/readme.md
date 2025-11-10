# e2e 测试目录结构
test
├── java
│   └── com
│       └── example
│           └── myapp
│               ├── base/
│               │   └── BaseApiTest.java    # (所有 API 测试的基类)
│               ├── config/
│               │   └── TestConfig.java     # (加载和提供配置)
│               ├── e2e/
│               │   ├── UserApiE2ETest.java   # (用户相关的 E2E 测试)
│               │   └── OrderApiE2ETest.java  # (订单相关的 E2E 测试)
│               └── helpers/
│                   ├── ApiClient.java      # (可选：封装通用 API 调用的客户端)
│                   └── DtoFactory.java     # (可选：用于创建请求 Body 的工厂)
└── resources
    ├── config.properties       # (核心配置文件，存放 API URL)
    └── logback-test.xml        # (日志配置文件)

# 核心工具栈
| 类别 | 推荐工具 | 作用 |
| :--- | :--- | :--- |
| **核心框架** | `JUnit 5` | 现代 Java 的标准测试框架。使用 `@Test`, `@BeforeAll` 等注解管理测试生命周期。 |
| **套件管理** | `JUnit 5` (基类与 `@Tag`) | 使用 `BaseApiTest` 基类处理全局设置 (如 `RestAssured.baseURI`)，使用 `@Tag` 对测试进行分组。 |
| **断言库** | `AssertJ` | 提供流式 (fluent) 且可读性极强的断言，如 `assertThat(user.getId()).isNotNull()`。 |
| **配置管理** | `Properties` 文件 + 自定义 `TestConfig` 类 | 从 `.properties` 文件中读取配置 (如 `app.api.baseUrl`)，实现配置与代码分离。 |
| **API 交互** | `REST Assured` | Java 中事实上的 API 测试标准。提供强大的 DSL 进行 HTTP 请求、响应解析和验证。 |
| **数据模型** | `Jackson` + POJOs | 使用 POJO (普通 Java 对象) 自动序列化/反序列化 JSON，取代易碎的字符串，提高可维护性。 |
| **构建/运行** | `Maven` (Surefire 插件) | 管理所有依赖，并提供命令行 (如 `mvn test`) 来执行测试套件。 |

# 文件内容和代码示例
1. src/test/resources/config.properties
这是核心配置文件

```
# ----------------------------------------------------
# Environment URLs (Managed by your external script)
# ----------------------------------------------------
app.api.baseUrl=http://localhost:8080/api/v1

# ----------------------------------------------------
# API Keys or Tokens (if needed, though env vars are safer)
# ----------------------------------------------------
# api.key=your-static-api-key
```

2. src/test/java/com/example/myapp/config/TestConfig.java
加载和提供配置

```
package com.example.myapp.config;

import java.io.IOException;
import java.io.InputStream;
import java.util.Properties;

/**
 * Loads and provides access to test configuration.
 * Uses a singleton pattern to load properties only once.
 */
public class TestConfig {

    private static final Properties properties = new Properties();
    private static final String CONFIG_FILE = "config.properties";

    // Load properties on class initialization
    static {
        try (InputStream input = TestConfig.class.getClassLoader().getResourceAsStream(CONFIG_FILE)) {
            if (input == null) {
                System.err.println("Sorry, unable to find " + CONFIG_FILE);
                throw new RuntimeException("Cannot find " + CONFIG_FILE);
            }
            properties.load(input);
        } catch (IOException ex) {
            ex.printStackTrace();
            throw new RuntimeException("Failed to load config", ex);
        }
    }

    /**
     * Gets a property value by key.
     * @param key The property key
     * @return The property value
     */
    public static String getProperty(String key) {
        // Allow overriding from System properties (for CI/CD)
        return System.getProperty(key, properties.getProperty(key));
    }

    // Convenience method
    public static String getApiBaseUrl() {
        return getProperty("app.api.baseUrl");
    }
}
```

3. src/test/java/com/example/myapp/base/BaseApiTest.java
API 测试基类

```
package com.example.myapp.base;

import com.example.myapp.config.TestConfig;
import io.rest-assured.RestAssured;
import io.rest-assured.builder.RequestSpecBuilder;
import io.rest-assured.filter.log.LogDetail;
import io.rest-assured.filter.log.RequestLoggingFilter;
import io.rest-assured.filter.log.ResponseLoggingFilter;
import io.rest-assured.http.ContentType;
import io.rest-assured.specification.RequestSpecification;
import org.junit.jupiter.api.BeforeAll;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * Base class for all API E2E tests.
 * Handles one-time setup:
 * 1. Sets the base URI from config.
 * 2. Creates a reusable RequestSpecification (spec).
 */
public abstract class BaseApiTest {

    protected static final Logger log = LoggerFactory.getLogger(BaseApiTest.class);

    // Reusable specification for all requests
    protected static RequestSpecification spec;

    @BeforeAll
    public static void globalApiSetup() {
        log.info("Setting up global API test configuration...");

        // 1. Set the base URI for all tests
        RestAssured.baseURI = TestConfig.getApiBaseUrl();
        log.info("REST Assured Base URI set to: {}", RestAssured.baseURI);

        // 2. Create a reusable RequestSpecification
        // This spec will be used by all tests extending this class
        spec = new RequestSpecBuilder()
                .setContentType(ContentType.JSON) // Default content type
                .setAccept(ContentType.JSON)     // Default accept header
                // Add authentication here if it's static (e.g., API key)
                // .addHeader("X-API-KEY", TestConfig.getProperty("api.key"))
                .build();

        // 3. Add global logging filters (optional but recommended)
        // This logs all requests and responses
        RestAssured.filters(
            new RequestLoggingFilter(LogDetail.ALL),
            new ResponseLoggingFilter(LogDetail.ALL)
        );
    }
}
```

4. src/test/java/com/example/myapp/helpers/User.java
我们使用 POJO (Plain Old Java Objects) 来处理 JSON 请求和响应

POJO 示例
```
// src/test/java/com/example/myapp/helpers/User.java
package com.example.myapp.helpers;

// Using Jackson annotations for serialization/deserialization
import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import com.fasterxml.jackson.annotation.JsonInclude;

@JsonIgnoreProperties(ignoreUnknown = true) // Ignore fields from response not defined here
@JsonInclude(JsonInclude.Include.NON_NULL) // Don't send null fields in requests
public class User {
    private String id;
    private String username;
    private String email;

    // Default constructor needed by Jackson
    public User() {}

    // Constructor for creating new users
    public User(String username, String email) {
        this.username = username;
        this.email = email;
    }

    // Getters and Setters...
    public String getId() { return id; }
    public void setId(String id) { this.id = id; }
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
}
```

5. src/test/java/com/example/myapp/e2e/UserApiE2ETest.java
测试类

```
package com.example.myapp.e2e;

import com.example.myapp.base.BaseApiTest;
import com.example.myapp.helpers.User; // Import our POJO
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.Tag;

// Import static methods for BDD syntax and AssertJ
import static io.rest-assured.RestAssured.*;
import static org.hamcrest.Matchers.*;
import static org.assertj.core.api.Assertions.*;

@Tag("api-e2e")
public class UserApiE2ETest extends BaseApiTest {

    @Test
    @DisplayName("API E2E: Should create a new user and retrieve it (using POJOs)")
    void shouldCreateAndVerifyUser() {
        // 1. Define test data using the POJO
        User newUserRequest = new User("e2e_pojo_user", "pojo@example.com");

        // 2. Create the user (POST)
        // REST Assured automatically serializes the POJO to JSON
        User createdUser = given()
            .spec(spec) // Use the base specification
            .body(newUserRequest) // Pass the POJO
        .when()
            .post("/users")
        .then()
            .statusCode(201)
            .body("username", equalTo(newUserRequest.getUsername()))
            .extract().as(User.class); // Deserialize the response back into a User POJO

        log.info("Created new user with ID: {}", createdUser.getId());

        // AssertJ assertion on the extracted object
        assertThat(createdUser.getId()).isNotNull().isNotEmpty();
        assertThat(createdUser.getUsername()).isEqualTo(newUserRequest.getUsername());

        // 3. Verify the user (GET)
        User fetchedUser = given()
            .spec(spec)
            .pathParam("userId", createdUser.getId())
        .when()
            .get("/users/{userId}")
        .then()
            .statusCode(200)
            .extract().as(User.class); // Deserialize

        // AssertJ assertions on the fetched object
        assertThat(fetchedUser.getId()).isEqualTo(createdUser.getId());
        assertThat(fetchedUser.getEmail()).isEqualTo(newUserRequest.getEmail());

        // 4. Cleanup (DELETE)
        given()
            .spec(spec)
            .pathParam("userId", createdUser.getId())
        .when()
            .delete("/users/{userId}")
        .then()
            .statusCode(204);

        // 5. Verify Cleanup
        given()
            .spec(spec)
            .pathParam("userId", createdUser.getId())
        .when()
            .get("/users/{userId}")
        .then()
            .statusCode(404);
    }
}
```