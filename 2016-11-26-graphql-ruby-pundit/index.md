# GraphQL Ruby and Authorization With Pundit


Recently I began exploring GraphQL for use with Ruby on Rails. There is an
excellent gem `graphql` created by [Robert
Mosolgo](https://github.com/rmosolgo), that provides all the workings of
GraphQL https://github.com/rmosolgo/graphql-ruby. Notably is the excellent
documentation accompanying this gem https://rmosolgo.github.io/graphql-ruby.

With all the documentation you should easily be able to get up and running with
GraphQL in Ruby. However, authorizing user access to records via
[Pundit](https://github.com/varvet/pundit) may require a bit of exploration,
seeing as it's original intended purpose is to authenticate by controller
actions. I'm going to quickly outline how I achieved this with a few code
snippets. First add [Pundit](https://github.com/varvet/pundit) to you gemfile:
`gem 'pundit`.

Then in your `Application Controller`:

{{< gist nickpoorman cbd39734fbdcc66049a69d5309844045 >}}

Now you'll need a controller to serve the GraphQL queries. Let's call it
`graphql_controller.rb`. As you can see below, `GraphqlController` inherits from
`ApplicationController`, thus giving us
[Pundit](https://github.com/varvet/pundit) capabilities. To make things simple,
we have passed in a context with the `current_user` and `pundit` -which will
give us access to the Pundit `authorize` method in the GraphQL handlers. Also,
by passing the instance of the controller into the context, we are able to add
`after_action :verify_authorized` to [ensure policies and scopes are
used](https://github.com/varvet/pundit#ensuring-policies-and-scopes-are-used)
somewhere in the GraphQL handlers.

{{< gist nickpoorman 41d431a9be23838a1d9bba5f5f83dfdb >}}

Next, if you're using Relay you should put a Pundit `authorize` in your schema
where you decode the object id. You can see that we have used the `pundit`
instance passed into the context (`ctx`). Because we are simply looking up a
record here, we have decided to use the `:show` policy action.

{{< gist nickpoorman 0f90dc9ef84f476afe79a4b47485cb01 >}}

If you happen to come across this [demo
app](https://github.com/rmosolgo/graphql-ruby-demo), you’ll notice an
[abstraction for fetching
records](https://github.com/rmosolgo/graphql-ruby-demo/blob/master/app/graph/fields/fetch_field.rb)
called `FetchFeild`. I’ll add an example below in case you also have chosen this
route. In the `resolve` method below, we add in a Pundit `authorize` call to
ensure the user has access to the record that is being fetched. Pundit will
automatically lookup the policy by model name (class name) and authorize
accordingly.

{{< gist nickpoorman d9cbdcb44f6deaf46ae3426ef13afd9c >}}

Lastly, when doing mutations (or even queries), you’ll need to add your
authorization to create or update the record. Below we have a mutation to create
a Post. We add the `authorize` call in the `resolve` proc. Note that we are
using a [headless policy for
Pundit](https://github.com/varvet/pundit#headless-policies) here: i.e.
`authorize :post`.

{{< gist nickpoorman 968aa61452b22e75a057e3d46414191b >}}

Hopefully this will get you moving along with Pundit authorization a bit faster.

