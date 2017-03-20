# Rails patterns from the wild

## Introduction

This document outlines several patterns that helped me in the past in structuring Rails applications. 
This is not by any means the true and only way to do such thing it is just what have work for me. 
There are several other good documents on the Internet talking about same topics and I will encorage the reader to also compare. I will try to update this document whenever I found new patterns through my own experimentations and research.

## Inspiration

### Talks

- [Hexagonal Rails](https://www.youtube.com/watch?v=CGN4RFkhH2M)
- [Domain Driven Rails](https://vimeo.com/106759024)
- [Decoupling from Rails](https://www.youtube.com/watch?v=tg5RFeSfBM4)
- [Architecture the lost years](https://www.youtube.com/watch?v=WpkDN78P884)

### Articles

- [Seven Patterns to Refactor Fat ActiveRecord Models](http://blog.codeclimate.com/blog/2012/10/17/7-ways-to-decompose-fat-activerecord-models/)
- [Hexagonal Architecture](http://alistair.cockburn.us/Hexagonal+architecture)

### Books
- [Domain Driven Design by Eric Evans](http://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215)

### Others 

- [Housetrip Engineering guidelines](https://github.com/HouseTrip/guidelines)
- [Hanami Framework](http://hanamirb.org/)

This is not the entire list but I think this is a good start.

## The problem

The main goal of these patterns are not to solve the problem of slow test suites which I think can be solve with
tools like spring or parallel testing. The issues that this patterns try to focus on are related to improving the underlying architecture of a big Rails app to help with maintanibility and extensibility of such application.

Of course the document is not a guide on how to build any type of web applications. There are reasons to use the Rails way sometimes and the reader must be the decider on which approach to follow.

Using this new concepts in most application will lead us to think more in terms of the actual business domain and not about
the implementation details of how we are deliverying data to an end user. Remember that Rails is just a delivery mechanism it doesn't come with any details about the business that we are trying to provide a solution for.

## Use Case 

Of course we need an use case to define all this patterns in terms of that particular use case. It shouldn't be big though, however it should be complex enough to demostrate the benefits of these patterns. 


## Use Case Objects / Interactors / Service Objects

This objects are the entry point to our application they will represent a set of steps to do
a particular business requirement:

```ruby
module TimeOff
	class RequestCreator
		attr_reader :request

		def initialize(
			request_params,
			request_day_params,
			company,
			worker
		)
			@request_params = request_params.except(:id)
			@request_day_params = request_day_params
			@company = company
			@worker = worker
		end

		def create
			@request = company.save_time_off_request_for(request_params)
			request.set_request_days_from_params(request_day_params)
						worker.perform_async(
				:created,
				request.id,
				current_state.serialized
			)

			if !request.time_off_plan.requires_approval
				request.calculate_carryover_and_approve
			end
			::TimeOff::Presenters::Request.new(request, current_state)
		end

		private

		attr_reader :request_params, :request_day_params, :company, :worker
	end
end
```

If we are going to solve any issue with a particular use case for the app we 
will have just one place to look and solve it and not several in a lot of
controllers across the app. Not just that we are going to be removing
responsibilities from the controller which will lead to an easy to test
controller and also business logic.

## Persistence Layer

There are some guidelines for this that we will have to follow and that should
be trying not to access any complex query logic from the outside world. In
other words we need to stop using `.where` from controllers or even `Use case
objects` it is better to wrap that logic in a well defined scope inside the
related model:

```ruby
scope :limited, -> { where(unlimited: false) }
```

If we get to the point in which the query is not simple at all then it will be
a good idea to write a query object. We already have this concept in our code
base but I think the location of those objects are not the proper one. Those
objects are still persistence logic so why not just putting then inside the
`app/model` directory? The idea behind this is that we are explicitly going to
be separating what's framework details from what is our own business logic.

So how to write those query objects. The best approach is to be explicit
through the naming of them:

```ruby
module TimeOffTypes
	class LatestBuckectsByProfile
		def self.call(relation)
			new(relation)
		end

		def initialize(relation=TimeOffType.scoped)
			@relation = relation
		end

		def call
			relation.time_off_buckets
				.joins(:profile)
				.joins('INNER JOIN user_statuses ON user_statuses.id = profiles.user_status_id')
				.where('user_statuses.is_closer' => false)
				.includes(:time_off_plan, profile: [:company_divisions, :company, :company_categories, :job_title])
				.latest_by_profile
		end

		private

		attr_reader :relation
	end
end
```

So for us not to loose all the advantages of calling an scope we can use this
query objects like so:

```ruby
scope :latest_buckets_by_profile, TimeOffTypes::LatestBucketByProfile
```

Rails will magically call the right thing when needed. For more information on
this read; [Delegating to query objects using scopes](http://craftingruby.com/posts/2015/06/29/query-objects-through-scopes.html).

### Value Objects

This are supporting objects for our persistence and business layer. _A value object
is a small object that represents a simple entity whose equality is not based
on identity - Wikipedia_. We can decouple a lot of the business logic inside
models if we start using this objects. Examples:

```ruby
module Queries
	class ProfileReader
		...
		def limit(from: limit)
			if from.to_s.downcase == 'all'
				nil
			elsif from.to_i > 0
				from.to_i
			else
				DEFAULT_LIMIT
			end
		end
	 ...
	end
end
```

```ruby
class ProfileReader
	class Limit
		DEFAULT_LIMIT = 10
		def self.for(limit:)
			if from.to_s.downcase == 'all'
				new(nil)
			elsif from.to_i > 0
				new(from.to_i)
			else
				new(DEFAULT_LIMIT)
			end
		end

		def initialize(limit)
			@limit = limit
		end

		def to_i
			@limit
		end
	end
end
```

Probably you are thinking; why to introduce a new class when a method will
suffice? And probably you are right this is a very simplistic way of using
a value object but the advantage is that we are going to be able to test this
logic which have 6 different branches in isolation without actually loading
Rails, apart from that now we have a new bucket in the system in which we can
put related logic. To me that's a win.

## View Models

There are a lot of logic on most of our views the way that we are handling the
testing of that logic is through controller specs or helper specs. Even if
that's good is not good enough. There are several issues of that approach:

* We are still have to do a lot of smaller changes in a lot of places when
	changing business logic
* We need to load Rails to test the interaction between the logic and the
	helper method.
* We need to put conditional logic inside the views.

A better approach is to introduce the concept of view logic. Instead of
returning an active record entity is better to return all the transformation
that we need inside a view model from the use case object:

```ruby
module TimeOff
	class RequestBuilder
		attr_reader :request, :current_state, :policies

		include ::Policies::PolicyMixin
		alias_method :policy_subject, :request

		def initialize(params, current_state)
			@current_state = current_state
			@company = current_state.company
			@params = Params.new(params)
		end

		def build!(build_request_days: true)
			@policies = [::Policies::TimeOff::Enabled]
			edit_if_time_off_request_exists
			build_if_time_off_request_doesnt_exist

			request.valid!
			validate!
			request.build_request_days if build_request_days
			::TimeOff::Presenters::Request.new(request, current_state)
		end

...
```

```ruby
module TimeOff
	module Presenters
		class Request
			alias :read_attribute_for_serialization :send

			extend Forwardable

			delegate [
				:id,
				:time_off_type_id,
				:employee_notes,
				:manager_notes,
				:approved,
				:status,
				:pending?,
				:profile,
				:requester,
				:profile_bucket,
				:time_off_plan,
				:user,
				:related_bucket_and_usage,
				:related_profiles,
				:calculate_carryover_and_approve,
			] => :request

			delegate [
				:is_hourly,
				:hourly_multiplier,
				:requires_approval,
			] => :time_off_plan

			attr_reader :request, :current_state, :lambdas

			def initialize(request, current_state, lambdas = nil)
				@request = request
				@current_state = current_state
				@lambdas = lambdas || Hash.new(->(*args){}) # unknown keys default to no_op lambda
			end
		end
	end
end
```

Now we can test all the view logic in isolation.

## Policies / Permissions

One of the pain points inside our current code base to my knowledge is
understanding the permissions layer that we have. Right now there is a lot of
logic across different places in the application. My suggestion is to start
using the notion of policies to expose hidden concepts with names that are easy
to understand:

```ruby
module Policies
	module TimeOff
		class CanRequest < Policy
			def valid?(*args)
				user.has_profile? &&
				ability.can?(:user_read, user, :time_off_request)
			end

			def valid!(*args)
				unless valid?(args)
					raise Denied.new('You are not authorized to request time off')
				end
			end
		end
	end
end
```

Again you can test all this logic in isolation.

## How to start building on top of this concepts?

This I really don't know, apart from pair programming and code reviews we don't
have a proper way to start adding this concepts in a day to day basics. My
suggestion is that every time you touch the code just try to improve it
a little bit on the basics of whatever concept we decide on using. This is not
a definitive list of things that I think are mandatory for a successful Rails
application this are just things that helped me in the past to share knowledge
across members of a team and also to deliver and solve problems in an easy and
quicker manner.
