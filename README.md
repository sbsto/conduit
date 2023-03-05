# Conduit
A rails gem for generating partials with boilerplate Turbo Streams functions.

## Motivation
Turbo Streams are quite flexible, which is great, but often we end up using them for the same thing: reflecting asyncronous states on the clinet. 

A simple example is the steps taken when presenting a search modal to the client:

1. The client submits a search query
2. A loading state is shown to the client and the server does some work to match results to that query
3. The results are shown on the client

This means that we frequently end up rewriting this workflow, just for different parts of the client. 

> **_NOTE:_** If all you want is to stream and update changes that are directly associated with instances of your rails models, you do not need to use this. There are great helper methods that are already included in the `Turbo::Broadcastable` modules in [hotwired/turbo-rails](https://github.com/hotwired/turbo-rails) that will make this super easy. This gem is designed as a layer on top of that module to allow a similar level of ease with partials that are not related to models (e.g. search results) 

## Design
You can use conduit by running

`rails generate conduit [conduit_partial_name] [controller_name]`

This will generate:

- `_conduit_partial_name_initial.html.erb`
- `_conduit_partial_name_loading.html.erb`
- `_conduit_partial_name_loaded.html.erb`

in the directory `app/views/controller_name/conduit_partial_name`.

It will also add a line

	`<%= turbo_stream_from current_user %>`

to `app/views/controller_name/index.html.erb`. `current_user` should be replaced with some unique identifier for that session.

It will also generate private methods in the specified controller:

`broadcast_conduit_partial_name_loading` and `broadcast_conduit_partial_name_loaded`. 

Calling these methods will broadcast the loaded and loading state of the generated partials. For example:

```ruby
class SearchController < ApplicationController
  before_action :authenticate_user!

  # e.g. GET /search_popover with data-turbo-stream: true
  def search_popover
    # Show loading version as soon as search starts
    broadcast_popover_loading
    @previously_added_songs = previously_added_songs
    @results = get_results(5)

    # Show loaded version now with results
    broadcast_popover_loaded(@results, @previously_added_songs)
  end

  private

  # Call this when we want the loaded partial to show with new data
  def broadcast_popover_loaded(results, previously_added_songs)
    Turbo::StreamsChannel.broadcast_replace_to(
      current_user,
      target: "search_popover",
      partial: "search/search_popover",
      locals: {
        results: results,
        previously_added_songs: previously_added_songs,
        query: params[:query],
      },
    )
  end

  # Call this when we want the loading partial to show
  def broadcast_popover_loading
    Turbo::StreamsChannel.broadcast_replace_to(
      current_user,
      target: "search_popover",
      partial: "search/search_popover_loading",
    )
  end
	
  def get_results
    ...
  end

  def previously_added_songs
    ...
  end
	
end
```

As you can see in the above example, you can also add locals to these partials as needed. 