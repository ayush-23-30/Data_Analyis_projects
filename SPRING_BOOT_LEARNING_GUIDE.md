# Spring Boot Learning Guide for Beginners

## Complete Learning Plan + Practical API Creation

---

## Table of Contents

1. [Part 1: Understanding Spring Boot Fundamentals](#part-1-understanding-spring-boot-fundamentals)
2. [Part 2: Analyzing an Existing Spring Boot Project](#part-2-analyzing-an-existing-spring-boot-project)
3. [Part 3: Step-by-Step REST API Creation](#part-3-step-by-step-rest-api-creation)
4. [Part 4: Using GitHub Copilot Effectively](#part-4-using-github-copilot-effectively)
5. [Part 5: Best Practices & Common Patterns](#part-5-best-practices--common-patterns)
6. [Part 6: Resources & Further Learning](#part-6-resources--further-learning)

---

# Part 1: Understanding Spring Boot Fundamentals

## What is Spring Boot?

**Spring Boot** is a framework built on top of Spring Framework that makes creating production-ready Java applications simple. Think of it as:

- **Spring Framework** = Powerful but requires lots of manual configuration
- **Spring Boot** = Spring with conventions, auto-configuration, and embedded servers built-in

### Key Concepts 

#### 1. **Dependency Injection (DI)**

Java classes often depend on other classes. Instead of creating them manually, Spring automatically injects dependencies.

```java
// WITHOUT Spring (manual dependencies) ---- 
public class EmployeeService {
    private EmployeeRepository repo = new EmployeeRepository(); // tight coupling
}

// WITH Spring (Dependency Injection)
@Service
public class EmployeeService {
    @Autowired
    private EmployeeRepository repo; // Spring injects this automatically
}
```

**Benefits**:

- Easy to test (swap real dependencies with mocks)
- Loose coupling (change implementations without rewriting code)
- Centralized management

#### 2. **Annotations - Spring's Magic**

Annotations are markers that tell Spring what to do with a class. -- Pre define Classes 

| Annotation               | Purpose                           | Example                                        |
| ------------------------ | --------------------------------- | ---------------------------------------------- |
| `@SpringBootApplication` | Main entry point                  | `public class CoreEmsHrServicesApplication {}` |
| `@Controller`            | Handles HTTP requests             | `public class EmployeeController {}`           |
| `@RestController`        | Returns JSON/XML (not HTML pages) | `public class ApiController {}`                |
| `@Service`               | Business logic layer              | `public class EmployeeService {}`              |
| `@Repository`            | Database access layer             | `public class EmployeeRepository {}`           |
| `@Component`             | Generic component                 | For custom utilities                           |
| `@Autowired`             | Inject dependency                 | `@Autowired private SomeService service;`      |
| `@RequestMapping`        | Map URL to method                 | `@RequestMapping("/employees")`                |
| `@GetMapping`            | Map GET request                   | `@GetMapping("/{id}")`                         |
| `@PostMapping`           | Map POST request                  | `@PostMapping`                                 |
| `@PutMapping`            | Map PUT request                   | `@PutMapping`                                  |
| `@DeleteMapping`         | Map DELETE request                | `@DeleteMapping`                               |

#### 3. **Three-Tier Architecture**

Your application is organized in layers:

```
User Request
    ↓
[CONTROLLER] ← Handles HTTP requests, validates input
    ↓
[SERVICE] ← Contains business logic
    ↓
[REPOSITORY] ← Accesses database
    ↓
[DATABASE] ← Stores data
```

#### 4. **DTO (Data Transfer Object)**

Objects that carry data between layers:

- `Request DTO`: Data coming FROM the client
- `Response DTO`: Data going TO the client

```java
// Request DTO - client sends this
public class CreateEmployeeRequest {
    @NotBlank
    private String name;
    private String email;
}

// Response DTO - we send this back
public class EmployeeResponse {
    private Long id;
    private String name;
    private String email;
}
```

#### 5. **JPA (Java Persistence API) & Hibernate**

Automatically converts Java objects to database tables and back.

```java
@Entity
@Table(name = "employees")
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "name")
    private String name;
}
// Hibernate automatically creates/maps this to the database
```

---

# Part 2: Analyzing an Existing Spring Boot Project

## Project Structure Overview

Your existing project has this structure:

```
src/main/java/com/ems/hr/service/
├── CoreEmsHrServicesApplication.java (Entry point)
├── config/                           (Configuration classes)
├── controller/                       (HTTP endpoints)
├── dto/
│   ├── request/                      (Input DTOs)
│   ├── response/                     (Output DTOs)
│   └── common/                       (Shared DTOs)
├── entity/                           (Database models)
├── repository/                       (Database access)
├── service/                          (Business logic)
├── security/                         (JWT, authentication)
├── exception/                        (Error handling)
└── util/                             (Helper utilities)

resources/
├── application.properties            (Main configuration)
├── application-prod.properties       (Production config)
└── application-test.properties       (Test config)
```

## How to Navigate & Understand Code

### Step 1: Find the Entry Point

```java
// File: CoreEmsHrServicesApplication.java
@SpringBootApplication
public class CoreEmsHrServicesApplication {
    public static void main(String[] args) {
        SpringApplication.run(CoreEmsHrServicesApplication.class, args);
    }
}
```

This is where everything starts. `@SpringBootApplication` combines:

- `@Configuration` - This is a config class
- `@EnableAutoConfiguration` - Auto-setup based on dependencies
- `@ComponentScan` - Find all `@Service`, `@Controller` classes

### Step 2: Check Configuration (application.properties)

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/ems_hr
spring.datasource.username=root
spring.datasource.password=your_password
spring.jpa.hibernate.ddl-auto=update
server.port=8080
```

**What this means:**

- Connect to MySQL database at localhost
- Use Hibernate to auto-update schema
- Server runs on port 8080

### Step 3: Trace a Request End-to-End

**Example: Getting an employee by ID**

```
1. CLIENT REQUEST
  GET/companies/{companyId}/employees/{employeeId}

2. CONTROLLER (handles request)
   @GetMapping("/{employeeId}")
   public ResponseEntity<EmployeeResponse> getEmployee(
       @PathVariable Long employeeId
   ) {...}

3. SERVICE (business logic)
   public Employee findById(Long employeeId) {
       return employeeRepository.findById(employeeId);
   }

4. REPOSITORY (database)
   Employee findById(Long id);
   // JPA auto-generates SQL: SELECT * FROM employees WHERE id = ?

5. DATABASE
   Returns Employee record

6. RESPONSE
   Controller converts Employee to EmployeeResponse JSON
   Sends back to client
```

### Step 4: Understand Key Patterns in Your Project

**Multi-Tenancy Pattern** (Row-Level Isolation)

```java
// Every query filters by company_id
public List<Employee> findByCompanyIdAndDeletedAtIsNull(Long companyId) {
    // Only returns employees for this specific company
}

// Controller enforces tenant checks
@PreAuthorize("@tenantValidator.isValidCompany(#companyId)")
public ResponseEntity<?> getEmployees(@PathVariable UUID companyId) {...}
```

**Soft Delete Pattern**

```java
// Instead of deleting, mark as deleted
@Column(name = "deleted_at")
private LocalDateTime deletedAt;

// Query only non-deleted records
findByCompanyIdAndDeletedAtIsNull(companyId);
```

**Caching Pattern**

```java
@Cacheable(value = "employeeProfile", key = "#employeeUuid")
public Employee getEmployee(String employeeUuid) {
    // Result is cached in Redis, fast retrieval
}

@CacheEvict(value = "employeeProfile", key = "#employeeUuid")
public void updateEmployee(String employeeUuid) {
    // Cache is cleared when data changes
}
```

---

# Part 3: Step-by-Step REST API Creation

## Building a Simple "Task Management" API

Let's build a simple REST API from scratch that manages tasks. This will teach you all the fundamentals.

### Prerequisites

- Java 11+ installed
- Maven or Gradle
- MySQL running
- VS Code with Java extensions

### Project Setup

#### Step 1: Create Gradle Project Structure

**File: `build.gradle`**

```gradle
plugins{ 
    id 'java'
    id 'org.springframework.boot' version '2.7.14'
    id 'io.spring.dependency-management' version '1.0.15.RELEASE'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'

repositories {
    mavenCentral()
}

dependencies {
    // Spring Boot Web (REST APIs)
    implementation 'org.springframework.boot:spring-boot-starter-web'

    // Database (JPA + MySQL)
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'com.mysql:mysql-connector-j'

    // Lombok (reduces boilerplate)
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    // Validation
    implementation 'org.springframework.boot:spring-boot-starter-validation'

    // Testing
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

#### Step 2: Create Folder Structure

```
src/main/java/com/example/taskapi/
├── TaskApiApplication.java
├── config/
├── controller/ -- http request
│   └── TaskController.java
├── dto/
│   ├── request/ -- in and out 
│   │   └── CreateTaskRequest.java
│   └── response/
│       └── TaskResponse.java
├── entity/ -- db model 
│   └── Task.java
├── repository/ -- db access
│   └── TaskRepository.java
├── service/ -- business logic
│   └── TaskService.java
└── exception/ -- error handling
    └── ResourceNotFoundException.java

src/main/resources/
└── application.properties
```

#### Step 3: Create the Entity (Database Model)

**File: `src/main/java/com/example/taskapi/entity/Task.java`**

```java
package com.example.taskapi.entity;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import javax.persistence.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "tasks")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Task {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(length = 500)
    private String description;

    @Column(nullable = false)
    @Enumerated(EnumType.STRING)
    private TaskStatus status = TaskStatus.PENDING; // PENDING, IN_PROGRESS, COMPLETED

    @Column(nullable = false)
    private Integer priority = 1; // 1=low, 2=medium, 3=high

    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }

    public enum TaskStatus {
        PENDING, IN_PROGRESS, COMPLETED, BLOCKED
    }
}
```

**Explanation:**

- `@Entity` - This is a database table
- `@Table(name = "tasks")` - Maps to "tasks" table in DB
- `@Id` - Primary key
- `@GeneratedValue` - Auto-increment ID
- `@Column` - Column properties
- `@Enumerated` - Store enum as string
- `@PrePersist/@PreUpdate` - Auto-set timestamps
- `@Data` (Lombok) - Auto-generates getters, setters, toString
- `@Builder` - Allows Task.builder().title("...").build()

#### Step 4: Create the Repository (Database Access)

**File: `src/main/java/com/example/taskapi/repository/TaskRepository.java`**

```java
package com.example.taskapi.repository;

import com.example.taskapi.entity.Task;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.List;

@Repository
public interface TaskRepository extends JpaRepository<Task, Long> {

    // JPA automatically implements these methods
    List<Task> findByStatus(Task.TaskStatus status);

    List<Task> findByPriority(Integer priority);

    List<Task> findByTitleContainingIgnoreCase(String title);
}
```

**How JPA Works:**
Spring Data JPA reads your method names and generates SQL automatically!

- `findByStatus(status)` → `SELECT * FROM tasks WHERE status = ?`
- `findByPriority(priority)` → `SELECT * FROM tasks WHERE priority = ?`
- `findByTitleContainingIgnoreCase(title)` → `SELECT * FROM tasks WHERE LOWER(title) LIKE LOWER(?)`

#### Step 5: Create Request DTOs

**File: `src/main/java/com/example/taskapi/dto/request/CreateTaskRequest.java`**

```java
package com.example.taskapi.dto.request;

import lombok.Data;
import javax.validation.constraints.NotBlank;
import javax.validation.constraints.Size;

@Data
public class CreateTaskRequest {

    @NotBlank(message = "Title is required")
    @Size(min = 3, max = 200, message = "Title must be between 3 and 200 characters")
    private String title;

    @Size(max = 500, message = "Description max 500 characters")
    private String description;

    private Integer priority = 1;
}
```

**File: `src/main/java/com/example/taskapi/dto/request/UpdateTaskRequest.java`**

```java
package com.example.taskapi.dto.request;

import com.example.taskapi.entity.Task.TaskStatus;
import lombok.Data;

@Data
public class UpdateTaskRequest {
    private String title;
    private String description;
    private TaskStatus status;
    private Integer priority;
}
```

#### Step 6: Create Response DTO

**File: `src/main/java/com/example/taskapi/dto/response/TaskResponse.java`**

```java
package com.example.taskapi.dto.response;

import com.example.taskapi.entity.Task;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import java.time.LocalDateTime;

@Data
@Builder
@AllArgsConstructor
public class TaskResponse {
    private Long id;
    private String title;
    private String description;
    private String status;
    private Integer priority;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    // Convert Entity to DTO
    public static TaskResponse fromEntity(Task task) {
        return TaskResponse.builder()
                .id(task.getId())
                .title(task.getTitle())
                .description(task.getDescription())
                .status(task.getStatus().toString())
                .priority(task.getPriority())
                .createdAt(task.getCreatedAt())
                .updatedAt(task.getUpdatedAt())
                .build();
    }
}
```

#### Step 7: Create Service (Business Logic)

**File: `src/main/java/com/example/taskapi/service/TaskService.java`**

```java
package com.example.taskapi.service;

import com.example.taskapi.dto.request.CreateTaskRequest;
import com.example.taskapi.dto.request.UpdateTaskRequest;
import com.example.taskapi.entity.Task;
import com.example.taskapi.repository.TaskRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class TaskService {

    @Autowired
    private TaskRepository taskRepository;

    // Create a new task
    public Task createTask(CreateTaskRequest request) {
        Task task = Task.builder()
                .title(request.getTitle())
                .description(request.getDescription())
                .priority(request.getPriority() != null ? request.getPriority() : 1)
                .status(Task.TaskStatus.PENDING)
                .build();
        return taskRepository.save(task);
    }

    // Get all tasks
    public List<Task> getAllTasks() {
        return taskRepository.findAll();
    }

    // Get task by ID
    public Task getTaskById(Long id) {
        return taskRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Task not found with id: " + id));
    }

    // Get tasks by status
    public List<Task> getTasksByStatus(Task.TaskStatus status) {
        return taskRepository.findByStatus(status);
    }

    // Update task
    public Task updateTask(Long id, UpdateTaskRequest request) {
        Task task = getTaskById(id);

        if (request.getTitle() != null) {
            task.setTitle(request.getTitle());
        }
        if (request.getDescription() != null) {
            task.setDescription(request.getDescription());
        }
        if (request.getStatus() != null) {
            task.setStatus(request.getStatus());
        }
        if (request.getPriority() != null) {
            task.setPriority(request.getPriority());
        }

        return taskRepository.save(task);
    }

    // Delete task
    public void deleteTask(Long id) {
        Task task = getTaskById(id);
        taskRepository.delete(task);
    }
}
```

#### Step 8: Create Controller (HTTP Endpoints)

**File: `src/main/java/com/example/taskapi/controller/TaskController.java`**

```java
package com.example.taskapi.controller;

import com.example.taskapi.dto.request.CreateTaskRequest;
import com.example.taskapi.dto.request.UpdateTaskRequest;
import com.example.taskapi.dto.response.TaskResponse;
import com.example.taskapi.entity.Task;
import com.example.taskapi.service.TaskService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import javax.validation.Valid;
import java.util.List;
import java.util.stream.Collectors;

@RestController
@RequestMapping("/api/tasks")
public class TaskController {

    @Autowired
    private TaskService taskService;

    // Create Task
    // POST /api/tasks
    @PostMapping
    public ResponseEntity<TaskResponse> createTask(@Valid @RequestBody CreateTaskRequest request) {
        Task task = taskService.createTask(request);
        return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(TaskResponse.fromEntity(task));
    }

    // Get All Tasks
    // GET /api/tasks
    @GetMapping
    public ResponseEntity<List<TaskResponse>> getAllTasks() {
        List<Task> tasks = taskService.getAllTasks();
        List<TaskResponse> responses = tasks.stream()
                .map(TaskResponse::fromEntity)
                .collect(Collectors.toList());
        return ResponseEntity.ok(responses);
    }

    // Get Task By ID
    // GET /api/tasks/{id}
    @GetMapping("/{id}")
    public ResponseEntity<TaskResponse> getTaskById(@PathVariable Long id) {
        Task task = taskService.getTaskById(id);
        return ResponseEntity.ok(TaskResponse.fromEntity(task));
    }

    // Get Tasks by Status
    // GET /api/tasks?status=IN_PROGRESS
    @GetMapping(params = "status")
    public ResponseEntity<List<TaskResponse>> getTasksByStatus(@RequestParam Task.TaskStatus status) {
        List<Task> tasks = taskService.getTasksByStatus(status);
        List<TaskResponse> responses = tasks.stream()
                .map(TaskResponse::fromEntity)
                .collect(Collectors.toList());
        return ResponseEntity.ok(responses);
    }

    // Update Task
    // PUT /api/tasks/{id}
    @PutMapping("/{id}")
    public ResponseEntity<TaskResponse> updateTask(
            @PathVariable Long id,
            @Valid @RequestBody UpdateTaskRequest request) {
        Task task = taskService.updateTask(id, request);
        return ResponseEntity.ok(TaskResponse.fromEntity(task));
    }

    // Delete Task
    // DELETE /api/tasks/{id}
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteTask(@PathVariable Long id) {
        taskService.deleteTask(id);
        return ResponseEntity.noContent().build();
    }
}
```

**Explanation:**

- `@RestController` - Returns JSON/XML (not HTML)
- `@RequestMapping("/api/tasks")` - Base path for all endpoints
- `@PostMapping`, `@GetMapping`, etc. - HTTP method + path mapping
- `@PathVariable` - Get value from URL (`/tasks/{id}`)
- `@RequestParam` - Get value from query string (`?status=PENDING`)
- `@Valid` - Validate request using annotations
- `ResponseEntity` - Control HTTP status, headers, body

#### Step 9: Create Configuration

**File: `src/main/resources/application.properties`**

```properties
# Database Configuration
spring.datasource.url=jdbc:mysql://localhost:3306/task_api
spring.datasource.username=root
spring.datasource.password=your_password
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# JPA/Hibernate Configuration
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL8Dialect

# Server
server.port=8080

# Logging
logging.level.root=INFO
logging.level.com.example.taskapi=DEBUG
```

#### Step 10: Create Main Application Class

**File: `src/main/java/com/example/taskapi/TaskApiApplication.java`**

```java
package com.example.taskapi;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class TaskApiApplication {

    public static void main(String[] args) {
        SpringApplication.run(TaskApiApplication.class, args);
    }
}
```

### Running Your API

```bash
# Build
./gradlew clean build

# Run
./gradlew bootRun

# Server starts on http://localhost:8080
```

### Testing Your API

Using curl or Postman:

```bash
# Create a task
curl -X POST http://localhost:8080/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Learn Spring Boot", "description": "Complete tutorial", "priority": 3}'

# Get all tasks
curl http://localhost:8080/api/tasks

# Get task by ID
curl http://localhost:8080/api/tasks/1

# Get tasks by status
curl "http://localhost:8080/api/tasks?status=PENDING"

# Update task
curl -X PUT http://localhost:8080/api/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{"status": "IN_PROGRESS"}'

# Delete task
curl -X DELETE http://localhost:8080/api/tasks/1
```

---

# Part 4: Using GitHub Copilot Effectively

## What is GitHub Copilot?

GitHub Copilot is an AI coding assistant that:

- **Predicts** what you're typing and suggests completions
- **Generates** boilerplate code from comments
- **Explains** code functionality
- **Fixes** bugs and suggests improvements

## How to Use Copilot for Spring Boot Learning

### 1. **Code Generation from Comments**

Type a comment describing what you want, Copilot generates code:

```java
// Service method example
public class EmployeeService {
    @Autowired
    private EmployeeRepository repository;

    // Get all employees sorted by name
    public List<Employee> getAllEmployeesSortedByName() {
        // Copilot suggests: return repository.findAll(Sort.by("name"));
    }
}
```

**How to use:**

1. Type comment describing the method
2. Press `Ctrl+Enter` to open suggestions
3. Arrow keys to browse, `Tab` to accept

### 2. **Method Signature Completion**

Start typing a method, Copilot completes it:

```java
public List<Employee> findByCompanyAndStatus(Long companyId, String status) {
    // Copilot suggests the implementation
}
```

### 3. **Repository Method Generation**

Copilot understands JPA naming conventions:

```java
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    // Type: List<Employee> findBy
    // Copilot suggests: findByCompanyIdAndStatus()
    // Or: findByDepartmentAndSalaryGreaterThan()
}
```

### 4. **DTO Creation**

Describe what you need:

```java
// DTO for creating a new employee with validation
public class CreateEmployeeRequest {
    // Copilot auto-generates with @NotBlank, @Email, etc.
}
```

### 5. **Test Case Generation**

```java
// Test method for creating employee
@Test
public void testCreateEmployeeSuccess() {
    // Copilot suggests full test with mocks and assertions
}
```

## Best Practices with Copilot

### DO:

✅ **Use specific comments** - "Get employees earning > $50k sorted by salary"
✅ **Show examples** - Provide similar code patterns above
✅ **Review suggestions** - Don't blindly accept all suggestions
✅ **Use for repetitive code** - DTOs, entity mappings, test boilerplate
✅ **Ask for refactoring** - "Extract this to a separate method"
✅ **Learn from suggestions** - Understand why it suggests certain approaches

### DON'T:

❌ **Don't rely 100%** - Verify logic, especially for business rules
❌ **Don't accept wrong suggestions** - Red squiggly lines? Fix before proceeding
❌ **Don't skip security** - Always review authentication/authorization code
❌ **Don't forget tests** - Copilot suggestions may not have edge cases covered
❌ **Don't ignore deprecations** - Check if suggested classes are outdated

## Copilot Prompting Tips for Spring Boot

### Technique 1: Show Context

```java
// Pattern in file:
@Service
public class EmployeeService {
    @Autowired
    private EmployeeRepository repository;

    public Employee getById(Long id) {
        return repository.findById(id).orElseThrow(() -> new ResourceNotFoundException("Employee not found"));
    }

    // Now start typing next method - Copilot knows your pattern
    public List<Employee> getActive() {
        // Copilot will suggest filtered list with custom exception
    }
}
```

### Technique 2: Comment-Driven Development

```java
// Bad comment (too vague):
// Get data

// Better comment:
// Retrieve all active employees for given company, sorted by hire date, exclude soft-deleted

public List<Employee> getActiveEmployeesByCompany(Long companyId) {
    // Copilot generates SQL/JPQL accurately from description
}
```

### Technique 3: Describe Error Handling

```java
// Create employee with validation, throw specific exceptions
public Employee createEmployee(CreateEmployeeRequest request) throws DuplicateEmailException, InvalidDataException {
    // Copilot generates proper validation checks
}
```

### Technique 4: Chain-Prompt for Complex Logic

```java
@Transactional
public PayslipResponse generatePayslip(Long employeeId, YearMonth month) {
    // Step 1: Fetch employee and validate exists
    // Step 2: Calculate gross salary from employee grade
    // Step 3: Calculate deductions (tax, insurance)
    // Step 4: Save payslip record
    // Step 5: Clear related cache
    // Step 6: Return response

    // Copilot will implement all steps when you press Alt+\
}
```

---

# Part 5: Best Practices & Common Patterns

## 1. **Proper Exception Handling**

❌ **Bad:**

```java
public Employee getEmployee(Long id) {
    return repo.findById(id).get(); // Throws NoSuchElementException
}
```

✅ **Good:**

```java
@Service
public class EmployeeService {
    public Employee getEmployee(Long id) {
        return repository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Employee", "id", id));
    }
}

// Create custom exception
@ResponseStatus(HttpStatus.NOT_FOUND)
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String resource, String field, Object value) {
        super(String.format("%s not found with %s: '%s'", resource, field, value));
    }
}
```

## 2. **Proper Validation**

✅ **Validate at DTO level:**

```java
@Data
public class CreateEmployeeRequest {
    @NotBlank(message = "Name cannot be blank")
    @Size(min = 2, max = 100)
    private String name;

    @Email(message = "Invalid email format")
    private String email;

    @Positive(message = "Salary must be greater than 0")
    private BigDecimal salary;
}

// Controller automatically validates
@PostMapping
public ResponseEntity<EmployeeResponse> create(@Valid @RequestBody CreateEmployeeRequest request) {
    // request is guaranteed valid here
}
```

## 3. **Proper Pagination**

```java
// Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    Page<Employee> findByCompanyId(Long companyId, Pageable pageable);
}

// Service
public Page<EmployeeResponse> getEmployees(Long companyId, int page, int size) {
    Pageable pageable = PageRequest.of(page, size, Sort.by("name"));
    return employeeRepository.findByCompanyId(companyId, pageable)
            .map(EmployeeResponse::fromEntity);
}

// Controller
@GetMapping
public ResponseEntity<Page<EmployeeResponse>> list(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size) {
    return ResponseEntity.ok(employeeService.getEmployees(companyId, page, size));
}

// Client calls: GET /employees?page=0&size=10
```

## 4. **Proper Transaction Management**

```java
@Service
public class PayrollService {
    @Autowired
    private EmployeeRepository employeeRepository;

    @Autowired
    private PayslipRepository payslipRepository;

    // All or nothing: if generatePayslip fails, employee update rolls back
    @Transactional
    public void processMonthlyPayroll(YearMonth month) {
        List<Employee> employees = employeeRepository.findActive();
        for (Employee employee : employees) {
            Payslip payslip = generatePayslip(employee, month);
            payslipRepository.save(payslip);
            // If exception here, everything rolls back
        }
    }
}
```

## 5. **Proper Caching**

```java
@Service
public class EmployeeService {

    // Cache result for 1 hour
    @Cacheable(value = "employeeProfile", key = "#employeeId")
    public EmployeeResponse getEmployee(Long employeeId) {
        return employeeRepository.findById(employeeId)
                .map(EmployeeResponse::fromEntity)
                .orElseThrow();
    }

    // Clear cache when updating
    @CacheEvict(value = "employeeProfile", key = "#employeeId")
    public EmployeeResponse updateEmployee(Long employeeId, UpdateRequest request) {
        Employee employee = employeeRepository.findById(employeeId).orElseThrow();
        // ... update logic
        return EmployeeResponse.fromEntity(employeeRepository.save(employee));
    }

    // Clear all cache of this value
    @CacheEvict(value = "employeeProfile", allEntries = true)
    public void refreshAllEmployees() {
        // Called when significant data changes
    }
}
```

## 6. **Proper Logging**

✅ **Good logging:**

```java
@Service
public class EmployeeService {

    private static final Logger logger = LoggerFactory.getLogger(EmployeeService.class);

    public void deleteEmployee(Long id) {
        logger.info("Deleting employee with id: {}", id);
        try {
            employeeRepository.deleteById(id);
            logger.info("Successfully deleted employee with id: {}", id);
        } catch (Exception e) {
            logger.error("Error deleting employee with id: {}", id, e);
            throw new ServiceException("Failed to delete employee", e);
        }
    }
}
```

## 7. **Proper API Versioning**

```java
// API v1
@RestController
@RequestMapping("/api/v1/employees")
public class EmployeeControllerV1 { ... }

// API v2 with new fields
@RestController
@RequestMapping("/api/v2/employees")
public class EmployeeControllerV2 { ... }

// Both available simultaneously, clients can migrate gradually
```

## 8. **Proper DTO Mapping**

```java
// Bad: Manual mapping everywhere
EmployeeResponse response = new EmployeeResponse();
response.setId(employee.getId());
response.setName(employee.getName());
// ... 20 more lines

// Better: Centralized conversion
public class EmployeeResponse {
    public static EmployeeResponse fromEntity(Employee employee) {
        return EmployeeResponse.builder()
                .id(employee.getId())
                .name(employee.getName())
                // ...
                .build();
    }
}

// Or use MapStruct library for complex mappings
```

## 9. **Proper Testing Structure**

```java
// Unit Test (test business logic in isolation)
@SpringBootTest
class EmployeeServiceTest {
    @MockBean
    private EmployeeRepository repository;

    @InjectMocks
    private EmployeeService service;

    @Test
    void testGetEmployeeSuccess() {
        // Arrange
        Employee expected = new Employee(1L, "John", "john@example.com");
        when(repository.findById(1L)).thenReturn(Optional.of(expected));

        // Act
        EmployeeResponse result = service.getEmployee(1L);

        // Assert
        assertEquals("John", result.getName());
        verify(repository).findById(1L);
    }
}

// Integration Test (test with real database)
@SpringBootTest
@TestPropertySource(locations = "classpath:application-test.properties")
class EmployeeControllerIntegrationTest {
    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void testCreateEmployeeEndToEnd() {
        // Makes real HTTP request to real database
        ResponseEntity<EmployeeResponse> response = restTemplate.postForEntity(
            "/api/employees",
            new CreateEmployeeRequest("John", "john@example.com", BigDecimal.valueOf(50000)),
            EmployeeResponse.class
        );

        assertEquals(HttpStatus.CREATED, response.getStatusCode());
    }
}
```

---

# Part 6: Resources & Further Learning

## Official Documentation

- **Spring Boot Docs**: https://spring.io/projects/spring-boot
- **Spring Data JPA**: https://spring.io/projects/spring-data-jpa
- **Spring Security**: https://spring.io/projects/spring-security
- **Spring REST Docs**: https://spring.io/projects/spring-restdocs

## Learning Platforms

1. **Udemy Courses**

   - "Spring Boot 3 and Spring Framework 6" - Udemy
   - "Master Spring Boot Development" - Spring Academy

2. **YouTube Channels**

   - Java Brains - Deep dive into Spring concepts
   - Tech Primers - Spring Boot beginner tutorials
   - Programming with Mosh - Full Spring Boot course

3. **Interactive Learning**

   - Spring Academy (https://spring.academy) - Free official courses
   - LeetCode + Spring Boot Problems - Practice APIs
   - Baeldung Tutorial Blog - Specific Spring topics

4. **Books**
   - "Spring in Action" (6th Edition) by Craig Walls
   - "Learning Spring Boot" by Greg L. Turnquist

## Practice Projects (Difficulty: Easy → Hard)

### Project 1: Blog API (Beginner)

- Create posts, comments, tags
- User authentication
- Pagination & search
- **Skills**: CRUD, JPA, Basic validation

### Project 2: E-Commerce API (Intermediate)

- Products, categories, shopping cart
- Orders and order history
- User roles (customer, admin)
- **Skills**: Complex relationships, authorization, transactions

### Project 3: Project Management (Advanced)

- Multi-user projects
- Task management with status tracking
- Time logging
- Reports & analytics
- **Skills**: Advanced queries, caching, multi-tenancy

### Project 4: Social Media API (Expert)

- User profiles, followers, posts
- Comments and likes
- Real-time notifications
- **Skills**: WebSockets, advanced caching, complex queries

## Copilot Learning Exercises

### Exercise 1: Generate CRUD Service

```java
// Write comment, let Copilot generate service
// Create complete CRUD service for Product entity
// Includes: create, read, update, delete, getAllPaginated, search
```

### Exercise 2: Write Repository Methods

```java
// Challenge: Describe complex queries in comments
// Let Copilot write method signatures
// Example: "Find products by category, price range, in stock, sorted by rating"
```

### Exercise 3: Generate Test Cases

```java
// Write test class structure
// Let Copilot suggest all test methods
// Review and understand test patterns
```

### Exercise 4: Refactor Legacy Code

```java
// Copy code with issues
// Ask Copilot: "Refactor this to use spring security properly"
// Compare before/after
```

## Quick Reference: Common Annotations

| Annotation               | Purpose                   |
| ------------------------ | ------------------------- |
| `@SpringBootApplication` | Main entry point          |
| `@RestController`        | REST API endpoints        |
| `@Service`               | Business logic            |
| `@Repository`            | Database access           |
| `@Autowired`             | Dependency injection      |
| `@GetMapping`            | Handle GET requests       |
| `@PostMapping`           | Handle POST requests      |
| `@RequestMapping`        | Base URL path             |
| `@PathVariable`          | Extract from URL          |
| `@RequestParam`          | Extract from query string |
| `@RequestBody`           | Parse JSON body           |
| `@Valid`                 | Validate request          |
| `@Entity`                | Database table            |
| `@Table`                 | Table name mapping        |
| `@Id`                    | Primary key               |
| `@GeneratedValue`        | Auto-increment            |
| `@Column`                | Column properties         |
| `@OneToMany`             | Relationship (1:N)        |
| `@ManyToOne`             | Relationship (N:1)        |
| `@Transactional`         | Transaction management    |
| `@Cacheable`             | Cache result              |
| `@CacheEvict`            | Clear cache               |
| `@PreAuthorize`          | Method-level security     |

## Debugging Tips

### 1. Enable Debug Logging

```properties
logging.level.org.springframework.web=DEBUG
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

### 2. Use VS Code Debugger

- Set breakpoints (click left of line number)
- F5 to start debugging
- F10 = step over, F11 = step into
- Inspect variables in Variables panel

### 3. Common Errors & Solutions

| Error                    | Cause                             | Solution                                     |
| ------------------------ | --------------------------------- | -------------------------------------------- |
| `NoSuchElementException` | `.get()` on empty Optional        | Use `.orElseThrow()` or `.orElse(null)`      |
| `DataAccessException`    | Database connection issue         | Check application.properties, MySQL running  |
| `404 Not Found`          | Wrong URL or mapping              | Check `@RequestMapping`, `@GetMapping` paths |
| `400 Bad Request`        | Invalid JSON or validation failed | Check JSON format, validation annotations    |
| `Circular dependency`    | Beans depend on each other        | Refactor to break cycle, use `@Lazy`         |

---

## Summary: Learning Path Timeline

**Week 1:** Understand fundamentals (annotations, DI, MVC)
**Week 2:** Build Task API from scratch
**Week 3:** Add validation, caching, error handling
**Week 4:** Study your existing project structure
**Week 5:** Small features in existing project using Copilot
**Week 6-8:** Build independent project with team feedback

---

**Now you're ready to start learning Spring Boot!** Begin with the Task API project, use Copilot for suggestions, and gradually move to your actual project.
