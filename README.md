# Form Application Architecture

_Our best practice architecture for developing complex online form applications._

**⚠️  TODO: This repository is a work in progress.**

## Indication

You need to build a transactional service where users answer a series of questions, and the
questions shown and their order depend on answers to previous questions.

## Motivation

Many of our past approaches to developing large, form-based applications suffered from pain
points, e.g.:

- Questions get re-ordered constantly and developers keep playing catch-up with user research
  and service design
- A lack of shared language within the (dev and wider) team to describe the application and
  content structure
- Complex logic determining which steps to show, and in which order, and no "natural" place to
  put that logic in typical MVC frameworks
- We keep reinventing the wheel with every new service
- Difficult to onboard new team members due to complex custom code

Our primary guiding intention for this architecture was to allow any developer who is new to a
project using it to be productive immediately. We avoid behind-the-scenes magic and prefer
slight verbosity and duplication where it makes sense, in line with the following guideline
from the MoJ development principles:

> Code should be **correct**, **clear**, and **concise** – in that order.

## Scope

This architecture is best suited to online services that guide users through a large number of
steps, with business logic determining which steps are shown and in which order. It includes a
simple domain language to talk about form apps, as well as a set of lightweight implementation
patterns.

### What this is not

We deliberately did not create a domain-specific language (DSL) or drop-in components because
that would tie this architecture down to a specific programming language or web framework.
This way developers who are new to a project do not have to learn (or remember, if they come
back to it later) a set of potentially leaky abstractions.

### Counterindications

A different (potentially simpler) architecture may be preferable if your service:

- Has a small number of steps
- Has a fixed, linear flow through the steps with no business logic considerations in the step
  transitions
- Primarily has non-flow user interaction (although you may want to consider extracting the step
  flow parts of your app into a microservice using these patterns)

## Technology Choice

The reference implementation of this architecture is in Ruby on Rails, however there is no reason
some or all of these patterns could not be used in other MVC web frameworks.

## Structure

![Diagram](https://github.com/ministryofjustice/form-app-architecture/raw/master/architecture.png)
<p align="center"><em>Figure 1: Architectural diagram</em></p>

### Structural elements

#### Intents

An intent is a user activity from when they first visit your service to when they've achieved
what they intended to do.

Examples:
- Appeal an HMRC decision with the Tax Tribunal
- Book a prison visit
- Apply for a fishing licence

> Many services will just have one intent, but they may have more. Intents are distinct activities
> and the user will usually complete exactly one intent as part of using your service.

#### Tasks

A task is a logical group of questions that belong together.

Examples:
- Determine the type of your appeal
- Check your eligibility for this service 
- Enter your personal details

> Tasks can be shared between intents – in the _Appeal to the Tax Tribunal_ service, there is a
> task that captures a users personal details, and if they are represented by someone else, that
> entity's details too. This task is used in both the "Appeal an HMRC decision" and the "Apply to
> close an ongoing enquiry" intents.

#### Steps

An [individual question](https://www.gov.uk/service-manual/design/form-structure#start-with-one-thing-per-page)
or a set of closely related fields.

Examples:
- What kind of tax is your appeal about?
- How much of a penalty have you been charged?
- What kind of fish do you want to fish?
- Enter your address details
- Check your answers

> A step is usually a form, but it can also be a static page. For example, the first step in a
> task could contain an overview of what the user is supposed to do in the following steps, and
> the last step in a task could be a "Check your answers" page listing all the answers given by
> the user. A step could also be an end point of your service, e.g. a page informing the user
> that based on their answers, they are not eligible to use your service.

### Components

#### Session Model

There is no prescribed way of keeping track of your session state, although it would be preferable
to not spread it out too much as the combined session state is passed into form objects and
decision trees in the `StepController#update_and_advance` method.

The simplest way to start out will be to keep track of your session state in a single ActiveRecord
model, and provide a method of retrieving an instance based on the user session in your
`ApplicationController`. For example, in a service handling passport applications, you may have
a `PassportApplication` model whose ID is stored in the session cookie, and a
`current_passport_application` method that retrieves the relevant object as and when required.

> In the _Appeal to the Tax Tribunal_ service, we decided on staying with this approach even as
> the number of fields on the object grew to around ~40 – there were no particular performance
> considerations as it was a low volume service, and a single database model was a reasonable
> place to keep track of this data.

#### Controllers

Controllers come in two forms, depending on whether the step involves user input or not (in
other words, whether or not there is a [form object](#form-objects) for the step).

A static step controller will only have a single `#show` (`GET`) action, which will show a
page such as a start or end page for a given task.

A step controller with user input will have an `#edit` (`GET`) and an `#update` (`PUT`)
action:

- `#edit` initialises a form object with current state and renders the view
- `#update` runs the `StepController#update_and_advance` method

**⚠️ TODO: Code!**

#### Value Objects

We use value objects extensively in this architecture to encapsulate primitives in a more
semantically meaningful way.

A value object is immutable, and is equal to another value object if and only if they are of the
same type and contain the same primitive value(s). For example, a `Duration(5 days)` is equal to
a `Duration(5 days)`, but not to a `Duration(5 weeks)` or a `DeliveryTimeEstimate(5 days)`:

```ruby
def ==(other)
  other.is_a?(self.class) && other.value == value
end
```

An example of a simple value object to represent a user's preferred method of contact could look
like this:

```ruby
class PreferredContactType < ValueObject
  VALUES = [
    EMAIL = new(:email),
    PHONE = new(:phone),
    POST = new(:post)
  ].freeze

  def self.values
    VALUES
  end
end
```

ActiveRecord's [`#composed_of`](http://api.rubyonrails.org/classes/ActiveRecord/Aggregations/ClassMethods.html)
is a quick and easy way to persist value objects against a model object, and in most cases your
value objects will simply wrap a single symbol, e.g. the user's choice out of a range of options.

For this, we add a simple helper method to `ApplicationRecord` to handle transparent serialisation
and deserialisation of value objects against your session model:

```ruby
def self.has_value_object(value_object, constructor: nil, class_name: nil)
  composed_of value_object,
    allow_nil:   true,
    mapping:     [[value_object.to_s, 'value']],
    constructor: constructor,
    class_name:  class_name
end
```

This allows you to simply add something like the following to your model:

```ruby
class Inquiry < ApplicationRecord
  has_value_object :preferred_contact_type
end
```

#### Form Objects

We use form objects as a layer of abstraction between a controller handling user input and the
model. They handle:

- setting up the range of choices for multiple- or single-choice forms
- validation of user input
- casting choisen option(s) into value object(s) if applicable
- persisting new state against the session model

**⚠️ TODO: Code!**

#### Decision Trees

Decision trees encapsulate the business logic of moving between steps – given a step that has just
been completed, the answer(s) given by the user, and the current state of the session model, they
determine what step to show next.

The decision tree's `#destination` method receives the session state and current step as an input
from the `StepController#update_and_advance` method, and can return any object suitable for Rails
routing (e.g. a `{controller:, action:}` hash).

**⚠️ TODO: Code!**

## Further considerations

> Stumbling blocks? Quirks?
> Notes on testing
> Notes on generators

## Reference implementation

> Add sample code to this repo

## MoJ projects using this architecture

- [Appeal to the Tax Tribunal - Data Capture](https://github.com/ministryofjustice/tax-tribunals-datacapture)

## Further reading

> Add more links
