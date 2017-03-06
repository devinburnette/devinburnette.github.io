---
layout: post
title: Be Prepared!
---

If you're using Ruby on Rails, then you're probably familiar with Active Record. Active Record uses the Object-Relational Mapping (ORM) pattern to form the "M" in "MVC". Having an ORM is essential because it allows a developer to just write code and it will handle complicated database queries automatically behind the scenes. While all this sounds great, there are some things I bet you didn't even know it was doing. Things that were meant as good intentions, but have catastrophic consequences. Okay maybe I'm exaggerating a bit, but bare with me.

![](http://3.bp.blogspot.com/-h2H0nSyKAWg/VbprTMnXXtI/AAAAAAAAFXQ/INfLZ1zC8dw/s1600/BePrepared.gif)

Let's talk Prepared Statements. In SQL, it's common practice to utilize prepared statements where appropriate for both security and performance reasons.

This is a SQL query without using prepared statements:

`INSERT INTO foo VALUES(1, 'Pride Rock', 't', 'Simba');`

This is a SQL query using prepared statements:

`PREPARE fooplan (int, text, bool, text) AS
    INSERT INTO foo VALUES($1, $2, $3, $4);
EXECUTE fooplan(1, 'Pride Rock', 't', 'Simba');`

When prepared statements are used, the structure of the query is cached. So now subsequent queries with a similar structure just need to perform execution interpolating the bound variables needed to complete the statement. While providing a minimal boost in performance, this functionality also makes an application less vulnerable to SQL injection attacks.

The problem is, Active Record automatically turns your queries into prepared statements by default. This may not seem like much of a problem with smaller applications that don't really have too many database transactions happening, but for a large application this can be undesirable. There is a fundamental flaw in this automated logic generating prepared statements. It seems there are only certain types of queries where all columns can be bound as variables in a prepared statement. For Example:

I run this inside my Rails Console:

`User.find_by(email: 'foo@bar.com')`

Then I see this in my Rails Log:

`User Load (8.9ms) SELECT  "users".* FROM "users" WHERE "users"."email" = $1 LIMIT 1  [["email", "foo@bar.com"]]`

So we can already see from the logs that ActiveRecord is going to attempt to bind the variable `email` to `$1`.

Now if I go check my prepared statements cache in my database, I see this:

`ActiveRecord::Base.connection.execute('select * from pg_prepared_statements').values
   (0.3ms)  select * from pg_prepared_statements
 => [["a1", "SELECT  \"users\".* FROM \"users\" WHERE \"users\".\"email\" = $1 LIMIT 1", "2017-03-06 04:36:44.478737+00", "{text}", "f"]]`

 So this will keep an eye out for another query within the same session/connection that looks similar to this structure and bind the given variable to $1 if it matches the text data type.

`User.find_by(email: 'bar@foo.com')`

`User Load (4.5ms) SELECT  "users".* FROM "users" WHERE "users"."email" = $1 LIMIT 1  [["email", "bar@foo.com"]]`

So we can see that subsequent queries matching the same structure and data types take half the time. This is how prepared statements are supposed to work.

But let's say we run this query instead:

`User.where(sign_in_count: 1).where('last_seen_at > ?', Time.now - 7.days)`

`User Load (0.3ms)  SELECT "users".* FROM "users" WHERE "users"."sign_in_count" = $1 AND (last_seen_at > '2017-02-27 05:27:30.218160')  [["sign_in_count", 1]]`

Now Let's check our prepared statements cache:

`ActiveRecord::Base.connection.execute('select * from pg_prepared_statements').values
   (0.5ms)  select * from pg_prepared_statements
 => [["a1", "SELECT  \"users\".* FROM \"users\" WHERE \"users\".\"email\" = $1 LIMIT 1", "2017-03-06 04:36:44.478737+00", "{text}", "f"],
     ["a2", "SELECT \"users\".* FROM \"users\" WHERE \"users\".\"sign_in_count\" = $1 AND (last_seen_at > '2017-02-27 05:27:30.218160')", "2017-03-06 05:27:30.220899+00", "{integer}", "f"]]`

So we've generated another prepared statement for a different type of query, cool, or is it?

Let's try running that same query again and see if our prepared statement gets used from the cache:

`User.where(sign_in_count: 2).where('last_seen_at > ?', Time.now - 7.days)`

`User Load (0.5ms)  SELECT "users".* FROM "users" WHERE "users"."sign_in_count" = $1 AND (last_seen_at > '2017-02-27 06:19:53.351531')  [["sign_in_count", 2]]`

`ActiveRecord::Base.connection.execute('select * from pg_prepared_statements').values
   (0.4ms)  select * from pg_prepared_statements
 => [["a1", "SELECT  \"users\".* FROM \"users\" WHERE \"users\".\"email\" = $1 LIMIT 1", "2017-03-06 04:36:44.478737+00", "{text}", "f"],
     ["a2", "SELECT \"users\".* FROM \"users\" WHERE \"users\".\"sign_in_count\" = $1 AND (last_seen_at > '2017-02-27 05:27:30.218160')", "2017-03-06 05:27:30.220899+00", "{integer}", "f"],
     ["a3", "SELECT \"users\".* FROM \"users\" WHERE \"users\".\"sign_in_count\" = $1 AND (last_seen_at > '2017-02-27 05:31:20.098760')", "2017-03-06 05:31:20.099189+00", "{integer}", "f"]]`

Oh no! What's happening. If you look at the last 2 prepared statements you'll see that Active Record has hardcoded the timestamp into the cache instead of making a bound variable for it like `$2`. EVEN FOR THE SAME EXACT QUERY! This isn't good. This means our prepared statements cache will get filled up with a bunch of one time use queries which defeats the purpose of a cache in the first place.

Why is this bad? Well, every time we tell our database to remember a query, it takes up Memory. And remember, prepared statements are only shared within the same connection. So if you're using a multi-server environment with Unicorn or Passenger, it's likely the problem is exacerbated by the number of workers you're using to run your application. By default Rails will generate up to 1,000 prepared statements per connection and that's not a low number by any means. If you don't take precaution, this seamlessly hidden Active Record feature will manifest itself with a nasty Memory Leak occurring in your database ulimately crashing with Out Of Memory Errors.

So what can you do? Well this is supposed to be fixed in Adequate Record, the new and improved ORM introduced with Rails 5. But until you're ready to make that move, if you're hitting memory limits on your database, there's a couple of configs you should be aware of. In your `config/database.yml` file, you can either lower the `statement_limit` (default: 1,000) for each connection or you can just disabled `prepared_statements` altogether if you like.

`production:
  adapter: postgresql
  statement_limit: 200`

  or

`production:
  adapter: postgresql
  prepared_statements: false`
