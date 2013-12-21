# There's only one default object

I was working on a problem from [exercism.io](http://exercism.io) that involved saving items to a hash of arrays. It seemed obvious to me at the time to intitalize the hash with a default object of array: 

`hash = Hash.new([])`

Then, I can simply append all of the items when they were added: 

`hash[:key] << :value`

And ... failing tests. 

So I popped into irb to see what was going on.

```
irb(main):001:0> test_hash = Hash.new([])
=> {}
irb(main):002:0> test_hash[:key]
=> []
irb(main):003:0> test_hash[:key] << :value
=> [:value]
irb(main):004:0> test_hash[:key]
=> [:value]
```

So far so good. 

```
irb(main):005:0> test_hash
=> {}
irb(main):006:0> test_hash.keys
=> []
```

So I was appending `:value` to an array, and the array was being returned when I sent `:key` to `hash`, but it wasn't quite the array I was expecting. 

I had actually appeneded `:value` to the default array that I used to initialize hash:

```
irb(main):007:0> test_hash[:key_1]
=> [:value]
```

The default object returned for a non-existent `:key_1` is `[:value]`. 

`<<` mutates the object it acts on, so it appended the value to the actual array that ruby was using as the default object. It didn't save any values at all for `:key`. I then tried: 

```
irb(main):007:0> test_hash[:key_1] <<= :value_1
=> [:value, :value_1]
irb(main):008:0> test_hash
=> {:key_1=>[:value, :value_1]}
```

And using `<<=` did assign the result to `:key_1`, but it is still mutating the default object. So now the default object has both values.

## The solution


The proper way to append to an default array in a hash like this is to use `+=`. [`Array#+`](http://www.ruby-doc.org/core-2.0.0/Array.html#method-i-2B) returns a copy of the array, and will not modify the default object. This requires casting the value as an array (ie, `[value]`).

```
irb(main):001:0> hash = Hash.new([])
=> {}
irb(main):002:0> hash[:key]
=> []
irb(main):003:0> hash[:key] += [:value]
=> [:value]
irb(main):004:0> hash
=> {:key=>[:value]}
irb(main):005:0> hash[:key_2]
=> []
```


Just for fun, I tried passing a `Proc` in to `Hash.new`, hoping that I could get a new array each time. 

```
irb(main):001:0> ary = Proc.new { Array.new }
=> #<Proc:0x007fb01b3f68f8@(irb):1>
irb(main):002:0> ary.call
=> []
irb(main):003:0> hash = Hash.new(ary)
=> {}
```


But, ruby doesn't call `call`, so all I got was a lousy `Proc`.

```
irb(main):004:0> hash[:key]
=> #<Proc:0x007fb01b3f68f8@(irb):1>
```








