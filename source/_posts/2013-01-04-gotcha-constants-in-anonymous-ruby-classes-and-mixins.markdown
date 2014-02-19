---
layout: post
title: "Gotcha: Constants in anonymous Ruby classes and mixins"
date: 2013-01-04 14:17
comments: true
categories: 
---

I thought I had a reasonable grasp of Ruby and its constants, rarely
these days having a wat? moment,  but today
I stumbled on something I find really quite surprising.  Particularly so as
I've never come across it before and I've read my fair share of Ruby
books, posts and articles.

Have you ever used constants in anonymous mixins and classes?  Well
read on ...

<!-- more -->

I use (create) anonymous mixins and classes all the time, they are the rule for
me rather than the exception.  If I need to create a named class (e.g.
*MyClass*) from the anonymous variable (e.g. *my\_anon\_class*), I do
an explicit assignment (e.g. *MyClass = my\_anon_class*).

Most Rubyists will be familiar with using constants in classes and
mixins, there are an easy way of providing instances methods with
access to class-level data.  (Class variables do the same but bring
some baggage.)

For example, a trivial example:

```ruby
class MyClass

  MYDATA = [1, 2, 3]

  def show_data
    puts("MYDATA is #{MYDATA}")
  end
  
end

my_inst = MyClass.new

my_inst.show_data

```

This produces the expected result:

```bash
MYDATA is [1, 2, 3]
```

Let's try this with an anonymous class:

```ruby
my_anon_class = Class.new do

  self::MYDATA = [1, 2, 3]

  def show_data
    puts("MYDATA is #{MYDATA}")
  end

end

MyClass = my_anon_class

my_inst = MyClass.new

my_inst.show_data
```

This fails with the following:

```bash
testcon2.rb:6:in `show_data': uninitialized constant MYDATA (NameError)
	from testcon2.rb:16:in `<main>'
```

As it the norm for anonymous classes and mixins, I've explicitly
bound *MYDATA* to *self* and the constant has really been created - a
bit of extra code proves it:

```ruby
my_anon_class = Class.new do

  self::MYDATA = [1, 2, 3]

  def show_data
    puts("MYDATA is #{MYDATA}")
  end

end

MyClass = my_anon_class

MyClass.constants.each {|c| v = MyClass.const_get(c, false); puts("MyClass c #{c} v #{v}") }

my_inst = MyClass.new

my_inst.show_data
```

produces:

```bash
MyClass c MYDATA v [1, 2, 3]
testcon3.rb:6:in `show_data': uninitialized constant MYDATA (NameError)
	from testcon3.rb:19:in `<main>'
```

The explicit *const_get* finds *MYDATA* (and any other constants) no problem.

To labour the point, a minor change to the syntax of the *show_data*
method to explicitly access the class-level constant also
demonstrates that the constant is really available in an instance method:

```ruby
my_anon_class = Class.new do

  self::MYDATA = [1, 2, 3]

  def show_data
    puts("MYDATA is #{self.class::MYDATA}")
  end
  
end

MyClass = my_anon_class

my_inst = MyClass.new

my_inst.show_data

```

produces the expected output:

```bash
MYDATA is [1, 2, 3]
```

I've done a complementary set of tests with mixins as well.

The take-away: *bare* constant names (as in the
first example) just don't work in anonymous
classes and mixins.  This seems to be a compilation issue and it
appears impossible to *persuade* Ruby (e.g. by explicitly re-setting
the constant post class creation) to find the  constant using its
*bare* name.

Which is tedious.

[Ruby]: http://www.ruby-lang.org/

<!--  LocalWords:  mixins
 -->
