---
layout: post
title: "Rails Authentication Via Sessions"
date: 06 October 2018
tags: rails authentication sessions
except: <p>Read on</p>
---

Recently, I've been looking into API development with Rails (when I have a spare minute), as well as the use of JSON Web Tokens (JWTs) in API authentication. With Rails 5, all it takes to get started with a new boilerplate API is using the `--api` flag when creating your project. 

Understanding how Rails manages sessions can seem irrelevant to a beginner, especially if you've already gone through [Hartl's tutorial](https://www.railstutorial.org/book) once, and are trying to get practice with Devise, which handles the minutiae of authentication. Nevertheless, I'd like to take a minute to review sessions, and the best place to start is with an explanation as to why we need them in the first place.

<!--more-->

# HTTP and statelessness
Regardless of where you are in your current webdev journey, you've doubtless heard of the [HyperText Transfer Protocol](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol#HTTP_session), or HTTP. It is the underlying protocol used for communication on the internet, where a user agent sends a request, such as for a website or a picture, and a server responds with a status code and the information requested, if successful. It's the mechanism that allows an individual to access and use your Rails web app via a browser.

The HTTP protocol is a stateless protocol, however, which presents an issue. HTTP servers are not required to keep information about a user agent once a response has been returned. Think about the process of logging into a website to update your profile. When authenticating over a HTTP protocol, you are merely asking the server to verify that you are a signed up user. Once a server has responded and finished the request/response cycle, how would you update your profile? How would the server know how you are, and which profile is being updated? How would the server authorize your actions, or what you can and cannot do, if it can't remember you? This is the challenge that Rails session stores are meant to address.

# Rails sessions
Rails provides a good overview of how sessions work in the [guides](https://guides.rubyonrails.org/action_controller_overview.html#session). The default storing mechanism is `ActionDispatch::Session::CookieStore`, which is a hash-like object that can be used to store information on a tamper-proof cookie. Most commonly, a user's ID is stored in this cookie. This means that basic authentication might look like this, assuming you are treating sessions like a RESTful resource and using bcrypt's `authenticate` method:

```ruby
# app/controllers/sessions_controller.rb

def create
  user = User.find_by(username: params[:session][:username])

  if user && user.authenticate(params[:session][:password])
    session[:user_id] = user.id
    # what do
  else
    # what do
  end
end
```

The data stored on `session` is now encrypted in a cookie that is sent with every request to your Rails server, which naturally decrypts it. The session data is accessible in any other controller/view. Thus, a user might update their profile like so:

```ruby
# app/controllers/users_controller.rb

def update
  user = User.find(session[:user_id])

  if user.update_attributes(user_params)
    # what do
  else
    # what do
  end
end
```

That's all there is to it, in a nutshell. Rails puts data in a cookie, Rails pulls data out of a cookie!