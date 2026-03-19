# Sales Project - Part 2: Sales, SaleLines & AutoMapper

## Goal

Extend the API built in Part 1 to support **Sales** and **Sale Lines**, introducing:

- New entities: `Sale` and `SaleLine` with a parent-child relationship
- An enum `SaleType` to distinguish Sales from Quotes
- **AutoMapper** to eliminate manual DTO mapping in all controllers (including refactoring Part 1 controllers)
- Server-side calculation of line totals and sale totals
- A dedicated endpoint to convert a Quote into a Sale
- Business rules: only Quotes can be deleted; Sales return 409

By the end of this part, you will have a fully functional sales API with quote-to-sale conversion, automatic mapping, and calculated totals.

---

## 1. Data Model

### Entity Diagram (Part 2 — Updated)

```
┌──────────────┐       ┌──────────────────┐
│  Categories  │       │    Products       │
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

┌─────────────────────┐       ┌─────────────────────┐
│       Sales         │       │     SaleLines       │
├─────────────────────┤       ├─────────────────────┤
│ Id (PK)             │──┐    │ Id (PK)             │
│ CustomerId (FK)     │  └───>│ SaleId (FK)         │
│ SaleType            │       │ ProductId (FK)      │
│ TotalAmount         │       │ Quantity            │
│ Notes               │       │ UnitPrice           │
│ CreatedAt           │       │ Total               │
│ UpdatedAt           │       └─────────────────────┘
└─────────────────────┘
        │
        └───> Customer
```

**Relationships:**
- A **Sale** belongs to one **Customer** (`CustomerId` FK)
- A **Sale** has many **SaleLines** (one-to-many)
- A **SaleLine** references one **Product** (`ProductId` FK)

### SaleType Enum

```csharp
public enum SaleType
{
    Sale = 0,
    Quote = 1
}
```

> `SaleType` determines the document type. Quotes can be deleted and converted to Sales. Sales cannot be deleted (return 409).

### Entity Definitions in C\#

#### Sale

```csharp
public class Sale
{
    public int Id { get; set; }

    [Required]
    public int CustomerId { get; set; }

    public SaleType SaleType { get; set; } = SaleType.Sale;

    [Column(TypeName = "decimal(14,4)")]
    public decimal TotalAmount { get; set; }

    [MaxLength(500)]
    public string? Notes { get; set; }

    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }

    // Navigation properties
    public Customer Customer { get; set; } = null!;
    public ICollection<SaleLine> SaleLines { get; set; } = new List<SaleLine>();
}
```

#### SaleLine

```csharp
public class SaleLine
{
    public int Id { get; set; }

    [Required]
    public int SaleId { get; set; }

    [Required]
    public int ProductId { get; set; }

    [Required]
    [Range(1, int.MaxValue, ErrorMessage = "Quantity must be at least 1")]
    public int Quantity { get; set; }

    [Column(TypeName = "decimal(14,4)")]
    public decimal UnitPrice { get; set; }

    [Column(TypeName = "decimal(14,4)")]
    public decimal Total { get; set; }

    // Navigation properties
    public Sale Sale { get; set; } = null!;
    public Product Product { get; set; } = null!;
}
```

> **Note:** `UnitPrice` and `Total` are **server-calculated**. The client can optionally send a custom `UnitPrice`; if omitted, the API uses `Product.Price`. `Total = Quantity * UnitPrice` is always calculated server-side.

---

## 2. AutoMapper

### Why AutoMapper?

In Part 1, we mapped entities to DTOs manually in every controller action:

```csharp
// Manual mapping — repetitive and error-prone
var dto = new ProductResponseDto
{
    Id = product.Id,
    Name = product.Name,
    Sku = product.Sku,
    Description = product.Description,
    Price = product.Price,
    IsActive = product.IsActive,
    CategoryId = product.CategoryId,
    CategoryName = product.Category.Name,
    CreatedAt = product.CreatedAt,
    UpdatedAt = product.UpdatedAt
};
```

AutoMapper eliminates this boilerplate by defining mapping rules once in a **Profile** and using them everywhere via dependency injection.

### Install AutoMapper

```bash
cd SalesProject.Api
dotnet add package AutoMapper.Extensions.Microsoft.DependencyInjection
```

### Create Mapping Profiles

Create a `Mappings/` folder for all profile classes.

#### MappingProfile.cs

```csharp
using AutoMapper;

public class MappingProfile : Profile
{
    public MappingProfile()
    {
        // ---- Categories ----
        CreateMap<Category, CategoryResponseDto>();
        CreateMap<CreateCategoryDto, Category>();

        // ---- Products ----
        CreateMap<Product, ProductResponseDto>()
            .ForMember(dest => dest.CategoryName,
                       opt => opt.MapFrom(src => src.Category.Name));
        CreateMap<CreateProductDto, Product>();
        CreateMap<UpdateProductDto, Product>();

        // ---- Customers ----
        CreateMap<Customer, CustomerResponseDto>();
        CreateMap<CreateCustomerDto, Customer>();

        // ---- Sales ----
        CreateMap<Sale, SaleResponseDto>()
            .ForMember(dest => dest.CustomerName,
                       opt => opt.MapFrom(src =>
                           string.IsNullOrWhiteSpace(src.Customer.LastName)
                               ? src.Customer.Name
                               : $"{src.Customer.Name} {src.Customer.LastName}"));
        CreateMap<CreateSaleDto, Sale>();
        CreateMap<UpdateSaleDto, Sale>();

        // ---- SaleLines ----
        CreateMap<SaleLine, SaleLineResponseDto>()
            .ForMember(dest => dest.ProductName,
                       opt => opt.MapFrom(src => src.Product.Name));
        CreateMap<CreateSaleLineDto, SaleLine>();
        CreateMap<UpdateSaleLineDto, SaleLine>();
    }
}
```

### Register AutoMapper in Program.cs

```csharp
// Add before var app = builder.Build();
builder.Services.AddAutoMapper(typeof(Program));
```

This scans the assembly for all classes that inherit from `Profile` and registers them.

### Refactor Part 1 Controllers

After setting up AutoMapper, inject `IMapper` into your existing controllers and replace manual mapping.

**Before (manual):**

```csharp
public class ProductsController : ControllerBase
{
    private readonly SalesDbContext _context;

    public ProductsController(SalesDbContext context)
    {
        _context = context;
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<ProductResponseDto>> GetProduct(int id)
    {
        var product = await _context.Products
            .Include(p => p.Category)
            .FirstOrDefaultAsync(p => p.Id == id);

        if (product == null) return NotFound();

        var dto = new ProductResponseDto
        {
            Id = product.Id,
            Name = product.Name,
            Sku = product.Sku,
            Description = product.Description,
            Price = product.Price,
            IsActive = product.IsActive,
            CategoryId = product.CategoryId,
            CategoryName = product.Category.Name,
            CreatedAt = product.CreatedAt,
            UpdatedAt = product.UpdatedAt
        };

        return Ok(dto);
    }
}
```

**After (AutoMapper):**

```csharp
public class ProductsController : ControllerBase
{
    private readonly SalesDbContext _context;
    private readonly IMapper _mapper;

    public ProductsController(SalesDbContext context, IMapper mapper)
    {
        _context = context;
        _mapper = mapper;
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<ProductResponseDto>> GetProduct(int id)
    {
        var product = await _context.Products
            .Include(p => p.Category)
            .FirstOrDefaultAsync(p => p.Id == id);

        if (product == null) return NotFound();

        return Ok(_mapper.Map<ProductResponseDto>(product));
    }
}
```

> **Tip:** Refactor `CategoriesController`, `ProductsController`, and `CustomersController` to use `IMapper` before building the new Sales controllers. This keeps the entire codebase consistent.

---

## 3. DTOs

### Sale DTOs

```csharp
// Request - Create
public class CreateSaleDto
{
    [Required]
    public int CustomerId { get; set; }

    public SaleType SaleType { get; set; } = SaleType.Sale;

    [MaxLength(500)]
    public string? Notes { get; set; }

    [Required]
    [MinLength(1, ErrorMessage = "A sale must have at least one line")]
    public List<CreateSaleLineDto> Lines { get; set; } = new();
}

// Request - Update (header only — lines are managed separately)
public class UpdateSaleDto
{
    [Required]
    public int CustomerId { get; set; }

    [MaxLength(500)]
    public string? Notes { get; set; }
}

// Response
public class SaleResponseDto
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
    public string CustomerName { get; set; } = string.Empty;
    public SaleType SaleType { get; set; }
    public string SaleTypeName => SaleType.ToString();
    public decimal TotalAmount { get; set; }
    public string? Notes { get; set; }
    public List<SaleLineResponseDto> Lines { get; set; } = new();
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}
```

### SaleLine DTOs

```csharp
// Request - Create
public class CreateSaleLineDto
{
    [Required]
    public int ProductId { get; set; }

    [Required]
    [Range(1, int.MaxValue, ErrorMessage = "Quantity must be at least 1")]
    public int Quantity { get; set; }

    /// <summary>
    /// Optional custom unit price. If null or 0, the product's current price is used.
    /// </summary>
    public decimal? UnitPrice { get; set; }
}

// Request - Update
public class UpdateSaleLineDto
{
    [Required]
    [Range(1, int.MaxValue, ErrorMessage = "Quantity must be at least 1")]
    public int Quantity { get; set; }

    /// <summary>
    /// Optional custom unit price. If null or 0, the product's current price is used.
    /// </summary>
    public decimal? UnitPrice { get; set; }
}

// Response
public class SaleLineResponseDto
{
    public int Id { get; set; }
    public int SaleId { get; set; }
    public int ProductId { get; set; }
    public string ProductName { get; set; } = string.Empty;
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
    public decimal Total { get; set; }
}
```

---

## 4. User Stories

### US-04: Sale/Quote Management

**As** a sales user,
**I want** to create, view, update, and delete sales and quotes (with their line items),
**so that** I can manage sales transactions.

**Acceptance Criteria:**

- [ ] `POST /api/sales` — Create a sale or quote with its lines. Returns 201.
  - Validate that `CustomerId` exists (return 400 if not).
  - Validate that at least one line is provided.
  - For each line, validate that `ProductId` exists (return 400 if not).
  - If `UnitPrice` is not sent (null) or is 0, use `Product.Price`.
  - Calculate `Total = Quantity * UnitPrice` for each line (server-side).
  - Calculate `TotalAmount` as the sum of all line totals (server-side).
  - Set `CreatedAt` and `UpdatedAt` to `DateTime.UtcNow`.
- [ ] `GET /api/sales` — List sales with support for:
  - **Filters:** by `customerId`, by `saleType` (0=Sale, 1=Quote).
  - **Pagination:** `page` and `pageSize` parameters (default: page=1, pageSize=10, max: 50).
  - Return pagination metadata in the response body.
  - Include customer name in the response (no lines in the list view).
- [ ] `GET /api/sales/{id}` — Get a sale by ID including customer info and all lines. Returns 200 or 404.
- [ ] `PUT /api/sales/{id}` — Update the sale header (customer, notes). Returns 200 or 404.
  - Validate that `CustomerId` exists.
  - Does **not** modify lines (lines are managed via separate endpoints).
  - Recalculate `TotalAmount` is not needed here since lines are unchanged.
- [ ] `DELETE /api/sales/{id}` — Delete a sale. Returns 204, 404, or 409.
  - Only Quotes (`SaleType = Quote`) can be deleted.
  - If `SaleType = Sale`, return **409 Conflict** with message: `"Sales cannot be deleted. Only quotes can be deleted."`
  - Deleting a quote also deletes all its associated lines (cascade).

**Endpoints:**

```
POST   /api/sales
GET    /api/sales
GET    /api/sales?customerId=1&saleType=0&page=1&pageSize=10
GET    /api/sales/{id}
PUT    /api/sales/{id}
DELETE /api/sales/{id}
```

**Line management endpoints (nested under a sale):**

- [ ] `POST /api/sales/{saleId}/lines` — Add a line to an existing sale. Returns 201.
  - Validate `ProductId` exists.
  - Apply UnitPrice logic (custom or from product).
  - Calculate line `Total` and update the sale's `TotalAmount`.
- [ ] `PUT /api/sales/{saleId}/lines/{lineId}` — Update a line. Returns 200 or 404.
  - Recalculate line `Total` and sale's `TotalAmount`.
- [ ] `DELETE /api/sales/{saleId}/lines/{lineId}` — Remove a line. Returns 204 or 404.
  - Do not allow deletion if it is the last line (return 409 with message: `"Cannot delete the last line of a sale."`).
  - Recalculate sale's `TotalAmount`.

```
POST   /api/sales/{saleId}/lines
PUT    /api/sales/{saleId}/lines/{lineId}
DELETE /api/sales/{saleId}/lines/{lineId}
```

---

### US-05: Quote to Sale Conversion

**As** a sales user,
**I want** to convert a quote into a sale,
**so that** I can formalize a quoted transaction.

**Acceptance Criteria:**

- [ ] `POST /api/sales/{id}/convert` — Convert a Quote to a Sale. Returns 200 with the updated sale.
  - The sale must exist (return 404 if not).
  - The sale must be a Quote (`SaleType = Quote`). If already a Sale, return **409 Conflict** with message: `"This document is already a sale."`.
  - Change `SaleType` from `Quote` to `Sale`.
  - Update `UpdatedAt` to `DateTime.UtcNow`.
  - Lines and totals remain unchanged.

**Endpoint:**

```
POST   /api/sales/{id}/convert
```

**Controller example:**

```csharp
[HttpPost("{id}/convert")]
public async Task<ActionResult<SaleResponseDto>> ConvertQuoteToSale(int id)
{
    var sale = await _context.Sales
        .Include(s => s.Customer)
        .Include(s => s.SaleLines)
            .ThenInclude(sl => sl.Product)
        .FirstOrDefaultAsync(s => s.Id == id);

    if (sale == null)
        return NotFound();

    if (sale.SaleType == SaleType.Sale)
        return Conflict(new { message = "This document is already a sale." });

    sale.SaleType = SaleType.Sale;
    sale.UpdatedAt = DateTime.UtcNow;

    await _context.SaveChangesAsync();

    return Ok(_mapper.Map<SaleResponseDto>(sale));
}
```

---

## 5. Business Rules

### 5.1 Server-Side Calculations

All monetary calculations are performed on the server. The client never sends `Total` or `TotalAmount`.

| Field | Calculation | When |
|---|---|---|
| `SaleLine.UnitPrice` | `dto.UnitPrice ?? Product.Price` | On line creation (if client sends null or 0, use product price) |
| `SaleLine.Total` | `Quantity * UnitPrice` | On line creation and update |
| `Sale.TotalAmount` | `Sum of all SaleLine.Total` | On sale creation, line add/update/delete |

**Helper method for recalculating the sale total:**

```csharp
private async Task RecalculateSaleTotal(int saleId)
{
    var sale = await _context.Sales
        .Include(s => s.SaleLines)
        .FirstOrDefaultAsync(s => s.Id == saleId);

    if (sale != null)
    {
        sale.TotalAmount = sale.SaleLines.Sum(sl => sl.Total);
        sale.UpdatedAt = DateTime.UtcNow;
    }
}
```

### 5.2 Deletion Rules

| SaleType | DELETE Behavior |
|---|---|
| `Quote` (1) | Allowed — deletes the quote and all its lines (cascade) |
| `Sale` (0) | **Not allowed** — returns 409 Conflict |

### 5.3 Validation Summary

| Rule | HTTP Status | Message |
|---|---|---|
| Customer not found | 400 Bad Request | `"Customer with ID {id} not found."` |
| Product not found (in line) | 400 Bad Request | `"Product with ID {id} not found."` |
| No lines provided | 400 Bad Request | Model validation error |
| Delete a Sale | 409 Conflict | `"Sales cannot be deleted. Only quotes can be deleted."` |
| Delete the last line | 409 Conflict | `"Cannot delete the last line of a sale."` |
| Convert an already-Sale | 409 Conflict | `"This document is already a sale."` |

---

## 6. Technical Considerations

### 6.1 Transactions

When creating a sale with lines, the entire operation should be atomic. If any line fails validation, nothing should be saved.

```csharp
using var transaction = await _context.Database.BeginTransactionAsync();
try
{
    // 1. Create the sale entity (without lines)
    var sale = _mapper.Map<Sale>(createSaleDto);
    sale.CreatedAt = DateTime.UtcNow;
    sale.UpdatedAt = DateTime.UtcNow;

    _context.Sales.Add(sale);
    await _context.SaveChangesAsync(); // Get the Sale.Id

    // 2. Process each line
    foreach (var lineDto in createSaleDto.Lines)
    {
        var product = await _context.Products.FindAsync(lineDto.ProductId);
        if (product == null)
        {
            await transaction.RollbackAsync();
            return BadRequest(new { message = $"Product with ID {lineDto.ProductId} not found." });
        }

        var line = new SaleLine
        {
            SaleId = sale.Id,
            ProductId = lineDto.ProductId,
            Quantity = lineDto.Quantity,
            UnitPrice = (lineDto.UnitPrice.HasValue && lineDto.UnitPrice.Value > 0)
                ? lineDto.UnitPrice.Value
                : product.Price,
        };
        line.Total = line.Quantity * line.UnitPrice;

        _context.SaleLines.Add(line);
    }

    await _context.SaveChangesAsync();

    // 3. Calculate total
    sale.TotalAmount = sale.SaleLines.Sum(sl => sl.Total);
    await _context.SaveChangesAsync();

    await transaction.CommitAsync();

    // 4. Reload with navigation properties for the response
    var createdSale = await _context.Sales
        .Include(s => s.Customer)
        .Include(s => s.SaleLines)
            .ThenInclude(sl => sl.Product)
        .FirstAsync(s => s.Id == sale.Id);

    return CreatedAtAction(nameof(GetSale), new { id = sale.Id },
        _mapper.Map<SaleResponseDto>(createdSale));
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

### 6.2 Eager Loading

Sales always need related data. Use `.Include()` and `.ThenInclude()` to avoid N+1 queries:

```csharp
// For detail view (single sale)
var sale = await _context.Sales
    .Include(s => s.Customer)
    .Include(s => s.SaleLines)
        .ThenInclude(sl => sl.Product)
    .FirstOrDefaultAsync(s => s.Id == id);

// For list view (no lines, just customer name)
var query = _context.Sales
    .Include(s => s.Customer)
    .AsQueryable();
```

### 6.3 Cascade Delete Configuration

Configure cascade delete for sale lines in `OnModelCreating`:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // ... existing configuration from Part 1 ...

    // Cascade delete: when a Sale is deleted, delete its SaleLines
    modelBuilder.Entity<SaleLine>()
        .HasOne(sl => sl.Sale)
        .WithMany(s => s.SaleLines)
        .HasForeignKey(sl => sl.SaleId)
        .OnDelete(DeleteBehavior.Cascade);
}
```

### 6.4 AutoMapper DI Pattern

All controllers follow the same pattern — inject both `SalesDbContext` and `IMapper`:

```csharp
[ApiController]
[Route("api/[controller]")]
public class SalesController : ControllerBase
{
    private readonly SalesDbContext _context;
    private readonly IMapper _mapper;

    public SalesController(SalesDbContext context, IMapper mapper)
    {
        _context = context;
        _mapper = mapper;
    }
}
```

### 6.5 Updated DbContext

Add the new `DbSet` properties:

```csharp
public class SalesDbContext : DbContext
{
    public SalesDbContext(DbContextOptions<SalesDbContext> options) : base(options) { }

    public DbSet<Product> Products => Set<Product>();
    public DbSet<Category> Categories => Set<Category>();
    public DbSet<Customer> Customers => Set<Customer>();
    public DbSet<Sale> Sales => Set<Sale>();
    public DbSet<SaleLine> SaleLines => Set<SaleLine>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>()
            .HasIndex(p => p.Sku)
            .IsUnique();

        modelBuilder.Entity<Category>()
            .HasIndex(c => c.Name)
            .IsUnique();

        modelBuilder.Entity<SaleLine>()
            .HasOne(sl => sl.Sale)
            .WithMany(s => s.SaleLines)
            .HasForeignKey(sl => sl.SaleId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}
```

### 6.6 Updated Folder Structure

```
SalesProject.Api/
├── Controllers/
│   ├── CategoriesController.cs
│   ├── CustomersController.cs
│   ├── ProductsController.cs
│   └── SalesController.cs          ← NEW
├── Data/
│   └── SalesDbContext.cs            ← UPDATED
├── Dtos/
│   ├── Categories/
│   │   ├── CreateCategoryDto.cs
│   │   └── CategoryResponseDto.cs
│   ├── Customers/
│   │   ├── CreateCustomerDto.cs
│   │   └── CustomerResponseDto.cs
│   ├── Products/
│   │   ├── CreateProductDto.cs
│   │   ├── UpdateProductDto.cs
│   │   └── ProductResponseDto.cs
│   └── Sales/                       ← NEW
│       ├── CreateSaleDto.cs
│       ├── UpdateSaleDto.cs
│       ├── SaleResponseDto.cs
│       ├── CreateSaleLineDto.cs
│       ├── UpdateSaleLineDto.cs
│       └── SaleLineResponseDto.cs
├── Mappings/                        ← NEW
│   └── MappingProfile.cs
├── Models/
│   ├── Category.cs
│   ├── Customer.cs
│   ├── Product.cs
│   ├── Sale.cs                      ← NEW
│   ├── SaleLine.cs                  ← NEW
│   └── SaleType.cs                  ← NEW (enum)
├── Program.cs                       ← UPDATED
├── appsettings.json
└── SalesProject.Api.csproj          ← UPDATED (new package)
```

---

## 7. Step-by-Step Implementation Guide

### Step 1: Install AutoMapper

```bash
cd SalesProject.Api
dotnet add package AutoMapper.Extensions.Microsoft.DependencyInjection
```

### Step 2: Create the SaleType enum

Create `Models/SaleType.cs`:

```csharp
public enum SaleType
{
    Sale = 0,
    Quote = 1
}
```

### Step 3: Create the Sale and SaleLine entities

Create the files in the `Models/` folder:
- `Models/Sale.cs` — use the entity definition from Section 1.
- `Models/SaleLine.cs` — use the entity definition from Section 1.

### Step 4: Update the DbContext

Add `DbSet<Sale>` and `DbSet<SaleLine>` to `SalesDbContext`. Configure cascade delete in `OnModelCreating`. See Section 6.5.

### Step 5: Create and apply the migration

```bash
dotnet ef migrations add AddSalesAndSaleLines
dotnet ef database update
```

### Step 6: Create the DTOs

Create the DTOs in `Dtos/Sales/`:
- `CreateSaleDto.cs`, `UpdateSaleDto.cs`, `SaleResponseDto.cs`
- `CreateSaleLineDto.cs`, `UpdateSaleLineDto.cs`, `SaleLineResponseDto.cs`

Use the DTO definitions from Section 3.

### Step 7: Create the AutoMapper profile

Create `Mappings/MappingProfile.cs` with all mappings (Categories, Products, Customers, Sales, SaleLines). See Section 2.

### Step 8: Register AutoMapper in Program.cs

Add to `Program.cs` before `var app = builder.Build();`:

```csharp
builder.Services.AddAutoMapper(typeof(Program));
```

### Step 9: Refactor Part 1 controllers

Update `CategoriesController`, `ProductsController`, and `CustomersController`:
1. Add `IMapper _mapper` to the constructor.
2. Replace all manual `new XxxResponseDto { ... }` blocks with `_mapper.Map<XxxResponseDto>(entity)`.
3. Replace all manual `new Entity { ... }` creation from DTOs with `_mapper.Map<Entity>(dto)`.
4. Test that all existing endpoints still work.

### Step 10: Create the SalesController

Implement `Controllers/SalesController.cs` with:
- `POST /api/sales` — Create sale with lines (use transaction, see Section 6.1).
- `GET /api/sales` — List with filters and pagination.
- `GET /api/sales/{id}` — Detail with customer and lines.
- `PUT /api/sales/{id}` — Update header.
- `DELETE /api/sales/{id}` — Delete with SaleType check.
- `POST /api/sales/{id}/convert` — Quote to Sale conversion.
- `POST /api/sales/{saleId}/lines` — Add a line.
- `PUT /api/sales/{saleId}/lines/{lineId}` — Update a line.
- `DELETE /api/sales/{saleId}/lines/{lineId}` — Remove a line.

### Step 11: Test with Swagger

Run the project and test all endpoints:

```bash
dotnet run
```

**Testing checklist:**

1. Create a Quote with 2+ lines → verify `TotalAmount` is calculated correctly.
2. Create a Sale with a custom `UnitPrice` on one line → verify it uses the custom price.
3. Create a Sale without `UnitPrice` → verify it uses `Product.Price`.
4. Add a line to an existing sale → verify `TotalAmount` is recalculated.
5. Update a line's quantity → verify `Total` and `TotalAmount` update.
6. Delete a line (not the last one) → verify `TotalAmount` updates.
7. Try to delete the last line → expect 409.
8. Try to delete a Sale → expect 409.
9. Delete a Quote → expect 204.
10. Convert a Quote to Sale → verify `SaleType` changes.
11. Try to convert an already-Sale → expect 409.

---

## 8. What You Will Learn in This Part

| Concept | Description |
|---|---|
| **AutoMapper** | Automatic mapping between entities and DTOs using profiles |
| **Mapping Profiles** | Defining mapping rules in a centralized class |
| **Dependency Injection (IMapper)** | Injecting AutoMapper into controllers |
| **Enums in EF Core** | Using `enum` types stored as integers in the database |
| **One-to-Many Relationships** | Sale → SaleLines parent-child pattern |
| **Eager Loading** | Using `.Include()` and `.ThenInclude()` to load related data |
| **Transactions** | Using `BeginTransactionAsync` for atomic multi-step operations |
| **Server-Side Calculations** | Computing UnitPrice, Total, and TotalAmount on the API |
| **Cascade Delete** | Configuring EF Core to delete children when parent is deleted |
| **Nested Resources** | REST endpoints for child resources (`/sales/{id}/lines`) |
| **Business Rule Enforcement** | Enforcing deletion rules based on entity state |
| **Custom Endpoints** | Non-CRUD actions like `/convert` |
| **409 Conflict** | Returning 409 for business rule violations |

---

## 9. What We Will NOT Cover in This Part (Coming Later)

- **Part 3:** Payments, payment applications, partial payments
- **Part 4:** Repository Pattern, Unit of Work, global error handling
- **Part 5:** JWT Authentication, authorization, auditing
- **Part 6:** Unit and integration tests
