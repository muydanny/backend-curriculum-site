# Consuming an API

## Learning Goals

By the end of this class, a student will be able to: 

- Set up and configure Faraday for use with a Rails application.
- Use Faraday to connect and retrieve information from third party external APIs.
- Parse the information retrieved from a third party API.

This lesson was built on Ruby 3.2.2 and Rails 7.0.5.1

## Warm Up

- How does a developer interact with an API?
- What are the benefits of using an API?

## Summary

What we are going to be working on today is creating an app that reaches out and consumes data from an external API, and then displays and formats that data on a web page. The API we will be using is the Congress.gov API, and we will be using it to grab a list of Representatives from Congress.

We will accomplish that by starting with a user story.

```markdown
As a user
When I visit "/"
And I select "Colorado" from the dropdown
And I click on "Locate Representatives"
Then my path should be "/search" with "state=Colorado" in the parameters
And I should see a message "3 Results"
And I should see a list of the representatives for Colorado
And I should see a name, party, and state for each member
```

As you can see, it lines out all that we will do. Let’s get started.

## Setup

We start by spinning up our rails app. We are going to call it House Salad.

Because we're getting information about Congress and the House of Representatives and we're gonna toss it around. Kind of.

```bash
$ git clone https://github.com/turingschool-examples/house-salad-7
$ cd house-salad-7
```

```bash
$ rails db:create
$ rails db:migrate
```

Yes, we haven't created any migrations, but running rails db:migrate will generate the `schema.rb` so we don't get an error when we start running tests.

## Testing, All Day, Every Day

So we've got the basic setup.

Let's create our first test files.

```bash
$ mkdir spec/features/
$ touch spec/features/user_can_search_by_state_spec.rb
```

Now let's open up that file and translate our user story into a test.

*spec/features/user_can_search_by_state_spec.rb*

```ruby
require 'rails_helper'

feature "user can search for members" do

  scenario "user submits valid state name" do
    # As a user
    # When I visit "/"
    visit '/'

    select "Colorado", from: :state
    # And I select "Colorado" from the dropdown
    click_on "Locate Representatives"
    # And I click on "Locate Representatives"
    expect(current_path).to eq(search_path)
    # I should see a list the members for Colorado

    within(first(".member")) do
      expect(page).to have_css(".name")
      expect(page).to have_css(".party")
      expect(page).to have_css(".state")
    end
    # And I should see a name, role, party, and state for each member
  end
end
```

And so we run our tests. We should get an error concerning a `search_path`.

Our form is sad about where we are trying to send information. So we are going to have to add a route.

*config/routes.rb*

```ruby
get "/search", to: "search#index"
```

And of course, we need to create a controller with an index action and then create a corresponding view.

*app/controllers/search_controller.rb*

```ruby
class SearchController < ApplicationController
  def index
  end
end
```

```bash
$ mkdir app/views/search
$ touch app/views/search/index.html.erb
```

Now we get the error, `expected to find css ".member" at least 1 time"`.

## Consuming the API

At this point, we are going to have to consume the Congress API to get the data we need. Read through the [Congress.gov API documentation](https://api.congress.gov/) and try to pull out the relevant pieces of information. Yes, you actually have to read it.

### API Keys

One thing you'll notice when reading the docs is that it requires us to sign up for an api key. An api key is a way for the api's owners to authenticate users. Most apis will require that you sign up for a key. This allows the api owners to track who is using their api and how much. Most apis limit the rate at which you can use the api for free, and typically you have to pay to increase this usage. You'll see an example of this in the Congress.gov API docs: "Usage is limited to 5000 requests per hour". In this case, avoid running your code 5000 times and you should be good.

If you haven't already, [sign up for a Congress.gov key](https://api.congress.gov/sign-up/).

### Authentication

Another key piece to pull out of the [Congress.gov documentation](https://github.com/LibraryOfCongress/api.congress.gov/?tab=readme-ov-file) is in the section on "Keys". Now that we have an API key, the included link tells us how to use it:

```bash
Pass the API key into the X-Api-Key header:

X-API-Key: CONGRESS_API_KEY
```

This API also allows us to pass our key with the request as a query parameter: 

```bash
GET /amendment/

Example Request

https://api.congress.gov/v3/amendment?api_key=[INSERT_KEY]
```

### Endpoints

We also need to find the documentation for the endpoints we will need. Explore the docs and see if you can find the endpoint.

Remember, we are trying to get a list of members from a particular state. There is a collapsible section under the 'member' header which says it returns a list of congressional members. That looks promising!

By reading through the documentation for this endpoint, we can determine that we'll need to send a request like:

```bash
GET https://api.congress.gov/v3/member
```

along with our API key in a header. Keep in mind that if you are not passing your API key as a `X-API-KEY` header, you will need to pass it as a query param:

```bash
GET https://api.congress.gov/v3/member?api_key=[CONGRESS_API_KEY]
```

Using this information, see if you can hit the API endpoint using Postman. I do want to note that with the request above, we are getting a list of all members, regardless of their state and their chamber (some will be from the Senate, and others from the House of Representatives).

### Make the Request

Let's run our tests to remind us of where we left off. Oh right, we're getting `expected to find css ".member" at least 1 time`.

Now that we know what request we want to send, we need to send it to get the data we want to display.

We will be using the [Faraday Gem](https://github.com/lostisland/faraday) to make HTTP requests using Ruby.

First, we will need to add `gem "faraday"` to our Gemfile. We don't want to add to a `:development`/`:test` block since we will need to make these API calls in all environments. After you add it to your Gemfile, run `bundle install`.

Now that we have it installed, lets use Faraday to make the API call. Rather than memorizing the syntax we use in this tutorial, make sure you get used to referencing documentation.

*app/controllers/search_controller.rb*

```ruby
class SearchController < ApplicationController
  def index
    state = params[:state]

    conn = Faraday.new(url: "https://api.congress.gov") do |faraday|
      faraday.headers["X-API-Key"] = '<YOUR API KEY>'
    end

    response = conn.get("/v3/member?limit=250")

    binding.pry
  end
end
```

Make sure you replace `<YOUR API KEY>` with the API key you signed up for earlier.

Since we want to get all possible results, we'll include a query param for the maximum limit available, as specified in the 'parameters' section of this endpoint's docs.

When we assign `conn`, does this make an HTTP request? What are these lines of code doing? (review the docs if you aren't sure)

In the code above, we set up a variable to hold the connection information, we tell it the name of the server, and our API Key, which is our password to be able to access the API. And then we use the `get` method on the connection and pass it the end point we want to access. We store that in the `response` local variable, and then we parse it.

When we run the code and hit the pry, we can visually inspect `response` and `response.body` to make sure it contains data and not an error message or something else unexpected.

Once we've verified our request was successful, we can parse the data and pass it to our view. We'll need to add some conditional logic so that we're only returning members from the state we have in `params`.

NOTE: This repo was updated between the 2403 and 2405 innings. If you cloned an older version, you'll need to update an existing helper module so that the `us_states` method looks like this:

*app/helpers/application_helper.rb*

```ruby
def us_states
  [
    'Alabama',
    'Alaska',
    'Arizona',
    'Arkansas',
    'California',
    'Colorado',
    'Connecticut',
    'Delaware',
    'District of Columbia',
    'Florida',
    'Georgia',
    'Hawaii',
    'Idaho',
    'Illinois',
    'Indiana',
    'Iowa',
    'Kansas',
    'Kentucky',
    'Louisiana',
    'Maine',
    'Maryland',
    'Massachusetts',
    'Michigan',
    'Minnesota',
    'Mississippi',
    'Missouri',
    'Montana',
    'Nebraska',
    'Nevada',
    'New Hampshire',
    'New Jersey',
    'New Mexico',
    'New York',
    'North Carolina',
    'North Dakota',
    'Ohio',
    'Oklahoma',
    'Oregon',
    'Pennsylvania',
    'Puerto Rico',
    'Rhode Island',
    'South Carolina',
    'South Dakota',
    'Tennessee',
    'Texas',
    'Utah',
    'Vermont',
    'Virginia',
    'Washington',
    'West Virginia',
    'Wisconsin',
    'Wyoming'
  ]
```

*app/controllers/search_controller.rb*

```ruby
class SearchController < ApplicationController
  def index
    state = params[:state]
    conn = Faraday.new(url: "https://api.congress.gov") do |faraday|
      faraday.headers["X-API-Key"] = "c8A5PNW7BC1ccEdzMVK0s5WZcJtZsFTUEaRVD3Up"
    end

    response = conn.get("/v3/member?limit=250")
    
    json = JSON.parse(response.body, symbolize_names: true)
    @members_by_state = []
    json[:members].each do |member_data|
      if member_data[:state] == state
        @members_by_state << member_data
      end
    end
  end
end
```

And then we need to add some code to our view:

*app/views/search/index.html.erb*

```html
<h1><%= @members.count %> Results</h1>
<% @members.each do |member| %>
  <ul class="member">
    <li class="name"><%= member[:name] %></li>
    <li class="party"><%= member[:partyName] %></li>
    <li class="state"><%= member[:state] %></li>
  </ul>
<% end %>
```

And now our test is passing. As always, run your server and check your work by hand in your development environment. It may not be pretty, but the data is there.

## Environment Variables

There's one more improvement we should make to our code. If you look in the controller, we have hard coded our API key. There's a couple reasons we don't want to do this:

1. It isn't secure. If someone gets access to this code (you should always assume this is possible, even if your project is closed-source), someone could copy our API key and then would be able to masquerade as our application. They could, for example, spam the Congress API with requests and force us over the rate limit we discussed earlier. If our API key has access to paid features, they could get this access for free.
2. It isn't flexible. If we need to change the API key, we'd need to go into the code base and manually configure it. If we use this API key in multiple places, we'd need to change it in each place.

What we really want is to put our environment configuration somewhere that is specific to this project. Luckily, Rails provides a seamless way to store environment variables via Rails Application Credentials.

To set up our API key, complete the following steps:

* Verify that you are able to launch VS Code from the command line by running `code`
  * If the following steps don’t work, you’ll need to follow these [Launching From the Command Line](https://code.visualstudio.com/docs/setup/mac#:~:text=Keep%20in%20Dock.-,Launching%20from%20the%20command%20line,code) steps to configure the command
* Generate what is called a ‘master key’ by running `EDITOR="code --wait" rails credentials:edit` in the command line
  * This will create a new key in `config/master.key` and a temporary YAML file which will open in your text editor
* Add your API Key to the opened file
  * Note the indentation in the example below. The tab before `key` is important, as it results in the ability to access this value under a congress "object".
  * The `secret_key_base` value is unique to YOUR repo. Use what is automatically generated and _don't_ copy this one.

```
congress:
  key: <Your API key here>

# Used as the base secret for all MessageVerifiers in Rails, including the one protecting cookies.
secret_key_base: ugsdfeadsfg98a7sd987asjkas98asd87asdkdwfdg876fgd
```

* Save and close the file, and you should see in your terminal that the file was encrypted and saved
* Note: To use these credentials and environment variables with a team you’ll need to share the contents of the `config/master.key` file with your teammates securely, and they’ll need to create this file with that key as the contents

[Here is a walkthrough video of the steps above, to help you set up your Rails Application Credentials.](https://drive.google.com/file/d/1Cy598b1W1d7nZ-gv6ur_gPmAGOmaD3Gi/view)

Next, you’ll have to replace the hardcoded key in your controller.

*app/controllers/search_controller.rb*

```ruby
class SearchController < ApplicationController
  def index
    state = params[:state]
    conn = Faraday.new(url: "https://api.congress.gov") do |faraday|
      faraday.headers["X-API-Key"] = Rails.application.credentials.congress[:key]
    end

    response = conn.get("/v3/member?limit=250")
    
    json = JSON.parse(response.body, symbolize_names: true)
    @members_by_state = []
    json[:members].each do |member_data|
      if member_data[:state] == state
        @members_by_state << member_data
      end
    end
  end
end
```

Run the tests again to confirm everything is working.

## Checks for Understanding.

- What does Faraday do?
- What is an API Key?
- What is a connection?
- What are headers?
- What don't you like about this code?
- Is our feature test enough?
- What are we missing?
- What do environment variables do? Why do we use them instead of hardcoding?
- Do you like the index action in the search controller?
- How would you start to refactor this?

Our next step is to refactor this.

You can find a completed working repo for this lesson here: [https://github.com/turingschool-examples/house-salad-7/tree/api-consumption-complete](https://github.com/turingschool-examples/house-salad-7/tree/api-consumption-complete)
