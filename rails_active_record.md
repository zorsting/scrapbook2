# Rails Active Record Scrapbook


## Joins, Includes and Eager Loading

Eager loading is responsible for prefetching data in one sql query 

    Product.order("name").includes(:category)

    Product.joins(:category, :reviews)
    Product.joins(:category, :reviews => :user)
    Product.joins("left outer join categories on category_id = categories.id")



**Fetching one column name from association**

```ruby
# app/models/user.rb
def client_name
  read_attribute("client_name") || client.name
end
```

```
u = User.order("clients.name").joins(:client).select('users.*, clients.name as client_name')
# => SELECT users.*, clients.name FROM `users` INNER JOIN `clients` ON `clients`.`id` = `users`.`client_id` WHERE (`users`.`deleted_at` IS NULL)
u.client_name
```

sources

* http://railscasts.com/episodes/22-eager-loading-revised
* http://guides.rubyonrails.org/active_record_querying.html

Rails: 3.2.13


## Scopes and Arel tricks

```ruby
scope :visible, where("hidden != ?", true)
scope :published, lambda { where("published_at <= ?", Time.zone.now) }
scope :recent, visible.published.order("published_at desc")
```


** Multiple or Arel scope**

```ruby
scope :with_owner_ids_or_global, lambda{ |owner_class, *ids|
  with_ids = where(owner_id: ids.flatten).where_values.reduce(:and)
  with_glob = where(owner_id: nil).where_values.reduce(:and)
  where(owner_type: owner_class.model_name).where(with_ids.or( with_glob ))
}
```


**complex scope example**

```ruby
class Document
  scope :with_latest_super_owner, lambda{ |o|
    raise "must be client or user instance" unless [User, Client].include?(o.class)
    joins(:document_versions, document_creator: :document_creator_ownerships).
    where(document_creator_ownerships: {owner_type: o.class.model_name, owner_id: o.id}).
    where(document_versions: {latest: true}).group('documents.id')
  }
end
# it can get kinda complex :)
```



**Multiple or with bracket separation**

```ruby
# app/model/candy.rb
class Candy < ActiveRecord::Base
  has_many :candy_ownerships
  has_many :clients, through: :candy_ownerships, source: :owner, source_type: 'Client'
  has_many :users,   through: :candy_ownerships, source: :owner, source_type: 'User'

  # ....
  scope :for_user_or_global, ->(user) do
    worldwide_candies  = where(type: 'WorldwideCandies').where_values.reduce(:and)
    client_candies     = where(type: 'ClientCandies', candy_ownerships: { owner_id: user.client.id, owner_type: 'Client'}).where_values.reduce(:and)
    user_candies       = where(type: 'UserCandies',   candy_ownerships: { owner_id: user.id,        owner_type: 'User'  }).where_values.reduce(:and)

    joins(:candy_ownerships).where( worldwide_candies.or( arel_table.grouping(client_candies) ).or( arel_table.grouping(user_candies) ) )
  end

  # ....
end
```

```irb
Candy.for_user_or_global(User.last)
#=> SELECT `candies`.* FROM `candies` INNER JOIN `candy_ownerships` ON `candy_ownerships`.`candy_id` = `candies`.`id` WHERE (`candies`.`deleted_at` IS NULL) AND (((`candies`.`type` = 'WorldwideCandies' OR (`candies`.`type` = 'ClientCandies' AND `candy_ownerships`.`owner_id` = 19 AND `candy_ownerships`.`owner_type` = 'Client')) OR (`candies`.`type` = 'UserCandies' AND `candy_ownerships`.`owner_id` = 121 AND `candy_ownerships`.`owner_type` = 'User')))
```

**Arel lower than**

```irb
Event.arel_table[:start_at].lt(Time.now).to_sql
=> "`events`.`start_at` < '2013-03-05 10:38:22'" 
```

**Arel not equal where statement**

```ruby
DocumentVersion.where( DocumentVersion.arel_table[:id].not_eq(11) )
```

**Arel IS NOT NULL**

```ruby
Foo.includes(:bar).where(Bar.arel_table[:id].not_eq(nil))

# non-arel example:
Foo.where('publication_id IS NOT NULL')
```

**Select Clients that have more that have existing documents**

```ruby
class Client < ActiveRecord::Base
  has_many :documents
  scope :with_existing_documents, ->{ Client.joins(:documents).where(Document.arel_table[:client_id].eq( Client.arel_table[:id]) ).uniq }
end
```

```ruby
class Document < ActiveRecord::Base
  belongs_to :client
end
```

however when you think about it `Client.joins(:documents).uniq` already do that job by it's own

```ruby
 # ...
 scope :with_existing_documents, ->{ Client.joins(:documents).uniq }
 #...
```

so 

```ruby
doc = Document.create
client_without  = Client.create
client_with_doc = Client.create( documents: [doc]
Client.with_existing_documents
# => [client_with_doc]
```


Sources:

* http://stackoverflow.com/a/16014142/473040
* https://github.com/rails/arel/tree/master/lib/arel/nodes
* https://github.com/rails/arel

Rails 3.2.13

## Updating attributes, columns, and touching stuff

* `update_attribute` skips validations, but will touch updated_at and execute callbacks.

* `update_column` skips validations, does not touch updated_at, and does not execute callbacks.

so  if you want to update column without triggering  anything do use `update_column`, good 
example is writing your own touch method

```ruby
def my_touch   
  update_column :cache_changed_at, send(:current_time_from_proper_timezone)
end
```

if you need to touch field with time

    # out of the box touch will run with validations
    touch(:cache_changed_at)  #watch out this will update `updated_at` as well

    UPDATE `documents` SET `updated_at` = '2013-08-20 11:46:28', `cache_canged_at` = '2013-08-20 11:46:28' WHERE `documents`.`id` = 10
    

with no args touch will touch `updated_at`

source : http://stackoverflow.com/a/10824249/473040, http://apidock.com/rails/ActiveRecord/Timestamp/touch



