# Point of Sale (POS) System with Ruby and SQLite

Here's a comprehensive guide to building a cross-platform POS system with Ruby and SQLite that includes all the requested functionality.

## System Architecture

```
POS System
├── Backend (Ruby)
│   ├── Models (ActiveRecord)
│   ├── Controllers
│   ├── Services
│   └── Database (SQLite)
└── Frontend (Shoes or GTK)
```

## Setup Instructions

### 1. Prerequisites

Install these on all platforms:
- Ruby (3.x recommended)
- SQLite3
- Bundler (`gem install bundler`)

### 2. Initialize the Project

```bash
mkdir pos_system
cd pos_system
bundle init
```

### 3. Edit Gemfile

```ruby
# Gemfile
source 'https://rubygems.org'

gem 'activerecord'
gem 'sqlite3'
gem 'bcrypt' # For authentication
gem 'pagy' # For pagination
gem 'ransack' # For search
gem 'aasm' # State machine for orders
gem 'roo' # For Excel exports
gem 'smarter_csv' # For CSV imports
gem 'gruff' # For charts

# Choose one GUI framework:
gem 'shoes' # Cross-platform but limited
# OR
gem 'gtk3' # More powerful but more complex
```

Run `bundle install`

## Database Setup

### 1. Database Configuration

```ruby
# config/database.rb
require 'active_record'

ActiveRecord::Base.establish_connection(
  adapter: 'sqlite3',
  database: 'db/pos_system.sqlite3'
)
```

### 2. Create Database Structure

```ruby
# db/migrate/001_create_tables.rb
class CreateTables < ActiveRecord::Migration[6.1]
  def change
    create_table :users do |t|
      t.string :username, null: false
      t.string :email, null: false
      t.string :password_digest
      t.string :role, default: 'cashier' # admin, manager, accountant, cashier
      t.boolean :active, default: true
      t.timestamps
    end

    create_table :products do |t|
      t.string :name, null: false
      t.string :barcode
      t.text :description
      t.decimal :price, precision: 10, scale: 2, null: false
      t.decimal :cost_price, precision: 10, scale: 2
      t.integer :stock_quantity, default: 0
      t.integer :reorder_threshold
      t.string :category
      t.timestamps
    end

    create_table :inventory_transactions do |t|
      t.references :product
      t.integer :quantity, null: false
      t.string :transaction_type # purchase, sale, return, adjustment
      t.text :notes
      t.references :user
      t.timestamps
    end

    create_table :orders do |t|
      t.string :status, default: 'pending' # pending, completed, cancelled
      t.decimal :total_amount, precision: 10, scale: 2
      t.decimal :tax_amount, precision: 10, scale: 2
      t.decimal :discount_amount, precision: 10, scale: 2
      t.references :user
      t.timestamps
    end

    create_table :order_items do |t|
      t.references :order
      t.references :product
      t.integer :quantity, null: false
      t.decimal :unit_price, precision: 10, scale: 2, null: false
      t.decimal :total_price, precision: 10, scale: 2, null: false
    end

    create_table :payments do |t|
      t.references :order
      t.decimal :amount, precision: 10, scale: 2, null: false
      t.string :payment_method # cash, card, mobile
      t.text :notes
      t.timestamps
    end

    create_table :financial_transactions do |t|
      t.string :transaction_type # sale, expense, refund
      t.decimal :amount, precision: 10, scale: 2, null: false
      t.text :description
      t.references :user
      t.references :order
      t.timestamps
    end
  end
end
```

Run the migration:
```bash
mkdir -p db/migrate
# Save the migration file then:
rake db:create
rake db:migrate
```

## Core Implementation

### 1. Authentication System

```ruby
# models/user.rb
class User < ActiveRecord::Base
  has_secure_password
  validates :username, presence: true, uniqueness: true
  validates :email, presence: true, uniqueness: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :role, inclusion: { in: %w[admin manager accountant cashier] }

  ROLES = %w[admin manager accountant cashier].freeze

  def admin?
    role == 'admin'
  end

  def manager?
    role == 'manager'
  end

  def accountant?
    role == 'accountant'
  end

  def cashier?
    role == 'cashier'
  end
end
```

### 2. Product Management

```ruby
# models/product.rb
class Product < ActiveRecord::Base
  validates :name, presence: true
  validates :price, presence: true, numericality: { greater_than: 0 }
  validates :stock_quantity, numericality: { only_integer: true, greater_than_or_equal_to: 0 }

  has_many :inventory_transactions
  has_many :order_items

  def self.low_stock
    where('stock_quantity <= reorder_threshold')
  end

  def update_stock(quantity, transaction_type, user)
    case transaction_type
    when 'purchase', 'return'
      self.stock_quantity += quantity
    when 'sale'
      self.stock_quantity -= quantity
    end

    if save
      inventory_transactions.create(
        quantity: quantity,
        transaction_type: transaction_type,
        user: user
      )
      true
    else
      false
    end
  end
end
```

### 3. Order Management

```ruby
# models/order.rb
class Order < ActiveRecord::Base
  include AASM

  belongs_to :user
  has_many :order_items, dependent: :destroy
  has_many :products, through: :order_items
  has_many :payments, dependent: :destroy

  validates :total_amount, numericality: { greater_than_or_equal_to: 0 }, allow_nil: true

  aasm column: 'status' do
    state :pending, initial: true
    state :completed
    state :cancelled

    event :complete do
      transitions from: :pending, to: :completed
    end

    event :cancel do
      transitions from: :pending, to: :cancelled
    end
  end

  def add_product(product, quantity)
    return false if quantity <= 0 || product.stock_quantity < quantity

    transaction do
      order_items.create(
        product: product,
        quantity: quantity,
        unit_price: product.price,
        total_price: product.price * quantity
      )

      update_totals
      product.update_stock(quantity, 'sale', user)
    end
    true
  rescue ActiveRecord::RecordInvalid
    false
  end

  def update_totals
    update(
      total_amount: order_items.sum(:total_price),
      tax_amount: calculate_tax,
      discount_amount: calculate_discount
    )
  end

  private

  def calculate_tax
    # Implement tax calculation logic
    total_amount * 0.1 # Example: 10% tax
  end

  def calculate_discount
    # Implement discount logic
    0 # No discount by default
  end
end
```

### 4. Reporting System

```ruby
# services/report_service.rb
class ReportService
  def self.sales_report(start_date, end_date)
    Order.completed.where(created_at: start_date..end_date)
         .group_by_day(:created_at)
         .sum(:total_amount)
  end

  def self.inventory_report
    Product.all.group_by(&:category).transform_values do |products|
      {
        count: products.count,
        total_value: products.sum { |p| p.price * p.stock_quantity }
      }
    end
  end

  def self.financial_report(start_date, end_date)
    {
      sales: Order.completed.where(created_at: start_date..end_date).sum(:total_amount),
      expenses: FinancialTransaction.expenses.where(created_at: start_date..end_date).sum(:amount),
      profits: calculate_profits(start_date, end_date)
    }
  end

  def self.export_to_csv(data, headers, file_path)
    CSV.open(file_path, 'w') do |csv|
      csv << headers
      data.each { |row| csv << row }
    end
  end

  private_class_method def self.calculate_profits(start_date, end_date)
    sales = Order.completed.where(created_at: start_date..end_date).sum(:total_amount)
    expenses = FinancialTransaction.expenses.where(created_at: start_date..end_date).sum(:amount)
    sales - expenses
  end
end
```

## User Interface (Using Shoes)

Here's a basic UI implementation using Shoes:

```ruby
# app/ui/main_app.rb
require 'shoes'
require_relative '../../config/database'
require_relative '../models/user'

class POSApp
  def initialize
    @current_user = nil
    login_screen
  end

  def login_screen
    Shoes.app(title: "POS System - Login", width: 400, height: 300) do
      stack(margin: 10) do
        title "POS System Login"
        
        flow do
          para "Username:"
          @username = edit_line(width: 200)
        end
        
        flow do
          para "Password:"
          @password = edit_line(width: 200, secret: true)
        end
        
        button "Login" do
          authenticate(@username.text, @password.text)
        end
      end
    end
  end

  def authenticate(username, password)
    user = User.find_by(username: username)
    if user&.authenticate(password)
      @current_user = user
      Shoes.app(title: "POS System - Dashboard", width: 800, height: 600) do
        dashboard_ui
      end
    else
      alert "Invalid username or password"
    end
  end

  def dashboard_ui
    # Implement main dashboard based on user role
    stack(margin: 10) do
      title "Welcome, #{@current_user.username} (#{@current_user.role})"
      
      if @current_user.admin? || @current_user.manager?
        button "Manage Products" do
          products_ui
        end
        
        button "View Reports" do
          reports_ui
        end
      end
      
      if @current_user.cashier? || @current_user.admin?
        button "New Sale" do
          new_sale_ui
        end
      end
    end
  end

  # Additional UI methods for products, sales, reports, etc.
  # ...
end

POSApp.new
```

## Packaging for Different Platforms

### 1. Using OCRA (Windows)

```bash
gem install ocra
ocra app/ui/main_app.rb --no-enc --add-all-core --windows --output pos_system.exe
```

### 2. Using Ruby Packer (Mac/Linux)

```bash
gem install ruby-packer
ruby-packer app/ui/main_app.rb -o pos_system
```

### 3. Creating Installers

For cross-platform distribution, consider using:

1. **Windows**: Inno Setup or NSIS
2. **Mac**: pkgbuild or create a .dmg
3. **Linux**: .deb or .rpm packages

## Additional Features to Implement

1. **Real-time Updates**: Use a background thread to poll for changes
2. **Barcode Scanning**: Integrate with barcode scanner libraries
3. **Receipt Printing**: Use system printing commands
4. **Data Backup**: Automatically backup SQLite database
5. **Multi-language Support**: Use i18n gem

## Security Considerations

1. Password hashing with bcrypt
2. Input validation for all models
3. Role-based access control
4. SQL injection prevention (ActiveRecord handles this)
5. Secure session management

This implementation provides a solid foundation for a cross-platform POS system with Ruby and SQLite. You can extend it further based on specific business requirements.