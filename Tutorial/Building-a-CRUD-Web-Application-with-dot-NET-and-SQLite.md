# Building a CRUD Web Application with .NET and SQLite

Here's a step-by-step guide to creating a basic CRUD (Create, Read, Update, Delete) web application using .NET and SQLite.

## Prerequisites
- .NET SDK (6.0 or later)
- Visual Studio, VS Code, or your preferred IDE
- SQLite (will be handled by EF Core)

## Step 1: Create a new ASP.NET Core Web App

```bash
dotnet new webapp -n CrudApp
cd CrudApp
```

## Step 2: Add Required Packages

```bash
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Microsoft.EntityFrameworkCore.Tools
```

## Step 3: Create a Model

Create a `Models` folder and add a `Product.cs` file:

```csharp
namespace CrudApp.Models;

public class Product
{
    public int Id { get; set; }
    public string? Name { get; set; }
    public string? Description { get; set; }
    public decimal Price { get; set; }
}
```

## Step 4: Create the Database Context

Create a `Data` folder and add `ApplicationDbContext.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using CrudApp.Models;

namespace CrudApp.Data;

public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    public DbSet<Product> Products { get; set; }
}
```

## Step 5: Configure the Database in Program.cs

Update `Program.cs`:

```csharp
using CrudApp.Data;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddRazorPages();

// Add DbContext
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlite(builder.Configuration.GetConnectionString("DefaultConnection")));

var app = builder.Build();

// Configure the HTTP request pipeline.
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();

app.UseRouting();

app.UseAuthorization();

app.MapRazorPages();

app.Run();
```

## Step 6: Add Connection String to appsettings.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Data Source=CrudApp.db"
  },
  // ... other configurations
}
```

## Step 7: Create Database Migration

Run these commands in the terminal:

```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```

## Step 8: Create a Products Service (Optional but Recommended)

Create a `Services` folder and add `ProductService.cs`:

```csharp
using CrudApp.Data;
using CrudApp.Models;
using Microsoft.EntityFrameworkCore;

namespace CrudApp.Services;

public class ProductService
{
    private readonly ApplicationDbContext _context;

    public ProductService(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<List<Product>> GetProductsAsync()
    {
        return await _context.Products.ToListAsync();
    }

    public async Task<Product?> GetProductByIdAsync(int id)
    {
        return await _context.Products.FindAsync(id);
    }

    public async Task CreateProductAsync(Product product)
    {
        _context.Products.Add(product);
        await _context.SaveChangesAsync();
    }

    public async Task UpdateProductAsync(Product product)
    {
        _context.Products.Update(product);
        await _context.SaveChangesAsync();
    }

    public async Task DeleteProductAsync(int id)
    {
        var product = await _context.Products.FindAsync(id);
        if (product != null)
        {
            _context.Products.Remove(product);
            await _context.SaveChangesAsync();
        }
    }
}
```

Register the service in `Program.cs`:

```csharp
builder.Services.AddScoped<ProductService>();
```

## Step 9: Create Razor Pages for CRUD Operations

### 9.1 Index Page (List Products)

Create `Pages/Products/Index.cshtml`:

```html
@page
@model CrudApp.Pages.Products.IndexModel
@{
    ViewData["Title"] = "Products";
}

<h1>Products</h1>

<p>
    <a asp-page="Create" class="btn btn-primary">Create New</a>
</p>

<table class="table">
    <thead>
        <tr>
            <th>Name</th>
            <th>Description</th>
            <th>Price</th>
            <th>Actions</th>
        </tr>
    </thead>
    <tbody>
        @foreach (var product in Model.Products)
        {
            <tr>
                <td>@product.Name</td>
                <td>@product.Description</td>
                <td>@product.Price.ToString("C")</td>
                <td>
                    <a asp-page="./Edit" asp-route-id="@product.Id" class="btn btn-sm btn-warning">Edit</a>
                    <a asp-page="./Delete" asp-route-id="@product.Id" class="btn btn-sm btn-danger">Delete</a>
                </td>
            </tr>
        }
    </tbody>
</table>
```

And `Pages/Products/Index.cshtml.cs`:

```csharp
using CrudApp.Services;
using Microsoft.AspNetCore.Mvc.RazorPages;

namespace CrudApp.Pages.Products;

public class IndexModel : PageModel
{
    private readonly ProductService _productService;

    public IndexModel(ProductService productService)
    {
        _productService = productService;
    }

    public IList<Product> Products { get; set; } = default!;

    public async Task OnGetAsync()
    {
        Products = await _productService.GetProductsAsync();
    }
}
```

### 9.2 Create Page

Create `Pages/Products/Create.cshtml`:

```html
@page
@model CrudApp.Pages.Products.CreateModel
@{
    ViewData["Title"] = "Create Product";
}

<h1>Create Product</h1>

<div class="row">
    <div class="col-md-4">
        <form method="post">
            <div asp-validation-summary="ModelOnly" class="text-danger"></div>
            <div class="form-group">
                <label asp-for="Product.Name" class="control-label"></label>
                <input asp-for="Product.Name" class="form-control" />
                <span asp-validation-for="Product.Name" class="text-danger"></span>
            </div>
            <div class="form-group">
                <label asp-for="Product.Description" class="control-label"></label>
                <input asp-for="Product.Description" class="form-control" />
                <span asp-validation-for="Product.Description" class="text-danger"></span>
            </div>
            <div class="form-group">
                <label asp-for="Product.Price" class="control-label"></label>
                <input asp-for="Product.Price" class="form-control" />
                <span asp-validation-for="Product.Price" class="text-danger"></span>
            </div>
            <div class="form-group mt-3">
                <input type="submit" value="Create" class="btn btn-primary" />
                <a asp-page="Index" class="btn btn-secondary">Back to List</a>
            </div>
        </form>
    </div>
</div>
```

And `Pages/Products/Create.cshtml.cs`:

```csharp
using CrudApp.Services;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;

namespace CrudApp.Pages.Products;

public class CreateModel : PageModel
{
    private readonly ProductService _productService;

    public CreateModel(ProductService productService)
    {
        _productService = productService;
    }

    [BindProperty]
    public Product Product { get; set; } = default!;

    public IActionResult OnGet()
    {
        return Page();
    }

    public async Task<IActionResult> OnPostAsync()
    {
        if (!ModelState.IsValid)
        {
            return Page();
        }

        await _productService.CreateProductAsync(Product);

        return RedirectToPage("./Index");
    }
}
```

### 9.3 Edit Page

Create `Pages/Products/Edit.cshtml`:

```html
@page "{id:int}"
@model CrudApp.Pages.Products.EditModel
@{
    ViewData["Title"] = "Edit Product";
}

<h1>Edit Product</h1>

<div class="row">
    <div class="col-md-4">
        <form method="post">
            <div asp-validation-summary="ModelOnly" class="text-danger"></div>
            <input type="hidden" asp-for="Product.Id" />
            <div class="form-group">
                <label asp-for="Product.Name" class="control-label"></label>
                <input asp-for="Product.Name" class="form-control" />
                <span asp-validation-for="Product.Name" class="text-danger"></span>
            </div>
            <div class="form-group">
                <label asp-for="Product.Description" class="control-label"></label>
                <input asp-for="Product.Description" class="form-control" />
                <span asp-validation-for="Product.Description" class="text-danger"></span>
            </div>
            <div class="form-group">
                <label asp-for="Product.Price" class="control-label"></label>
                <input asp-for="Product.Price" class="form-control" />
                <span asp-validation-for="Product.Price" class="text-danger"></span>
            </div>
            <div class="form-group mt-3">
                <input type="submit" value="Save" class="btn btn-primary" />
                <a asp-page="./Index" class="btn btn-secondary">Back to List</a>
            </div>
        </form>
    </div>
</div>
```

And `Pages/Products/Edit.cshtml.cs`:

```csharp
using CrudApp.Services;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;

namespace CrudApp.Pages.Products;

public class EditModel : PageModel
{
    private readonly ProductService _productService;

    public EditModel(ProductService productService)
    {
        _productService = productService;
    }

    [BindProperty]
    public Product Product { get; set; } = default!;

    public async Task<IActionResult> OnGetAsync(int id)
    {
        var product = await _productService.GetProductByIdAsync(id);
        if (product == null)
        {
            return NotFound();
        }
        Product = product;
        return Page();
    }

    public async Task<IActionResult> OnPostAsync()
    {
        if (!ModelState.IsValid)
        {
            return Page();
        }

        await _productService.UpdateProductAsync(Product);

        return RedirectToPage("./Index");
    }
}
```

### 9.4 Delete Page

Create `Pages/Products/Delete.cshtml`:

```html
@page "{id:int}"
@model CrudApp.Pages.Products.DeleteModel
@{
    ViewData["Title"] = "Delete Product";
}

<h1>Delete Product</h1>

<h3>Are you sure you want to delete this?</h3>
<div>
    <h4>Product</h4>
    <hr />
    <dl class="row">
        <dt class="col-sm-2">
            @Html.DisplayNameFor(model => model.Product.Name)
        </dt>
        <dd class="col-sm-10">
            @Html.DisplayFor(model => model.Product.Name)
        </dd>
        <dt class="col-sm-2">
            @Html.DisplayNameFor(model => model.Product.Description)
        </dt>
        <dd class="col-sm-10">
            @Html.DisplayFor(model => model.Product.Description)
        </dd>
        <dt class="col-sm-2">
            @Html.DisplayNameFor(model => model.Product.Price)
        </dt>
        <dd class="col-sm-10">
            @Html.DisplayFor(model => model.Product.Price)
        </dd>
    </dl>
    
    <form method="post">
        <input type="hidden" asp-for="Product.Id" />
        <input type="submit" value="Delete" class="btn btn-danger" />
        <a asp-page="./Index" class="btn btn-secondary">Back to List</a>
    </form>
</div>
```

And `Pages/Products/Delete.cshtml.cs`:

```csharp
using CrudApp.Services;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;

namespace CrudApp.Pages.Products;

public class DeleteModel : PageModel
{
    private readonly ProductService _productService;

    public DeleteModel(ProductService productService)
    {
        _productService = productService;
    }

    [BindProperty]
    public Product Product { get; set; } = default!;

    public async Task<IActionResult> OnGetAsync(int id)
    {
        var product = await _productService.GetProductByIdAsync(id);
        if (product == null)
        {
            return NotFound();
        }
        Product = product;
        return Page();
    }

    public async Task<IActionResult> OnPostAsync(int id)
    {
        await _productService.DeleteProductAsync(id);
        return RedirectToPage("./Index");
    }
}
```

## Step 10: Add Navigation Link

Update `Pages/Shared/_Layout.cshtml` to include a link to the Products page in the navigation:

```html
<li class="nav-item">
    <a class="nav-link text-dark" asp-area="" asp-page="/Products/Index">Products</a>
</li>
```

## Step 11: Run the Application

```bash
dotnet run
```

Visit `https://localhost:5001/Products` in your browser to test the CRUD functionality.

## Additional Improvements

1. **Validation**: Add more robust validation to your model
2. **Error Handling**: Implement better error handling
3. **Pagination**: Add pagination for the product list
4. **Search**: Implement search functionality
5. **Authentication**: Add user authentication
6. **DTOs**: Use Data Transfer Objects for better separation of concerns
7. **Logging**: Add logging for important operations

This completes a basic CRUD application with .NET and SQLite. You can now create, read, update, and delete products from your database.