ActsAsIcontact
==============
ActsAsIcontact connects Ruby applications with the [iContact e-mail marketing service](http://icontact.com) using the iContact API v2.0.  Building on the [RestClient](http://rest-client.heroku.com) gem, it offers two significant feature sets:

* Simple, consistent access to all resources in the iContact API; and
* Automatic synchronizing between ActiveRecord models and iContact contact lists for Rails applications.

Prerequisites
-------------
You'll need the following to use this gem properly:

1. **Ruby 1.9** Yeah, we know, change is _scary_ and a zillion of your other gems only work in 1.8. But ActsAsIcontact makes use of a few 1.9 features for efficiency, such as Enumerators.  It's _possible_ that this might work in 1.8.7 if you install the **JSON** gem and `require 'enumerator'` explicitly -- but the author hasn't tested it.  If you need it to work in 1.8, speak up.  Or better yet, make it work and submit a patch.

2. **Rails 2.1** _(If using Rails integration)_ We use ActiveRecord's 'dirty fields' feature that first appeared in 2.1 to determine whether iContact needs updating.  If you're on a version of Rails older than this, it's probably worth your while to update anyway.

3. **rest-client** This gem _should_ install when you install the **acts\_as\_icontact** gem.  But we include it here for completeness.

Setting Up
----------
Using ActsAsIcontact is easy, but going through iContact's authorization process requires jumping a couple of hoops.  Here's how to get going quickly:

1. Install the gem.

       $ sudo gem install acts_as_icontact

2. _Optional but recommended:_ Go to <http://beta.icontact.com> and sign up for an iContact Beta account. ActsAsIcontact will use your Beta credentials and API URL by default if the `RAILS_ENV` or `RACK_ENV` variables are set to anything but 'production', or if you tell it to in configuration.

3. Enable the ActsAsIcontact gem for use with your iContact account. The URL and credentials you'll use are different between the beta and production environments:  
  
     * **BETA:** Go to <http://app.beta.icontact.com/icp/core/externallogin> and enter `Ml5SnuFhnoOsuZeTOuZQnLUHTbzeUyhx` for the Application Id. Choose a password for ActsAsIcontact that's different from your account password.  
  
     * **PRODUCTION:** Go to [http://app.icontact.com/icp/core/externallogin](http://app.icontact.com/icp/core/externallogin) and enter `IYDOhgaZGUKNjih3hl1ItLln7zpAtWN2` for the Application Id. Choose a password for ActsAsIcontact that's different from your account password.
  
4. Set your _(beta, if applicable)_ account username and the password you just chose for API access. You can either set the environment variables `ICONTACT_USERNAME` and `ICONTACT_PASSWORD`, or you can explicitly do it with calls to the Config module:  

        require 'rubygems'
        require 'acts_as_icontact'
    
        ActsAsIcontact::Config.beta = true
        ActsAsIcontact::Config.username = my_beta_username
        ActsAsIcontact::Config.password = my_api_password  
  
    If you're using Rails, the recommended approach is to require the gem with `config.gem 'acts_as_icontact'` in your **config/environment.rb** file, and then set up an initializer (i.e. **config/initializers/acts\_as\_icontact.rb**) with the above code.  See more about Rails below.

5. Rinse and repeat with production credentials when you're ready to move out of the beta environment.  

API Access
----------
Whether or not you're using Rails, retrieving and modifying iContact resources is simple.  The gem autodiscovers your account and client folder IDs (you only have one of each unless you're an 'agency' account), so you can jump straight to the good parts:

     contacts = ActsAsIcontact::Contact.find(:all)  # => <#ActsAsIcontact::ResourceCollection>
     c = contacts.first    # => <#ActsAsIcontact.Contact>
     c.firstName           # => "Bob"
     c.lastName            # => "Smith"
     c.email               # => "bob@example.org"
     c.lastName = "Smith-Jones"   # Bob gets married and changes his name
     c.save                # => true
     history = c.actions   # => <#ActsAsIcontact::ResourceCollection>
     a = history.first     # => <#ActsAsIcontact::Action>
     a.actionType          # => "EditFields"
  
  
### Nesting
The interface is deliberately as "ActiveRecord-like" as possible, with methods linking resources that are either nested in iContact's URLs or logically related.  Messages have a Message#bounces method.  Lists have List#subscribers to list the Contacts subscribed to them, and Contacts have Contact#lists.  Read the documentation for each class to find out what you can do:

* ActsAsIcontact::Account
  * ActsAsIcontact::ClientFolder
* ActsAsIcontact::Contact
  * ActsAsIcontact::History _(documented as "Contact History")_
* ActsAsIcontact::Message
  * ActsAsIcontact::Bounce
  * ActsAsIcontact::Click
  * ActsAsIcontact::Open
  * ActsAsIcontact::Unsubscribe
  * ActsAsIcontact::Statistics
* ActsAsIcontact::List
  * ActsAsIcontact::Segment
      * ActsAsIcontact::Criterion  _(documented as "Segment Criteria")_
* ActsAsIcontact::Subscription
* ActsAsIcontact::Campaign
* ActsAsIcontact::CustomField
* ActsAsIcontact::Send
* ActsAsIcontact::Upload
* ActsAsIcontact::User
  * ActsAsIcontact::Permission
* ActsAsIcontact::Time

### Searching
Searches are handled in a sane way using the same query options that iContact accepts.  The following are all valid:

`Messages.all` -- *Same as `Messages.find(:all)`*  
`Messages.first` -- *Same as `Messages.find(:first)`*  
`Messages.find(:all, :limit => 20)` -- *First 20 messages*  
`Messages.find(:all, :limit => 20, :offset => 40)` -- *Messages 41-60*  
`Messages.first(:subject => "Fnord")` -- *First message with the given subject*  
`Messages.all(:orderby => createDate, :desc => true)` -- *Messages ordered by most recent first*  
`Messages.all(:messageType => "welcome", :campaignId => 11)` -- *Welcome messages from the given campaign*  

At this time, special searches are not yet supported.  Fields requiring dates must also be given a string corresponding to the ISO8601 timestamp (e.g. `2006-09-16T14:30:00-06:00`).  Proper date/time conversion will happen soon.

### Updating

Again, think ActiveRecord.  When you initialize an object, you can optionally pass it a hash of values:

    c = Contact.new(:firstName => "Bob", 
                    :lastName => "Smith-Jones", 
                    :email => "bob@example.org")
    c.address = "123 Test Street"

Each resource class has a `#save` method which returns true or false.  If false, the `#error` method contains the reply back from iContact about what went wrong.  (Which may or may not be informative, but we can't tell you more than they do.)  There's also a`#save!`method which throws an exception instead.

Nested resources can be created using the `build_` method (which returns an object but doesn't save it right away) or `create_` method (which does save it upon creation).  The full panoply of ActiveRecord association methods are not implemented yet.  (Hey, we said it was AR-_like._)

The `#delete` method on each class works as you'd expect, assuming iContact allows deletes on that resource.  Resource collections containing the resource are not updated, however, so you may need to requery.

Multiple-record updates are not implemented at this time.

Rails Integration
-----------------
The _real_ power of ActsAsIcontact is its automatic syncing with ActiveRecord.  At this time this feature is focused entirely on Contacts.  

### Activation
First add the line `config.gem 'acts_as_icontact'` to your **config/environment.rb** file.  Then create an initializer (e.g. **config/initializers/acts\_as\_icontact.rb**) and set it up with your username and password.  If applicable, you can give it both the beta _and_ production credentials:

    module ActsAsIcontact
      if Config.beta?
        Config.username = my_beta_username
        Config.password = my_beta_password
      else
        Config.username = my_production_username
        Config.password = my_production_password
      end
    end

The default behavior is to set `beta` to _false_ if `RAILS_ENV` is equal to "production" and _true_ if it's anything else.

Finally, enable one of your models to synchronize with iContact with a simple declaration:

    class Person < ActiveRecord::Base
      acts_as_icontact
    end
  
There are some options, of course; we'll get to those in a bit.

### What Happens
When you call the `acts_as_icontact` method in an ActiveRecord class declaration, the gem does several useful things:

1. Creates callbacks to post changes to iContact's API after a record is saved or deleted.
2. Defines an `icontact_sync!` method to pull the contact's data _from_ iContact and make any changes.
3. Defines other methods such as `icontact_lists` and `icontact_history` to make related data accessible.
4. If an `icontact_status` field exists, creates named scopes on the model class for each iContact status.  _(Pending)_

### Options
Option values and field mappings can be passed to the `acts_as_icontact` declaration to set default behavior for the model class.  Right now there's only one option:

`default_lists` -- _The name or ID number of a list to subscribe new contacts to automatically, or an array of said names or numbers_

### Field Mappings
You can add contact integration to any ActiveRecord model that tracks an email address.  (If your model _doesn't_ include email but you want to use iContact with it, you are very, very confused.)

Any fields that are named the same as iContact's personal information fields, or custom fields you've previously declared, will be autodiscovered.  Otherwise you can map them:

    class Customer < ActiveRecord::Base
      acts_as_icontact :default_lists => ['New Customers', 'All Users']  # Puts new contact on two lists
                       :given_name => :firstName, # Key is Rails field, value is iContact field
                       :family_name => :lastName,
                       :address1 => :street,
                       :address2 => :street2,
                       :id => :rails_id, # Custom field created in iContact
                       :preferred? => :preferred_customer # Custom field
    end

A few iContact-specific fields are exceptions, and have different autodiscovery names to avoid collisions with other attributes in your application:

`icontact_id` -- _Corresponds to `contactId` in iContact.  Highly recommended._  
`icontact_status` -- _Corresponds to `status` in iContact._  
`icontact_created` -- _Corresponds to `createDate` in iContact._
`icontact_bounces` -- _Corresponds to `bounceCount` in iContact._

You are welcome to create these fields in your model or omit them.  However, we _strongly_ recommend that you either include the `icontact_id` field to track iContact's primary key in your application, or map your own model's primary key to a custom field in iContact.  You can also do both for two-way associations.  If you don't establish a relationship with at least one ID, ActsAsIcontact will resort to using the email address for lookups -- which will be a problem if the email address changes.

### Lists
The reason to add contacts to iContact is to put them on mailing lists.  We know this.  The `default_list` option (see above) is one way to do it automatically.  The following methods are also defined on the model for your convenience:

`icontact_lists` -- _An array of List objects to which the contact is currently subscribed_
`icontact_subscribe(list)` -- _Given a list name or ID number, subscribes the contact to that list immediately_
`icontact_unsubscribe(list)` -- _Given a list name or ID number, unsubscribes the contact from that list_


### Why Just Contacts?
iContact's interface is really quite good at handling pretty much every other resource.  Campaigns, segments, etc. can usually stand alone.  It's less likely that you'll need to keep copies of them in your Rails app.  But contacts are highly entangled.  If you're using iContact to communicate with your app's users or subjects, you'll want to keep iContact up-to-date when they change.  And if someone bounces or unsubscribes in iContact, odds are good you'll want to know about it.  So this is the strongest point of coupling and the highest priority feature.  (Lists will likely come next, followed by messages.)

Copyright
---------
Copyright (c) 2009 Stephen Eley. See LICENSE for details.