## Queuing

- in order to create a form for a file:
```
<%= form_with(url:'/upload', local:true, html: {multipart: true}) do |f| %>
  <%= f.text_field :name %>
  <%= f.file_field :file %>
  <%= f.submit %>
<% end %>
```
-  when uploaded, the file will live in the tmp directory for the duration of the controller method, and can be accessed via the params object, in this case `params[:file].path`

- you can also see all the methods you have on your file by starting up a binding.pry in the controller and then `cd params[:file]` followed by `ls`

- in order to copy the file into another directory, we can gain access to the root of the rails project with `Rails.root`

- we want to rename the file to something unique so it won't collide with another file named the same thing

- our controller action now looks like this:
```
def upload
  f = params[:file]
  unique_file_name = "#{Rails.root}/uploads/#{SecureRandom.hex(64)}"
  FileUtils.cp f.path, unique_file_name
end
```
- now that we have a file, we are going to use [Redis](https://redis.io).

- to start redis `sudo service redis-server start`

- `redis-cli` to use the redis command line and `KEYS *` should show something if redis is running

- to talk to redis with rails we want to use the sidekiq gem so add `gem 'sidekiq'` to your gemfile and `bundle install`

- add the following to your routes to view redis as a webserver (not something to do in prod):
```
require 'sidkiq/web'
mount Sidekiq::Web => '/sidekiq'
```

- within the class Application in your `/config/application.rb` add `config.active_job.queue_adapter = :sidekiq`

- `rails g job {JOB_NAME}` will create a file in `/app/jobs/`

- fill in the peform method in your job in order to execute code:
```
def perform(*args)
    p '*************'
    p args[0]
    p 'hello from my sidekiq job'
end
```

- in our controller we call the class and tell it to `perform_later`:
```
def upload
    f = params[:file]
    unique_file_name = "#{Rails.root}/uploads/#{SecureRandom.hex(64)}"
    FileUtils.cp f.path, unique_file_name
    #upload file to azure blog storage
    UploadImagesJob.perform_later('someParam')
    redirect_to root_path
  end
```

- run `sidekiq -c 3` in the console in order to start sidekiq and tell sidekiq how many jobs to run at the same time, in this case 3

- any uploaded jobs will be now be performed by the sidekiq process, if we had a job we would see the output:
```
"*************"
"someParam"
"hello from my sidekiq job"
```
- to talk to azure blobstorage we want to use `gem 'azure-storage'`, so add that to the gemfile and `bundle install`

- `Azure::Storage.setup(:storage_account_name => {YOUR_ACCOUNT_NAME}, :storage_access_key => {YOUR_ACCESS_KEY})` to setup your azure storage connection

- `blob = Azure::Storage::Blob::BlobService.new` to create a blob storage object

- `content = ::File.open({PATH_TO_FILE}, 'rb'){|f| f.read}` to read the file contents

- at the end, the job should look like this:
```
class UploadImagesJob < ApplicationJob
  queue_as :default

  def perform(*args)
    name, secure = args
    Azure::Storage.setup(:storage_account_name => 'railsclass', :storage_access_key => '2GrQJnBZIu3UDcgha9MAqmlRxkeqDfzWu1K3VqDc718ze6Wv0KmQzDlE0Y9KTqhU34j2qQLr1OVHkJsF/9udew==')
    blob = Azure::Storage::Blob::BlobService.new
    content = ::File.open(secure, 'rb'){|f| f.read}
    azurepath = "#{SecureRandom.hex(64)}/#{name}"
    blob.create_block_blob('classuploads', azurepath, content)
  end
end
```
- the controller:
```
class HomeController < ApplicationController
  def index
  end

  def upload
    f = params[:file]
    orig = f.original_filename
    secure = "#{Rails.root}/uploads/#{SecureRandom.hex(64)}"
    FileUtils.cp f.path, secure

    UploadImagesJob.perform_later(orig, secure)
    redirect_to root_path
  end
end
```
