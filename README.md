# About

The `fmap` library defines three methods on all object instances, `Object#fmap`, `Object#afmap`, and `Object#eqfmap`. All three methods are ways to "descend into" arbitrarily-nested data structures to an arbitrary depth, replacing each object in the structure with the return value of the supplied block. This makes these methods a generalization of `Enumerable#map`, that, instead of applying to all Enumerable objects, apply to all *functors*.

From the perspective of the `fmap` library, a *functor* is anything that has a `#functor?` method which returns true. This, by default, includes all built-in container objects, such as Array, Hash, Range, and Set, and also anything which imports the `Enumerable` module (where `Enumerable#fmap` is defined in terms of `Enumerable#map`.) It also includes `Object::Composite`, a module that is brought in by the `fmap` library which you can import to make any simple data structure class (where "simple" means that every instance variable is an exposed value of the container) `fmap`pable.

# Usage

`Object#fmap` is the most general method; it will simply yield every object in the data structure. However, this means that `Object#fmap` also yield the data structure itself. If you don't want this behavior, use `Object#afmap`, which will only yield "atomic" (non-*functor*) values. A comparison:

    require 'fmap'
    require 'set'
    
    stuff = Set[{[1, 2, 3] => [4, 5, 6, true]}, -3.5, :hmm]
    
    stuff.afmap{ |v| v.inspect } # => #<Set: {{["1", "2", "3"]=>["4", "5", "6", "true"]}, "-3.5", ":hmm"}>
    
    stuff.fmap{ |v| v.inspect } # => "#<Set: {\"{\\\"[\\\\\\\"1\\\\\\\", \\\\\\\"2\\\\\\\", \\\\\\\"3\\\\\\\"]\\\"=>\\\"[\\\\\\\"4\\\\\\\", \\\\\\\"5\\\\\\\", \\\\\\\"6\\\\\\\", \\\\\\\"true\\\\\\\"]\\\"}\", \"-3.5\", \":hmm\"}>"
    
The trivial use of `Object#fmap`, using the identity combinator (`id = lambda{ |v| v }`), produces a deep copy without using serialization:

    require 'fmap'

    stuff = Set[{[1, 2, 3] => [4, 5, 6, true]}, -3.5, :hmm]
    new_stuff = stuff.fmap{ |v| v }

`Object#eqfmap` takes a type object as an argument. The type object is used in an equivalence-class comparison (`Object#===`) to determine whether to yield each value.

    require 'fmap'

    stuff = Set[{[-1, 2, 3] => [4, :x, 82, 6, true]}, -3.5, :hmm]
    stuff_without_symbols = stuff.eqfmap(Symbol){ |v| v.to_s }
    stuff_without_symbols # => #<Set: {{[-1, 2, 3]=>[4, "x", 82, 6, true]}, -3.5, "hmm"}>

    ranged_stuff = stuff.eqfmap(Numeric){ |v| [[v, 0].max, 50].min }
    ranged_stuff # => #<Set: {{[0, 2, 3]=>[4, "x", 50, 6, true]}, 0, "hmm"}>


# Using `fmap` with your own data structures

You can add `fmap` support to your own data structures in three ways:

1. You can simply make your class `Enumerable`. When `Object#fmap` is called on your class's instances, they will be iterated through using `Enumerable#map`, and the result will be an Array.

2. You can include the module `Object::Composite` in your class. For each object of this type, a shallow clone will be made of it; then each instance variable set on the instance will be read, `fmap`ped itself, yielded, and then overwritten with the yielded value. This is very likely inefficient in most implementations.

3. You can define your own `#fmap` and `#functor?` methods. (`Object#afmap` and `Object#eqfmap` are both defined in terms of `Object#fmap`.)

An `#fmap`-supporting class looks like this:

    class Foo
      def initialize(a, b)
        @attr_a = a
        @attr_b = b
      end

      ...
    
      def functor?
        true
      end
      
      def fmap(&bl)
        new_attr_a  = @attr_a.fmap(&bl)
        new_attr_b  = @attr_b.fmap(&bl)
        new_inst = self.class.new(new_attr_a, new_attr_b)
        bl.call( new_inst )
      end
    end

There are three parts to the `#fmap` definition:

1. Call `#fmap` on your own values, passing on the block passed to you.
2. Create a new instance of yourself with the results of those `#fmap` calls as the new data.
3. Finally, yield that new instance to the block, and return the value returned from the block.