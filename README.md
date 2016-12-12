# SimpleFeed — Scalable, easy to use activity feed implementation.

[![Gem Version](https://badge.fury.io/rb/simple-feed.svg)](https://badge.fury.io/rb/simple-feed)
[![MIT licensed](https://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/kigster/simple-feed/master/LICENSE.txt)
[![Build Status](https://travis-ci.org/kigster/simple-feed.svg?branch=master)](https://travis-ci.org/kigster/simple-feed)
[![Code Climate](https://codeclimate.com/repos/58339a5b3d9faa74ac006b36/badges/8b899f6df4fc1ed93759/gpa.svg)](https://codeclimate.com/repos/58339a5b3d9faa74ac006b36/feed)
[![Test Coverage](https://codeclimate.com/repos/58339a5b3d9faa74ac006b36/badges/8b899f6df4fc1ed93759/coverage.svg)](https://codeclimate.com/repos/58339a5b3d9faa74ac006b36/coverage)
[![Issue Count](https://codeclimate.com/repos/58339a5b3d9faa74ac006b36/badges/8b899f6df4fc1ed93759/issue_count.svg)](https://codeclimate.com/repos/58339a5b3d9faa74ac006b36/feed)

This is a fast, pure-ruby implementation of an activity feed concept commonly used in social networking applications. The implementation is optimized for **read-time performance** and high concurrency (lots of users), and can be extended with custom backend providers. Two providers come bundled: the production-ready Redis provider, and a naive pure Hash-based provider.

__Important Notes and Acknowledgements:__

 * SimpleFeed *does not depend on Ruby on Rails* and is a __pure-ruby__ implementation. 
 * __SimpleFeed requires ruby 2.3 or later.__
 * SimpleFeed is currently live in production.
 * We'd like to thank __[Simbi, Inc — Symbiotic Economy](http://simbi.com)__ for their sponsorship of the development of this open source library.

## What is an activity feed?

> Activity feed is a visual representation of a time-ordered, reverse chronological list of events which can be:
>
> * personalized for a given user or a group, or global
> * filtered by a certain characteristic, such as, eg.
>   * the source of the events — i.e. people you follow
>   * type of event (i.e. posts, likes, and updates)
>   * the target of the event i.e. my own activity as opposed to from those I follow.
> * aggregated across several actors for a similar event type, eg. "John, Mary, etc.. followed George"

Here is an example of a text-based simple feed that is very common today on social networking sites.

[![Example](https://raw.githubusercontent.com/kigster/simple-feed/master/man/sf-example.png)](https://raw.githubusercontent.com/kigster/simple-feed/master/man/sf-example.png)

The _stories_ in the feed depend entirely on the application using this
library, therefore to integrate with SimpleFeed requires implementing
several _glue points_ in your code.

## Challenges

Activity feeds tend to be challenging due to the large number of event types that it typically includes, and the requirement for it to scale to massive numbers of concurrent users.  Therefore common implementations tend to focus on either:

 * optimizing the read time performance by pre-computing the feed for each user ahead of time
 * OR optimizing the various ranking algorithms by computing the feed at read time, with complex forms of caching addressing the performance requirements.
 
The first type of feed is much simpler to implement on a large scale (up to a point), and it scales well if the data is stored in a light-weight in-memory storage such as Redis. This is exactly the approach this library takes.

## Overview

The feed library aims to address the following goals:

* To define a minimalistic API for a typical event-based simple feed,
  without tying it to any concrete provider implementation
* To make it easy to implement and plug in a new type of provider,
  eg.using Couchbase or MongoDB
* To provide a scalable default provider implementation using Redis, which can support millions of users via sharding
* To support multiple simple feeds within the same application, but used for different purposes, eg. simple feed of my followers, versus simple feed of my own actions.

## Usage

First you need to configure the Feed with a valid provider
implementation and a name.

### Configuration

Below we configure a feed called `:newsfeed`, which presumably
will be populated with the events coming from the followers.

```ruby
require 'simplefeed'
require 'simplefeed/providers/redis'

# Let's define a Redis-based feed, and wrap Redis in a in a ConnectionPool. 
SimpleFeed.define(:newsfeed) do |f|
  f.provider   = SimpleFeed.provider(:redis, 
                                      redis: -> { ::Redis.new },
                                      pool_size: 10)
  f.per_page   = 50
  f.batch_size = 10 # default batch size
  f.namespace  = 'nf'
end
```

After the feed is defined, the gem creates a similarly named method
under the `SimpleFeed` namespace to access the feed. For example, given
a name such as `:newsfeed` the following are all valid ways of
accessing the feed:

 * `SimpleFeed.newsfeed`
 * `SimpleFeed.get(:newsfeed)`

You can also get a full list of currently defined feeds with `SimpleFeed.feed_names` method.

### Reading from and writing to the feed

For the impatient here is a quick way to get started with the
`SimpleFeed`.

```ruby
activity = SimpleFeed.get(:followers).activity(@current_user.id)

# Store directly the value and the optional time stamp
activity.store(value: 'hello')
# => true

# or equivalent:
@event = SimpleFeed::Event.new(value: 'hello', at: Time.now)
activity.store(event: @event)
# => false # false indicates that the same event is already in the feed.
```

As we've added events for this user, we can request them back, sorted by
the time and paginated. If you are using a distributed provider, such as
`Redis`, the events can be retrieved by any ruby process in your
application, not just the one that published the event (which is the
case for the "toy" `Hash::Provider`.

```ruby
activity.paginate(page: 1)
# => [ <SimpleFeed::Event#0x2134afa value='hello' at='2016-11-20 23:32:56 -0800'> ]
```

### The Two Forms of the API

The feed API is offered in a single-user and a batch (multi-user) forms.

The only and primary difference is in what the methods return. In the
single user case, the return of, say, `#total_count` is an `Integer`
value representing the total count for this user.

In the multi-user case, the return is a `SimpleFeed::Response` instance,
that can be thought of as a `Hash`, that has the user IDs as the keys,
and return results for each user as a value.

Please see further below the details about the [Batch API](#bach-api).

<a name="single-user-api"/>

#####  Single-User API 

This API should be used typically for _read_ operations, and is accessed
via the `SimpleFeed::Feed#for` instance method. Optimized for simplicity
of data retrieval of a single-user, this method strives for simplicity
and ease of use.

Below is a user session that demonstrates simple return values from the
Feed operations for a given user:

```ruby
require 'simplefeed'

# Define the feed using an in-memory Hash provider, which uses
# SortedSet to keep user's events sorted.
SimpleFeed.define(:notifications) do |f|
  f.provider = SimpleFeed.provider(:hash)
  f.per_page = 50
  f.per_page = 2
end

# Let's get the Activity instance that wraps this user_id
activity = SimpleFeed.get(:notifications).activity(user_id)
# => [... complex object removed for brevity ]

# let's clear out this feed to ensure it's empty
activity.wipe
# => true

# Let's verify that the counts for this feed are at zero
activity.total_count
#=> 0

activity.unread_count
#=> 0

# Store some events
activity.store(value: 'hello')
activity.store(value: 'goodbye')

# Now we can paginate the events, which by default resets "last_read" timestamp the user
activity.paginate(page: 1)
# [
#     [0] #<SimpleFeed::Event#70138821650220 {"value":"goodbye","at":1480475294.0579991}>,
#     [1] #<SimpleFeed::Event#70138821649420 {"value":"hello","at":1480475294.057138}>
# ]

# Now the unread_count should return 0 since the user just "viewed" the feed.
activity.unread_count
#=> 0
```

You can fetch all items in the feed using `#fetch`, and you can
`#paginate` without resetting the `last_read` timestamp by passing the
`peek: true` as a parameter.

<a name="batch-api"/>

#####  Batch (Multi-User) API 

This API should be used when dealing with an array of users (or, in the
future, a Proc or an ActiveRecord relation). 

> There are several reasons why this API should be preferred for
> operations that perform a similar action across a range of users:
> _various provider implementations can be heavily optimized for
> concurrency, and performance_.
> 
> The Redis Provider, for example, uses a notion of `pipelining` to send
> updates for different users asynchronously and concurrently.

Multi-user operations return a `SimpleFeed::Response` object, which can
be used as a hash (keyed on user_id) to fetch the result of a given
user.

```ruby
# Using the Feed API with, eg #find_in_batches
@event_producer.followers.find_in_batches do |group|
 
  # Convert a group to the array of IDs and get ready to store
  activity = SimpleFeed.get(:followers).activity(group.map(&:id))
  activity.store(value: "#{@event_producer.name} liked an article")
  
  # => [Response] { user_id1 => [Boolean], user_id2 => [Boolean]... } 
  # true if the value was stored, false if it wasn't.
end
```

##### DSL 

The library offers a convenient DSL for adding feed functionality into
your current scope.

To use the module, just include `SimpleFeed::DSL` where needed, which
exports just one primary method `with_activity'. You call this method
and pass an activity object created for a set of users (or a single
user), like so:

```ruby
require 'simplefeed/dsl'
include SimpleFeed::DSL

feed = SimpleFeed.newsfeed
activity = feed.activity(current_user.id)
data_to_store = %w(France Germany England)

def report(value)
  puts value
end

with_activity(activity, countries: data_to_store) do
  # we can use countries as a variable because it was passed above in **opts
  countries.each do |country|
    # we can call #store without a receiver because the block is passed to 
    # instance_eval
    store(value: country) { |result| report(result ? 'success' : 'failure') }
    # we can call #report inside the proc because it is evaluated in the 
    # outside context of the #with_activity
  end  
  printf "Activity counts are: %d unread of %d total\n", unread_count, total_count
end
```

<a name="api"/>

## Complete API

### Single User

For a single user, via the instance of 
`SimpleFeed::Activity::UserActivity` class:

```ruby
require 'simplefeed'

@ua = SimpleFeed.get(:news).activity(current_user.id)

@ua.store(event:)
@ua.store(value:, at:)
# => [Boolean] true if the value was stored, false if it wasn't.

@ua.delete(event:)
@ua.delete(value:, at:)
# => [Boolean] true if the value was removed, false if it didn't exist

@ua.delete_if do |user_id, event|
  # if the block returns true, the event is deleted
end

@ua.wipe
# => [Boolean] true

# Options:
# with options[:peak] = true it does not reset last_read
# with options[:with_total] = true it returns a hash with a total:
# @return: 
@ua.paginate(page:, per_page:, **options)
# @return: [Array]<Event> (without options[:with_total])
# @return: { events: [Array]<Event, total_count: 3242 }

@ua.fetch
# => [Array]<Event> – returns all events up to Feed.max_size

@ua.reset_last_read
# => [Time] last_read

@ua.total_count
# => [Integer] total_count

@ua.unread_count
# => [Integer] unread_count

@ua.last_read
# => [Time] last_read
```

#### Batch User API

Each API call at this level expects an array of user IDs, therefore the
return value is an object, `SimpleFeed::Response`, containing individual
responses for each user, accessible via `response[user_id]` method.

```ruby
@multi = SimpleFeed.get(:feed_name).activity(User.active.map(&:id))

@multi.store(value:, at:)
@multi.store(event:)
# => [Response] { user_id => [Boolean], ... } true if the value was stored, false if it wasn't.

@multi.delete(value:, at:)
@multi.delete(event:)
# => [Response] { user_id => [Boolean], ... } true if the value was removed, false if it didn't exist

@multi.delete_if do |user_id, event|
  # if the block returns true, the event is deleted
end

@multi.wipe
# => [Response] { user_id => [Boolean], ... } true if user activity was found and deleted, false otherwise

@multi.paginate(page:, per_page:, peek: false)
# => [Response] { user_id => [Array]<Event>, ... }
# With (peak: true) does not reset last_read, otherwise it does.

@multi.fetch
# => [Response] { user_id => [Array]<Event>, ... }

@multi.reset_last_read
# => [Response] { user_id => [Time] last_read, ... }

@multi.total_count
# => [Response] { user_id => [Integer] total_count, ... }

@multi.unread_count
# => [Response] { user_id => [Integer] unread_count, ... }

@multi.last_read
# => [Response] { user_id => [Time] last_read, ... }

```

## Providers

A provider is an underlying implementation that persists the events for each user, together with some meta-data for each feed.

It is the intention of this gem that:

 * it should be easy to swap providers
 * it should be easy to add new providers

Each provider must implement exactly the public API of a provider shown
above (the `Feed` version, that receives `user_ids:` as arguments).

Two providers are available with this gem:

 * `SimpleFeed::Providers::Redis::Provider` is the production-ready provider that uses the [sorted set Redis data type](https://redislabs.com/ebook/redis-in-action/part-2-core-concepts-2/chapter-3-commands-in-redis/3-5-sorted-sets) and their operations operations to store the events, scored by their time typically (but not necessarily). This provider is highly optimized for massive writes and can be sharded by using a _Twemproxy_ backend, and many small Redis shards.

 * `SimpleFeed::Providers::HashProvider` is a pure Hash-like implementation of a provider that can be useful in unit tests of a host application. This provider could be used to write and read events within a single ruby process, can be serialized to and from a YAML file, and is therefore intended primarily for Feed emulations in automated tests.
  

### Redis Provider

If you set environment variable `REDIS_DEBUG` to `true` and run the example (see below) you will see every operation redis performs. This could be useful in debugging an issue or submitting a bug report.
  
## Running the Examples

Source code for the gem contains the `examples` folder with an example file that can be used to test out the providers, and see what they do under the hood.

To run it, checkout the source of the library, and then:

```bash
git clone https://github.com/kigster/simple-feed.git
cd simple-feed
bundle
be rspec  # make sure tests are passing
ruby examples/redis_provider_example.rb 
```

The above command will help you download, setup all dependencies, and run the examples for a single user: 

[![Example](https://raw.githubusercontent.com/kigster/simple-feed/master/man/running-example.png)](https://raw.githubusercontent.com/kigster/simple-feed/master/man/running-example.png)

If you set `REDIS_DEBUG` variable prior to running the example, you will be able to see every single Redis command executed as the example works its way through. Below is a sample output:

[![Example with Debugging](https://raw.githubusercontent.com/kigster/simple-feed/master/man/running-example-redis-debug.png)](https://raw.githubusercontent.com/kigster/simple-feed/master/man/running-example-redis-debug.png)

### Generating Ruby API Documentation

```bash
rake doc
```

This should use Yard to generate the documentation, and open your browser once it's finished.

### Installation

Add this line to your application's Gemfile:

```ruby
gem 'simple-feed'
```

And then execute:

```
$ bundle
```

Or install it yourself as:

```
$ gem install simple-feed
```

### Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake spec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and tags, and push the `.gem` file to [rubygems.org](https://rubygems.org).

### Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/kigster/simple-feed

### License

The gem is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).

### Acknowledgements

 * This project is conceived and sponsored by [Simbi, Inc.](https://simbi.com).
 * Author's personal experience at [Wanelo, Inc.](https://wanelo.com) has served as an inspiration.


