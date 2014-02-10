# Roll your own Full-text Search with Rails and MySQL

At Groupize, we have a LOT of hotels (aka properties) in our database. We also have quite a few users, who are associated with those properties, and booking requests, that belong to those properties. We needed a way for users to be able to search through the properties, booking requests, and other users that they are associated with. 

## WHERE LIKE

The first thing we did was

```
class Property < ActiveRecord::Base
  def self.name_like(query)
    where('name LIKE ?' "%#{query}%")
  end
end
```

This works, but it is slow. MySql does a table scan on the indicated column; no indexing is available to help.

// EXPLAIN SELECT for LIKE query above

Once our property count was in the tens of thousands, this became too slow to be usable. We considered sledgehammer solutions like [thinking-sphinx](https://github.com/pat/thinking-sphinx) and [sunspot](https://github.com/sunspot/sunspot); however both of these felt like overkill. We have a lot of records but fairly simple search requirements: full text across multiple columns in a particular model, but not across models. 

## MATCH AGAINST

MySql provides a full-text search operation: `MATCH AGAINST`. It simply requires a full-text index on the columns of interest, and, of course, some hand-written SQL. `MATCH AGAINST` is MySql specific and isn't supported by the ActiveRecord adapter, so using it required us to roll our own. 

The first thing to do was to create the full-text index on `Property`:

`ALTER TABLE properties ADD FULLTEXT KEY (name);`

Simple enough. Now to execute the actual query. 


### Querying MATCH AGAINST

`MATCH AGAINST` has two modes: `BOOLEAN` and `NATURAL LANGUAGE`. `NATURAL LANGUAGE` is the default, and does more or less what is expected. It treats the search query as a single phrase and searches for it within the index. `BOOLEAN` essentially tokenizes the phrase by breaking it at spaces, and searches for the presence or absence of each token in records in the index. `BOOLEAN` also allows modifiers (such as `+` and `-` that stipulate a token be present or absent). Both types can also return a `SCORE`, by which the result set can be ordered, giving a set with the most relevant results first.

To construct the query, simply insert the `MATCH AGAINST` into the usual `SELECT` as below, passing the name of the full-text index and the query. For instance, if we wanted to search for 'Hotel California':

```
SELECT properties.*, MATCH(p.name) AGAINST('Hotel California' IN BOOLEAN MODE) AS score
  FROM properties
    ORDER BY score DESC;
```

Modifiers are interpolated into the search string. Requiring that 'California' be present requires prepending it with a `+`, like so:


```
SELECT pproperties.*, MATCH(p.name) AGAINST('Hotel +California' IN BOOLEAN MODE) AS score
  FROM properties
    ORDER BY score DESC;
```

Wildcarding a word requires appending it with a `*`:


```
SELECT properties.*, MATCH(p.name) AGAINST('Hotel Califor*' IN BOOLEAN MODE) AS score
  FROM properties
    ORDER BY score DESC;
```

`MATCH AGAINST` is much faster than `LIKE`

// EXPLAIN SELECT for MATCH AGAINST

## ActiveRecord

So this is cool, but how do I run this from rails? 

`ActiveRecord::Base.connection.select_all(sql_statement)`

of course. Simply build the search statement as a string, and pass it to `select_all`; ActiveRecord will return an array of hashes containing the requested attributes. `SELECT properties.*` will return an array of hashes whose keys are each of the attributes on `Property`.

Because the search terms will be provided by the user, we want to make sure that our SQL is properly escaped before we send it back to the database. `ActiveRecord::Base` has another method `sanitize_sql_array` that does exactly that. It takes an array that contains `[the_sql_statement, terms_to_be_interpolated_in_order]`. Unfortunately, it's a private method, but that never stopped anyone before:

```
ActiveRecord::Base.send(:sanitize_sql_array[sql_statement, search_term])
```

## Implementing the search

Now we have all the building blocks, but we still don't have a search that users can hit from the website and receive useful results from.

The first thing we'll do is build the query string. We'll get the query back from the user as a single string (from `params`) with whatever they entered in the search field. Everybody knows that users can't be trusted, so we do a little up front work to make sure the database doesn't barf. We remove all non-word characters from the string:

```
def remove_symbols
  term.gsub(/[^0-9a-z ]/i, ' ').squish
end
```

We chose `BOOLEAN` search, because we couldn't guarantee the the property name would match the phrase that the user entered. Searching for 'Boston Awesome Hotel' would not match 'Hotel Awesome - Boston' in a `NATURAL LANGUAGE` search because of the order within the phrase. Since we're using a `BOOLEAN` search, we go ahead and wildcard all the words, in case the user forgot some letters.

```
remove_symbols.split(' ').map do |word|
  "#{word}*"
  end.join(' ')
```

We also do a quick `nil` check, just in case.

```
def build_search_terms(term)
  term ||= ''
  remove_symbols(term).split(' ').map do |word|
    "#{word}*"
  end.join(' ')
end
```

Then we can pass `build_search_terms(user_supplied_term)` directly into `ActiveRecord::Base.send(:sanitize_sql_array, [query, build_search_term])`

### class Search

If we combine all this logic into a class, we might get something that looks like this:

```
class PropertySearch
  def initialize(term)
    @search_terms = build_search_terms(term)
  end

  def results
    ActiveRecord::Base.connection.select_all(sanitized_sql_statement)
  end

  private
  attr_reader :search_terms

  def sanitized_sql_statement
    ActiveRecord::Base.send(:sanitize_sql_array, [build_sql_statement, search_terms, scope.id, search_terms])
  end

  def build_sql_statement
    <<-SQL
      SELECT properties.*, MATCH(p.name) AGAINST('Hotel California' IN BOOLEAN MODE) AS score
        FROM properties
          ORDER BY score DESC;
    SQL
  end
  
  def build_search_terms(term)
    term ||= ''
    remove_symbols(term).split(' ').map do |word|
      "#{word}*"
    end.join(' ')
  end

  def remove_symbols(term)
    term.gsub(/[^0-9a-z ]/i, ' ').squish
  end
end
```

#### Almost there

So that almost works, but there are two major problems with this implementation. 

+ The result set is not an `ActiveRecord::Relation`
+ The results are not an `ActiveRecord::Base` model

The first problem prevents chaining additional queries, including pagination. The second problem is just generally annoying.

Luckily, we can solve both problems at the same time. All we need to do is grab all the ids and do a simple `.where(id: ids_in_result_set)` query.

If we modify the `results` method, we can do this easily:

```
def results
  ids = ActiveRecord::Base.connection.select_all(sanitized_sql_statement).map do |record|
    record['id']
  end
  Property.where(id: ids)
end
```

### It Works! Almost

Now we have a class that will run a full text search on `Property#name` in our database, and returns an `ActiveRecord::Relation` of all of the results. 

We pushed it up to staging and started playing with it. There was one small problem...

The order that we gained by using the `AS score ORDER score BY DESC` is lost when we call `Property.where(id: ids)` - the result from this is sorted by the order of or records in the database. It turns out that there is one more MySql specific method that we can use to work around this. The method is called `FIELD`, which takes a column name and an ordered list of values. The records are returned in the order of the provided list.

We can modify `results` one last time.

```
def results
  ids = ActiveRecord::Base.connection.select_all(sanitized_sql_statement).map do |record|
    record['id']
  end
  Property.where(id: ids).order("FIELD(id, #{ids.join(',')})")
end
```

The last two issues with the results are that `FIELD` is unhappy when `ids` is empty, and we decided that if the query is null, we should return all results. Again, some minor modifications to `results` can handle this.

```
def results
  if search_terms.empty?
    Property.all
  else
    ids = ActiveRecord::Base.connection.select_all(sanitized_sql_statement).map do |record|
      record['id']
    end
    if ids.empty?
      Property.none
    else
      Property.where(id: ids).order("FIELD(id, #{ids.join(',')})")
    end
  end
end
```

## Now it works!

A working class for full-text search for a MySql database. It gets called directly from the `PropertySearchesController` and returns the results to the user. 

```
class PropertySearchesController < ApplicationController
  respond_to :json

  def show
    respond_with PropertySearcher.new(params[:query]).results,
      each_serializer: SimplePropertySerializer
  end
end
```

We extended the searcher for our specific use by allowing it work for multiple models - `BookingRequest` and `User`, as shown above. The extension was not to allow a search to touch all three models, but to allow the user to search any one of the models. The only real difference here is the SQL statement itself and the model references in results; the rest of the class remains unchanged.

To make the searcher extensible across multiple models, we simply extracted the SQL statement and the model being searched into subclasses.

```
class PropertySearcher < Search
  def model
    Property
  end

  def build_sql_statement
    <<-SQL
       SELECT properties.*, MATCH(p.name) AGAINST(? IN BOOLEAN MODE) AS score
         FROM properties
           ORDER BY score DESC;
    SQL
  end
end
```

Then we simply replace the `Property` call in `Search` with a call to `model` in `PropertySearcher#results`.

```
def results
  if search_terms.empty?
    model.all
  else
    ids = ActiveRecord::Base.connection.select_all(sanitized_sql_statement).map do |row|
      row['id']
    end
    if ids.empty?
      model.none
    else
      model.where(id: ids).order("FIELD(id, #{ids.join(',')})")
    end
  end
end
```


