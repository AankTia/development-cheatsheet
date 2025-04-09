# Building a CRUD Web Application with C# and SQLite

This step-by-step guide will walk you through creating a simple CRUD (Create, Read, Update, Delete) web application using C# with SQLite as the database.

## Prerequisites
- .NET SDK (6.0 or later)
- Visual Studio or VS Code
- SQLite (will be installed via NuGet packages)

## Step 1: Create a New Project

1. Open a terminal/command prompt
2. Run:
   ```bash
   dotnet new webapp -o CrudApp
   cd CrudApp
   ```

## Step 2: Add Required NuGet Packages

Run these commands in your project directory:
```bash
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Microsoft.EntityFrameworkCore.Tools
```

## Step 3: Create a Model Class

Create a new folder called `Models` and add a `Product.cs` file:

```csharp
namespace CrudApp.Models;

public class Product
{
    public int Id { get; set; }
    public string? Name { get; set; }
    public string? Description { get; set; }
    public decimal Price { get; set; }
    public int StockQuantity { get; set; }
}
```

## Step 4: Create the Database Context

Create a new folder called `Data` and add `AppDbContext.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using CrudApp.Models;

namespace CrudApp.Data;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options)
    {
    }

    public DbSet<Product> Products { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Seed some initial data
        modelBuilder.Entity<Product>().HasData(
            new Product { Id = 1, Name = "Laptop", Description = "High performance laptop", Price = 999.99m, StockQuantity = 10 },
            new Product { Id = 2, Name = "Smartphone", Description = "Latest smartphone model", Price = 699.99m, StockQuantity = 15 }
        );
    }
}
```

## Step 5: Configure the Database Connection

In `Program.cs`, add the SQLite configuration:

```csharp
// Add this after the builder is created
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite(builder.Configuration.GetConnectionString("DefaultConnection")));

// Add this before app.Run()
using (var scope = app.Services.CreateScope())
{
    var services = scope.ServiceProvider;
    var dbContext = services.GetRequiredService<AppDbContext>();
    dbContext.Database.EnsureCreated();
}
```

Add the connection string to `appsettings.json`:

```json
"ConnectionStrings": {
  "DefaultConnection": "Data Source=crudapp.db"
}
```

## Step 6: Create a Products Controller

Create a new controller in the `Controllers` folder named `ProductsController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using CrudApp.Data;
using CrudApp.Models;

namespace CrudApp.Controllers;

public class ProductsController : Controller
{
    private readonly AppDbContext _context;

    public ProductsController(AppDbContext context)
    {
        _context = context;
    }

    // GET: Products
    public async Task<IActionResult> Index()
    {
        return View(await _context.Products.ToListAsync());
    }

    // GET: Products/Details/5
    public async Task<IActionResult> Details(int? id)
    {
        if (id == null)
        {
            return NotFound();
        }

        var product = await _context.Products
            .FirstOrDefaultAsync(m => m.Id == id);
        if (product == null)
        {
            return NotFound();
        }

        return View(product);
    }

    // GET: Products/Create
    public IActionResult Create()
    {
        return View();
    }

    // POST: Products/Create
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Create([Bind("Id,Name,Description,Price,StockQuantity")] Product product)
    {
        if (ModelState.IsValid)
        {
            _context.Add(product);
            await _context.SaveChangesAsync();
            return RedirectToAction(nameof(Index));
        }
        return View(product);
    }

    // GET: Products/Edit/5
    public async Task<IActionResult> Edit(int? id)
    {
        if (id == null)
        {
            return NotFound();
        }

        var product = await _context.Products.FindAsync(id);
        if (product == null)
        {
            return NotFound();
        }
        return View(product);
    }

    // POST: Products/Edit/5
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Edit(int id, [Bind("Id,Name,Description,Price,StockQuantity")] Product product)
    {
        if (id != product.Id)
        {
            return NotFound();
        }

        if (ModelState.IsValid)
        {
            try
            {
                _context.Update(product);
                await _context.SaveChangesAsync();
            }
            catch (DbUpdateConcurrencyException)
            {
                if (!ProductExists(product.Id))
                {
                    return NotFound();
                }
                else
                {
                    throw;
                }
            }
            return RedirectToAction(nameof(Index));
        }
        return View(product);
    }

    // GET: Products/Delete/5
    public async Task<IActionResult> Delete(int? id)
    {
        if (id == null)
        {
            return NotFound();
        }

        var product = await _context.Products
            .FirstOrDefaultAsync(m => m.Id == id);
        if (product == null)
        {
            return NotFound();
        }

        return View(product);
    }

    // POST: Products/Delete/5
    [HttpPost, ActionName("Delete")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> DeleteConfirmed(int id)
    {
        var product = await _context.Products.FindAsync(id);
        if (product != null)
        {
            _context.Products.Remove(product);
        }

        await _context.SaveChangesAsync();
        return RedirectToAction(nameof(Index));
    }

    private bool ProductExists(int id)
    {
        return _context.Products.Any(e => e.Id == id);
    }
}
```

## Step 7: Create the Views

Create a `Products` folder in the `Views` directory and add these views:

1. `Index.cshtml`:
```html
@model IEnumerable<CrudApp.Models.Product>

@{
    ViewData["Title"] = "Products";
}

<h1>Products</h1>

<p>
    <a asp-action="Create">Create New</a>
</p>
<table class="table">
    <thead>
        <tr>
            <th>
                @Html.DisplayNameFor(model => model.Name)
            </th>
            <th>
                @Html.DisplayNameFor(model => model.Description)
            </th>
            <th>
                @Html.DisplayNameFor(model => model.Price)
            </th>
            <th>
                @Html.DisplayNameFor(model => model.StockQuantity)
            </th>
            <th></th>
        </tr>
    </thead>
    <tbody>
@foreach (var item in Model) {
        <tr>
            <td>
                @Html.DisplayFor(modelItem => item.Name)
            </td>
            <td>
                @Html.DisplayFor(modelItem => item.Description)
            </td>
            <td>
                @Html.DisplayFor(modelItem => item.Price)
            </td>
            <td>
                @Html.DisplayFor(modelItem => item.StockQuantity)
            </td>
            <td>
                <a asp-action="Edit" asp-route-id="@item.Id">Edit</a> |
                <a asp-action="Details" asp-route-id="@item.Id">Details</a> |
                <a asp-action="Delete" asp-route-id="@item.Id">Delete</a>
            </td>
        </tr>
}
    </tbody>
</table>
```

2. `Create.cshtml`:
```html
@model CrudApp.Models.Product

@{
    ViewData["Title"] = "Create Product";
}

<h1>Create</h1>

<h4>Product</h4>
<hr />
<div class="row">
    <div class="col-md-4">
        <form asp-action="Create">
            <div asp-validation-summary="ModelOnly" class="text-danger"></div>
            <div class="form-group">
                <label asp-for="Name" class="control-label"></label>
                <input asp-for="Name" class="form-control" />
                <span asp-validation-for="Name" class="text-danger"></span>
            </div>
            <div class="form-group">
                <label asp-for="Description" class="control-label"></label>
                <input asp-for="Description" class="form-control" />
                <span asp-validation-for="Description" class="text-danger"></span>
            </div>
            <div class="form-group">
                <label asp-for="Price" class="control-label"></label>
                <input asp-for="Price" class="form-control" />
                <span asp-validation-for="Price" class="text-danger"></span>
            </div>
            <div class="form-group">
                <label asp-for="StockQuantity" class="control-label"></label>
                <input asp-for="StockQuantity" class="form-control" />
                <span asp-validation-for="StockQuantity" class="text-danger"></span>
            </div>
            <div class="form-group">
                <input type="submit" value="Create" class="btn btn-primary" />
            </div>
        </form>
    </div>
</div>

<div>
    <a asp-action="Index">Back to List</a>
</div>

@section Scripts {
    @{await Html.RenderPartialAsync("_ValidationScriptsPartial");}
}
```

3. `Edit.cshtml` (similar to Create but with hidden Id field)
4. `Details.cshtml` (display only)
5. `Delete.cshtml` (confirmation view)

## Step 8: Add Navigation Link

Add a link to the Products page in the shared layout (`Views/Shared/_Layout.cshtml`):

```html
<li class="nav-item">
    <a class="nav-link text-dark" asp-area="" asp-controller="Products" asp-action="Index">Products</a>
</li>
```

## Step 9: Run the Application

Run the application with:
```bash
dotnet run
```

Or from Visual Studio by pressing F5.

## Step 10: Test the Application

1. Navigate to `/Products`
2. Try creating, reading, updating, and deleting products
3. Verify the data persists between application restarts

## Additional Enhancements

1. **Validation**: Add data annotations to your model for better validation
2. **Pagination**: Implement pagination for the Index view
3. **Search**: Add search functionality
4. **Authentication**: Add user authentication
5. **Error Handling**: Improve error handling

This completes a basic CRUD application with C# and SQLite. The application uses Entity Framework Core for database operations and follows the MVC pattern.