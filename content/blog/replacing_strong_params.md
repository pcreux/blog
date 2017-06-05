+++
date = "2017-06-02T04:05:25-06:00"
Categories = ["Development","Ruby"]
Description = ""
Tags = ["Development","Ruby", "Functional"]
title = "Simple Functional Strong Params"
+++

Prior to Rails 3, the way to protect against mass-assignment you needed to use the `attr_accessor` class method on your ActiveRecord models.

This was kind of weird solution because your model which represents a database table has some knowledge of what kind of data your webserver is receiving.

The solution that the Rails community had found was the `strong_parameters` gem. The goal of this gem is to filter params at the controller level. This is a better solution but I find it hard to use when you need to filter complex data structures.  

Even this example found in the documention is pretty hard to understand.

```ruby
params.permit(:name, 
              {:emails => []}, 
              :friends => [ :name, 
                            { :family => [ :name ], 
                              :hobbies => [] }])
```

## Partial application

In order to understand the solution that I propose, you need understand partial application of functions, which is one of the key features of some functional languages such Haskell, OCaml, Elm etc.. Note that in this document when I talk about functions, I am talking about lambdas or procs.

Here is the definition of partial application taken from Wikipedia;

> In computer science, partial application (or partial function application) refers to the process of fixing a number of arguments to a function, producing another function of smaller arity.

To better understand what is partial application lets start with an example.

Let say that you have the `add` function that you define using lambda.

```ruby
add = -> a, b { a + b }
```

You can call this function this way:

```ruby
add.(1, 2)
#=> 3
```

But you can write it using partial application

```ruby
add = -> a { -> b { a + b } }
```

Now you can call the function this way

```ruby
add_1 = add.(1)
# => #<Proc:0x007fcf57a58a60@(irb):12 (lambda)>
add_1.(2)
# => 3
```
This means that we can partially apply 1 to the `add` function then apply 2. Note that when we pass the last param, the result gets computed.

So you can call the same function this way

```ruby
add.(1).(2)
# => 3
```

This process of converting a function with multiple parameters to function returning other functions is called currying. Currying can be painful if done manually. Ruby has a solution for that: it's the `curry` method that is defined on `Proc` objects.

So you can take a function with multiple params and convert it to a function with one params that returns a function with one param... up until there is no parameters left.

Here's an example:

```ruby
add_3_numbers = -> a, b ,c { a + b + c }.curry
```

Then your function is curried:

```ruby
add3_numbers.(1).(2).(3)
# => 6
```
This gives us a easy way to apply the parameters of that function in different contexts. This is a very powerful concept that allows you to "initialize" functions before running them.

## Functions to the rescue

### Hash

Let say that we define a function `filter_hash` which takes a list of `keys` to keep from the params hash and the `params` hash.

```ruby
filter_hash = -> keys, params {
  Hash[keys.map { |key| [key, param[key]] }]
}.curry
```

We can use this function to filter a params hash like this one:

```ruby
params = {name: "Joe", age: "23", pwd: "hacked_password"}

filter_hash.([:name, :age]).(params)
# => {name: "Joe", age: "23"}
```

This is great but if we have a more complicated params hash:

```ruby
user_params = {name: "Joe", age: "23", pwd: "hacked_password",
               contact: { address: "2342 St-Denis",
                          to_filter: ""}}
```

We might want to apply a filter to the contact hash. The `filter_hash` does not allow us to apply a filter on the values of the params hash.

```ruby
filter_hash.([:name, :age, :contact]).(params)
# => {name: "Joe", age: "23",
#     contact: { address: "2342 St-Denis",
#                to_filter: ""}}
```

Instead of using the `filter_hash` function, we can define the `hash_of` function which takes a `fields` and a `hash` param. The `fields` param is a hash that maps each key you want to keep to a function to be applied to the corresponding value in the params hash. The `hash` is the params hash to apply the filter.

Here is the definition

```ruby
hash_of = -> fields , hash {
  Hash[fields.map { |(key, fn)| [key, fn.(hash[key])] }]
}.curry
```

It's easier to understand using an example.

```ruby
user = hash_of.(name: -> a {a},
                age: -> a {a},
                contact: hash_of.(address: -> a {a}))
user.(user_params)
# => {name: "Joe", age: "23",
#     contact: { address: "2342 St-Denis" }
```

Note that the `-> a { a }` function is a very useful function in functional programming. It is called the `id` function, it takes a params and returns it as is. Since `id` is used in a different context in Rails, we will define the `same` function.

```ruby
same = -> a { a }
same.(2) # => 2
```

We can then rewrite the `user` function using that function:

```ruby
hash_of.(name: same,
         age: same,
         contact: hash_of.(address: same))
```
This very small function makes the filter functions more DSLish.

Note that these functions are very composable. We can put the `contact` filter in a variable:

```ruby
contact = hash_of.(address: same)
user = hash_of.(name: same, age: same,
                contact: hash_of.(contact))
```

With this you can reuse this filter in another controller very easily.

### Array

If we want to filter a field that contains an array. We can use the following function:

```ruby
array_of = -> fn, value { value.kind_of?(Array) ?  value.map(&fn) : [] }.curry
```

Let say that we have a list of contacts

```ruby
contacts_params = [{address: "21 Jump Street", remove: "me" },
                   {address: "24 Sussex", remove: "me too" }]
```

We can use our `array_of` to filter out our contacts.

```ruby
array_of.(contact).(contacts_params)
# => [{address: "21 Jump Street"},
#     {address: "24 Sussex:}]
```

### Default Values

With strong params it can be hard to set a default value on `blank` data. But using functions it gets very trivial. We can create a `default` function.

```ruby
default = -> default, a { a.blank? ? default : a  }.curry
```

```ruby
contact = hash_of.(address: default.("N/A"))
params = [{address: "21 Jump Street", remove: "me" },
          {address: ""}]
array_of.(contact).(params)
# => [{address: "21 Jump Street"},
#     {address: "N/A"}]
```

### Scalar Values

Using the `same` function can pose some security issues, because if you are expecting a string a you get a hash, the `same` function will keep the hash. A solution against that problem is the `scalar` function.

```ruby
scalar = -> a { a.kind_of?(Array) || a.kind_of?(Hash) ? nil : a }
```
Here's an example
```ruby
hash_of.(name: scalar}.(name: {hack: "coucou"})
# => { name: nil }

hash_of.(name: scalar}.(name: "Martin"})
# => { name: "Martin" }
```

### Strong params example

Here's a way to rewrite the example that we've extracted from the `strong_parameters` documentation.

```ruby
params.permit(:name, 
              {:emails => []}, 
              :friends => [ :name, 
                            { :family => [ :name ], 
                              :hobbies => [] }])
```

```ruby
friend = {name: scalar,
          family: hash_of.(name: scalar)
          hobbies: array_of.(scalar)})}
hash_of.({ name: scalar,
           emails: array_of.(scalar),
           friends: array_of.(friend))
```
Ok, it's a bit more code but it is a way simpler solution, easier to understand, easier to test, more extensible, more reusable and more composable. The only drawback is that you need to be familiar with partial application. However, knowing partial application can improve your code in different ways.

### Extensibility

It is very easy to extend this solution by creating your own functions. Nothing forbids you to create validations, cast functions etc. If you want to extend the `strong_parameters` gem you have to monkey path it, which is a pretty bad solution.

### Other contexts

You can also use this pattern in other context such as converting json structure coming from a remote api. Since you can set defaults or cast data like dates, this makes it a perfect solution to filter/transform complex json payloads structure.

### Conclusion

With only a few very simple functions you can do a good part of what the `strong_parameters` gem does.

```ruby
hash_of = -> fields , hash { 
  Hash[fields.map { |(key, fn)| [key, fn.(hash[key])] }] 
}.curry
array_of = -> fn, value { value.kind_of?(Array) ?  value.map(&fn) : [] }.curry
default = -> default, a { a.blank? ? default : a  }.curry
scalar = -> a { a.kind_of?(Array) || a.kind_of?(Hash) ? nil : a }
```

This solution is so simple that bugs are very less likely to exist.