---
title: The one on `let`, `let!`, `before(:all)` and `before(:each)`
description: These are very common helpers in RSpec. In this article, I share some examples explaining their basic behaviour and how they can work together.
tags: ["Testing", "RSpec"]
author: Sri Vishnu Totakura
layout: post
---

These helper methods are very commonly used in RSpec.
They are similar yet very different.
In this article, I will go through their basic behaviour with some examples
and show how they can work together.

Before you continue, it's good to understand the terminologies: `examples` and `example groups` in RSpec.
If you want to quickly read about them, here is one of my previous articles
that takes just two minutes to read:
[RSpec terminology: Groups and Examples]({% link _posts/2020-11-01-rspec-terminology-examples-and-example-groups.md %}).

## `before(:all)`

The block defined with `before(:all)` is run before all the examples within a group.
In other words, its only run once for all the examples inside the group.

Take the below example:

```ruby
RSpec.describe "explain before(:all)" do
  describe "group with before(:all)" do
    before(:all) do |group|
      puts "Running +before(:all)+ block for #{group}"
    end
    it "example 1" do
    end
    it "example 2" do
    end
  end
  describe "group without +before(:all)+" do
    it "example 3" do
    end
  end
end
```

```console
> rspec spec.rb
Running +before(:all)+ block for #<RSpec::ExampleGroups::ExplainBeforeAll::GroupWithBeforeAll:0x0000556d3f03c260>
...

Finished in 0.00178 seconds (files took 0.43658 seconds to load)
3 examples, 0 failures
```
When you run the above spec, `"Running +before(:each)+ block"` is printed once.

## `before(:each)`

Like it reads, `before(:each)` will run the block before every example in the
group in which it is defined.

```ruby
RSpec.describe "explain before(:each)" do
  describe "group with before(:each)" do
    before(:each) do |example|
      puts "Running +before(:each)+ block"
    end
    it "example 1" do
    end
    it "example 2" do
    end
  end
  describe "group without +before(:each)+" do
    it "example 3" do
    end
  end
end
```

```console
> rspec spec.rb
Running +before(:each)+ block for #<RSpec::Core::Example "example 1">
.Running +before(:each)+ block for #<RSpec::Core::Example "example 2">
..

Finished in 0.00308 seconds (files took 0.32345 seconds to load)
3 examples, 0 failures
```

When you run the above spec, `"Running +before(:each)+ block"` is printed twice.
Once for example 1 and once for example 2.

#### default

`before` without explicitly passing `:all` or `:each` will default to `:each`.

```ruby
before(:each) do
  ...
end
# is same as
before do
  ...
end
```

## `let`
A `let` block lets you define a helper method that is available in RSpec group's
examples.
The block passed to `let` is evaluated and the return value of the block will be 
the value returned by the helper method defined.
For example:

```ruby
RSpec.describe "explaining +let+" do
let(:count) {
  count = 0
  3.times { count += 1 }
  count
}
...
end
```
This defines a helper method `count` which returns the value `3`.
You can call this `count` method from the group's examples.

The important characteristic of `let` is that the block is **lazy-evaluated**
and **cached**.

#### lazy-evaluated

The `let` block is evaluated only when the helper method defined by it is
referenced in the spec for the first time.
Until then, it is not evaluated.
Take the following spec for example:

```ruby
RSpec.describe "explaining +let+" do
  let(:my_int) { 3 }

  describe "as string" do
    let(:int_as_string) {
      puts "in the +let+ block"
      my_int.to_s
    }

    it "is will be a string value" do
      puts "in the +it+ block"
      expect(int_as_string).to eq("3")      # first reference
      expect(int_as_string).to be_a(String) # second reference
    end
  end
end
```

```console
> rspec main.rb
in the +it+ block
in the +let+ block
.

Finished in 0.0018 seconds (files took 0.28244 seconds to load)
1 example, 0 failures
```
You notice that the `puts` line from within the `it` block is first
printed and only after the first reference of `int_as_string`, the
line from the `let` block is printed.

#### Cached

The +let+ block is evaluated only for the first time the helper method is referenced
and the value is memoized.
For all the other references, the memoized value is returned without evaluating
the block.

This is why, in the above example, you see `in the +let+ block` printed only once
for the first reference and not for the second reference.

Remember, the caching is only done for the example and that the `let` block is 
evaluated for every example. 
Let's add another `it` block into the above example:

```ruby
RSpec.describe "explaining +let+" do
  let(:my_int) { 3 }

  describe "as string" do
    let(:int_as_string) {
      puts "in the +let+ block"
      my_int.to_s
    }

    it "is will be a string value" do
      puts "in the +it+ block"
      expect(int_as_string).to eq("3")      # first reference
      expect(int_as_string).to be_a(String) # second reference
    end
    it "is is not empty" do
      puts "in the new +it+ block"
      expect(int_as_string).to_not be_empty
    end
  end
end
```

```console
> rspec main.rb
in the +it+ block
in the +let+ block
.in the new +it+ block
in the +let+ block
.

Finished in 0.00594 seconds (files took 0.38973 seconds to load)
2 examples, 0 failures
```

You see that the `let` block is evaluated twice.

## `let!`

This is same as `let` but the only difference is that the evaluation of the
block is done immediately regardless of whether the helper method defined by it
is referenced or not.

Modifying the previous example by changing `let(:int_as_string)` to `let!(:int_as_string)`,
prints the following:

```console
> rspec main.rb
in the +let+ block
in the +it+ block
.

Finished in 0.0018 seconds (files took 0.28244 seconds to load)
1 example, 0 failures
```

Notice the `let` block is evaluated before the `it` block.

## Referencing `let` helper methods in `before`

One similarity between `let`, `let!` and `before(:each)` is that their blocks
are evaluated once for every example.

This lets you reference the helper methods defined by `let` inside a `before(:each)`
even if the `let` is not defined before the `before` block.
The following will work completely fine.

```ruby
RSpec.describe "group" do
  before(:each) do
    puts "Value of +name+ is #{name}"
  end

  context "group1" do
    let(:name) { "John" }
    it "example1" do
      ...
    end

    context "group2" do
      let(:name) { "Doe" }
      it "example1" do
        ...
      end
    end
  end
end
```

However, if you are to reference the `let` defined helper methods from
`before(:all)`, it will not work.
This is because `before(:all)` executes once for all example while `let`
evaluates once for every example.

## Misc

### When to use `let` and `before`

Use `let` when you have to share the value of a block between different examples.

Use `before` when you have to set the state for the example or a group.

### Use `let` instead of instance variables

Main problem (and also a feature in some cases) with instance variables is that
they get created automatically with a `nil` value when they are referenced.
So, if you make a type in the name of the instance variable, you will not
see an error.
This will lead to some unexpected behaviour or errors that can be hard to debug.

If you use `let` and make a typo when referencing the helper method, it will
raise a `NameError` which makes it easy to identify the issues.

Another issue is with instance variables defined in `before` block. They are
always evaluated for every example (`it` block) regardless of whether
the variable is actually needed inside the examples.


