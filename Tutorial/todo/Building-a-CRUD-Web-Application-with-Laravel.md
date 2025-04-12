# Building a CRUD Web Application with Laravel

Here's a comprehensive guide to building a feature-rich Laravel application.

This comprehensive setup gives you:

- CRUD functionality with Laravel
- SQLite database
- Bootstrap styling
- User authentication
- Input validation
- Pagination
- Search functionality
- Role-based authorization
- File uploads
- API endpoints
- Real-time updates
- Docker configuration
- Deployment options for Heroku, Render, and AWS

---

## STEP 1: Project Setup

First, let's set up the Laravel project with the required technologies:

1. Create new Laravel project

```bash
composer create-project laravel/laravel crud-app
cd crud-app
```

2. Install SQLite (create database file)

```bash
touch database/database.sqlite
```

3. Install frontend dependencies

```bash
npm install bootstrap @popperjs/core
```

4. Configure `.env` to use SQLite:

```
DB_CONNECTION=sqlite
DB_DATABASE=/absolute/path/to/your/project/database/database.sqlite
```

## 2. Authentication Setup

Laravel Breeze provides a simple authentication scaffolding:

```bash
composer require laravel/breeze --dev
php artisan breeze:install
php artisan migrate
npm install && npm run dev
```

## 3. Database and Models

1. Create a model with migration for your main CRUD entity (e.g., `Product`):

```bash
php artisan make:model Product -m
```

2. Edit the migration file:

```php
// database/migrations/xxxx_create_products_table.php
public function up()
{
    Schema::create('products', function (Blueprint $table) {
        $table->id();
        $table->string('name');
        $table->text('description');
        $table->decimal('price', 8, 2);
        $table->string('image_path')->nullable();
        $table->foreignId('user_id')->constrained()->onDelete('cascade');
        $table->timestamps();
    });
}
```

3. Run migrations:

```bash
php artisan migrate
```

## 4. Role-Based Authorization

1. Create Role Table:

```bash
php artisan make:migration create_roles_table
```

2. Upadate Migration:

```php
Schema::create('roles', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->timestamps();
});

Schema::table('users', function (Blueprint $table) {
    $table->foreignId('role_id')->default(1)->constrained();
});
```

4. Create Role model:

```bash
php artisan make:model Role
```

5. Add relationship to User model:

```php
// app/Models/User.php
public function role()
{
    return $this->belongsTo(Role::class);
}
```

6. Create middleware for roles:

```bash
php artisan make:middleware CheckRole
```

7. Implement the middleware:

```php
public function handle(Request $request, Closure $next, ...$roles)
{
    if (! $request->user() || ! in_array($request->user()->role->name, $roles)) {
        abort(403);
    }
    return $next($request);
}
```

8. Register middleware in `Kernel.php`:

```php
protected $routeMiddleware = [
    // ...
    'role' => \App\Http\Middleware\CheckRole::class,
];
```

## 5. CRUD Implementation

1. Create controller:

```bash
php artisan make:controller ProductController --resource --model=Product
```

2. Controller methods:

```php
// app/Http/Controllers/ProductController.php
public function index()
{
    $products = Product::latest()->paginate(10);
    return view('products.index', compact('products'));
}

public function create()
{
    return view('products.create');
}

public function store(Request $request)
{
    $validated = $request->validate([
        'name' => 'required|max:255',
        'description' => 'required',
        'price' => 'required|numeric',
        'image' => 'nullable|image|max:2048'
    ]);

    if ($request->hasFile('image')) {
        $path = $request->file('image')->store('products', 'public');
        $validated['image_path'] = $path;
    }

    $request->user()->products()->create($validated);

    return redirect()->route('products.index');
}

// Implement other methods (show, edit, update, destroy)
```

## 6. Search Functionality

Add search to your index method:

```php
public function index(Request $request)
{
    $query = Product::query();

    if ($request->has('search')) {
        $query->where('name', 'like', '%'.$request->search.'%')
              ->orWhere('description', 'like', '%'.$request->search.'%');
    }

    $products = $query->latest()->paginate(10);

    return view('products.index', compact('products'));
}
```

## 7. File Uploads

1. Ensure you have configured filesystems in `config/filesystems.php`:

```php
'disks' => [
    'public' => [
        'driver' => 'local',
        'root' => storage_path('app/public'),
        'url' => env('APP_URL').'/storage',
        'visibility' => 'public',
    ],
    // ...
],
```

2. Create a symbolic link:

```bash
php artisan storage:link
```

## 8. API Endpoints

1. Create API routes in `routes/api.php`:

```php
Route::middleware('auth:sanctum')->group(function () {
    Route::apiResource('products', ProductController::class);
});
```

2. Add API token to your User model:

```php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
}
```

## 9. Real-time Updates with Laravel Echo

1. Install Laravel Echo and Pusher:

```bash
composer require pusher/pusher-php-server
npm install laravel-echo pusher-js
```

2. Configure broadcasting in `.env`:

```
BROADCAST_DRIVER=pusher
PUSHER_APP_ID=your_app_id
PUSHER_APP_KEY=your_app_key
PUSHER_APP_SECRET=your_app_secret
PUSHER_APP_CLUSTER=mt1
```

3. Create event:

```bash
php artisan make:event ProductUpdated
```

4. Broadcast the event in your controller:

```php
public function update(Request $request, Product $product)
{
    // ... update logic

    broadcast(new ProductUpdated($product))->toOthers();

    return redirect()->route('products.index');
}
```

## 10. Testing

1. Create tests:

```bash
php artisan make:test ProductTest
```

2. Example test:

```php
public function test_authenticated_user_can_create_product()
{
    $user = User::factory()->create();

    $response = $this->actingAs($user)
        ->post('/products', [
            'name' => 'Test Product',
            'description' => 'Test Description',
            'price' => 9.99
        ]);

    $response->assertRedirect('/products');
    $this->assertDatabaseHas('products', ['name' => 'Test Product']);
}
```

4. Run tests:

```bash
php artisan test
```

## 11. ImportMap Configuration

Laravel 9+ comes with Vite by default, but if you want to use ImportMap:

1. Install ImportMap:

```bash
composer require tightenco/importmap-laravel
php artisan importmap:install
```

2. Configure in `app.js`:

```javascript
import "bootstrap";
```

3. Update `importmap.php`:

```php
return [
    'paths' => [
        // ...
        'bootstrap' => 'vendor/twbs/bootstrap/dist/js/bootstrap.esm.js',
    ],
];
```


## 12. Docker Setup

1. Create `Dockerfile`:

```dockerfile
FROM php:8.2-fpm

# Install dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    zip \
    unzip \
    sqlite3

# Clear cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Install PHP extensions
RUN docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd

# Get latest Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Set working directory
WORKDIR /var/www

# Copy existing application directory contents
COPY . /var/www

# Install dependencies
RUN composer install

# Expose port 9000 and start php-fpm server
EXPOSE 9000
CMD ["php-fpm"]
```

2. Create `docker-compose.yml`:

```yaml
version: "3"

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    image: laravel-app
    container_name: app
    restart: unless-stopped
    tty: true
    environment:
      SERVICE_NAME: app
      SERVICE_TAGS: dev
    working_dir: /var/www
    volumes:
      - ./:/var/www
      - ./docker/php/php-ini-overrides.ini:/usr/local/etc/php/conf.d/99-overrides.ini
    networks:
      - laravel

  webserver:
    image: nginx:alpine
    container_name: webserver
    restart: unless-stopped
    tty: true
    ports:
      - "8000:80"
    volumes:
      - ./:/var/www
      - ./docker/nginx/conf.d/:/etc/nginx/conf.d/
    networks:
      - laravel

  db:
    image: sqlite
    container_name: db
    volumes:
      - ./database/database.sqlite:/db/database.sqlite
    networks:
      - laravel

networks:
  laravel:
    driver: bridge
```

## 13. Deployment

### Heroku Deployment

1. Create `Procfile`:

```
web: vendor/bin/heroku-php-nginx -C nginx_app.conf public/
```

2. Deploy:

```bash
heroku create
heroku addons:create heroku-postgresql
git push heroku main
heroku run php artisan migrate --seed
```

### Render Deployment

1. Create `render.yaml`:

```yaml
services:
  - type: web
    name: laravel-app
    runtime: php
    buildCommand: composer install --no-dev && php artisan optimize
    startCommand: php artisan serve --host=0.0.0.0 --port=10000
    envVars:
      - key: APP_ENV
        value: production
      - key: APP_KEY
        generateValue: true
      - key: DB_CONNECTION
        value: sqlite
```

### AWS Deployment (Elastic Beanstalk)

1. Create `Dockerrun.aws.json`:

```json
{
  "AWSEBDockerrunVersion": "1",
  "Image": {
    "Name": "laravel-app",
    "Update": "true"
  },
  "Ports": [
    {
      "ContainerPort": "8000"
    }
  ]
}
```

2. Deploy using EB CLI:

```bash
eb init -p php
eb create
```

