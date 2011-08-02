---
layout: post
title: "Simple URL Shortener"
date: 2011-08-02 08:48
comments: true
categories: ruby
---

We are going to be making a really simple URL shortener in Sinatra with Mongoid.

The first step is to make a model for the mapping between a URL and its unique
identifier. We could use ObjectIds, MongoDB's own unique identifier, but those
are fairly long for a URL shortener, so we will roll our own.

{% codeblock model.rb %}

class Url
	include Mongoid::Document
	field :url, :type => String
	field :ident, :type => String

	validates_presence_of :url
	validates_format_of :url, :with => URI::regexp(%w(http https)), :message => "not valid URL"
	validates_uniqueness_of :url, message: "already taken"

	before_create :set_ident

  private
	# Generate a unique identifier for the URL.
  def set_ident
    loop do
      ident = ActiveSupport::SecureRandom.urlsafe_base64(5)
      if self.class.where(ident: ident).count == 0 # Loop until it's unique.
        self.ident = ident
        break
      end
    end
  end
end
{% endcodeblock %}

Because a user will never set the identifier themselves, a unique validator
would not be useful, so we just make sure it's uniqueue ourselves.

Let's talk about our API. We have 2, maybe 3 actions we need to take. We need 
to make new URLS, and be redirected by existing ones. Let's make POST / the
route to make a new URL shortening, and /:ident the redirect method, because
that makes sense. First, the redirection creation route.

{% codeblock url_creation.rb %}
post '/' do
	url = Url.find_or_create_by(url: params["url"])
	if url.errors.empty
		url.ident
	else
		url.errors.to_json
	end
end
{% endcodeblock%}


Thanks to Mongoid, we can write consise like that. The redirection block is similarly simple.

{% codeblock redirection.rb %}
get '/:ident' do |ident|
	url = Url.where(ident: ident).last
	if url
		redirect url.url, "Go to #{url.url}\n"
	else
		404
	end
end
{% endcodeblock %}

If a redirection is not found, the block returns 404, which is interpreted as a 404 not found
error, which is handled by this block
{% codeblock error.rb%}
error 404 do
	"Not found"
end
{% endcodeblock %}

You can also add an optional GET route to provide information relating to the API, you don't 
need to.

This is the full code listing: 

{% gist 1120738 %}
