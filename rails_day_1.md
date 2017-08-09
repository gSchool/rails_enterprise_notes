# Ruby on Rails

## SQL

- `sudo service postgresql start` to start postgres
- `psql` to enter the postgres console
- `create database {DATABASE_NAME}` to create a database
- `\l` to show all databases
- `\c {DATABASE_NAME}` to connect to a database
- `\d` to show all tables in a database
- `create table {TABLE_NAME} ({COLUMN_NAME} {COLUMN_DATA_TYPE})` to create a table
- `select * from {TABLE_NAME}` to see all rows in the table
- `\q` to quit the psql console
- `drop database {DATABASE_NAME}` to delete a database

## New Rails Project

- `rails new {APP_NAME} -d postgresql --skip-action-mailer --skip-action-cable --skip-coffee --skip-test --skip-system-test`
- add to gemfile under group development, test:
```
gem 'pry'
gem 'pry-byebug'
gem 'pry-rails'
gem 'faker'
```
- run `bundle install` to install the new dependencies
- add `template: template0` to the `database.yml` under the default settings
- `rails db:create` to create databases
- `rails g migration CreateTasks` to create a new migration
- under db/migrate there should be a new file, update it with fields
```
class CreateTasks < ActiveRecord::Migration[5.1]
  def change
    create_table :tasks do |t|
      # All columns should be lowercase with underscores to separate words
      t.string :title
      t.string :category
      t.integer :priority
      t.boolean :is_complete
      t.date :due

      t.timestamps #this automatically adds a created_at and updated_at fields to the table
    end
  end
end
```
- `rails db:migrate` to run your migrations
- create a model in app/models called task.rb
```
class Task < ApplicationRecord
end
```
- `Task.create({COLUMN}: {VALUE})` to create and save a new record
- `Task.find 1` finds task 1 by id
- `Task.find_by and Task.where` play with them and read documentation to figure out how to use
- Update the seeds.rb file in the db directory:
```
Task.delete_all

25.times do |i|
  Task.create!(
    title: Faker::Lorem.sentence, # a random sentence from the lorem set in faker
    is_complete: [true, false].sample, #sample will randomly pick an element from the array
    priority: (0..9).to_a.sample,
    category: ['Home', 'Work', 'Play'].sample,
    due: Faker::Date.forward(90) # a random date in the future up to 90 days
  )
end
```
- `rails db:seed` to create seed the database
- either use `rails console` or connect to the database to verify that it worked
- in config/routes.rb:
```
Rails.application.routes.draw do
  resources :tasks #both of these must be plural
end
```
- `touch app/controllers/tasks_controller.rb` make sure the file name is snake case and tasks is plural, then update that file:
```
class TasksController < ApplicationController
  def index
  end

  def create
  end

  def new
  end

  def edit
  end

  def show
  end

  def update
  end

  def update
  end

  def destroy
  end
end
```
- `mkdir app/views/tasks` create a view folder for tasks
- `touch app/views/tasks/index.html.erb` and update the file with some text
- run the app with `HOST=0.0.0.0 rails s` to run on all ports
- preview and go to /tasks path to see your text
- update your controller's index method with `@tasks = Task.all` this exposes the `@tasks` variable in our view
- update the view with:
```
<h1>Task Manager</h1>

<table>
    <thead>
        <tr>
            <th>complete?</th>
            <th>id</th>
            <th>title</th>
            <th>priority</th>
            <th>category</th>
            <th>due</th>
        </tr>
    </thead>

    <tbody>
        <% @tasks.each do |t| %>
            <tr>
                <td><%= t.is_complete %></td>
                <td><%= t.id %></td>
                <td><%= t.title %></td>
                <td><%= t.priority %></td>
                <td><%= t.category %></td>
                <td><%= t.due %></td>
            </tr>
        <% end %>
    </tbody>
</table>
```
- add the following to your view
```
<%= form_with(model: @task, local: true) do |form| %>
    <div class="field">
        <%= form.label :title %>
        <%= form.text_field :title, id: :task_title %>
    </div>
    <div class="field">
        <%= form.submit %>
    </div>
<% end %>
```
- add to your controller under the create method:
```
t = Task.new(params[:task].permit(:title))
t.save
redirect_to tasks_url
```
- you should now be able to add tasks with a name
### Create
- Now update your controllers index and create methods:
```
def index
    @task = Task.new
    @tasks = Task.order(id: :desc)
end

def create
    t = Task.new(params[:task].permit(:title, :due, :priority, :is_complete, :category))
    t.save
    redirect_to tasks_url
end
```
- Update your view with all the fields:
```
<%= form_with(model: @task, local: true) do |form| %>
    <div class="field">
        <%= form.label :title %>
        <%= form.text_field :title, id: :task_title %>
    </div>

    <div class="field">
        <%= form.label :is_complete %>
        <%= form.check_box :is_complete, id: :task_is_complete %>
    </div>

    <div class="field">
        <%= form.label :priority %>
        <%= form.number_field :priority, id: :task_priority %>
    </div>

    <div class="field">
        <%= form.label :due %>
        <%= form.date_select :due, id: :task_due %>
    </div>

    <div class="field">
        <%= form.label :category %>
        <%= form.select(:category, id: :task_category) do %>
            <% ['Work', 'Play', 'Home'].each do |cat| %>
                <%= content_tag(:option, cat, value: cat) %>
            <% end %>
        <% end %>
    </div>

    <div class="field">
        <%= form.submit %>
    </div>
<% end %>
```

#### For full code with delete and update go to https://github.com/advanced-rails/06-todo/
