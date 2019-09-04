# Introduction 

Rails is a web application development framework written in the Ruby programming language. It is designed to make programming web applications easier by making assumptions about what every developer needs to get started. 

As of version 5, Rails core now supports API only applications - previously we would have relied on an external gem: `rails-api` which has now been merged to core rails. 

API only applications are slimmed down compared to traditional Rails web applications and according to the release notes, the API only application will: 

- Start the application with a limited set of middleware 
- Make the ApplicationController inherit from `ActionContorller::API` instead of `ActionController::Base` 
- Skip generation of view files 

This works to generate an API-centric framework excluding functionality that would otherwise be unused and unnecessary. 

This project is inspired by a Scotch.io tutorial and will be building a todo list API where users can manage their to-do lists and to-do items. 


# Prerequisites 

The application requires a ruby version >=2.2.2 and rails version 5. 

```
ruby -v 
rails -v 
``` 

Once this is completed we can get started 


## API Endpoints 

Here are the RESTful endpoints which will be exposed by our application 

**Endpoint**                        **Functionality** 

POST /signup                        Signup 
POST /auth/login                    Login 
GET /auth/logout                    Logout 
GET /todos                          List all todos 
POST /todos                         Create a new todo 
GET /todos/:id                      Get a todo 
PUT /todos/:id                      Update a todod 
DELETE /todods/:id                  Delete a todo and its items 
GET /todods/:id/items               Get a todo item 
PUT /todods/:id/items               Update a todo item 
DELETE /todos/:id/items             Delete a todo item 



## Project Setup 

Generate a new project called `todos-api` with: 

```
rails new todos-api --api -T
```

Note that the --api argument tells Rails that we want an API application and -T to exclude Minitest the default testing framework. We are going to be using RSpec instead to test our API. 


## Dependencies 

Here are the gems which we will be using for this project 

- `rspec-rails` - Testing framework 
- `factory_bot_rails` - Fixtures replacement with more straightforward syntax 
- `shoulda_matchers` - Provides RSpec with additional matchers 
- `database_cleaner` - Cleans the test database to ensure a clean state in each test suite 
- `faker` - library for generating fake data 

Now we are going to setup these dependencies - In the `Gemfile` : 

Add respec-rails to both `:development` and `:test` groups: 



```Ruby 
group :development, :test do 
  gem 'rspec-rails', '~> 3.5' 
end 
``` 

Next, we add `factory_bot_rails`, `shoulda_matchers`, `faker` and `database_cleaner` to the `:test` group 



```Ruby 
group :test do 
  gem 'factory_bot_rails', '~> 4.0'
  gem 'shoulda-matchers', '~> 3.1'
  gem 'faker'
  gem 'database_cleaner'
end 
```

Now we can install the gems by running: 

```
bundle install
```

Initialize the `spec` directory (where our tests will reside)

```
rails generate rspec:install
```

This will add the following files which can be used for configuration: 
- .rspec 
- spec/spec_helper.rb
- spec/rails_helper.rb 

Create a `factories` directory (factory bot uses this as the default). This is where we will define the model factories 

```
mkdir spec/factories 
```

In `spec/rails_helper.rb` we will have: 

```Ruby 
# require database cleaner 
require 'database_cleaner'

# configure shoulda matchers to use rspec as the test framework and full matcher libraries for rails
Shoulda::Matchers.configure do |config|
  config.integrate do |with|
    with.test_framework :rspec
    with.library :rails
  end
end


RSpec.configure do |config|
  # add `FactoryBot` methods
  config.include FactoryBot::Syntax::Methods

  # start by truncating all the tables but then use the faster transaction strategy the rest of the time.
  config.before(:suite) do
    DatabaseCleaner.clean_with(:truncation)
    DatabaseCleaner.strategy = :transaction
  end

  # start the transaction strategy as examples are run
  config.around(:each) do |example|
    DatabaseCleaner.cleaning do
      example.run
    end
  end

end
```

## Models 

We start by generating the `Todo` model

```
rails g model Todo title:string created_by:string
```

We included the model attributes in the model generation command so that we don't have to modify the migration file. The generator invokes `active record` and `rspec` to generate the migration, model, and `spec` respectively. 


```Ruby 
class CreateTodos < ActiveRecord::Migration[6.0]
  def change
    create_table :todos do |t|
      t.string :name
      t.boolean :done
      t.references :todo, foreign_key: true

      t.timestamps
    end
  end
end
```

We can run the migrations at this point: 
```
rails db:migrate
```

Following Test Driven Development, we write the model specs first: 

```Ruby 
require 'rails_helper'

RSpec.describe Todo, type: :model do
  # Association test 
  # ensure Todo model has 1:m relationship with the Item model 
  it {should have_many(:items).dependent(:destory)}
  # Validation tests
  # ensure columns title and created_by are present before saving 
  it {should validate_presence_of(:title)}
  it {should validate_presence_of(:created_by)}
end
```

RSpec has a very expressive DSL (Domain Specific Language), and the tests are almost read like a paragraph. The shoulda matcher gem provides RSpec with the association and validation matchers above.  

```Ruby 
require "rails_helper"

# Test suite for the Item model 
RSpec.describe Item, type: :model do 
    # Association test 
    # ensure an item record belongs to a single todo record 
    it {should belong_to(:todo)}
    # Validation test 
    # ensure column name is present before saving 
    it {should validate_presence_of(:name)}
end 
```

We can execute the specs as follows: 

```
bundle exec rspec
```

To no surprise, we have only one test passing and four failures. We will resolve these issues: 

```Ruby 
# app/models/todo.rb 
