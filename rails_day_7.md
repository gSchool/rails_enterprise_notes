# Testing in Rails

- `rails new dealer -d postgresql --skip-action-mailer --skip-action-cable --skip-coffee --skip-test --skip-system-test`

- update gem file:
```
group :development, :test do
  gem 'pry'
  gem 'pry-byebug'
  gem 'pry-rails'
  gem 'faker'
  gem 'annotate'
  gem 'railroady'
  gem 'launchy'
  gem 'guard'
  gem 'guard-rspec'
  gem 'rails-controller-testing'
  gem 'database_cleaner'
  gem 'rspec-rails'
  gem 'capybara'
  gem 'selenium-webdriver'
  # Call 'byebug' anywhere in the code to stop execution and get a debugger console
  gem 'byebug', platforms: [:mri, :mingw, :x64_mingw]
end
```
- `rails generate rspec:install` to generate rspec files and directories
- `guard init` to hook up guard
- create a migration `rails generate migration CreateCars`
- update migration:
```
class CreateCars < ActiveRecord::Migration[5.1]
  def change
    create_table :cars do |t|
      t.string :make, null: false
      t.string :model, null: false
      t.integer :year, null: false
      t.string :vin, null: false
      t.string :color, default: 'black'
      t.string :category, default: 'car'
      t.integer :cylinders, default: 4
      t.float :displacement, default: 0
      t.integer :mpg, default: 0
      t.integer :hp, default: 0
      t.timestamps
    end

    add_index :cars, [:vin], unique: true
    add_index :cars, [:make, :model, :year], unique: true
  end
end
```
- `rails db:create`, `rails db:migrate`

### Model tests

- create a test file
- run `rspec` to run tests
- write a test for construction:
```
RSpec.describe Car, type: :model do
  describe '.new' do
    it 'instantiates a Car object' do
      c = Car.new
      expect(c.is_a?(Car)).to be true
      expect(c.attributes.keys.count).to eql(13)
    end
  end
end
```
- make it pass with an empty model in `app/models/car.rb`
```
class Car < ApplicationRecord
end
```
- add a happy path test, this should pass:
```
describe '#save' do
  context 'happy path' do
    it 'saves a car and upcases the vin' do
      c = Car.new(make: 'Ford', model: 'Fusion', year: 2015, vin: '0123456789abcdefg')
      c.save
      expect(c.id).to_not be_nil
      expect(c.created_at).to_not be_nil
      expect(c.updated_at).to_not be_nil
    end
  end
end
```
- add some error case tests:
```
context 'invalid data' do
  it 'missing model, year - will not save' do
    c = Car.new(make: 'Ford', vin: 'abcd')
    c.save
    expect(c.id).to be_nil
    expect(c.errors[:model].first).to eql("can't be blank")
    expect(c.errors[:year].first).to eql("can't be blank")
  end
  it 'vin number is too short - will not save' do
    c = Car.new(make: 'Ford', model: 'Fusion', year: 2015, vin: 'abcd')
    c.save
    expect(c.id).to be_nil
    expect(c.errors[:vin].first).to eql("is the wrong length (should be 17 characters)")
  end
end
```
- make them pass by adding validators to the class
```
class Car < ApplicationRecord

  validates :make, :model, :year, :vin, presence: true
  validates :vin, length: {is: 17}

end
```
- add a test for incorrect vin syntax:
```
it 'vin number has incorrect syntax - will not save' do
  c = Car.new(make: 'Ford', model: 'Fusion', year: 2015, vin: '123456789ABCDE@GH')
  c.save
  expect(c.id).to be_nil
  expect(c.errors[:vin].first).to eql("invalid representation")
end
```
- make it pass by enhancing your model:
```
class Car < ApplicationRecord
  before_validation :upper_vin, :check_vin

  validates :make, :model, :year, :vin, presence: true
  validates :vin, length: {is: 17}

  private

  def upper_vin
    self.vin = self.vin.upcase if self.vin
  end

  def check_vin
    self.errors.add(:vin, 'invalid representation') if self.vin && self.vin !~ /^[0-9A-Z]{17}$/
  end
end
```
- to add a test for uniqueness first add to your seed file:
```
Car.delete_all
Car.create(make: 'Toyota', model: 'Camry', year: 2015, vin: 'A0000000000000001')
Car.create(make: 'Honda', model: 'Civic', year: 2014, vin: 'A0000000000000002')
Car.create(make: 'Suburu', model: 'Outback', year: 2013, vin: 'A0000000000000003')
```
- add to your `spec/spec_helper.rb` file to make seeds run before each test
```
config.before(:each) do
  Rails.application.load_seed
end
```
- add a test:
```
it 'vin number already taken - will not save' do
  c = Car.new(make: 'Ford', model: 'Fusion', year: 2015, vin: 'A0000000000000001')
  c.save
  expect(c.id).to be_nil
  expect(c.errors[:vin].first).to eql("has already been taken")
end
```
- make it pass by adding `uniqueness: true` to the vin validator on your model
- add tests for all of your validations (do this one test at a time, making sure all tests pass before writing a new one):
```
require 'rails_helper'

RSpec.describe Car, type: :model do
  describe '.new' do
    it 'instantiates a Car object' do
      c = Car.new
      expect(c.is_a?(Car)).to be true
      expect(c.attributes.keys.count).to eql(13)
    end
  end

  describe '#save' do
    context 'happy path' do
      it 'saves a car' do
        c = Car.new(make: 'Ford', model: 'Fusion', year: 2015, vin: '0123456789abcdefg', category: 'car', cylinders: 4)
        c.save
        expect(c.id).to_not be_nil
        expect(c.vin).to eql('0123456789ABCDEFG')
        expect(c.category).to eql('CAR')
        expect(c.created_at).to_not be_nil
        expect(c.updated_at).to_not be_nil
      end
    end
    context 'invalid data' do
      it 'missing model, year - will not save' do
        c = Car.new(make: 'Ford', vin: 'abcd')
        c.save
        expect(c.id).to be_nil
        expect(c.errors[:model].first).to eql("can't be blank")
        expect(c.errors[:year].first).to eql("can't be blank")
      end
      it 'vin number is too short - will not save' do
        c = Car.new(make: 'Ford', model: 'Fusion', year: 2015, vin: 'abcd')
        c.save
        expect(c.id).to be_nil
        expect(c.errors[:vin].second).to eql("is the wrong length (should be 17 characters)")
      end
      it 'vin number has incorrect syntax - will not save' do
        c = Car.new(make: 'Ford', model: 'Fusion', year: 2015, vin: '*12345678@ABC#EFZ')
        c.save
        expect(c.id).to be_nil
        expect(c.errors[:vin].first).to eql("invalid representation")
      end
      it 'vin number already taken - will not save' do
        c = Car.new(make: 'Ford', model: 'Fusion', year: 2015, vin: 'A0000000000000001')
        c.save
        expect(c.id).to be_nil
        expect(c.errors[:vin].first).to eql("has already been taken")
      end
      it 'make, model, year already taken - will not save' do
        c = Car.new(make: 'Toyota', model: 'Camry', year: 2015, vin: 'B0000000000000001')
        c.save
        expect(c.id).to be_nil
        expect(c.errors[:make].first).to eql("has already been taken")
      end
      it 'year is too old - will not save' do
        c = Car.new(make: 'Toyota', model: 'Camry', year: 1800, vin: 'B0000000000000001')
        c.save
        expect(c.id).to be_nil
        expect(c.errors[:year].first).to eql("is not between 1900 and 2020")
      end
      it 'category is invalid - will not save' do
        c = Car.new(make: 'Toyota', model: 'Camry', year: 1800, vin: 'B0000000000000001', category: 'bad')
        c.save
        expect(c.id).to be_nil
        expect(c.errors[:category].first).to eql("is not included in the list")
      end
      it 'cylinders is invalid - will not save' do
        c = Car.new(make: 'Toyota', model: 'Camry', year: 1800, vin: 'B0000000000000001', cylinders: 5)
        c.save
        expect(c.id).to be_nil
        expect(c.errors[:cylinders].first).to eql("is not one of 4, 6, 8 or 12")
      end
    end
  end
end
```
- make them pass (do this one test at a time, making sure all tests pass before writing a new one):
```
class Car < ApplicationRecord
  before_validation :upper, :check_vin

  validates :make, :model, :year, :vin, presence: true
  validates :vin, length: {is: 17}
  validates :vin, uniqueness: true
  validates :make, uniqueness: {scope: [:model, :year]}
  validates :year, inclusion: {in: 1900..2020, message: 'is not between 1900 and 2020'}
  validates :category, inclusion: {in: ['CAR', 'SPORT', 'SUV', 'TRUCK']}
  validates :cylinders, inclusion: {in: [4, 6, 8, 12], message: 'is not one of 4, 6, 8 or 12'}

  private

  def upper
    self.vin = self.vin.upcase if self.vin
    self.category = self.category.upcase if self.category
  end

  def check_vin
    self.errors.add(:vin, 'invalid representation') if self.vin && self.vin !~ /^[0-9A-Z]{17}$/
  end
end
```
### Feature tests

- to run a specific test you can point rspec to it with a file path and line number, ie `rspec spec/models/car_spec.rb:33`

- create a feature test via command line `rails g rspec:feature cars` and in `/spec/features/cars`
```
require 'rails_helper'

RSpec.feature "Cars", type: :feature do
  scenario 'When the user views the /cars page' do
    car = Car.first
    visit '/cars'
    expect(page).to have_content('Cars')
    expect(page).to have_content('Toyota')
    expect(page.html).to include('<h1>Cars</h1>')
    expect(page).to have_link('New', href: '/cars/new')
    expect(page).to have_link(car.vin, href: "/cars/#{car.id}")
  end
end
```
- make it pass by following the errors and solving each one individually

- drive out other features the same way

- Link to the full application: https://github.com/advanced-rails/09-dealer

### Controller tests

- `rails g rspec:controller cars`

- create a controller test:
```
require 'rails_helper'

RSpec.describe CarsController, type: :controller do
  describe 'Get /cars/:id' do
    it 'displays a car' do
      car = Car.first
      get :show, params: {id: car.id}
      expect(assigns(:car)).to eql car
      expect(response.status).to eql 200
    end
  end
end
```
