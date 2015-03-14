# Spyke

<p align="center">
  <img src="http://upload.wikimedia.org/wikipedia/en/thumb/2/21/Spyke.jpg/392px-Spyke.jpg" width="20%" />
  <br/>
  Interact with remote <strong>REST services</strong> in an <strong>ActiveRecord-like</strong> manner.
  <br /><br />
  <a href="https://rubygems.org/gems/spyke"><img src="https://badge.fury.io/rb/spyke.svg?style=flat" alt="Gem Version"></a>
  <a href='https://gemnasium.com/balvig/spyke'><img src="https://gemnasium.com/balvig/spyke.svg" /></a>
  <a href="https://codeclimate.com/github/balvig/spyke"><img src="https://codeclimate.com/github/balvig/spyke/badges/gpa.svg" /></a>
  <a href='https://coveralls.io/r/balvig/spyke?branch=master'><img src='https://img.shields.io/coveralls/balvig/spyke.svg?style=flat' /></a>
  <a href="https://travis-ci.org/balvig/spyke"><img src="https://travis-ci.org/balvig/spyke.svg?branch=master" /></a>
</p>

---

Spyke basically ~~rips off~~ takes inspiration :innocent: from [Her](https://github.com/remiprev/her), a gem which we sadly had to abandon as it gave us some performance problems and maintenance seemed to have gone stale.

We therefore made Spyke which adds a few fixes/features needed for our projects:

- Fast handling of even large amounts of JSON
- Proper support for scopes
- Ability to define custom URIs for associations
- ActiveRecord-like log output
- Handling of API-side validations
- Googlable name! :)

## Configuration

Add this line to your application's Gemfile:

```ruby
gem 'spyke'
```

Spyke uses Faraday to handle requests and expects it to parse the response body into a hash in the following format:

```ruby
{ data: { id: 1, name: 'Bob' }, metadata: {}, errors: {} }
```

So, for example for an API that returns JSON like this:

```json
{ "result": { "id": 1, "name": "Bob" }, "extra": {}, "errors": {} }
```

...the simplest possible configuration that could work is something like this:

```ruby
# config/initializers/spyke.rb

class JSONParser < Faraday::Response::Middleware
  def parse(body)
    json = MultiJson.load(body, symbolize_keys: true)
    {
      data: json[:result],
      metadata: json[:extra],
      errors: json[:errors]
    }
  end
end

Spyke::Base.connection = Faraday.new(url: 'http://api.com') do |c|
  c.request   :json
  c.use       JSONParser
  c.adapter   Faraday.default_adapter
end
```

## Usage

Adding a class and inheriting from `Spyke::Base` will allow you to interact with the remote service:

```ruby
class User < Spyke::Base
  has_many :posts
  scope :active, -> { where(active: true) }
end

User.all
# => GET http://api.com/users

User.active
# => GET http://api.com/users?active=true

User.where(age: 3).active
# => GET http://api.com/users?active=true&age=3

user = User.find(3)
# => GET http://api.com/users/3

user.posts
# => find embedded in returned JSON or GET http://api.com/users/3/posts

user.update_attributes(name: 'Alice')
# => PUT http://api.com/users/3 - { user: { name: 'Alice' } }

user.destroy
# => DELETE http://api.com/users/3

User.create(name: 'Bob')
# => POST http://api.com/users - { user: { name: 'Bob' } }
```

### Custom URIs

You can specify custom URIs on both the class and association level.
Set uri to `nil` for associations you only want to use embedded JSON
and never call out to the API.

```ruby
class User < Spyke::Base
  uri 'people/(:id)' # id optional, both /people and /people/4 are valid

  has_many :posts, uri: 'posts/for_user/:user_id' # user_id is required
  has_one :image, uri: nil # only use embedded JSON
end

class Post < Spyke::Base
end

user = User.find(3) # => GET http://api.com/people/3
user.image # Will only use embedded JSON and never call out to api
user.posts # => GET http://api.com/posts/for_user/3
Post.find(4) # => GET http://api.com/posts/4
```

### Custom requests

Custom request methods and the `with` scope methods allow you to
perform requests for non-REST actions:

The `.with` scope:

```ruby
Post.with('posts/recent') # => GET http://api.com/posts/recent
Post.with(:recent) # => GET http://api.com/posts/recent
Post.with(:recent).where(status: 'draft') # => GET http://api.com/posts/recent?status=draft
Post.with(:recent).post # => POST http://api.com/posts/recent
```

Custom requests from instance:

```ruby
Post.find(3).put(:publish) # => PUT http://api.com/posts/3/publish
```

Arbitrary requests (returns plain Result object):

```ruby
Post.request(:post, 'posts/3/log', time: '12:00')
# => POST http://api.com/posts/3/log - { time: '12:00' }
```

### API-side validations

Spyke expects errors to be formatted in the same way as the
[ActiveModel::Errors details hash](https://cowbell-labs.com/2015-01-22-active-model-errors-details.html), ie:

```ruby
{ title: [{ error: 'blank'}, { error: 'too_short', count: 10 }]}
```

If the API you're using returns errors in a different format you can
remap it in Faraday to match the above. Doing this will allow you to
show errors returned from the server in forms and f.ex using
`@post.errors.full_messages` just like ActiveRecord.

### Attributes-wrapping

Spyke, like Rails, by default wraps sent attributes in a root element,
but this can be disabled or customized:

```ruby
class Article < Spyke::Base
  # Default
  include_root_in_json  true # { article: { title: ...} }

  # Custom
  include_root_in_json :post # { post: { title: ...} }

  # Disabled
  include_root_in_json false # { title: ... }
end
```

### Using multiple APIs

If you need to use different APIs, instead of configuring `Spyke::Base`
you can configure each class individually:

```ruby
class Post < Spyke::Base
  self.connection = Faraday.new(url: 'http://sashimi.com') do |faraday|
    # middleware
  end
end
```

### Log output

When used with Rails, Spyke will automatically output helpful
ActiveRecord-like messages to the main log:

```bash
Started GET "/posts" for 127.0.0.1 at 2014-12-01 14:31:20 +0000
Processing by PostsController#index as HTML
  Parameters: {}
  Spyke (40.3ms)  GET http://api.com/posts [200]
Completed 200 OK in 75ms (Views: 64.6ms | Spyke: 40.3ms | ActiveRecord: 0ms)
```

### Other examples

For more examples of how Spyke can be used, check out [fixtures.rb](https://github.com/balvig/spyke/blob/master/test/support/fixtures.rb) and the
[test suite](https://github.com/balvig/spyke/tree/master/test).


## Contributing

If possible please take a look at the [tests marked "wishlisted"](https://github.com/balvig/spyke/search?l=ruby&q=wishlisted&utf8=%E2%9C%93)!
These are features/fixes I'd like to implement but haven't gotten around to doing yet :)
