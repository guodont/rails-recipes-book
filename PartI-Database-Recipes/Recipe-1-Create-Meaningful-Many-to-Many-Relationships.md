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

