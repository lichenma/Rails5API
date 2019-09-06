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


The tests will not be working until we define the todo **items controller**: 

```Ruby 
# app/controllers/items_controller.rb 
class ItemsController < ApplicationController
  before_action :set_todo
  before_action :set_todo_item, only: [:show, :update, :destroy]

  # GET /todos/:todo_id/items
  def index
    json_response(@todo.items)
  end

  # GET /todos/:todo_id/items/:id
  def show
    json_response(@item)
  end

  # POST /todos/:todo_id/items
  def create
    @todo.items.create!(item_params)
    json_response(@todo, :created)
  end

  # PUT /todos/:todo_id/items/:id
  def update
    @item.update(item_params)
    head :no_content
  end

  # DELETE /todos/:todo_id/items/:id
  def destroy
    @item.destroy
    head :no_content
  end

  private

  def item_params
    params.permit(:name, :done)
  end

  def set_todo
    @todo = Todo.find(params[:todo_id])
  end

  def set_todo_item
    @item = @todo.items.find_by!(id: params[:id]) if @todo
  end
end
```

That's all for the first part, so far we: 
- Generated an API application with Rails 5 
- Setup `RSpec` testing framework with `Factory Bot`, `Database Cleaner`, `Shoulda Matchers` and `Faker` 
- Build models and controllers with TDD (Test Driven Development)
- Make HTTP requests to an API using CURL 


## For this next part we will implement token-based authentication with JWT

Our API should also be able to support user accounts with each user having the ability to manage their own resources. We start by generating a user model: 

```
$ rails g model User name:string email:string password_digest:string
# run the migrations
$ rails db:migrate
# make sure the test environment is ready
$ rails db:test:prepare
```

* Note: A migration in Rails is a change to the database schema but first we must generate a table (this is created when we create a model)


Let's define the user model spec for testing: 

```Ruby 
# spec/models/user_spec.rb
require 'rails_helper'

# Test suite for User model
RSpec.describe User, type: :model do
  # Association test
  # ensure User model has a 1:m relationship with the Todo model
  it { should have_many(:todos) }
  # Validation tests
  # ensure name, email and password_digest are present before save
  it { should validate_presence_of(:name) }
  it { should validate_presence_of(:email) }
  it { should validate_presence_of(:password_digest) }
end
```

Users should be able to manage their own todo lists, thus the user model should have a one to many relationship with the todo model. We also want to make sure that every user account creation has all of the required credentials. 

Now, we create a user factory which will be used by the test suite to create test users. 


```Ruby 
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    name { Faker::Name.name }
    email 'foo@bar.com'
    password 'foobar'
  end
end
```

**Next we will implement the user model**

```Ruby 
# app/models/user.rb 
class User < ApplicationRecord
  # encrypt password
  has_secure_password

  # Model associations
  has_many :todos, foreign_key: :created_by
  # Validations
  validates_presence_of :name, :email, :password_digest
end
```

Our user model defines a 1:m relationship with the todo model also adds field validations. Note that the user also calls the method `has_secure_password` which is a method providing ways to authenticate against a `bcrypt` password. It is this mechanism that requires us to have a `password_digest` field instead of a normal `password` field. 

Thus we need to have the `bcrypt` gem as a dependency. 

```
# Gemfile 
gem 'bcrypt', '~> 3.1.7'
```

Now we can install the gem and run the tests: 

```
bundle install 
bundle exec rspec 
```

Now the model is all setup to save users and we can add the rest of the service classes for authentication: 
- `JsonWebToken`: Encode and decode `jwt` tokens 
- `AuthorizeApiRequest`: Authorize each API request
- `AuthenticateUser`: Authenticate Users 
- `AuthenticationController`: Orchestrate authentication process 


# JSON Web Token 

We will be using the `jwt` gem to manage JSON web tokens: 

```Ruby 
# Gemfile 
gem 'jwt' 
```

```
bundle install 
```

Our class will be in the `lib` directory since it is not domain specific and if we were to move it to a different application it should work with minimal configuration. However, as of Rails 5, **autoloading is disabled in production because of thread safety**.

This is an issue as `lib` is part of auto-load paths and so to counter this we will add our `lib` in `app` since all code in the app is auto-loaded in development and eager-loaded in production. 

```
$ mkdir app/lib
$ touch app/lib/json_web_token.rb
```

In `json_web_token.rb` we define the jwt singleton: 

```Ruby 
# app/lib/json_web_token.rb 
class JsonWebToken 
  # secret to encode and decode token 
  HMAC_SECRET = Rails.application.secrets.secret_key_base

  def self.encode(payload, exp = 24.hours.from_now)
    # set expiry to 24 hours from creation time
    payload[:exp] = exp.to_i
    # sign token with application secret
    JWT.encode(payload, HMAC_SECRET)
  end

  def self.decode(token)
    # get payload; first index in decoded Array
    body = JWT.decode(token, HMAC_SECRET)[0]
    HashWithIndifferentAccess.new body
    # rescue from all decode errors
  rescue JWT::DecodeError => e
    # raise custom error to be handled by custom handler
    raise ExceptionHandler::InvalidToken, e.message
  end
end
```

This singleton wraps `JWT` and provides token encoding and decoding methods. The encode method is responsible for creating tokens based on the user id payload and expiration period. Since every Rails application has a unique secret key, we will use that as our secret to sign tokens. The decode method, on the other hand, accepts a token and attempts to decode it using the same secret used in encoding. In the event decoding fails (possibly from expiration/validation), `JWT` will raise respective exceptions which will be caught and handled by the `Exception Handler` module. 



```Ruby 
module ExceptionHandler
  extend ActiveSupport::Concern

  # Define custom error subclasses - rescue catches `StandardErrors`
  class AuthenticationError < StandardError; end
  class MissingToken < StandardError; end
  class InvalidToken < StandardError; end

  included do
    # Define custom handlers
    rescue_from ActiveRecord::RecordInvalid, with: :four_twenty_two
    rescue_from ExceptionHandler::AuthenticationError, with: :unauthorized_request
    rescue_from ExceptionHandler::MissingToken, with: :four_twenty_two
    rescue_from ExceptionHandler::InvalidToken, with: :four_twenty_two

    rescue_from ActiveRecord::RecordNotFound do |e|
      json_response({ message: e.message }, :not_found)
    end
  end

  private

  # JSON response with message; Status code 422 - unprocessable entity
  def four_twenty_two(e)
    json_response({ message: e.message }, :unprocessable_entity)
  end

  # JSON response with message; Status code 401 - Unauthorized
  def unauthorized_request(e)
    json_response({ message: e.message }, :unauthorized)
  end
end
```

We have added custom `Standard Error` sub-classes to help handle exceptions raised. By defining error classes as sub-classes of standard error, we are able to `rescue_from` them once raised 

# Authorize API Request 

This class will be responsible for authorizing all API requests and making sure that all requests have a valid token and user payload. 

This is an authentication service class so it will be in `app/auth`: 

```
# create auth folder to house auth services
$ mkdir app/auth
$ touch app/auth/authorize_api_request.rb
# Create corresponding spec files
$ mkdir spec/auth
$ touch spec/auth/authorize_api_request_spec.rb
```

Defining the specifications for testing the authorize API: 
```Ruby
# spec/auth/authorize_api_request_spec.rb
require 'rails_helper'

RSpec.describe AuthorizeApiRequest do
  # Create test user
  let(:user) { create(:user) }
  # Mock `Authorization` header
  let(:header) { { 'Authorization' => token_generator(user.id) } }
  # Invalid request subject
  subject(:invalid_request_obj) { described_class.new({}) }
  # Valid request subject
  subject(:request_obj) { described_class.new(header) }

  # Test Suite for AuthorizeApiRequest#call
  # This is our entry point into the service class
  describe '#call' do
    # returns user object when request is valid
    context 'when valid request' do
      it 'returns user object' do
        result = request_obj.call
        expect(result[:user]).to eq(user)
      end
    end

    # returns error message when invalid request
    context 'when invalid request' do
      context 'when missing token' do
        it 'raises a MissingToken error' do
          expect { invalid_request_obj.call }
            .to raise_error(ExceptionHandler::MissingToken, 'Missing token')
        end
      end

      context 'when invalid token' do
        subject(:invalid_request_obj) do
          # custom helper method `token_generator`
          described_class.new('Authorization' => token_generator(5))
        end

        it 'raises an InvalidToken error' do
          expect { invalid_request_obj.call }
            .to raise_error(ExceptionHandler::InvalidToken, /Invalid token/)
        end
      end

      context 'when token is expired' do
        let(:header) { { 'Authorization' => expired_token_generator(user.id) } }
        subject(:request_obj) { described_class.new(header) }

        it 'raises ExceptionHandler::ExpiredSignature error' do
          expect { request_obj.call }
            .to raise_error(
              ExceptionHandler::InvalidToken,
              /Signature has expired/
            )
        end
      end

      context 'fake token' do
        let(:header) { { 'Authorization' => 'foobar' } }
        subject(:invalid_request_obj) { described_class.new(header) }

        it 'handles JWT::DecodeError' do
          expect { invalid_request_obj.call }
            .to raise_error(
              ExceptionHandler::InvalidToken,
              /Not enough or too many segments/
            )
        end
      end
    end
  end
end
```

The `AuthorizeApiRequest` service should have an entry method `call` that returns a valid user object when the request is valid and raises an error when invalid. Note the helper methods used for test: 

- `token_generator`: generate test token 
- `expired_token_generator`: generate expired token 

We will define these functions in `spec/support`: 

```
# create module file
$ touch spec/support/controller_spec_helper.rb
```

Here are the contents of the file: 

```Ruby
# spec/support/controller_spec_helper.rb
module ControllerSpecHelper
  # generate tokens from user id
  def token_generator(user_id)
    JsonWebToken.encode(user_id: user_id)
  end

  # generate expired tokens from user id
  def expired_token_generator(user_id)
    JsonWebToken.encode({ user_id: user_id }, (Time.now.to_i - 10))
  end

  # return valid headers
  def valid_headers
    {
      "Authorization" => token_generator(user.id),
      "Content-Type" => "application/json"
    }
  end

  # return invalid headers
  def invalid_headers
    {
      "Authorization" => nil,
      "Content-Type" => "application/json"
    }
  end
end
```

We are also using additional test helpers to generate headers. In order to make use of these helper methods, we need to include the module in `rails helper`. 



```Ruby
RSpec.configure do |config|
  # [...]
  # previously `config.include RequestSpecHelper, type: :request`
  config.include RequestSpecHelper
  config.include ControllerSpecHelper
  # [...]
end
```


Now having written up our tests, we can define the class: 

```Ruby 
# app/auth/authorize_api_request.rb
class AuthorizeApiRequest
  def initialize(headers = {})
    @headers = headers
  end

  # Service entry point - return valid user object
  def call
    {
      user: user
    }
  end

  private

  attr_reader :headers

  def user
    # check if user is in the database
    # memoize user object
    @user ||= User.find(decoded_auth_token[:user_id]) if decoded_auth_token
    # handle user not found
  rescue ActiveRecord::RecordNotFound => e
    # raise custom error
    raise(
      ExceptionHandler::InvalidToken,
      ("#{Message.invalid_token} #{e.message}")
    )
  end

  # decode authentication token
  def decoded_auth_token
    @decoded_auth_token ||= JsonWebToken.decode(http_auth_header)
  end

  # check for token in `Authorization` header
  def http_auth_header
    if headers['Authorization'].present?
      return headers['Authorization'].split(' ').last
    end
      raise(ExceptionHandler::MissingToken, Message.missing_token)
  end
end
```

The `AuthorizeApiRequest` service gets the token from the authorization headers, attempts to decode it to return a valid user object. We also have a singleton `Message` to house all our messages. This is an easier way to manage all of the application messages. We will define it in `app/lib` since it is non-domain-specific. 


```Ruby 
# app/lib/message.rb
class Message
  def self.not_found(record = 'record')
    "Sorry, #{record} not found."
  end

  def self.invalid_credentials
    'Invalid credentials'
  end

  def self.invalid_token
    'Invalid token'
  end

  def self.missing_token
    'Missing token'
  end

  def self.unauthorized
    'Unauthorized request'
  end

  def self.account_created
    'Account created successfully'
  end

  def self.account_not_created
    'Account could not be created'
  end

  def self.expired_token
    'Sorry, your token has expired. Please login to continue.'
  end
end
```


Now we can run the authentication specs and everything should pass. 

```
bundle exec rspec spec/auth -fd
```


# Authenticate User 

This class will be responsible for authenticating users via email and password. 

This is also an authentication service class so it will live in `app/auth` 


```
$ touch app/auth/authenticate_user.rb
# Create corresponding spec file
$ touch spec/auth/authenticate_user_spec.rb
```

Now let's define the specifications: 

```Ruby 
# spec/auth/authenticate_user_spec.rb
require 'rails_helper'

RSpec.describe AuthenticateUser do
  # create test user
  let(:user) { create(:user) }
  # valid request subject
  subject(:valid_auth_obj) { described_class.new(user.email, user.password) }
  # invalid request subject
  subject(:invalid_auth_obj) { described_class.new('foo', 'bar') }

  # Test suite for AuthenticateUser#call
  describe '#call' do
    # return token when valid request
    context 'when valid credentials' do
      it 'returns an auth token' do
        token = valid_auth_obj.call
        expect(token).not_to be_nil
      end
    end

    # raise Authentication Error when invalid request
    context 'when invalid credentials' do
      it 'raises an authentication error' do
        expect { invalid_auth_obj.call }
          .to raise_error(
            ExceptionHandler::AuthenticationError,
            /Invalid credentials/
          )
      end
    end
  end
end
```

The `AuthenticateUser` service also has an entry point `#call`. It should return a token when the user credentials are valid and raise an error when they are not. Running the auth specs and they should fail with a load error. To fix that we will go ahead and implement the class. 


```Ruby
# app/auth/authenticate_user.rb
class AuthenticateUser
  def initialize(email, password)
    @email = email
    @password = password
  end

  # Service entry point
  def call
    JsonWebToken.encode(user_id: user.id) if user
  end

  private

  attr_reader :email, :password

  # verify user credentials
  def user
    user = User.find_by(email: email)
    return user if user && user.authenticate(password)
    # raise Authentication error if credentials are invalid
    raise(ExceptionHandler::AuthenticationError, Message.invalid_credentials)
  end
end
```


The `AuthenticateUser` service accepts a user email and password, checks if they are valid and then creates a token with the user id as the payload. 

```
$ bundle exec rspec spec/auth -fd
```


# Authentication Controller 

This controller is responsible for orchestrating the authentication process making use of the **authentication service** we have just created. 


```
# generate the Authentication Controller
$ rails g controller Authentication
```

First, we create a new test: 

```Ruby
# spec/requests/authentication_spec.rb
require 'rails_helper'

RSpec.describe 'Authentication', type: :request do
  # Authentication test suite
  describe 'POST /auth/login' do
    # create test user
    let!(:user) { create(:user) }
    # set headers for authorization
    let(:headers) { valid_headers.except('Authorization') }
    # set test valid and invalid credentials
    let(:valid_credentials) do
      {
        email: user.email,
        password: user.password
      }.to_json
    end
    let(:invalid_credentials) do
      {
        email: Faker::Internet.email,
        password: Faker::Internet.password
      }.to_json
    end

    # set request.headers to our custon headers
    # before { allow(request).to receive(:headers).and_return(headers) }

    # returns auth token when request is valid
    context 'When request is valid' do
      before { post '/auth/login', params: valid_credentials, headers: headers }

      it 'returns an authentication token' do
        expect(json['auth_token']).not_to be_nil
      end
    end

    # returns failure message when request is invalid
    context 'When request is invalid' do
      before { post '/auth/login', params: invalid_credentials, headers: headers }

      it 'returns a failure message' do
        expect(json['message']).to match(/Invalid credentials/)
      end
    end
  end
end
```

Next we will define the **authentication controller class** which will expose an `/auth/login` endpoint that accepts user credentials and returns a JSON response with the result. 


```Ruby 
# app/controllers/authentication_controller.rb
class AuthenticationController < ApplicationController
  # return auth token once user is authenticated
  def authenticate
    auth_token =
      AuthenticateUser.new(auth_params[:email], auth_params[:password]).call
    json_response(auth_token: auth_token)
  end

  private

  def auth_params
    params.permit(:email, :password)
  end
end
```

Notice that the authentication controller is very compact and this is due to all of the helper functions which we have defined in the service architecture. We use the authentication controller to piece everything together ... to control authentication. Now we need to add routing for the authentication action. 


```Ruby 
# config/routes.rb
Rails.application.routes.draw do
  # [...]
  post 'auth/login', to: 'authentication#authenticate'
end
```


In order to have users authenticate we need to have them signup first - this will be handled by the users controller. 


```
# generate users controller
$ rails g controller Users
# generate users request spec
$ touch spec/requests/users_spec.rb
```

Next we write up the testing specifications for user signup: 

```Ruby 
# spec/requests/users_spec.rb
require 'rails_helper'

RSpec.describe 'Users API', type: :request do
  let(:user) { build(:user) }
  let(:headers) { valid_headers.except('Authorization') }
  let(:valid_attributes) do
    attributes_for(:user, password_confirmation: user.password)
  end

  # User signup test suite
  describe 'POST /signup' do
    context 'when valid request' do
      before { post '/signup', params: valid_attributes.to_json, headers: headers }

      it 'creates a new user' do
        expect(response).to have_http_status(201)
      end

      it 'returns success message' do
        expect(json['message']).to match(/Account created successfully/)
      end

      it 'returns an authentication token' do
        expect(json['auth_token']).not_to be_nil
      end
    end

    context 'when invalid request' do
      before { post '/signup', params: {}, headers: headers }

      it 'does not create a new user' do
        expect(response).to have_http_status(422)
      end

      it 'returns failure message' do
        expect(json['message'])
          .to match(/Validation failed: Password can't be blank, Name can't be blank, Email can't be blank, Password digest can't be blank/)
      end
    end
  end
end
```

The user controller should expose a `/signup` endpoint that accepts user information and returns a JSON response with the result. Add the signup route: 



```Ruby 
# config/routes.rb
Rails.application.routes.draw do
  # [...]
  post 'signup', to: 'users#create'
end
```

Now let's implement the **user controller**: 


```Ruby 
# app/controllers/users_controller.rb
class UsersController < ApplicationController
  # POST /signup
  # return authenticated token upon signup
  def create
    user = User.create!(user_params)
    auth_token = AuthenticateUser.new(user.email, user.password).call
    response = { message: Message.account_created, auth_token: auth_token }
    json_response(response, :created)
  end

  private

  def user_params
    params.permit(
      :name,
      :email,
      :password,
      :password_confirmation
    )
  end
end
```

The users controller attempts to create a user and returns a JSON response with the result. We use Active Record's `create!` method so that in the event there's an error, an exception will b raised and handled in the exception handler. 


Current we have wired up the user authentication portion but the API is still open - it does not authorize requests with a token. To fix this, we need to make sure that on every request (except authentication) our API checks for a valid token. To achieve this we will implement a **callback in the application controller that authenticates every request. Since all controllers inherit from application controller, it will be propagated to all controllers**. 


First we have the controller specifications: 

```Ruby 
# spec/controllers/application_controller_spec.rb
require "rails_helper"

RSpec.describe ApplicationController, type: :controller do
  # create test user
  let!(:user) { create(:user) }
   # set headers for authorization
  let(:headers) { { 'Authorization' => token_generator(user.id) } }
  let(:invalid_headers) { { 'Authorization' => nil } }

  describe "#authorize_request" do
    context "when auth token is passed" do
      before { allow(request).to receive(:headers).and_return(headers) }

      # private method authorize_request returns current user
      it "sets the current user" do
        expect(subject.instance_eval { authorize_request }).to eq(user)
      end
    end

    context "when auth token is not passed" do
      before do
        allow(request).to receive(:headers).and_return(invalid_headers)
      end

      it "raises MissingToken error" do
        expect { subject.instance_eval { authorize_request } }.
          to raise_error(ExceptionHandler::MissingToken, /Missing token/)
      end
    end
  end
end
```

Now that we have the tests, we can implement the **authorization controller**: 

```Ruby 
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  include Response
  include ExceptionHandler

  # called before every action on controllers
  before_action :authorize_request
  attr_reader :current_user

  private

  # Check for valid request token and return user
  def authorize_request
    @current_user = (AuthorizeApiRequest.new(request.headers).call)[:user]
  end
end
```

On every request, the application will verify the request by calling the request authorization service. If the request is authorized, it will set the `current user` object to be used in other controllers. 


The error handling implementation used in this controller also allows us to have minimal guard clauses and conditionals in our controllers. 


Remember that when signing up and authenticating a user we won't need a token. We only require user credentials. Thus, we can skip request authentication for these two actions. 


First the authentication action: 


```Ruby 
# app/controllers/authentication_controller.rb
class AuthenticationController < ApplicationController
  skip_before_action :authorize_request, only: :authenticate
  # [...]
end
```


Then the user signup action: 


```Ruby
# app/controllers/users_controller.rb
class UsersController < ApplicationController
  skip_before_action :authorize_request, only: :create
  # [...]
end
```


Right now when we run the tests we should see that the Todo and TodoItems API is failing. This is exactly as we expected - it means that our request authorization is working as intended. Now let's update the API to cater for this. 


In the Todos request spec, we will make a partial update to all of our requests to have authorization headers and a JSON payload. 


```Ruby 
# spec/requests/todos_spec.rb
require 'rails_helper'

RSpec.describe 'Todos API', type: :request do
  # add todos owner
  let(:user) { create(:user) }
  let!(:todos) { create_list(:todo, 10, created_by: user.id) }
  let(:todo_id) { todos.first.id }
  # authorize request
  let(:headers) { valid_headers }

  describe 'GET /todos' do
    # update request with headers
    before { get '/todos', params: {}, headers: headers }

    # [...]
  end

  describe 'GET /todos/:id' do
    before { get "/todos/#{todo_id}", params: {}, headers: headers }
    # [...]
    end
    # [...]
  end

  describe 'POST /todos' do
    let(:valid_attributes) do
      # send json payload
      { title: 'Learn Elm', created_by: user.id.to_s }.to_json
    end

    context 'when request is valid' do
      before { post '/todos', params: valid_attributes, headers: headers }
      # [...]
    end

    context 'when the request is invalid' do
      let(:invalid_attributes) { { title: nil }.to_json }
      before { post '/todos', params: invalid_attributes, headers: headers }

      it 'returns status code 422' do
        expect(response).to have_http_status(422)
      end

      it 'returns a validation failure message' do
        expect(json['message'])
          .to match(/Validation failed: Title can't be blank/)
      end
  end

  describe 'PUT /todos/:id' do
    let(:valid_attributes) { { title: 'Shopping' }.to_json }

    context 'when the record exists' do
      before { put "/todos/#{todo_id}", params: valid_attributes, headers: headers }
      # [...]
    end
  end

  describe 'DELETE /todos/:id' do
    before { delete "/todos/#{todo_id}", params: {}, headers: headers }
    # [...]
  end
end
```

Our todos controller also doesn't know about users yet so let's fix that: 

```Ruby 
# app/controllers/todos_controller.rb
class TodosController < ApplicationController
  # [...]
  # GET /todos
  def index
    # get current user todos
    @todos = current_user.todos
    json_response(@todos)
  end
  # [...]
  # POST /todos
  def create
    # create todos belonging to current user
    @todo = current_user.todos.create!(todo_params)
    json_response(@todo, :created)
  end
  # [...]
  private

  # remove `created_by` from list of permitted parameters
  def todo_params
    params.permit(:title)
  end
  # [...]
end
```

Now let's update the Items API with the same: 

```Ruby 
# spec/requests/items_spec.rb
require 'rails_helper'

RSpec.describe 'Items API' do
  let(:user) { create(:user) }
  let!(:todo) { create(:todo, created_by: user.id) }
  let!(:items) { create_list(:item, 20, todo_id: todo.id) }
  let(:todo_id) { todo.id }
  let(:id) { items.first.id }
  let(:headers) { valid_headers }

  describe 'GET /todos/:todo_id/items' do
    before { get "/todos/#{todo_id}/items", params: {}, headers: headers }

    # [...]
  end

  describe 'GET /todos/:todo_id/items/:id' do
    before { get "/todos/#{todo_id}/items/#{id}", params: {}, headers: headers }

    # [...]
  end

  describe 'POST /todos/:todo_id/items' do
    let(:valid_attributes) { { name: 'Visit Narnia', done: false }.to_json }

    context 'when request attributes are valid' do
      before do
        post "/todos/#{todo_id}/items", params: valid_attributes, headers: headers
      end

      # [...]
    end

    context 'when an invalid request' do
      before { post "/todos/#{todo_id}/items", params: {}, headers: headers }

      # [...]
    end
  end

  describe 'PUT /todos/:todo_id/items/:id' do
    let(:valid_attributes) { { name: 'Mozart' }.to_json }

    before do
      put "/todos/#{todo_id}/items/#{id}", params: valid_attributes, headers: headers
    end

    # [...]
    # [...]
  end

  describe 'DELETE /todos/:id' do
    before { delete "/todos/#{todo_id}/items/#{id}", params: {}, headers: headers }

    # [...]
  end
end
```

Now if we re-run the tests then everything should be working properly. At this point the application is able to implement token based authentication with JWT. 

We can also test locally and manually: 

```
# Attempt to access API without a token
$ http :3000/todos
# Signup a new user - get token from here
$ http :3000/signup name=ash email=ash@email.com password=foobar password_confirmation=foobar
# Get new user todos
$ http :3000/todos \
> Authorization:'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjozLCJleHAiOjE0ODg5MDEyNjR9.7txvLgDzFdX5NIUGYb3W45oNIXinwB_ITu3jdlG5Dds'
# create todo for new user
$ http POST :3000/todos title=Beethoven \
> Authorization:'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjozLCJleHAiOjE0ODg5MDEyNjR9.7txvLgDzFdX5NIUGYb3W45oNIXinwB_ITu3jdlG5Dds'
# Get create todos
$ http :3000/todos \
Authorization:'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjozLCJleHAiOjE0ODg5MDEyNjR9.7txvLgDzFdX5NIUGYb3W45oNIXinwB_ITu3jdlG5Dds'
```


Next we will cover `API versioning`, `pagination` and `serialization`. 


# Versioning 

When building an API whether public or internal facing, it is highly recommended that developers use versioning. When the API is public facing it is in our best interest to establish a contract with your clients - every breaking change should be a new version. 

In order to version a Rails API, we require two things 

1. A route constraint - this will select a version based on the request headers 
2. Namespace the controllers - have different controller namespaces to handle different versions 


Rails routing supports `advanced constraints`. Provided that an object responds to `matches?`, we can control which controller handles a specific route. 


We will define a class `ApiVersion` that checks the API version from the request headers and routes to the appropriate controller module. The class will live in `app/lib` since it's non-domain-specific. 


```
# create the class file
$ touch app/lib/api_version.rb
```

Implement the class `ApiVersion`: 

* Note: In ruby any variables declared with the `@` symbol are instance variables which are available to all methods within the class 

* Note: Normal variables declared without the `@` symbol are local variables which only exists within its scope (current block)
  

```Ruby
# app/lib/api_version.rb
class ApiVersion
  attr_reader :version, :default

  def initialize(version, default = false)
    @version = version
    @default = default
  end

  # check whether version is specified or is default
  def matches?(request)
    check_headers(request.headers) || default
  end

  private

  def check_headers(headers)
    # check version from Accept headers; expect custom media type `todos`
    accept = headers[:accept]
    accept && accept.include?("application/vnd.todos.#{version}+json")
  end
end
```

The `ApiVersion` class accepts a version and a default flag on initialization. In accordance with Rails constraints, we implement an instance method `matches?`. This method will be called with the request object upon initialization. From the request obejct, we can access the `Accept`. This process is called **content negotiation**.



# Content Negotiation 

REST is closely tied to the HTTP specification. HTTP defines mechanisms that **make it possible to serve different versions (representations) of a resource at the same URI. This is called content negotiation.**

Our `ApiVersion` class inplements server-driven content negotiation where the client (user agent) informs the server what media types it understands by providing an **Accept HTTP head**. 

According to the `Media Type Specification`, you can **define your own media types using the vendor tree**, `application/vnd.example.resource+json`. 


* Note: The `vendor tree` is used for media types associated with publicly available products. It uses the "vnd" facet 


Thus, we define a custom vendor media type `application/vnd.todos.{version_number}+json` giving clients the ability to choose which API version they require. 


Now that we have the constraint class, let's change our routing to accomodate this. Since we don't want to have the version number as part of the URI (this is argued as an anti-pattern), we will make use of the **module scope to namespace our controllers**.


Let's move the exisiting todos and todo-items resources into a `v1` namespace. 



```Ruby
# config/routes
Rails.application.routes.draw do
  # For details on the DSL available within this file, see http://guides.rubyonrails.org/routing.html

  # namespace the controllers without affecting the URI
  scope module: :v1, constraints: ApiVersion.new('v1', true) do
    resources :todos do
      resources :items
    end
  end


  post 'auth/login', to: 'authentication#authenticate'
  post 'signup', to: 'users#create'
end
```


We have set the version constraint at the namespace level. This means that it will be applied to all resources within it. We have also defined `v1` as the default version in cases where the version is not provided. 


















