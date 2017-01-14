+++
date = "2017-01-14T12:12:59-08:00"
title = "How to Skip ActiveModel Callbacks"
draft = false
tags = ["ruby"]
+++

Over the years, I've been increasingly opposed to the use of model callbacks.
There are better ways to achieve the same result (such as service objects) and
they always seem to get in the way.

At work, we've recently run into a issue when migrating some data where we
didn't want callbacks to run.  The official method of skipping callbacks is to
wrap your save in `skip_callback` and `set_callback`:

```ruby
User.skip_callback(:save, :before, :send_welcome_email)
user.update_attributes(email: "user@example.com")
User.skip_callback(:save, :before, :send_welcome_email)
```

The biggest problems with this method is that it disables the callback globally
and that it is not thread safe.  It could have unintended side effects on the
rest of the system.  You also have to run this for each callback that you want
to skip.

We have come up with a better solution that skips all callbacks only for the
instance we are saving.  This makes use of monkey patching, which is not ideal,
but as long as you aren't constantly updating your version of ActiveModel you
should be ok.

```ruby
module SkipCallbacks
  def run_callbacks(kind, *args, &block)
    yield(*args) if block_given?
  end
end
```

```ruby
user.extend(SkipCallbacks)
user.update_attributes(email: "user@example.com")
```
