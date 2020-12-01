---
title: When to use Concerns, Helpers and Services
description: Summarizing some issues that I noticed in my experience or while reading about these topics and the approach I take to deciding on when to use Conerns, Helpers and Services during development or refactoring.
tags: ["Ruby on Rails", "Architecture"]
author: Sri Vishnu Totakura
layout: post
---

As a Ruby on Rails developer, you will need to make decisions to move some code into concerns, helpers, modules, classes or maybe also services. As I started developing in Ruby on Rails, I wasn't always sure what to choose in such cases and I always found myself making a decision and later changing it to something else. From my experience and some reading, I gained some insights and formed opinions on when to use one of these in my applications and I tried to write them down in this article.

I tried to summarise the basic use-cases of classes, modules (helpers, concerns) and services along with some considerations to make sure you are not making decisions that will come to bite you back in the future.

## Classes

I'm sure you are very well aware of what classes are in Object Oriented Programming. I just want to quickly say a few words and create some context.

Classes are generally used to instantiate objects that have **state** and **behaviour**.

In the example below, `title` and `starts_at` represent the *state* and the method `send_notification` represents *behaviour*.

```ruby
class LearningSession
  attr_accessor :title, :start_at

  def initialize(title, starts_at)
    @title, @starts_at = title, starts_at
  end

  def send_notification
    # implementation
  end
end
```

## Modules

Modules are used in quite a few ways in Ruby. However, there are two primary use cases of Modules: *Namespacing* and *Mixin functionality (Sharing behaviour)*.

#### Namespacing

Namespacing allows using same name clashes for constants (classes, modules etc.):

```ruby
module LearningSessions
  class Attendee
  end
end

module Conferences
  class Attendee
  end
end
```
#### Mixin functionality (sharing behaviour)

Share behaviour between classes. This is also in a way, an alternative to multiple inheritance in Ruby.


```ruby
module Trackable
  def track
    # implementation
  end
end

module Attendable
  def attendees
    @attendees ||= []
  end

  def attend(attendee)
    self.attendees << attendee
  end
end

class LearningSession
  include Trackable
  include Attendable
end

class Conference
  include Trackable
  include Attendable
end
```

## Helpers

When Rails developers talk about helpers, they are mostly referring to view helpers. This shows a clear use-case for using helpers in Rails: *when you need to expose certain data through methods to views.*

However, helpers as a pattern can be applied outside this use-case. To offer generic helper methods that are needed through out your Ruby application. For example, a helper to generate a URL for  my domain models `LearningSession` or `Conference`.

```ruby
module UrlHelpers
  BASE_URL = "https://app.zage.life"

  def self.learning_session_url(id)
    BASE_URL + "/learning_sessions/#{id}"
  end

  def self.conference_url(id)
    BASE_URL + "/conferences/#{id}"
  end
end

UrlHelpers.learning_session_url(1)
# => "https://app.zage.life/learning_sessions/1"
```
*Hint: The URL and path helpers that you use often in your Rails application are basically methods on a module. Check by typing `Rails.application.routes.url_helpers` in Rails console.*

####  When do I use Helper modules?

There sure are different use cases for such helper modules. However, as a simple guideline, I tend to use them when the meet the following conditions:
1. ***Standalone methods that don't need state***
The `_url` helper methods in above example don't need the state from the respective objects.
2. ***Methods that don't necessarily have to be the behaviour on a object*** (or that don't necessarily fit into any specific object)
In the above example, `LearningResource` domain object doesn't necessarily need to have the behaviour of generating an URL. This way, the `LearningResource` can exist without any knowledge about the environment that it is being used in.

## Concerns

Concerns are same as modules. They simply offer two benefits:
1. Syntactic sugar to access base class that includes the concern and
2. Gracefully handle module dependencies.

Get more details on that [here](https://api.rubyonrails.org/classes/ActiveSupport/Concern.html).

Apart from those benefits they are just like any other module.

#### When do I use concerns?

Only when I'm writing modules that are to be used only in my `ActiveRecord` models or controllers. Even then, I don't really see why I need concerns unless some of these concerns have inter-dependencies.

## Common issues with Modules (or Concerns)

Imagine a `LearningResource` model class including many modules that add lot of behaviour to the `LearningResource` objects.

```ruby
class LearningResource
  include Attendable
  include Trackable
  include Purchasable
  include CalendarManagement
  include Bookmarkable
  include Validations

  def save
    # write to database
  end
end
```
#### Annoying guess work

If you see a method call on a `learning_resource` object, you will have to guess to find the correct module that defines that method. Say, `learning_resource.purchage_order` is fairly easy to guess but that's not always the case and we all know that.

#### Confusion when modules override the same methods

My big issue when use modules is when these modules override the same methods. From the above example, lets look at two modules overriding the `save` method:

```ruby
module Trackable
  def save
     return false if @logger.nil?
     @logger.log("saving Learning resource: #{id}")
     super
  end
end

module Validations
  def save
    return false if !valid?
    super
  end

  def valid?
    # implementation
  end
end
```

When a call to `learning_resource.save` returns `false`, it's going to be hard to understand where the issue is coming from. Obviously, in the above case, you will add proper error messages but as the complexity of these modules increase, it won't be that simple to find out what is happening. A developer debugging such methods will need to open all those modules and read each one of those.

#### Circular dependency among modules

Circular dependency among modules occur when two or more modules depend on each other and cannot be used independently.

If we are creating modules to share behaviour in multiple classes, you might want to be able to use them independently from other modules. If there are circular dependencies, you will be forced to include all those modules into a class. In such cases, I would question the modules themselves and see if they can be combined and renamed to something else.


### Considerations when using modules (or concerns)

#### Try not to use them to simply reduce the class size

It is easy for developers to think of extracting similar methods into modules to reduce the number of lines and methods in a class file. By doing this, you are not reducing the complexity of the code but are distributing it into different files. In some cases, this can be even worse.

So, what to do when you encounter a fat class? Try to locate any "hidden abstractions" in the class. There might be such an abstraction that is present in the class but you just didn't give it a name yet. If you find such an abstraction, you might be able to move some methods into a meaningful objects. This will improve the readability of your codebase. Take for example, the `LearningResource` class with the following methods.

```ruby
class LearningResource
  # ... Other methods hidden for the sake of readbility

  def mark_done(user)
    # implementation
  end

  def unmark_done(user)
    # implementation
  end

  def done?(user)
    # implementation
  end

  def marked_done_at?(user)
    # implementation
  end
end
```
The methods relating to a user marking a `LearningResource` done seem to naturally belong to this class but there clearly is an abstraction for all these methods. You should be able to see that you can abstract them away into a `Completion` class as follows:

```ruby
class Completion
  def initialize(user, resource)
    @user, @resource = user, resource
    @created_at = Time.now
  end

  def save
    # implementation
  end

  def delete
    # implementation
  end
end
```

This way, `Completion` can exist itself as a domain model without the `LearningResource` having to know a lot about the completion logic.

`Completion` is a domain specific abstraction that was hidden until we looked carefully at our code.

#### Prefer composition over inheritance

When you are including multiple modules into a Ruby class, you are adding them into the inheritance chain of that class. This means, the class is inheriting behaviour from all those modules.

Composition, on the other hand, is when you try to break your problem into different objects with distinct responsibilities and encapsulate the behaviour into those separate objects.  The above example the `Completion` class encapsulation all completion related responsibilities. Every time the completion information is needed, the `LearningResource` objects will use the `Completion` objects. This way, the dependancy between these objects is clear and explicit.

Read more about Composition vs Inheritance [here](https://thoughtbot.com/blog/reusable-oo-composition-vs-inheritance).


## Services

Services (or service objects) originated from Domain driven development. In many cases, Services are created when you can't find a domain object to which the behaviour obviously belongs to or when co-ordinating behaviour from different domain objects or external services to respond to a request. I think they are just that simple! Somehow everyone makes a big deal about them. I guess that's mainly because that's where most of their applications' business or domain logic is.

Let's talk about them with an example. Take `LearningSessionAttendanceController` which is called when a user clicks on "Join now" button on a Learning session.

```ruby
class LearningSession::AttendanceController < AppplicationController
  def create
    learning_session = LearningSession.find(param[:learning_session_id])
    if !learning_session.full?
      learning_session.add_attendee(attendee)
      LearningSessions::AuthorMailer.send_attendee_joined(learning_session.author)
      LearningSessions::UserMailer.send_calendar_invitation(current_user)
      Tracking.log_event(:learning_session_joined, current_user)
    else
      learning_session.add_attendee(attendee, is_waiting: true)
      LearningSessions::UserMailer.send_calendar_invitation_email(current_user, is_waiting: true)
      Tracking.log_event(:learning_session_waitlist_joined, current_user)
    end
    render :ok
  end
end
```

If it were me, I would be completely fine with something like this in a controller and don't bother moving this to a service because it is simple enough to understand. However, for sake of this article, let us assume that this is a complex business logic that doesn't belong to the controller.

The main characteristic that I want you to look at in the above code is that it is calling behaviour on different objects and that's all. It's basically co-ordinating the calls. This makes it a candidate for a service. I call it `AttendanceCreation`:

```ruby
# app/services/learning_sessions/attendance_creation.rb
class Services::LearningSessions::AttendanceCreation < Service
  def call(learning_session, attendee)
	if !learning_session.full?
      learning_session.add_attendee(attendee)
      LearningSessions::AuthorMailer.send_attendee_joined(learning_session.author)
      LearningSessions::UserMailer.send_calendar_invitation(attendee)
      Tracking.log_event(:learning_session_joined, attendee)
    else
      learning_session.add_attendee(attendee, is_waiting: true)
      LearningSessions::UserMailer.send_calendar_invitation_email(attendee, is_waiting: true)
      Tracking.log_event(:learning_session_waitlist_joined, attendee)
    end
  end
end

# app/controllers/learning_sessions/attendance_controller.rb
class LearningSessions::AttendanceController < AppplicationController
  def create
    learning_session = LearningSession.find(param[:learning_session_id])
    Services::LearningSessions::AttendanceCreation.call(learning_session, current_user)
	render :ok
  end
end
```

That's all. As a pattern, I think services should be just that simple: making decisions and triggering the correct behaviour on the domain objects.

In the code above, I hid the details of `Service` base class that allows calling the services and returning responses from them. Every developer has their own way of writing services and the actually implementation of services is a little out of scope for this article but if you are interested, I recommend [this](https://hackernoon.com/the-3-tenets-of-service-objects-c936b891b3c2) article.

### Considerations when using services

#### Try to keep behaviour on domain objects

In our `AttendanceCreation` service above, we called methods on domain objects so that the respective objects are responsible for their behaviour. This will ensure the proper use of Object Oriented Programming principles and offer more flexibility.

In the above example, suppose we didn't have the `LearningSession#add_attendee` method and we wrote the logic to add an attendee in our service directly. Later, if we need to let the admins add attendees without all the email notifications, we cannot do that without rewriting the logic elsewhere. Having the `LearningSession#add_attendee` method gives the flexibility to call it from anywhere when needed.

#### You might not need services always

Once your codebase has the Services layer, you might be tempted to use Service objects whenever you encounter some long blocks of code and you want to reduce the number of lines in a class. Before you take that step, ask yourself if the code is just a procedure or an actually domain logic that needs a service. Lets look at the example below (don't bother knowing what's happening in the code):

```ruby
class Admin::LearningSessionReports < ApplicationController
  def create
    sessions = LearningSession.where("starts_at > ? AND starts_at < ?", 1.week.ago, Time.now)
    users = User.where(attendance: { learning_session: sessions })
    csv_file_path = "reports/learning_sessions/weekly-#{Time.now}.csv"
    CSV.open(csv_file_path, "wb") do |csv|
      csv << [
        "signup month",
        "company",
        "sessions attended",
        "has paid subscription"
      ]
      user.each { |user|
        csv << [
          user.created_at.strftime("%B"),
          user.company,
          user.learning_sessions_count,
          user.has_paid_subscription?
        ]
      }
    end
    Admin.all.each { |admin|
      admin.send_report(:learning_sessions_weekly, csv_file_path)
    }
  end
end
```

Now, ask yourself, does it makes sense to create a service for this? Is there really some logic in that piece of code? It's just simple program performing sequence of actions: *A procedure*. Compare this to the previous example of `AttendanceCreation` service which has some logic of deciding whether to add user to a waiting list (it's not a lot of logic but a real world example might have more logic).

By moving this report creation procedure to a service, you will be polluting your services. So, if you have to move this procedure out of the controller, maybe create a simple module in your `app/lib/` directory and call that:

```ruby
# app/lib/reports/learning_session_weekly.rb
module Reports::LearningSessionWeekly
  def self.generate
    sessions = LearningSession.where("starts_at > ? AND starts_at < ?", 1.week.ago, Time.now)
    users = User.where(attendance: { learning_session: sessions })
    # ...
  end
end

# app/controllers/admin/learning_session_reports_controller.rb
class Admin::LearningSessionReports < ApplicationController
  def create
    Reports::LearningSessionWeekly.generate
  end
end
```

### When do I use services


### Guidelines I follow to use services

* Use services when you need a place to contain all your business (domain) specific logic that doesn't necessarily
fit into any object.
This is usually the case when you have some logic that needs to trigger behaviour on different objects.
* Don’t dump all logic into these services. Try to move behaviour to the domain objects as much as possible and only trigger that behaviour from the services.

## Alright then, how do I deal with my fat models and controllers?

Glad you asked! When you encounter fat models, controllers or classes, my advise is to follow these steps to refactoring:

1. Look for hidden abstractions like we did in the example with creating a `Completion` domain model. There is a good chance that you are able to create more domain models and abstract away responsibilities.
2. Once you perform step 1, look for shared behaviour that can moved to either concerns, modules (or helpers). Remember to keep in mind the problems we looked at above with these and try to avoid them.
3. See if there is any critical domain/business logic that you might want to move into services.


## References
* [https://api.rubyonrails.org/classes/ActiveSupport/Concern.html](https://api.rubyonrails.org/classes/ActiveSupport/Concern.html)
* [https://thoughtbot.com/blog/reusable-oo-composition-vs-inheritance](https://thoughtbot.com/blog/reusable-oo-composition-vs-inheritance)
* [https://gist.github.com/ryanb/4172391](https://gist.github.com/ryanb/4172391)
* [https://www.codewithjason.com/used-intelligently-rails-concerns-great/](https://www.codewithjason.com/used-intelligently-rails-concerns-great/)
