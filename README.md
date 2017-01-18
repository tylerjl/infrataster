# Infrataster
[![Gitter](https://badges.gitter.im/Join Chat.svg)](https://gitter.im/ryotarai/infrataster?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

[![Gem Version](https://badge.fury.io/rb/infrataster.png)](http://badge.fury.io/rb/infrataster)
[![Code Climate](https://codeclimate.com/github/ryotarai/infrataster.png)](https://codeclimate.com/github/ryotarai/infrataster)

Infrastructure Behavior Testing Framework.


## Basic Usage with Vagrant

First, create `Gemfile`:

```ruby
source 'https://rubygems.org'

gem 'infrataster'
```

Install gems:

```
$ bundle install
```

Install Vagrant: [Official Docs](http://docs.vagrantup.com/v2/installation/index.html)

Create Vagrantfile:

```ruby
# Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "hashicorp/precise64"

  config.vm.define :proxy do |c|
    c.vm.network "private_network", ip: "192.168.33.10"
    c.vm.network "private_network", ip: "172.16.33.10", virtualbox__intnet: "infrataster-example"
  end

  config.vm.define :app do |c|
    c.vm.network "private_network", ip: "172.16.33.11", virtualbox__intnet: "infrataster-example"
  end
end
```

Start VMs:

```
$ vagrant up
```

Initialize rspec directory:

```
$ rspec --init
  create   spec/spec_helper.rb
  create   .rspec
```

`require 'infrataster/rspec'` and define target servers for testing in `spec/spec_helper.rb`:

```ruby
# spec/spec_helper.rb
require 'infrataster/rspec'

Infrataster::Server.define(
  :proxy,           # name
  '192.168.0.0/16', # ip address
  vagrant: true     # for vagrant VM
)
Infrataster::Server.define(
  :app,             # name
  '172.16.0.0/16',  # ip address
  vagrant: true,    # for vagrant VM
  from: :proxy      # access to this machine via SSH port forwarding from proxy
)

# Code generated by `rspec --init` is following...
```
Or

```ruby
# spec/spec_helper.rb
require 'infrataster/rspec'

Infrataster::Server.define(:proxy) do |server|
    server.address = '192.168.0.0/16'
    server.vagrant = true
end
Infrataster::Server.define(:app) do |server|
    server.address = '172.16.0.0/16'
    server.vagrant = true
    server.from = :proxy
end

# Code generated by `rspec --init` is following...
```

Then, you can write spec files:

```ruby
# spec/example_spec.rb
require 'spec_helper'

describe server(:app) do
  describe http('http://app') do
    it "responds content including 'Hello Sinatra'" do
      expect(response.body).to include('Hello Sinatra')
    end
    it "responds as 'text/html'" do
      expect(response.headers['content-type']).to eq("text/html")
    end
  end
end
```

Run tests:

```
$ bundle exec rspec
2 examples, 2 failures
```

Currently, the tests failed because the VM doesn't respond to HTTP request.

It's time to write provisioning instruction like Chef's cookbooks or Puppet's manifests!

## Server

"Server" is a server you tests. This supports Vagrant, which is very useful to run servers for testing. Of course, you can test real servers.

You should define servers in `spec_helper.rb` like the following:

```ruby
Infrataster::Server.define(
  # Name of the server, this will be used in the spec files.
  :proxy,
  # IP address of the server
  '192.168.0.0/16',
  # If the server is provided by vagrant and this option is true,
  # SSH configuration to connect to this server is got from `vagrant ssh-config` command automatically.
  vagrant: true,
)

Infrataster::Server.define(
  # Name of the server, this will be used in the spec files.
  :app,
  # IP address of the server
  '172.16.0.0/16',
  # If the server is provided by vagrant and this option is true,
  # SSH configuration to connect to this server is got from `vagrant ssh-config` command automatically.
  vagrant: true,
  # Which gateway is used to connect to this server by SSH port forwarding?
  from: :proxy,
  # options for resources
  mysql: {user: 'app', password: 'app'},
)
```

You can specify SSH configuration manually too:

```ruby
Infrataster::Server.define(
  # ...
  ssh: {host_name: 'hostname', user: 'testuser', keys: ['/path/to/id_rsa']}
)
```

### fuzzy IP address

Infrataster has "fuzzy IP address" feature. You can pass IP address which has netmask (= CIDR) to `Infrataster::Server#define`. This needs `vagrant` option or `ssh` option which has `host_name` because this fetches all IP address via SSH and find the matching one.

```ruby
Infrataster::Server.define(
  :app,
  # find IP address matching 172.16.0.0 ~ 172.16.255.255
  '172.16.0.0/16',
)
```

Of course, you can set fully-specified IP address too.

```ruby
Infrataster::Server.define(
  :app,
  '172.16.11.22',
  # or
  '172.16.11.22/32',
)
```

### #ssh_exec

You can execute a command on the server like the following:

```ruby
describe server(:proxy) do
  let(:time) { Time.now }
  before do
    current_server.ssh_exec "echo 'Hello' > /tmp/test-#{time.to_i}"
  end
  it "executes a command on the current server" do
    result = current_server.ssh_exec("cat /tmp/test-#{time.to_i}")
    expect(result.chomp).to eq('Hello')
  end
end
```

This is useful to test cases which depends on the status of the server.

## Resources

"Resource" is what you test by Infrataster. For instance, the following code describes `http` resource.

```ruby
describe server(:app) do
  describe http('http://example.com') do
    it "responds content including 'Hello Sinatra'" do
      expect(response.body).to include('Hello Sinatra')
    end
  end
end
```

### `http` resource

`http` resource tests HTTP response when sending HTTP request.
It accepts `method`, `params` and `header` as options.

```ruby
describe server(:app) do
  describe http(
    'http://app.example.com',
    method: :post,
    params: {'foo' => 'bar'},
    headers: {'USER' => 'VALUE'}
  ) do
    it "responds with content including 'app'" do
      expect(response.body).to include('app')

      # `response` is a instance of `Faraday::Response`
      # See: https://github.com/lostisland/faraday/blob/master/lib/faraday/response.rb
    end
  end

  # Gzip support
  describe http('http://app.example.com/gzipped') do
    it "responds with content deflated by gzip" do
      expect(response.headers['content-encoding']).to eq('gzip')
    end
  end

  describe http('http://app.example.com/gzipped', inflate_gzip: true) do
    it "responds with content inflated automatically" do
      expect(response.headers['content-encoding']).to be_nil
      expect(response.body).to eq('plain text')
    end
  end

  # Redirects
  describe http('http://app.example.com/redirect', follow_redirects: true) do
    it "follows redirects" do
      expect(response.status).to eq(200)
    end
  end

  # Custom Faraday middleware
  describe http('http://app.example.com', faraday_middlewares: [
    YourMiddleware,
    [YourMiddleware, options]
  ]) do
    it "uses the middlewares" do
      expect(response.status).to eq(200)
    end
  end
end
```

### `capybara` resource

`capybara` resource tests your web application by simulating real user's interaction.

```ruby
describe server(:app) do
  describe capybara('http://app.example.com') do
    it 'shows food list' do
      visit '/'
      click_link 'Foods'
      expect(page).to have_content 'Yummy Soup'
    end
  end
end
```

### `mysql_query` resource

`mysql_query` resource is now in [infrataster-plugin-mysql](https://github.com/ryotarai/infrataster-plugin-mysql).

### `pgsql_query` resource

`pgsql_query` resource sends a query to PostgreSQL server.

`pgsql_query` is provided by [infrataster-plugin-pgsql](https://github.com/SnehaM/infrataster-plugin-pgsql) by [@SnehaM](https://github.com/SnehaM).

### `dns` resource

`dns` resource sends a query to DNS server.

`dns` is provided by [infrataster-plugin-dns](https://github.com/otahi/infrataster-plugin-dns) by [@otahi](https://github.com/otahi).

### `memcached` resource

`memcached` resource sends a query to memcached server.

`memcached` is provided by [infrataster-plugin-memecached](https://github.com/rahulkhengare/infrataster-plugin-memcached) by [@rahulkhengare](https://github.com/rahulkhengare).

### `redis` resource

`redis` resource sends a query to redis server.

`redis` is provided by [infrataster-plugin-redis](https://github.com/rahulkhengare/infrataster-plugin-redis) by [@rahulkhengare](https://github.com/rahulkhengare).

### `firewall` resource

`firewall` resource tests your firewalls.

`firewall` is provided by [infrataster-plugin-firewall](https://github.com/otahi/infrataster-plugin-firewall) by [@otahi](https://github.com/otahi).

## Example

* [example](example)
* [spec/integration](spec/integration)

## Tests

### Unit Tests

Unit tests are under `spec/unit` directory.

```
$ bundle exec rake spec:unit
```

### Integration Tests

Integration tests are under `spec/integration` directory.

```
$ bundle exec rake spec:integration:prepare
$ bundle exec rake spec:integration
```

## Presentations and Articles

* https://speakerdeck.com/ryotarai/introducing-infrataster

<a href="https://speakerdeck.com/ryotarai/introducing-infrataster">
<img src="http://i.gyazo.com/305b8b041acd3569268ece4e07e2517a.png" alt="Introducing Infrataster" width="300px">
</a>

* https://speakerdeck.com/ryotarai/infrataster-infra-behavior-testing-framework-number-oedo04
* [Infratasterでリバースプロキシのテストをする](http://techlife.cookpad.com/entry/2014/11/19/151557)
  * Google-Translated: [Testing reverse proxy with Infrataster](https://translate.google.com/translate?hl=ja&sl=ja&tl=en&u=http%3A%2F%2Ftechlife.cookpad.com%2Fentry%2F2014%2F11%2F19%2F151557)

## Changelog

[Changelog](CHANGELOG.md)

## Contributing

1. Fork it ( http://github.com/ryotarai/infrataster/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
