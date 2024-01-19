---
layout: post
title:  "When returning a value... doesn't"
date:   2024-01-19 07:06:52 -0500
categories: ruby rails activerecord rails7 ruby3
---
So this was interesting:

{% highlight ruby %}
class WordleSolver < ApplicationRecord
  MAX_LETTERS = 5
  validates :letters_guessed, length: { maximum: MAX_LETTERS }

  # ... snip

  def []=(index, value)
    if !value.nil? && !value&.match?(/^.$/u) # any single character
      puts "-=-=-=> #{self.class.name}##{__method__} -> here 1"
      return "NOOOOOOO"
    end
    puts "-=-=-=> #{self.class.name}##{__method__} -> here 2"

    self.letters_guessed[index] = value
  end
end
{% endhighlight %}

This was meant to be throwaway code: set a character in a string based on an index, but exit early if needed. Easy peasy. 
Or so I thought:

{% highlight ruby %}
irb(main):001> ws = WordleSolver.first
irb(main):002> ws.letters_guessed
=> "abcde"
irb(main):003> ws[2] = 'yz'
-=-=-=> WordleSolver#[]= -> here 1
=> 'yz'
{% endhighlight %}

Wait. Why did it print `yz`? Didn't we *explicitly tell it* to return a crazed "NOOOOOOO" when multiple characters were given for `value`? Why is it returing "yz" instead? The second `puts` call never gets executed, so it is (or appears to be) returning when we expect it to, but the return value is definitely not what you would expect.

Another bit of weirdness: it works as expected when the `[]=` method is called using `send`:

{% highlight ruby %}
irb(main):004> ws.send(:[]=, 2, 'yz')
-=-=-=> WordleSolver#[]= -> here 1
=> "NOOOOOOO"
{% endhighlight %}

What is going on here?

My first thought was it has something to do with the naming of the `[]=` method. ActiveRecord provides that method so you can set model attributes using symbols:

{% highlight ruby %}
u = User.new # A model with a first_name attribute
u[:first_name] = "James" # Same(-ish) as u.first_name = "James"
=> "James"
{% endhighlight %}

To investigate this further I renamed the method from `[]=` to `set_value`. The arguments and method body otherwise remained the same. After doing this:

{% highlight ruby %}
irb(main):004> ws.set_value(2, 'yz')
-=-=-=> WordleSolver#set_value -> here 1
=> "NOOOOOOO"
{% endhighlight %}

So renaming the method also works. This means the problem lays somewhere with the behavior around `[]=`, with that specific name.

Maybe it's something with the `inspect` or `to_s` when interacting with irb. Let's make a test case:

{% highlight ruby %}
test "array index setter only accepts a single character" do
  wordle_solver = WordleSolver.new(letters_guessed: "abcde")
  assert_equal "NOOOOOOO", (wordle_solver[0] = "ab")
end
{% endhighlight %}

Unfortunately:
{% highlight ruby %}
WordleSolverTest#test_array_index_setter_only_accepts_a_single_character [/Users/jchilders/work/jchilders/solver_web/test/models/wordle_solver_test.rb:24]:
Expected: "NOOOOOOO"
  Actual: "ab"
{% endhighlight %}

I'm still investigating this and will provide updates soon.
