---
layout: default
---

# Operation::Controller

The `Operation::Controller` module provides four shorthand methods to run and present operations. It works in Rails but should also be fine in Sinatra.

## Generics

Before the operation is invoked, the controller method `process_params!` is run. You can override that to normalize the incoming parameters.

Each method will set `@operation`, `@model` and `@form` on the controller which allows using them in views, too. Each method returns the operation instance.

You need to include the `Controller` module.

{% highlight ruby %}
class ApplicationController < ActionController::Base
  include Trailblazer::Operation::Controller
end
{% endhighlight %}


## Run

Use `#run` to invoke the operation.

{% highlight ruby %}
class CommentsController < ApplicationController
  def create
    run Comment::Create
  end
end
{% endhighlight %}

Internally, this will do as follows.

{% highlight ruby %}
process_params!(params)

result, op = Comment::Create.run(params)
@operation = op
@model     = op.model
@form      = op.contract
{% endhighlight %}

First, you have the chance to normalize parameters. The controller's `params` hash is then passed into the operation run. After that, the three instance variables on the controller are set.

An optional block for `#run` is run only when the operation was valid.

{% highlight ruby %}
class CommentsController < ApplicationController
  def create
    run Comment::Create do |op|
      flash[:notice] = "Success!" # only run for successful/valid operation.
    end
  end
end
{% endhighlight %}

## Present

To setup an operation without running its `#process` method, use `#present`.

{% highlight ruby %}
class CommentsController < ApplicationController
  def show
    present Comment::Update
    puts @model.id # this is available now.
  end
end
{% endhighlight %}

Instead of running the operation, this will call `::present`, pass in the controller's `params` and hence only run the model finding logic (or, more precise, everything inside `Operation#setup!`).

{% highlight ruby %}
Comment::Update.present(params)
{% endhighlight %}

Everything else of the call stack is identical to `#run`.

## Form

To render the operation's form, use `#form`.

{% highlight ruby %}
class CommentsController < ApplicationController
  def show
    form Comment::Create
  end
end
{% endhighlight %}

This is identical to `#present` with one addition: after the operation, model and form are set up, the form's `prepopulate!` method is called.

{% highlight ruby %}
op = Comment::Create.present(params)
op.contract.prepopulate!
{% endhighlight %}

If you don't want `prepopulate!` to be invoked, simply use `#present` in place of `#form`.

## Respond

Rails-specific.

{% highlight ruby %}
class CommentsController < ApplicationController
  respond_to :json

  def create
    respond Comment::Create
  end
end
{% endhighlight %}

This will do the same as `#run`, invoke the operation and then pass it to `#respond_with`.

{% highlight ruby %}
op = Comment::Create.(params)
respond_with op
{% endhighlight %}

The operation needs to be prepared for the responder as the latter makes weird assumptions about the object being passed to `respond_with`. For example, the operation needs to respond to `to_json` in a JSON request. Read about [Representer](representer.html) here.

In a non-HTML request (e.g. for `application/json`) the params hash will be slightly modified. As the operation's model key, the request body document is passed into the operation.

{% highlight ruby %}
params[:comment] = request.body
Comment::Create.(params)
{% endhighlight %}

By doing so the operation's representer will automatically parse and deserialize the incoming document, bypassing Rails' `ParamsParser`.

If you want the responder to compute URLs with namespaces, pass in the `:namespace` option.

{% highlight ruby %}
respond Comment::Create, namespace: [:api]
{% endhighlight %}

This will result in a call `respond_with :api, op`.

## Normalizing Params

Override #process_params! to add or remove values to params before the operation is run. This is called in #run, #respond and #present and #form.

{% highlight ruby %}
class CommentsController < ApplicationController
private
  def process_params!(params)
    params.merge!(current_user: current_user)
  end
end
{% endhighlight %}

This centralizes params normalization and doesn't require you to do that in every action manually.