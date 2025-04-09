# Building a Comprehensive CRUD Web Application with Rails

Here's a complete guide to building a feature-rich Rails application with all the requested functionality. Let's break this down into manageable steps.

## 1. Project Setup

First, let's create a new Rails application with the required technologies:

```bash
rails new my_app -d sqlite3 -j esbuild -c bootstrap
cd my_app
```

Add these gems to your Gemfile:

```ruby
# Authentication
gem 'devise'
gem 'pundit' # for authorization

# File uploads
gem 'activestorage'
gem 'image_processing', '~> 1.2'

# Pagination
gem 'pagy'

# Real-time updates
gem 'turbo-rails'
gem 'stimulus-rails'

# Testing
gem 'rspec-rails'
gem 'factory_bot_rails'
gem 'faker'
gem 'capybara'
gem 'shoulda-matchers'

# Deployment
gem 'rails_12factor' # For Heroku
```

Run `bundle install` and set up the basic configuration.

## 2. User Authentication with Devise

Set up Devise for user authentication:

```bash
rails generate devise:install
rails generate devise User
rails db:migrate
```

Configure Devise by editing `config/initializers/devise.rb` and setting your mailer defaults.

## 3. Role-Based Authorization with Pundit

Set up Pundit for authorization:

```bash
rails generate pundit:install
```

Create a role enum in your User model:

```ruby
# app/models/user.rb
class User < ApplicationRecord
  enum role: [:user, :admin]
  after_initialize :set_default_role, if: :new_record?

  private

  def set_default_role
    self.role ||= :user
  end
end
```

Create a migration to add the role column:

```bash
rails generate migration AddRoleToUsers role:integer
rails db:migrate
```

## 4. Setting Up the Main Resource

Let's assume we're building a blog application with Articles:

```bash
rails generate scaffold Article title:string content:text user:references
rails db:migrate
```

Set up the association in the User model:

```ruby
# app/models/user.rb
has_many :articles, dependent: :destroy
```

## 5. File Uploads with Active Storage

Set up Active Storage:

```bash
rails active_storage:install
rails db:migrate
```

Add file attachment to your Article model:

```ruby
# app/models/article.rb
class Article < ApplicationRecord
  has_one_attached :image
  validates :title, presence: true, length: { minimum: 5 }
  validates :content, presence: true
  
  # For image validation
  validate :acceptable_image

  private

  def acceptable_image
    return unless image.attached?
    
    unless image.byte_size <= 1.megabyte
      errors.add(:image, "is too big")
    end

    acceptable_types = ["image/jpeg", "image/png"]
    unless acceptable_types.include?(image.content_type)
      errors.add(:image, "must be a JPEG or PNG")
    end
  end
end
```

Update your form to include file upload:

```erb
<%= form.file_field :image, accept: "image/png,image/jpeg" %>
```

## 6. Pagination with Pagy

Add Pagy to your application controller:

```ruby
# app/controllers/application_controller.rb
include Pagy::Backend
```

In your view:

```erb
<% @pagy, @articles = pagy(Article.all) %>

<% @articles.each do |article| %>
  <!-- display article -->
<% end %>

<%= render partial: 'shared/pagination', locals: { pagy: @pagy } %>
```

Create a shared partial for pagination:

```erb
# app/views/shared/_pagination.html.erb
<div class="pagination">
  <%== pagy_nav(@pagy) %>
</div>
```

## 7. Search Functionality

Add search to your Article model:

```ruby
# app/models/article.rb
def self.search(search_term)
  if search_term.present?
    where('title LIKE ? OR content LIKE ?', "%#{search_term}%", "%#{search_term}%")
  else
    all
  end
end
```

In your controller:

```ruby
def index
  @q = params[:q]
  @pagy, @articles = pagy(Article.search(@q).order(created_at: :desc))
end
```

In your view:

```erb
<%= form_with url: articles_path, method: :get do |form| %>
  <%= form.text_field :q, value: params[:q] %>
  <%= form.submit "Search" %>
</form>
```

## 8. API Endpoints

Create an API controller:

```ruby
# app/controllers/api/v1/articles_controller.rb
module Api
  module V1
    class ArticlesController < ApplicationController
      before_action :authenticate_user!
      respond_to :json

      def index
        @articles = Article.all
        render json: @articles
      end

      def show
        @article = Article.find(params[:id])
        render json: @article
      end

      # Other CRUD actions...
    end
  end
end
```

Update your routes:

```ruby
# config/routes.rb
namespace :api do
  namespace :v1 do
    resources :articles
  end
end
```

## 9. Real-time Updates with Turbo Streams

Turbo is included by default in Rails 7. To use it for real-time updates:

In your controller:

```ruby
def create
  @article = current_user.articles.new(article_params)

  respond_to do |format|
    if @article.save
      format.turbo_stream
      format.html { redirect_to @article, notice: 'Article was successfully created.' }
      format.json { render :show, status: :created, location: @article }
    else
      format.html { render :new }
      format.json { render json: @article.errors, status: :unprocessable_entity }
    end
  end
end
```

Create a Turbo Stream view:

```erb
# app/views/articles/create.turbo_stream.erb
<%= turbo_stream.prepend "articles", @article %>
<%= turbo_stream.update Article.new, "" %>
```

## 10. Testing Setup

Set up RSpec:

```bash
rails generate rspec:install
```

Configure Factory Bot in `spec/rails_helper.rb`:

```ruby
RSpec.configure do |config|
  config.include FactoryBot::Syntax::Methods
end
```

Create a sample test:

```ruby
# spec/models/article_spec.rb
require 'rails_helper'

RSpec.describe Article, type: :model do
  let(:user) { create(:user) }
  
  it "is valid with valid attributes" do
    article = build(:article, user: user)
    expect(article).to be_valid
  end

  it "is not valid without a title" do
    article = build(:article, title: nil, user: user)
    expect(article).not_to be_valid
  end
end
```

## 11. Docker Setup

Create a Dockerfile:

```dockerfile
# Dockerfile
FROM ruby:3.2.2

RUN apt-get update -qq && apt-get install -y nodejs npm postgresql-client
RUN npm install -g yarn

WORKDIR /myapp
COPY Gemfile /myapp/Gemfile
COPY Gemfile.lock /myapp/Gemfile.lock
RUN bundle install

COPY . /myapp

# Add a script to be executed every time the container starts.
COPY entrypoint.sh /usr/bin/
RUN chmod +x /usr/bin/entrypoint.sh
ENTRYPOINT ["entrypoint.sh"]
EXPOSE 3000

CMD ["rails", "server", "-b", "0.0.0.0"]
```

Create an entrypoint.sh:

```bash
#!/bin/bash
set -e

# Remove a potentially pre-existing server.pid for Rails.
rm -f /myapp/tmp/pids/server.pid

# Then exec the container's main process (what's set as CMD in the Dockerfile).
exec "$@"
```

Create a docker-compose.yml:

```yaml
version: '3.8'
services:
  db:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: password
  web:
    build: .
    command: bash -c "rm -f tmp/pids/server.pid && bundle exec rails s -p 3000 -b '0.0.0.0'"
    volumes:
      - .:/myapp
    ports:
      - "3000:3000"
    depends_on:
      - db
    environment:
      - DATABASE_URL=postgres://postgres:password@db:5432/myapp_development

volumes:
  postgres_data:
```

## 12. Deployment

### Heroku Deployment:

1. Install Heroku CLI
2. Create a new Heroku app:
   ```bash
   heroku create
   ```
3. Set up production database:
   ```bash
   heroku addons:create heroku-postgresql:hobby-dev
   ```
4. Configure Rails for Heroku:
   ```ruby
   # config/environments/production.rb
   config.assets.compile = true
   ```
5. Deploy:
   ```bash
   git push heroku main
   heroku run rails db:migrate
   ```

### Render Deployment:

1. Create a render.yaml file:
   ```yaml
   services:
     - type: web
       name: myrailsapp
       runtime: ruby
       buildCommand: "./bin/render-build.sh"
       startCommand: "bundle exec puma -C config/puma.rb"
       envVars:
         - key: DATABASE_URL
           fromDatabase:
             name: myrailsappdb
             property: connectionString
         - key: RAILS_MASTER_KEY
           sync: false
   ```
2. Create a render-build.sh:
   ```bash
   #!/usr/bin/env bash
   set -o errexit
   bundle install
   bundle exec rails assets:precompile
   bundle exec rails assets:clean
   bundle exec rails db:migrate
   ```

### AWS Deployment (Elastic Beanstalk):

1. Install EB CLI
2. Initialize EB:
   ```bash
   eb init -p ruby-3.2 rails-app
   ```
3. Create environment:
   ```bash
   eb create rails-env --database.engine postgres --database.username ebroot --database.password password
   ```
4. Configure environment variables in AWS console
5. Deploy:
   ```bash
   eb deploy
   ```

## Final Notes

This comprehensive setup gives you a Rails application with:
- User authentication (Devise)
- Role-based authorization (Pundit)
- CRUD operations with input validation
- File uploads (Active Storage)
- Pagination (Pagy)
- Search functionality
- API endpoints
- Real-time updates (Turbo Streams)
- Docker support
- Deployment configurations for Heroku, Render, and AWS

Remember to:
1. Secure your application with proper authorization checks
2. Write tests for all critical functionality
3. Monitor your application in production
4. Regularly update your dependencies
5. Implement proper error handling and logging

Would you like me to elaborate on any specific part of this setup?