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

**Recall that models are Ruby classes that talk to the database, store and validate data, perform the business logic and otherwise do the heavy lifting.** 

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
class Todo < ApplicationRecord
    # model association 
    has_many :items, dependent: :destroy

    # validations 
    validates_presence_of :title, :created_by
end
```

```Ruby
# app/models/item.rb

class Item < ApplicationRecord
  # model association
  belongs_to :todo

  # validation 
  validates_presence_of :name 
  
end
```


# Controllers 

Now that the models are setup, we can generate the controllers 

```
rails g controller Todos 
rails g controller Items 
``` 

Generating controllers by default generates controller specs so we will instead be writing request specs. 

Request specs are designed to drive behavior through the full stack, including routing. This means that they can **hit the applications' HTTP endpoints as opposed to controller specs which call methods directly**. Since we are building an API application, this is exactly the kind of behavior that we want from the tests. 

We add a `requests` folder to the `spec` directory with the corresponding spec files

```
mkdir spec/requests && touch spec/requests/{todos_spec.rb,items_spec.rb} 
```

Before defining the request specs, we will add model factories which will be used to provide the test data. Here are the needed factory files: 

```
touch spec/factories/{todos.rb,items.rb}
```

We define the factories as follows: 

```Ruby 
# spec/factories/todos.rb
FactoryBot.define do
  factory :todo do
    title { Faker::Lorem.word }
    created_by { Faker::Number.number(10) }
  end
end
```
```Ruby 
# spec/factories/items.rb
FactoryBot.define do
  factory :item do
    name { Faker::StarWars.character }
    done false
    todo_id nil
  end
end
```

Wrapping the faker methods in a block ensures that the fake generates dynamic data every time the factory is invoked. This ensures unique data. 

Here is the spec for testing API endpoints: 

```ruby 
# spec/requests/todos_spec.rb
require 'rails_helper'

RSpec.describe 'Todos API', type: :request do
  # initialize test data 
  let!(:todos) { create_list(:todo, 10) }
  let(:todo_id) { todos.first.id }

  # Test suite for GET /todos
  describe 'GET /todos' do
    # make HTTP get request before each example
    before { get '/todos' }

    it 'returns todos' do
      # Note `json` is a custom helper to parse JSON responses
      expect(json).not_to be_empty
      expect(json.size).to eq(10)
    end

    it 'returns status code 200' do
      expect(response).to have_http_status(200)
    end
  end

  # Test suite for GET /todos/:id
  describe 'GET /todos/:id' do
    before { get "/todos/#{todo_id}" }

    context 'when the record exists' do
      it 'returns the todo' do
        expect(json).not_to be_empty
        expect(json['id']).to eq(todo_id)
      end

      it 'returns status code 200' do
        expect(response).to have_http_status(200)
      end
    end

    context 'when the record does not exist' do
      let(:todo_id) { 100 }

      it 'returns status code 404' do
        expect(response).to have_http_status(404)
      end

      it 'returns a not found message' do
        expect(response.body).to match(/Couldn't find Todo/)
      end
    end
  end

  # Test suite for POST /todos
  describe 'POST /todos' do
    # valid payload
    let(:valid_attributes) { { title: 'Learn Elm', created_by: '1' } }

    context 'when the request is valid' do
      before { post '/todos', params: valid_attributes }

      it 'creates a todo' do
        expect(json['title']).to eq('Learn Elm')
      end

      it 'returns status code 201' do
        expect(response).to have_http_status(201)
      end
    end

    context 'when the request is invalid' do
      before { post '/todos', params: { title: 'Foobar' } }

      it 'returns status code 422' do
        expect(response).to have_http_status(422)
      end

      it 'returns a validation failure message' do
        expect(response.body)
          .to match(/Validation failed: Created by can't be blank/)
      end
    end
  end

  # Test suite for PUT /todos/:id
  describe 'PUT /todos/:id' do
    let(:valid_attributes) { { title: 'Shopping' } }

    context 'when the record exists' do
      before { put "/todos/#{todo_id}", params: valid_attributes }

      it 'updates the record' do
        expect(response.body).to be_empty
      end

      it 'returns status code 204' do
        expect(response).to have_http_status(204)
      end
    end
  end

  # Test suite for DELETE /todos/:id
  describe 'DELETE /todos/:id' do
    before { delete "/todos/#{todo_id}" }

    it 'returns status code 204' do
      expect(response).to have_http_status(204)
    end
  end
end
```

This script starts by populating the database a list of 10 todo records using factory bot. We also have a helper method `json` which parses the JSON response to a Ruby Hash which is easier to work with in our tests. 


Right now if we run the tests we get failing routing errors - this is because we haven't defined the routes yet. We define them in `config/routes.rb`. 

```Ruby 
# config/routes.rb
Rails.application.routes.draw do
  resources :todos do
    resources :items
  end
end
```

In our route definition, we are creating todo resource with a nested item resource. This enforces the 1:m (one to many) associations at the routing level. To view the routes, we can run the following: 

```
rails routes
```

Now the routing error is gone, we can now work on fixing the controller failures. Here is the definition of the controller methods 

# Controller Methods 

**Recall controllers in Rails do the work of parsing user requests, data submissions, cookies, sessions and the "browser stuff". They are the managers that orders employees around and will ususally ask the model to retrieve resources** 

```Ruby 
class TodosController < ApplicationController 
  before_action :set_todo, only: [:show, :update, :destroy]

  # GET /todos 
  def index 
    @todos = Todo.all 
    json_response(@todos)
  end 

  # POST /todos 
  def create 
    @todo = Todo.create!(todo_params)
    json_response(@todo, :created)
  end 

  # GET /todos/:id 
  def show 
    json_response(@todo)
  end 

  # PUT /todos/:id
  def update 
    @todo.update(todo_params)
    head :no_content
  end 

  # DELETE /todos/:id
  def destroy
    @todo.destroy
    head :no_content
  end 

  private 

  def todo_params 
    # whitelist params 
    params.permit(:title, :created_by)
  end 

  def set_todo
    @todo = Todo.find(params[:id])
  end 
end 
```

You will notice that we have multiple helper functions: 

- `json_response` which responds with JSON and an HTTP status code (200 by default). We define this method in concerns folder 


```Ruby 
# app/controllers/concerns/response.rb
module Response
  def json_response(object, status = :ok)
    render json: object, status: status
  end
end
```

- `set_todo`: callback method to find a todo by id. In the case where the record does not exist, ActiveRecord will throw an exception `ActiveRecord::RecordNotFound`. We will rescue from this exception and return a `404` message 

* Note: `rescue` is similar to the try catch block from Java and there are methods for handling errors in Ruby on Rails. Without `rescue`, invalid inputs which should return `400 invalid input` instead respond with `500 server error` 


```Ruby 
# app/controllers/concerns/exception_handler.rb 
module ExceptionHandler
  # provides the more graceful `included` method 
  extend ActiveSupport::Concern 

  included do 
    rescue_from ActiveRecord::RecordNotFound do |e|
      json_response({message: e.message}, :not_found)
    end 

    rescue_from ActiveRecord::RecordInvalid do |e|
      json_response({message: e.message}, :unprocessable_entity)
    end 
  end 
end 
```

In our `create` method in `TodosController` we are using `!create` instead of just `create`. This will make the model raise an exception `ActiveRecord::RecordInvalid` and this allows us to avoid deep nested if statements in the controller. We can then resue this exception in the `ExceptionHandler` module. 

At the moment, the controller classes do not know about these helpers and we can fix that by including these modules in the application controller

```Ruby 
# app/controllers/application_controller.rb 
class ApplicationController < ActionController::API
  include Response 
  include ExceptionHandler 
end 
``` 

Now the tests should be working and we can try manual testing by starting the server 

```
rails s 
```
 
# TodoItems API 

Here is the spec for testing items API: 

```Ruby 
# spec/requests/items_spec.rb
require 'rails_helper'

RSpec.describe 'Items API' do
  # Initialize the test data
  let!(:todo) { create(:todo) }
  let!(:items) { create_list(:item, 20, todo_id: todo.id) }
  let(:todo_id) { todo.id }
  let(:id) { items.first.id }

  # Test suite for GET /todos/:todo_id/items
  describe 'GET /todos/:todo_id/items' do
    before { get "/todos/#{todo_id}/items" }

    context 'when todo exists' do
      it 'returns status code 200' do
        expect(response).to have_http_status(200)
      end

      it 'returns all todo items' do
        expect(json.size).to eq(20)
      end
    end

    context 'when todo does not exist' do
      let(:todo_id) { 0 }

      it 'returns status code 404' do
        expect(response).to have_http_status(404)
      end

      it 'returns a not found message' do
        expect(response.body).to match(/Couldn't find Todo/)
      end
    end
  end

  # Test suite for GET /todos/:todo_id/items/:id
  describe 'GET /todos/:todo_id/items/:id' do
    before { get "/todos/#{todo_id}/items/#{id}" }

    context 'when todo item exists' do
      it 'returns status code 200' do
        expect(response).to have_http_status(200)
      end

      it 'returns the item' do
        expect(json['id']).to eq(id)
      end
    end

    context 'when todo item does not exist' do
      let(:id) { 0 }

      it 'returns status code 404' do
        expect(response).to have_http_status(404)
      end

      it 'returns a not found message' do
        expect(response.body).to match(/Couldn't find Item/)
      end
    end
  end

  # Test suite for PUT /todos/:todo_id/items
  describe 'POST /todos/:todo_id/items' do
    let(:valid_attributes) { { name: 'Visit Narnia', done: false } }

    context 'when request attributes are valid' do
      before { post "/todos/#{todo_id}/items", params: valid_attributes }

      it 'returns status code 201' do
        expect(response).to have_http_status(201)
      end
    end

    context 'when an invalid request' do
      before { post "/todos/#{todo_id}/items", params: {} }

      it 'returns status code 422' do
        expect(response).to have_http_status(422)
      end

      it 'returns a failure message' do
        expect(response.body).to match(/Validation failed: Name can't be blank/)
      end
    end
  end

  # Test suite for PUT /todos/:todo_id/items/:id
  describe 'PUT /todos/:todo_id/items/:id' do
    let(:valid_attributes) { { name: 'Mozart' } }

    before { put "/todos/#{todo_id}/items/#{id}", params: valid_attributes }

    context 'when item exists' do
      it 'returns status code 204' do
        expect(response).to have_http_status(204)
      end

      it 'updates the item' do
        updated_item = Item.find(id)
        expect(updated_item.name).to match(/Mozart/)
      end
    end

    context 'when the item does not exist' do
      let(:id) { 0 }

      it 'returns status code 404' do
        expect(response).to have_http_status(404)
      end

      it 'returns a not found message' do
        expect(response.body).to match(/Couldn't find Item/)
      end
    end
  end

  # Test suite for DELETE /todos/:id
  describe 'DELETE /todos/:id' do
    before { delete "/todos/#{todo_id}/items/#{id}" }

    it 'returns status code 204' do
      expect(response).to have_http_status(204)
    end
  end
end
```



