---
layout: post
title: "Integration testing SMTP with Serverspec"
date: 2014-04-11 18:15:55 +0200
comments: true
categories: 
---

When testing for proper working services it is usually not enough to
check whether some port is open and listening and/or some service is up.
Checking if your SMTP server is not an open relay is actually as
important as having the service up and running.

Fortunately tools exist to allow this kind of testing the easy way.

<!-- more -->

[Serverspec][serverspec] is a great tool to check if your server is
properly configured. It is pretty easy to check for listening ports and
running services:

``` ruby postfix_spec.rb
require 'spec_helper'

describe package('postfix') do
  it { should be_installed }
end

describe service('postfix') do
  it { should be_enabled   }
  it { should be_running   }
end

describe port(25) do
  it { should be_listening }
end
```

The first *describe* block checks that postfix is installed, the second
verifies that is up and running and the third checks if the port 25 is
listening. Easy uh?

But if you need to check (and you do!) that your server is not an open
relay, or if it has a proper SSL/TLS configuration you need to talk some
SMTP to him.

Fortunately Serverspec is running on top of [RSpec] so we can add ruby
code and use some gems to enhance our tests.

## Testing SMTP with RSpec and Pony

Many gems exists to send email but [pony] is lean and does the job. We
can add this to our spec:

``` ruby postfix_spec.rb
describe "Smtp" do

		Pony.options = { 
				:to => 'mytest@maildomain.com',
				:from => 'mytest@maildomain.com',
				:subject => 'Test from Serverspec',
				:body => 'Test from Serverspec',
				:via => :smtp,
				:via_options => { 
						:address => 'mail.maildomain.com',
						:port => '25',
						:enable_starttls_auto => false,
						:user_name => 'myusername',
						:password => 'mypassword',
						:authentication => :plain,
						:domain => 'maildomain.com'
				}
 	 	}

		it "should allow sending a mail" do
				expect { Pony.mail({}) }.to be_true
		end

		it "should disallow sending mail on wrong password" do
				old_options = Pony.options
				Pony.options[:via_options][:password] = 'falsepass'
				expect { Pony.mail({}) }.to raise_error
				Pony.options = old_options
		end
end
```

That's it! You now can check for Pony to pass or fail any test you want.

Some ideas for further testing:

* Check for open relay
* Check for proper SSL certificate
* Check for sending quotas

Have fun!

[serverspec]: http://serverspec.org/
[RSpec]: http://rspec.info/
[pony]: https://github.com/benprew/pony 
