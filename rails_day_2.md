# Ruby on Rails

## Starting a new rails app

1. `rails new {APP_NAME} -d postgresql --skip-action-mailer --skip-action-cable --skip-coffee --skip-test --skip-system-test`
1. create empty repo in github
1. add remote from github repo `git remote add origin {GITHUB PATH}`
1. add - `git add .`, commit - `git commit -m "{YOUR MESSAGE HERE, initial commit maybe?}"`, push - `git push -u origin master`
1. add any gems you want to the gemfile
1. `bundle install`
1. add, commit and push

### Gems for development and test

- inside the group `group :development, :test do` add:
```
gem 'pry'
gem 'pry-byebug'
gem 'pry-rails'
gem 'faker'
```

### Build a Schema - use migrations

- tables are plural and lowercase, for example `accounts`
- do not use type as a column in a rails database
- `rails g migration createAccounts` and then update the migration
```
class CreateAccounts < ActiveRecord::Migration[5.1]
  def change
    create_table :accounts do |t|
      t.string :name, null: false
      t.string :category, null: false
      t.decimal :balance, default: 0, null: false
      t.integer :flags, default: 0, null: false
      t.boolean :is_suspended, default: false, null: false

      t.timestamps
    end

    add_index :accounts, [:name], unique: true
  end
end
```
- `rails g migration createTransactions` and then update the migration
```
class CreateTransactions < ActiveRecord::Migration[5.1]
  def change
    create_table :transactions do |t|
      t.decimal :amount, null: false
      t.string :category, null: false
      t.belongs_to :account, index: true, null: false

      t.timestamps
    end

    add_foreign_key :transactions, :accounts
  end
end
```
- create a model for each of these tables in the app/models folder
account.rb:
```
class Account < ApplicationRecord
  has_many :transactions
end
```
transaction.rb:
```
class Transaction < ApplicationRecord
  belongs_to :account
end
```

### Validations

- You can add validations to models in rails
- in app/models/account.rb:
```
class Account < ApplicationRecord
  has_many :transactions
  validates :name, uniqueness: true
  validates :name, :category, presence: true
end
```

### Business logic belongs in models
- create a deposit method in your account models
```
def deposit(amount)
  self.errors.add(:amount, 'not a number') and return if !amount.is_a? Numeric
  self.errors.add(:amount, 'must be a positive value') and return if amount <= 0

  ActiveRecord::Base.transaction do
    self.update!(balance: self.balance + amount)

    Transaction.create!(amount: amount, category: 'Deposit', account_id: self.id)
  end
end
```
- now add the withdraw method
```
def withdraw(amount)
  return if invalid_amount?(amount)

  if self.balance >= amount
    ActiveRecord::Base.transaction do
      self.update!(balance: self.balance - amount)
      Transaction.create!(amount: amount, category: 'Withdraw', account_id: self.id)
    end
  else
    ActiveRecord::Base.transaction do
      self.update!(balance: self.balance - 50)
      Transaction.create!(amount: 50, category: 'Overdraft', account_id: self.id)
    end
  end
end
```
- after adding some helper functions and some extra validations your model should look something like:
```
class Account < ApplicationRecord
    has_many :transactions
    validates :name, uniqueness: true
    validates :name, :category, presence: true
    after_save :check_suspension

    def deposits
        self.transactions.where(category: 'Deposit')
    end

    def withdraws
        self.transactions.where(category: 'Withdraw')
    end

    def overdrafts
        self.transactions.where(category: 'Overdraft')
    end

    def deposit(amount)
        return if amount_is_not_valid(amount)

        ActiveRecord::Base.transaction do
            self.update!(balance: self.balance + amount)
            Transaction.create!(amount: amount, category: 'Deposit', account_id: self.id)
        end
    end

    def withdraw(amount)
        return if amount_is_not_valid(amount)
        return if account_is_suspended

        if self.balance >= amount
            ActiveRecord::Base.transaction do
                self.update!(balance: self.balance - amount)
                Transaction.create!(amount: amount, category: 'Withdraw', account_id: self.id)
            end
        else
            ActiveRecord::Base.transaction do
                fee = 50
                self.update!(balance: self.balance - fee, flags: self.flags + 1)
                Transaction.create!(amount: fee, category: 'Overdraft', account_id: self.id)
            end
        end
    end

    def clear_suspension
        return if insufficient_funds

        ActiveRecord::Base.transaction do
            fee = 100
            self.update!(balance: self.balance - fee, is_suspended: false)
            Transaction.create!(amount: fee, category: 'Unfreeze', account_id: self.id)
        end
    end

    private

    def check_suspension
        if self.flags > 3
            self.update!(is_suspended: true, flags: 0)
        end
    end

    def insufficient_funds
        self.errors.add(:balance, 'does not have sufficient funds') if self.balance < 100
        self.errors.any?
    end

    def account_is_suspended
        self.errors.add(:account, 'is suspended due to overdrafts') if self.is_suspended
        self.errors.any?
    end

    def amount_is_not_valid(amount)
        if !amount.is_a? Numeric
            self.errors.add(:amount, 'not a number')
            return
        end

        self.errors.add(:amount, 'less than zero') if amount <= 0
        self.errors.any?
    end
end
```
- add a controller `touch app/controllers/accounts_controller.rb` and give it an index method
```
class AccountsController < ApplicationController
  def index
    @accounts = Account.order(id: :desc)
  end
end
```
- add `resources :accounts, only: [:index]` to your config/routes.rb file
- add `root to: 'accounts#index'` to your config/routes.rb file
- create a view in app/views/accounts/index.html.erb
```
<h1>Accounts</h1>

<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Type</th>
            <th>Balance</th>
            <th>Flags</th>
            <th>Suspsended?</th>
        </tr>
    </thead>

    <tbody>
        <% @accounts.each do |a| %>
            <tr>
                <td><%= a.id %></td>
                <td><%= a.name %></td>
                <td><%= a.category %></td>
                <td><%= number_to_currency a.balance %></td>
                <td><%= a.flags %></td>
                <td><%= a.is_suspended ? 'Y' : 'N' %></td>
            </tr>
        <% end %>
    </tbody>
</table>
```
### Boostrap

- add boostrap dependencies to your gemfile:
```
source 'https://rails-assets.org' do
   gem 'rails-assets-tether', '>= 1.3.3'
end
gem 'jquery-rails'
gem 'bootstrap', '~> 4.0.0.alpha6'
```
- change your app/assets/stylesheets/application.css filename to application.scss
- delete the contents of app/assets/stylesheets/application.scss and add the line `@import "bootstrap"`
- change the contents of your app/assets/javascripts/application.js file to:
```
//= require jquery
//= require tether
//= require bootstrap
//= require rails-ujs
//= require turbolinks
//= require_tree .
```
- update your app/views/layouts/application.html.erb with boostrap divs:
```
<!DOCTYPE html>
<html>
  <head>
    <title>Atm</title>
    <%= csrf_meta_tags %>

    <%= stylesheet_link_tag    'application', media: 'all', 'data-turbolinks-track': 'reload' %>
    <%= javascript_include_tag 'application', 'data-turbolinks-track': 'reload' %>
  </head>

  <body>
    <div class="container">
      <div class="row">
        <div class="col-12">
          <%= yield %>
        </div>
      </div>
    </div>
  </body>
</html>
```
- add `class="table table-striped"` to your table html element in index.html
- add a navbar to your application in a partial at `app/views/application/_nav.html.erb` found at https://v4-alpha.getbootstrap.com/components/navbar/
- add the line `<%= render 'nav' >` under the body tag in your app/views/layouts/application.html.erb
- update your navbar to:
```
<nav class="navbar navbar-toggleable-md navbar-light bg-faded">
  <button class="navbar-toggler navbar-toggler-right" type="button" data-toggle="collapse" data-target="#navbarSupportedContent" aria-controls="navbarSupportedContent" aria-expanded="false" aria-label="Toggle navigation">
    <span class="navbar-toggler-icon"></span>
  </button>
  <a class="navbar-brand" href="/">ATM</a>

  <div class="collapse navbar-collapse" id="navbarSupportedContent">
    <ul class="navbar-nav mr-auto">
      <li class="nav-item">
        <a class="nav-link" href="/">Home</a>
      </li>
    </ul>
  </div>
</nav>
```

### member routes
- update your routes.rb file with a member route:
```
Rails.application.routes.draw do
  root to: 'accounts#index'
  resources :accounts, only: [:index] do
    member do
      get '/atm', to: 'accounts#atm'
    end
  end
end
```
- run `rails routes` to verify that it worked
- change the name in your table to a link: `<td><%= link_to a.name, atm_account_path(a) %></td>`
- update your AccountsController with the atm method:
```
def atm
  @account = Account.find(params[:id])
end
```
- create your view in app/views/atm.html.erb:
```
<h1 class="display-4">ATM for <%= @account.name %></h1>

<div class="row">
  <div class="col">
    <% if @account.errors.any? %>
      <h2><%= @account.errors.count %> error.</h2>
      <ul>
        <% @account.errors.full_messages.each do |msg| %>
          <li><%= msg %></li>
        <% end %>
      </ul>
    <% end %>
  </div>
</div>

<div class="row">
    <div class="col">
        <%= form_with(url: deposit_account_path(@account), local: true, method: 'post') do |form| %>
            <div class="form-group">
                <%= form.label :amount %>
                <%= form.text_field :amount, id: :amount, class: 'form-control' %>
            </div>
            <div class="form-group">
                <%= form.submit 'Deposit', class: "btn btn-success" %>
            </div>
        <% end %>
    </div>
    <div class="col"></div>
    <div class="col"></div>
</div>
```
- update with the deposit method:
```
def deposit
  @account = Account.find(params[:id])
  @account.deposit(params[:amount].to_f)

  if @account.errors.any?
    render 'atm'
  else
    redirect_to root_url
  end
end
```
- REPO found at https://github.com/advanced-rails/07-atm
