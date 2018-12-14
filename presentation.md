# Performance problems and how to fix them

![main_image](main_image.jpg)

## Agenda
**Part 1:**
- Pagination
- N+1 and how to avoid them
- Caching (service level, request level, Redis, conditional Get)
- stupid logic problems
- Preliminary work (aggregating some data to DB, pre-filling cache)
- Too long requests to Database (e.g. Use index, Luke!)
-------
**Part 2:**
- how to avoid such problems
- when to optimize
- flamegraph

## 1. Pagination

![gateway_timeout](1_pagination/gateway_timeout.png)

## 2. Excessive(Stupid) work

### 2.1 N+1 requests

#### Problem:

```ruby
clients = Client.limit(10)
 
clients.each do |client|
  puts client.address.postcode
end
```
-> 11 requests:

```sql
SELECT * FROM clients LIMIT 10;
SELECT addresses.* FROM addresses  WHERE (addresses.client_id = 1);
SELECT addresses.* FROM addresses  WHERE (addresses.client_id = 2);
SELECT addresses.* FROM addresses  WHERE (addresses.client_id = 3);
SELECT addresses.* FROM addresses  WHERE (addresses.client_id = 4);
SELECT addresses.* FROM addresses  WHERE (addresses.client_id = 5);
SELECT addresses.* FROM addresses  WHERE (addresses.client_id = 6);
SELECT addresses.* FROM addresses  WHERE (addresses.client_id = 7);
SELECT addresses.* FROM addresses  WHERE (addresses.client_id = 8);
SELECT addresses.* FROM addresses  WHERE (addresses.client_id = 9);
SELECT addresses.* FROM addresses  WHERE (addresses.client_id = 10);
```

#### Solution:

```ruby
clients = Client.includes(:address).limit(10)
 
clients.each do |client|
  puts client.address.postcode
end
```
-> 2 requests:
```sql
SELECT * FROM clients LIMIT 10;
SELECT addresses.* FROM addresses
  WHERE (addresses.client_id IN (1,2,3,4,5,6,7,8,9,10));
```

it doesn't seem scary, but with time, you end with:

```ruby
hash = {}
BulkUpload::File.all.each {|f| hash[f.id] = f.file_content.content.length}
```
=>
![n_plus_1_big_problem](2_n_plus_one/n_plus_1_big_problem.png)
#### another example:

![includes_vs_joins_1](2_n_plus_one/includes_vs_joins_1.png)
(it's not all the page)

result:
![includes_vs_joins_2](2_n_plus_one/includes_vs_joins_2.png)
---
 after fix:
![includes_vs_joins_3](2_n_plus_one/includes_vs_joins_3.png)
![includes_vs_joins_4](2_n_plus_one/includes_vs_joins_4.png)

#### How to fix:
For ruby : https://github.com/flyerhzm/bullet

configuration:
```ruby
config.after_initialize do
  Bullet.enable = true
  Bullet.sentry = true
  Bullet.alert = true
  Bullet.bullet_logger = true
  Bullet.console = true
  Bullet.growl = true
  Bullet.xmpp = { :account  => 'bullets_account@jabber.org',
                  :password => 'bullets_password_for_jabber',
                  :receiver => 'your_account@jabber.org',
                  :show_online_status => true }
  Bullet.rails_logger = true
  Bullet.honeybadger = true
  Bullet.bugsnag = true
  Bullet.airbrake = true
  Bullet.rollbar = true
  Bullet.add_footer = true
  Bullet.stacktrace_includes = [ 'your_gem', 'your_middleware' ]
  Bullet.stacktrace_excludes = [ 'their_gem', 'their_middleware', ['my_file.rb', 'my_method'], ['my_file.rb', 16..20] ]
  Bullet.slack = { webhook_url: 'http://some.slack.url', channel: '#default', username: 'notifier' }
end
```


**!!!** I'm sure something similar is already implemented for your language

### 2.2 No Caching

#### 2.2.1 Simple caching service, memoisation
```ruby
def fibonacci(n)
   n <= 1 ? n :  fibonacci( n - 1 ) + fibonacci( n - 2 ) 
end
(0..10).map {|n| fibonacci(n)} # executes almost instantly
(0..40).map {|n| fibonacci(n)} # takes 26 seconds
```
(Yes, it's slow Ruby, but it will hang on every lang with big number)

**solution:**
```ruby
class FibonacciCaching
  attr_reader :cached_values
  def initialize
    @cached_values = { 0 => 0, 1 => 1 }
  end

  def get(n)
    return cached_values[n] if n < 2
    cached_values[n] ||= cached_values[n-1] + cached_values[n-2]
  end
end
```

**results:**
```ruby
require 'benchmark'

max = 40
Benchmark.bm do |x|
  x.report { (0..max).map {|n| fibonacci(n)} }
  x.report { fc = FibonacciCaching.new; (0..max).map {|n| fc.get(n)} }
  x.report { fc = FibonacciCaching.new; (0..50000).map {|n| fc.get(n)} }
end

#      user     system      total        real
# 26.400000   0.000000  26.400000 ( 26.405840)
#  0.000000   0.000000   0.000000 (  0.000052)
#  0.140000   0.020000   0.160000 (  0.166776) 
```

#### 2.2.2 Caching something for the whole request

for ruby: https://github.com/steveklabnik/request_store


#### 2.2.3 Redis (one love)
```ruby
class Product < ApplicationRecord
  def competing_price
    Rails.cache.fetch("#{cache_key}/competing_price", expires_in: 12.hours) do
      Competitor::API.find_price(id)
    end
  end
end

```

#### 2.2.4 Conditional GET (HTTP `ETag` and `If-None-Match` headers)
! doesn't work together with previous example

```ruby
class ProductsController < ApplicationController
 
  def show
    @product = Product.find(params[:id])
 
    # If the request is stale according to the given timestamp and etag value
    # (i.e. it needs to be processed again) then execute this block
    if stale?(last_modified: @product.updated_at.utc, etag: @product.cache_key_with_version)
      respond_to do |wants|
        # ... normal response processing
      end
    end
    # after `stale?` call, Rails returns HTTP 304 Not Modified by default
  end
end
```

### 2.3 Problems with logic

Example:
- service_1 process up to 50k transfers
- system should validate that any of the transfers has amount less than some limit, if not - render "At least one transfer exceeds your allowed limit"
- such validation endpoint is on service_2, accepts ONE value
-------------
how to implement?

## 3. Preliminary work
- save to database
- save to cache

## 4. Database
EXPLAIN ANALYZE - your best friend
- Understand difference between INDEX FULL SCAN / INDEX UNIQUE SCAN / TABLE SCAN etc.
- Understand what index types does your DB support (not only B-tree exist, for example indexes for geo  data (lat/long))
- Check indexes on all heavy queries fields used in `where`, `on`, `group by`

## 5. How to understand, what to optimize and what to not

### 5.1 Amount of data
always test with the amount of data compared to how much you'll have on prod
 - fill DB with scripts generating fake data
 - upload big files

### 5.2 Profiling and optimisation
You MUST do profiling before doing optimisation, to be sure you're not just wasting time,
Unless you're sure in advance it's 100% required in this particular place (comes with years of work experience)

profiling:
- Datadog/newRelic (good for production, costly)
- something like Flamegraph (good for local debugging, open-source)


### 5.3 Flamegraph

#### 5.3.1. problem:
this is a stacktrace of profiling some bash script:
![cpu-bash-profile-1800](flamegraph/cpu-bash-profile-1800.jpg)

#### 5.3.2. solution:
now it's much better (when you understand how to interprete it):
![cpu-bash-flamegraph](flamegraph/cpu-bash-flamegraph.png)

#### 5.3.3 How it works:
https://www.slideshare.net/brendangregg/blazing-performance-with-flame-graphs

It visualises a collection of stacktraces
- **X axis:** alphabetical sort **(!!!NOT TIME, THIS IS VERY IMPORTANT !!!)**, to maximize merging
- **Y axis:** stack depth
- **colors:** random

![flamegraph_simple_example](flamegraph/simple_example.png)