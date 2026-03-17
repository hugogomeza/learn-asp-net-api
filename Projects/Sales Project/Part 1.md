# Sales Project - Part 1: ASP.NET Web API Fundamentals

## Goal

Build a Web API to manage a product catalog, customers, and the basic operations of a sales system. In this first part we focus on:

- Creating the ASP.NET Web API project
- Defining the data model with Entity Framework Core (Code First)
- Implementing Controllers with CRUD operations
- Using HTTP methods correctly
- Implementing search filters and pagination
- Validating input data

By the end of this part, you will have a functional API with endpoints for **Products**, **Categories**, and **Customers**.

---

## 1. Data Model

### Entity Diagram (Part 1)

```
┌──────────────┐       ┌──────────────────┐
│  Categories  │       │    Products      │
├──────────────┤       ├──────────────────┤
│ Id (PK)      │──┐    │ Id (PK)          │
│ Name         │  └───>│ CategoryId (FK)  │
│ CreatedAt    │       │ Name             │
│ UpdatedAt    │       │ Sku              │
│              │       │ Description      │
│              │       │ Price            │
│              │       │ IsActive         │
│              │       │ CreatedAt        │
│              │       │ UpdatedAt        │
└──────────────┘       └──────────────────┘

┌─────────────────────┐
│     Customers       │
├─────────────────────┤
│ Id (PK)             │
│ Name                │
│ LastName            │
│ Email               │
│ PhoneNumber         │
│ CompanyName         │
│ CreatedAt           │
│ UpdatedAt           │
└─────────────────────┘
```

> **Note:** The `Sale`, `SaleLine`, `Payment`, and `PaymentApplication` entities will be introduced in Part 2 and Part 3. For now, we focus on the catalog and customers.

### Entity Definitions in C\#

#### Category

```csharp
public class Category
{
    public int Id { get; set; }

    [Required]
    [MaxLength(100)]
    public string Name { get; set; } = string.Empty;

    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }

    // Navigation properties
    public ICollection<Product> Products { get; set; } = new List<Product>();
}
```

#### Product

```csharp
public class Product
{
    public int Id { get; set; }

    [Required]
    [MaxLength(250)]
    public string Name { get; set; } = string.Empty;

    [Required]
    [MaxLength(45)]
    public string Sku { get; set; } = string.Empty;

    [MaxLength(500)]
    public string? Description { get; set; }

    [Required]
    [Column(TypeName = "decimal(14,4)")]
    public decimal Price { get; set; }

    public bool IsActive { get; set; } = true;

    public int CategoryId { get; set; }

    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }

    // Navigation properties
    public Category Category { get; set; } = null!;
}
```

#### Customer

```csharp
public class Customer
{
    public int Id { get; set; }

    [Required]
    [MaxLength(120)]
    public string Name { get; set; } = string.Empty;

    [MaxLength(120)]
    public string? LastName { get; set; }

    [MaxLength(50)]
    [EmailAddress]
    public string? Email { get; set; }

    [MaxLength(20)]
    public string? PhoneNumber { get; set; }

    [MaxLength(200)]
    public string? CompanyName { get; set; }

    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}
```

---

## 2. Naming Conventions

### General C\# / .NET Conventions

| Element | Convention | Example |
|---|---|---|
| Classes and Records | PascalCase | `Product`, `SaleLine` |
| Public properties | PascalCase | `FirstName`, `IsActive` |
| Local variables | camelCase | `productName`, `totalAmount` |
| Method parameters | camelCase | `int categoryId` |
| Constants | PascalCase | `MaxPageSize` |
| Interfaces | PascalCase with "I" prefix | `IProductRepository` |
| Methods | PascalCase | `GetProducts()`, `CreateSale()` |
| Private fields | camelCase with "_" prefix | `_context`, `_logger` |
| Enums | PascalCase (singular) | `PaymentStatus`, `PaymentType` |
| Enum values | PascalCase | `PaymentStatus.Pending` |

### Project-Specific Conventions

| Element | Convention | Example |
|---|---|---|
| Project name | `SalesProject.Api` | — |
| Models folder | `Models/` | `Models/Product.cs` |
| Controllers folder | `Controllers/` | `Controllers/ProductsController.cs` |
| Controller names | Plural + "Controller" | `ProductsController`, `CategoriesController` |
| DbContext name | Domain + "DbContext" | `SalesDbContext` |
| DTO names | Entity + Action + "Dto" | `CreateProductDto`, `ProductResponseDto` |
| API routes | Plural, kebab-case for compounds | `/api/products`, `/api/sale-lines` |
| DB tables | Plural, PascalCase (EF default) | `Products`, `Categories` |

### Recommended Folder Structure

```
SalesProject.Api/
├── Controllers/
│   ├── CategoriesController.cs
│   ├── CustomersController.cs
│   └── ProductsController.cs
├── Data/
│   └── SalesDbContext.cs
├── Models/
│   ├── Category.cs
│   ├── Customer.cs
│   └── Product.cs
├── Dtos/
│   ├── Categories/
│   │   ├── CreateCategoryDto.cs
│   │   └── CategoryResponseDto.cs
│   ├── Customers/
│   │   ├── CreateCustomerDto.cs
│   │   └── CustomerResponseDto.cs
│   └── Products/
│       ├── CreateProductDto.cs
│       ├── UpdateProductDto.cs
│       └── ProductResponseDto.cs
├── Program.cs
├── appsettings.json
└── SalesProject.Api.csproj
```

---

## 3. Technical Considerations

### 3.1 Entity Framework Core - Code First

**Why Code First?**
We define entities as C# classes and EF Core generates the database from them. This allows us to:
- Version the DB schema alongside the code
- Use migrations to evolve the DB in a controlled manner
- Keep the entities as the single source of truth

**Required NuGet packages:**

```bash
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.SqlServer    # or .Sqlite for local development
dotnet add package Microsoft.EntityFrameworkCore.Tools
dotnet add package Microsoft.EntityFrameworkCore.Design
```

**DbContext:**

```csharp
public class SalesDbContext : DbContext
{
    public SalesDbContext(DbContextOptions<SalesDbContext> options) : base(options) { }

    public DbSet<Product> Products => Set<Product>();
    public DbSet<Category> Categories => Set<Category>();
    public DbSet<Customer> Customers => Set<Customer>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Configure unique index on Product.Sku
        modelBuilder.Entity<Product>()
            .HasIndex(p => p.Sku)
            .IsUnique();

        // Configure unique index on Category.Name
        modelBuilder.Entity<Category>()
            .HasIndex(c => c.Name)
            .IsUnique();
    }
}
```

**Registration in Program.cs:**

```csharp
builder.Services.AddDbContext<SalesDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
```

**Migrations:**

```bash
# Create the initial migration
dotnet ef migrations add InitialCreate

# Apply the migration to the database
dotnet ef database update
```

### 3.2 HTTP Methods and Their Correct Usage

| Method | Usage | Example | Successful Response |
|---|---|---|---|
| `GET` | Retrieve resources | `GET /api/products` | 200 OK |
| `GET` | Retrieve a resource by ID | `GET /api/products/5` | 200 OK or 404 Not Found |
| `POST` | Create a new resource | `POST /api/products` | 201 Created |
| `PUT` | Update an entire resource | `PUT /api/products/5` | 200 OK or 204 No Content |
| `DELETE` | Delete a resource | `DELETE /api/products/5` | 204 No Content |

> **Tip:** `POST` is not idempotent (each call creates a new resource). `PUT` and `DELETE` are idempotent (calling them multiple times produces the same result).

### 3.3 Common HTTP Status Codes

| Code | Meaning | When to Use |
|---|---|---|
| 200 | OK | Successful response with body |
| 201 | Created | Resource created successfully |
| 204 | No Content | Successful operation without response body |
| 400 | Bad Request | Invalid input data |
| 404 | Not Found | Resource not found |
| 409 | Conflict | Conflict (e.g., duplicate SKU) |
| 500 | Internal Server Error | Unexpected server error |

### 3.4 Dependency Injection

ASP.NET Core has built-in dependency injection (DI). The `DbContext` is registered as a service and injected into controllers:

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly SalesDbContext _context;

    public ProductsController(SalesDbContext context)
    {
        _context = context;
    }
}
```

> In this first part, we use the DbContext directly in controllers. In Part 4, we will migrate to the Repository Pattern.

---

## 4. User Stories

### US-01: Category Management

**As** a system administrator,
**I want** to create, view, update, and delete product categories,
**so that** I can organize the product catalog.

**Acceptance Criteria:**

- [ ] `POST /api/categories` — Create a category. Returns 201 with the created category.
  - Validate that `Name` is required and does not exceed 100 characters.
  - Validate that no other category with the same name exists (return 409).
- [ ] `GET /api/categories` — List all categories. Returns 200.
- [ ] `GET /api/categories/{id}` — Get a category by ID. Returns 200 or 404.
- [ ] `PUT /api/categories/{id}` — Update a category. Returns 200 or 404.
  - Validate that `Name` is required.
  - Validate that no other category with the same name exists (excluding the current one).
- [ ] `DELETE /api/categories/{id}` — Delete a category. Returns 204 or 404.
  - Do not allow deletion if the category has associated products (return 409).

**Endpoints:**

```
POST   /api/categories
GET    /api/categories
GET    /api/categories/{id}
PUT    /api/categories/{id}
DELETE /api/categories/{id}
```

**Suggested DTOs:**

```csharp
// Request
public class CreateCategoryDto
{
    [Required]
    [MaxLength(100)]
    public string Name { get; set; } = string.Empty;
}

// Response
public class CategoryResponseDto
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}
```

---

### US-02: Product Management

**As** a system administrator,
**I want** to register, view, update, and delete products,
**so that** I can keep the catalog up to date.

**Acceptance Criteria:**

- [ ] `POST /api/products` — Create a product. Returns 201 with the created product.
  - Validate required fields: `Name`, `Sku`, `Price`, `CategoryId`.
  - Validate that `Sku` is unique (return 409 if it already exists).
  - Validate that `CategoryId` exists (return 400 if it doesn't).
  - Validate that `Price` is greater than 0.
- [ ] `GET /api/products` — List products with support for:
  - **Filters:** by `name` (contains), by `categoryId`, by `isActive`.
  - **Pagination:** `page` and `pageSize` parameters (default: page=1, pageSize=10, max: 50).
  - Return pagination metadata in the response body.
- [ ] `GET /api/products/{id}` — Get a product by ID including its category. Returns 200 or 404.
- [ ] `PUT /api/products/{id}` — Update a product. Returns 200 or 404.
  - Same validations as creation.
- [ ] `DELETE /api/products/{id}` — Delete a product. Returns 204 or 404.

**Endpoints:**

```
POST   /api/products
GET    /api/products
GET    /api/products?name=coca&categoryId=1&isActive=true&page=1&pageSize=10
GET    /api/products/{id}
PUT    /api/products/{id}
DELETE /api/products/{id}
```

**Suggested DTOs:**

```csharp
// Request - Create
public class CreateProductDto
{
    [Required]
    [MaxLength(250)]
    public string Name { get; set; } = string.Empty;

    [Required]
    [MaxLength(45)]
    public string Sku { get; set; } = string.Empty;

    [MaxLength(500)]
    public string? Description { get; set; }

    [Required]
    [Range(0.01, double.MaxValue, ErrorMessage = "Price must be greater than 0")]
    public decimal Price { get; set; }

    [Required]
    public int CategoryId { get; set; }

    public bool IsActive { get; set; } = true;
}

// Request - Update
public class UpdateProductDto
{
    [Required]
    [MaxLength(250)]
    public string Name { get; set; } = string.Empty;

    [MaxLength(500)]
    public string? Description { get; set; }

    [Required]
    [Range(0.01, double.MaxValue)]
    public decimal Price { get; set; }

    [Required]
    public int CategoryId { get; set; }

    public bool IsActive { get; set; } = true;
}

// Response
public class ProductResponseDto
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Sku { get; set; } = string.Empty;
    public string? Description { get; set; }
    public decimal Price { get; set; }
    public bool IsActive { get; set; }
    public int CategoryId { get; set; }
    public string CategoryName { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}
```

**Paged response model:**

```csharp
public class PagedResponse<T>
{
    public List<T> Items { get; set; } = new();
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalCount { get; set; }
    public int TotalPages => (int)Math.Ceiling((double)TotalCount / PageSize);
    public bool HasPreviousPage => Page > 1;
    public bool HasNextPage => Page < TotalPages;
}
```

**Filter and pagination example in the controller:**

```csharp
[HttpGet]
public async Task<ActionResult<PagedResponse<ProductResponseDto>>> GetProducts(
    [FromQuery] string? name,
    [FromQuery] int? categoryId,
    [FromQuery] bool? isActive,
    [FromQuery] int page = 1,
    [FromQuery] int pageSize = 10)
{
    // Limit page size
    pageSize = Math.Min(pageSize, 50);

    // Build query with filters
    var query = _context.Products
        .Include(p => p.Category)
        .AsQueryable();

    if (!string.IsNullOrWhiteSpace(name))
        query = query.Where(p => p.Name.Contains(name));

    if (categoryId.HasValue)
        query = query.Where(p => p.CategoryId == categoryId.Value);

    if (isActive.HasValue)
        query = query.Where(p => p.IsActive == isActive.Value);

    // Get total count before paginating
    var totalCount = await query.CountAsync();

    // Apply pagination
    var products = await query
        .OrderBy(p => p.Name)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .ToListAsync();

    // Map to DTOs
    var items = products.Select(p => new ProductResponseDto
    {
        Id = p.Id,
        Name = p.Name,
        Sku = p.Sku,
        Description = p.Description,
        Price = p.Price,
        IsActive = p.IsActive,
        CategoryId = p.CategoryId,
        CategoryName = p.Category.Name,
        CreatedAt = p.CreatedAt,
        UpdatedAt = p.UpdatedAt
    }).ToList();

    var response = new PagedResponse<ProductResponseDto>
    {
        Items = items,
        Page = page,
        PageSize = pageSize,
        TotalCount = totalCount
    };

    return Ok(response);
}
```

---

### US-03: Customer Management

**As** a system administrator,
**I want** to register, view, update, and delete customers,
**so that** I can associate them with sales.

**Acceptance Criteria:**

- [ ] `POST /api/customers` — Create a customer. Returns 201 with the created customer.
  - Validate that `Name` is required.
  - Validate `Email` format if provided.
- [ ] `GET /api/customers` — List customers with support for:
  - **Filters:** by `name` (searches in Name and LastName), by `email`.
  - **Pagination:** `page` and `pageSize` parameters.
- [ ] `GET /api/customers/{id}` — Get a customer by ID. Returns 200 or 404.
- [ ] `PUT /api/customers/{id}` — Update a customer. Returns 200 or 404.
- [ ] `DELETE /api/customers/{id}` — Delete a customer. Returns 204 or 404.

**Endpoints:**

```
POST   /api/customers
GET    /api/customers
GET    /api/customers?name=john&email=mail&page=1&pageSize=10
GET    /api/customers/{id}
PUT    /api/customers/{id}
DELETE /api/customers/{id}
```

**Suggested DTOs:**

```csharp
// Request
public class CreateCustomerDto
{
    [Required]
    [MaxLength(120)]
    public string Name { get; set; } = string.Empty;

    [MaxLength(120)]
    public string? LastName { get; set; }

    [MaxLength(50)]
    [EmailAddress]
    public string? Email { get; set; }

    [MaxLength(20)]
    public string? PhoneNumber { get; set; }

    [MaxLength(200)]
    public string? CompanyName { get; set; }
}

// Response
public class CustomerResponseDto
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string? LastName { get; set; }
    public string FullName => string.IsNullOrWhiteSpace(LastName) ? Name : $"{Name} {LastName}";
    public string? Email { get; set; }
    public string? PhoneNumber { get; set; }
    public string? CompanyName { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}
```

---

## 5. Step-by-Step Implementation Guide

### Step 1: Create the project

```bash
# Create the project directory
mkdir SalesProject
cd SalesProject

# Create the solution and Web API project
dotnet new sln -n SalesProject
dotnet new webapi -n SalesProject.Api --use-controllers
dotnet sln add SalesProject.Api/SalesProject.Api.csproj
```

> The `--use-controllers` flag generates the project with controllers instead of Minimal APIs.

### Step 2: Install Entity Framework packages

```bash
cd SalesProject.Api

dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools
dotnet add package Microsoft.EntityFrameworkCore.Design
```

> **Alternative for local development:** If you don't have SQL Server installed, you can use SQLite:
> ```bash
> dotnet add package Microsoft.EntityFrameworkCore.Sqlite
> ```

### Step 3: Create the entities

Create the files in the `Models/` folder:
- `Models/Category.cs`
- `Models/Product.cs`
- `Models/Customer.cs`

Use the entity definitions from Section 1.

### Step 4: Create the DbContext

Create `Data/SalesDbContext.cs` with the configuration shown in Section 3.1.

### Step 5: Configure the connection

In `appsettings.json`:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=SalesProjectDb;Trusted_Connection=true;TrustServerCertificate=true;"
  }
}
```

> **For SQLite:**
> ```json
> {
>   "ConnectionStrings": {
>     "DefaultConnection": "Data Source=SalesProject.db"
>   }
> }
> ```

In `Program.cs`, add before `var app = builder.Build();`:

```csharp
builder.Services.AddDbContext<SalesDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
```

### Step 6: Create the initial migration

```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```

### Step 7: Create the Controllers

Implement the controllers following the user stories:

1. `CategoriesController.cs` — Full CRUD (US-01)
2. `ProductsController.cs` — CRUD with filters and pagination (US-02)
3. `CustomersController.cs` — CRUD with filters and pagination (US-03)

### Step 8: Test with Swagger

When you run the project with `dotnet run`, Swagger UI will be available at:
```
https://localhost:{port}/swagger
```

Use Swagger to test all endpoints before moving on.

---

## 6. What You Will Learn in This Part

| Concept | Description |
|---|---|
| **ASP.NET Web API** | How to create a REST API with ASP.NET Core |
| **Controllers** | How to organize endpoints using `ControllerBase` and route attributes |
| **HTTP Methods** | Correct usage of GET, POST, PUT, DELETE |
| **Entity Framework Core** | Code First, DbContext, migrations, LINQ queries |
| **Data Annotations** | Model validation with `[Required]`, `[MaxLength]`, `[Range]` |
| **DTOs** | Separating the internal data model from the API contract |
| **Filters** | Implementing search and filters using `IQueryable` |
| **Pagination** | Implementing pagination with `Skip()` and `Take()` |
| **HTTP Status Codes** | Returning appropriate status codes |
| **Dependency Injection** | Basic usage of ASP.NET Core's DI container |
| **Swagger** | Automatic API documentation |

---

## 7. What We Will NOT Cover in This Part (Coming Later)

- **Part 2:** Sales and sale lines, advanced DTOs, AutoMapper
- **Part 3:** Payments and business logic (payments, payment applications)
- **Part 4:** Repository Pattern, Unit of Work, global error handling
- **Part 5:** JWT Authentication, authorization, auditing
- **Part 6:** Unit and integration tests
