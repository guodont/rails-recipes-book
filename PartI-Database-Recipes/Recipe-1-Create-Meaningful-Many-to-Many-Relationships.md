##Create Meaningful Many-to-Many Relationships

###Problem

>Sometimes, a relationship between two models is just a relationship. For example, a person has and belongs to many pets, and you can leave it at that. This kind of relationship is straightforward. The association is all there is to track.

>But relationships usually have their own data and their own meaning within a domain. For example, a magazine has (and belongs to) many readers by way of their subscriptions. Subscriptions are interesting entities in their own right that a magazine-related application would probably want to track. A subscription might have a price or an end date. It might even have its own business rules. Thinking about the connections between entities as you model them can create a richer, more fluent domain model. How can you create meaningful many-to-many relationships between your models?

###Solution
To model rich many-to-many relationships in Rails, use join models to leverage Active Record’s ```has_many :through() ``` macro.

When modeling many-to-many relationships in Rails, many newcomers assume they should use the ```has_and_belongs_to_many() ( habtm ) ```macro with its associated join table. For years, application developers have been creating strangely named join tables in order to simply connect two tables. But habtm is best suited to relationships that have no attributes or meaning of their own. And, given some thought, almost every relationship in a Rails model deserves its own name to represent its function in the domain being modeled.

For the majority of many-to-many relationships in Rails, we use join models. Don’t panic: this isn’t a whole new type of model you have to learn. You’ll still be using and extending ```ActiveRecord::Base``` . In fact, join models are more of a technique or design pattern than they are a technology. The idea with join models is that if your many-to-many relationship needs to have some richness in the association, instead of putting a simple, dumb join table in the middle of the relationship, you can put a full table with an associated Active Record model.

Let’s look at an example. We’ll model a magazine and its readership. Magazines (their owners hope) have many readers, and readers can potentially have many magazines. We might first choose to use habtm to model this relationship. Here’s a sample schema to implement this approach:

>rr2/many_to_many/beginning_schema.rb

```
create_table :magazines do |t|
	t.string :title
	t.datetime :created_at
	t.datetime :updated_at
end

create_table :readers do |t|
	t.string :name
	t.datetime :created_at
	t.datetime :updated_at
end

create_table :magazines_readers do |t|
	:id => false
	t.integer :magazine_id
	t.integer :reader_id
end
```
As you see here, the table joining the two sides of the relationship is named after the tables it joins, with the two names appearing in alphabetical order and separated by an underscore. You would then say that the Magazine model has_and_belongs_to_many :readers, and vice versa. This relationship does the trick, enabling you to write code such as this:
```
magazine = Magazine.create(:title => "The Ruby Language Journal") 
matz = Reader.find_by_name("Matz")
magazine.readers << matz matz.magazines.size # => 1
```

Now imagine you need to track not only current readers but everyone who has ever been a regular reader of your magazine. The natural way to do this would be to think in terms of subscriptions. People who have subscriptions are the readers of your magazine. Subscriptions have their own attributes, such as a length and a date of last renewal.

It is possible with Rails to add these attributes to a habtm relationship and to store them in the join table (magazines_readers in this case) along with the foreign keys for the associated Magazine and Reader entities.

However, this technique relegates a real, concrete, first-class concept in our domain to what amounts to an afterthought. We’d be taking what should be its own class and making it hang together as a set of attributes hanging from an association. It feels like an afterthought because it is.

This is where join models come in. Using a join model, we can maintain the convenient, directly accessible association between magazines and readers while representing the relationship itself as
a first-class object: a Subscription in this case.

Let’s put together a new version of our schema, but this time supporting Subscription as a join model. Assuming we already have a migration that set up the previous version, here’s the new migration:

>rr2/many_to_many/db/migrate/20101127162741_convert_to_join_model.rb

```
def self.up
drop_table :magazines_readers create_table :subscriptions do |t|
	t.column :reader_id, :integer
	t.column :magazine_id, :integer 
	t.column :last_renewal_on, :date 
	t.column :length_in_issues, :integer
end end
```
Our new schema uses the existing magazines and readers tables but replaces the magazines_readers join table with a new table called subscriptions. Now we’ll also need to generate a Subscription model and modify all three models to set up their associations. Here are all three models:
rr2/many_to_many/app/models/subscription.rb
class Subscription < ActiveRecord::Base belongs_to :reader
belongs_to :magazine
end

>rr2/many_to_many/app/models/reader.rb

```
class Reader < ActiveRecord::Base has_many :subscriptions
	has_many :magazines, :through => :subscriptions
end
```
>rr2/many_to_many/app/models/magazine.rb

```
class Magazine < ActiveRecord::Base has_many :subscriptions
	has_many :readers, :through => :subscriptions
end
``` 
Subscription has a many-to-one relationship with both Magazine and Reader, mak- ing the implicit relationship between Magazine and Reader a many-to-many relationship.
We can now specify that a Magazine object has_many() readers through their associated subscriptions. This is both a conceptual association and a technical one. Let’s load the console to see how it works:
```
$ rails c
>> magazine = Magazine.create(:title => "Ruby Illustrated")
=> #<Magazine id: 1, title: "Ruby Illustrated", ...>
>> reader = Reader.create(:name => "Anthony Braxton")
=> #<Reader id: 1, name: "Anthony Braxton", ... >
>> subscription = Subscription.create(:last_renewal_on => Date.today,
:length_in_issues => 6)
=> #<Subscription id: 1, reader_id: nil, magazine_id: nil,
last_renewal_on: "2010-11-27",
length_in_issues: 6>
>> magazine.subscriptions << subscription
=> [#<Subscription id: 1, reader_id: nil, magazine_id: 1,
last_renewal_on: "2010-11-27",
length_in_issues: 6>]
>> reader.subscriptions << subscription
=> [#<Subscription id: 1, reader_id: 1,
magazine_id: 1,
last_renewal_on: "2010-11-27",
length_in_issues: 6>]
>> subscription.save
=> true
```
This doesn’t contain anything new yet. But now that we have this association set up, look what we can do:
```
>> magazine.reload
>> reader.reload
>> magazine.readers
=> [#<Reader id: 1, name: "Anthony Braxton", ...>]
>> reader.magazines
=> [#<Magazine id: 1, title: "Ruby Illustrated", ...>]
```
Though we never explicitly associated the reader to the magazine, the associ- ation is implicit through the :through parameter of the has_many() declarations.
Behind the scenes, Active Record generates a SQL select that joins the tables for us. For example, calling reader.magazines generates the following:
```
SELECT "magazines".* FROM "magazines"
INNER JOIN "subscriptions" ON "magazines".id = "subscriptions".magazine_id
WHERE (("subscriptions".reader_id = 1))
```
With a join model relationship, you still have access to all the same has_many options you would normally use.#3 For example, if we wanted an easy accessor for all of a magazine’s semiannual subscribers, we could add the following to the Magazine model:
>ManyToManyWithAttributesOnTheRelationship/app/models/magazine.rb

```
class Magazine < ActiveRecord::Base has_many :subscriptions
	has_many :readers, :through => :subscriptions has_many :semiannual_subscribers,
										:through => :subscriptions,
										:source => :reader,
										:conditions => ['length_in_issues = 6']
end
```

We could now access a magazine’s semiannual subscribers as follows:
```
$ rails c
>>  Magazine.first.semiannual_subscribers
=> [#<Reader id: 1, name: "Anthony Braxton", ... >]
```
Sometimes, the name of a relationship isn’t obvious to you. For example, aren’t users just in groups? Over years of working with join models, I’ve learned that the step of trying to name the relationships helps flesh out my domain model in a positive way. Indeed, users are in groups, but that rela- tionship is a membership. Are there other missing domain models you can think of?

\#3.One exception to this is the :class_name option. When creating a join model, you should instead use :source, which should be set to the name of the association to use, instead of the class name.

