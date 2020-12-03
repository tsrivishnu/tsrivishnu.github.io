---
title: "RSpec terminology: Examples and Example groups"
description: I often wondered what groups and examples are in RSpec and I wanted to summarise my learning in this article
tags: ["Testing", "RSpec"]
author: Sri Vishnu Totakura
layout: post
---

Whenever we discuss RSpec, we often use the terms "examples" and "example groups".
They might be evident for a developer using RSpec but in the beginning of my
career, it was not very evident to me.
So, I summarised what I learnt about them in this article.

## Examples

The `it` blocks in your spec file are referred to examples.
Imagine these as testing the example behaviour of an object.
For example:

```ruby
RSpec.describe Account do
  describe "#full_name" do
    let(:account) { Accout.new(first_name: "John", last_name: "Doe") }

    it "combines +first_name+ and +last_name+" do
      expect(account.full_name).to eq("John Doe")
    end
  end
end
```

Here, we are testing an example behaviour of the `Account` object's
`full_name` method.
Therefore, *"combines +first_name+ and +last_name+"* is an ***example***.

## Example groups

The usage of `describe` and `context` creates example groups.
Example groups (or simply referred as groups), are simply a way to organise and group the examples.
These provide meaning and context to the example so that it's easier for the developer or the reader of the tests to understand the specs better.

```ruby
RSpec.describe "example group 1" do
  describe "nested group 1" do
    it "example 1" do
      ...
    end
    it "example 2" do
      ...
    end
  end
  describe "nested group 2" do
    it "example 1" do
      ...
    end
  end
end
```

This is all for this post.
Hope this clarified the terminology for you.
