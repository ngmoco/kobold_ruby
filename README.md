#kobold

A collection of 4 helper libraries, with some interdependencies.  These libraries, in summary:

kobold - Some basic ruby helpers for hashes and lists.  Also, a comparison helper for deep-comparing two data structures.

kobold_test - Some test helpers for ruby, including some custom assertions and a helper for comparing HTTP responses.

kobold_sinatra_test - Test helpers for Sinatra applications.  Includes a function that issues an HTTP request and validates the response (by code, payload, response headers, etc.).

kobold_rails_test - Test helpers for Ruby on Rails applications. the HTTP request-and-validate function from kobold_sinatra_test.  Also, activeresource_fake, which can be used to isolate tests from external SOA dependencies, where the interface for communicating with these dependencies is activeresource.

The inter-dependencies are:

kobold_test depends on kobold

kobold has no real dependencies, but its tests depend on kobold_test

kobold_sinatra_test depends on kobold_test and kobold

kobold_rails_test depends on kobold_test and kobold

##kobold

Modularized helpers.  No import side-effects.  In principle, the only effect of pulling kobold into your project is that
a few extra modules are added to your namespace.  No putting extra methods on built-in types, no implicit changes
to existing behavior.

This is antithetical to activesupport.  It has useful functionality, but pulling it in makes all sorts of changes
to built-in types.  In general, anything that we find useful in activesupport can be pulled into a helper in kobold,
so that we can use the functionality without all the buffoonery that activesupport imposes upon us.

###HashHelper

Standalone functions that work on hashes.  These functions make sense as functions attached to a hash object, but including
them as standalone functions whose first argument is a hash makes just as much sense.

####project_keys(hash_in, list_of_keys)

Takes a hash and a list of keys.  Creates a new hash with only those keys and returns it.  A block can be 
optionally supplied.  If supplied, the block is called for each key in the list_of_keys and its corresponding value.  
The block returns a bool that tells us whether or not to include the key in the hash.  If the block is not supplied, 
we assume that every key is included in the resulting hash.

####safe_merge(hash_one, hash_two)

Takes two hashes and returns the union of the two hashes.  We do this by iterating over the keys in both hashes, putting
the corresponding value in the return hash.  If we encounter a key that we've already encountered, we prepend the key
with an underscore so that we preserve the values from both hashes.

####flatten(to_flatten)

Takes a hash and returns the hash in its "flattened" form.  What that means is that the keys from any arbitrarily-deeply
nested hash are pulled into the top-level hash.  safe_merge is used, so if we ever encounter keys that already exist
in the top-level we use the key prepended with an underscore.

####symbolize_keys(hash)

You may recognize this from ActiveSupport - it's basically the same function, just pulled off of the built-in Hash and put
into the HashHelper module.  Takes a hash, returns the "symbolized" version of the hash.  Any key in the hash that 
responds to to_sym is turned into a symbol.

###ListHelper

Standalone functions that work on lists.

####create_csv(list_of_hashes, sep=',', keys=nil)

Takes a list of hashes, and returns a string representing a CSV.  Each hash represents a "row" in the resulting csv.  
The columns are the keys from the first hash in the list (if other hashes have other keys, they will be ignored).

###ComparisonHelper

Standalone functions to compare various data structures.  Only one of these functions - compare - is actually used.
It delegates to the appropriate helper functions based on the types of the arguments.

####compare(expected, actual, options={})

#####Parameters

Takes two arbitrary data structures to compare and an optional hash of options.  The two structures must either be
a list, a hash, some data structure that can be sanely compared with ==, or a "DontCare" (I'll talk about that in a 
minute).  

#####Return Values

If the two things match, we return the symbol :match.  If they don't match, we return a list of two elements: the
differences in the first, and the differences in the second.  How these differences are represented depends
on the types that were used.

If both things were hashes, we return two hashes, both containing only the keys that were different.  The first hash 
maps those keys to the values in the first input, and the second hash does likewise for the second input.

```ruby
ComparisonHelper::compare({:a => 1, :b => 2}, {:b => 3, :c => 4})
# => [{:a=>1, :b=>2, :c=>nil}, {:a=>nil, :b=>3, :c=>4}]
```

If both things were lists, we return two lists.  The positions of the lists that matched are represented with an 
underscore.  The positions of the lists that didn't match contain the thing that was different.

```ruby
ComparisonHelper::compare([1, 2, 3], [1, 4, 3])
# => [["_", 2, "_"], ["_", 4, "_"]] 
```

If both things are neither lists nor hashes, we compare them with ==, which is fairly straightforward:

```ruby
ComparisonHelper::compare(1, 2)
# => [1, 2]
```

Keep in mind that the compare is recursive.  If we compare two hashes, we are actually comparing the values of each
key.  If the value for one of these keys is another hash, we will end up calling compare on each of the keys in that 
hash.  The nesting is demonstrated here:

```ruby
ComparisonHelper::compare({:a => [{:a => 1, :b => 2}, {:c => 3, :d => 4}]}, 
                          {:a => [{:a => 2, :b => 2}, {:c => 4, :d => 4}]})
# => [{:a=>[{:a=>1}, {:c=>3}]}, {:a=>[{:a=>2}, {:c=>4}]}] 
```

#####DontCare

When used in testing, the first data structure in the compare is the value that the test expects, and the second
data structure is the value that the test actually got.  In this context, you will sometimes not care about certain data
in the second data structure.  Imagine, for instance, that there is a :create_date key that maps to some specific 
datetime that you have no way of knowing in your test.  You know that it's an ISO 8601 parseable datetime, and it shouldn't
be nil, but the exact value isn't even particularly relevant.  In times like these, you can put a DontCare in the 
first data structure to signify that a comparison should not be done.  You can supply a "rule" to the DontCare to 
make sure the data conforms to some expectation.  

```ruby
ComparisonHelper::compare({:create_date => ComparisonHelper::DontCare.new(:rule => :iso8601_datetime)}, 
                          {:create_date => "2013-06-25T10:32:00Z"})
# => :match
```

The documented rules are as follows:

:not_nil_or_missing - pass as long as the value is not nil, or :missing (this is the default)

:array - pass as long as the value is an array.  If :length is supplied as an option to the constructor, assert that it 
is the right length as well.

:json - pass as long as it parses as json

:iso8601_datetime - pass as long as it parses as a DateTime of the form "%Y-%m-%dT%H:%M:%S%z"

:no_rules - just pass, no matter what

You can also use DontCares with the :dontcare symbol.  If you do that, the rule is set to the default 
(not_nil_or_missing)

#####Options

compare takes a hash of optional arguments.  Defaults are:

```ruby
default_type_compare = {:hash => :full,
                        :ordered => true}
```

The :hash key can be :full or :existing.  This option really affects the keys that are used to compare hashes.
If :hash is set to :full, the list of keys we compare is the union of keys from both hashes.  If :hash is set to
:existing, we only compare the keys from the first hash.  The idea with an :existing compare is that you are only 
validating a subset of the keys, and don't care about extra keys that might be in the actual result.

:ordered can be true or false, and affects array comparisons.  There are times when you want to compare two lists
as if they were sets - that is, you don't care about the order that the items appear in the lists as long as the elements
in one list all occur in the second.  :ordered defaults to true.

The cost of doing an unordered comparison is that we lose partial diffing if the list contains some nested
structure.  Basically, we only know if there is a match in the other list or not.  If there is not, we don't know 
which element in the other list we should diff against.  To clarify, this example:

```ruby
ComparisonHelper::compare([{:a => 1, :b => 2}, {:a => 3, :b => 4}], 
                          [{:a => 2, :b => 2}, {:a => 3, :b => 5}])
# => [[{:a=>1}, {:b=>4}], [{:a=>2}, {:b=>5}]] 
ComparisonHelper::compare([{:a => 1, :b => 2}, {:a => 3, :b => 4}], 
                          [{:a => 2, :b => 2}, {:a => 3, :b => 5}], 
                          :ordered => false)
# => [[{:a=>1, :b=>2}, {:a=>3, :b=>4}], [{:a=>2, :b=>2}, {:a=>3, :b=>5}]] 

```

##kobold_test

###TestHelpers::CustomAssertions

####assert_deep_compare(expected, actual, options={})

This calls into ComparisonHelper::compare(expected, actual).  If the result is not :match, we raise an 
AssertionError.

###TestHelpers::Response

####create_response_stub(options)

Create an HTTP Response stub.

####assert_response_matches(options)

Compare the an expected response with the actual response, in terms of status code, payload and headers

Takes a hash, options, with the following keys:

method - :get, :post, :put or :delete

path - Optional, path to the endpoint 

expected - Hash of :code, :headers and :payload

parsed_body and/or response_body

response_headers 

response_code 

response_flash (Optional, Rails only)

##kobold_rails_test

###TestHelpers::ActiveResourceFake

Instead of relying on an external service for ActiveResource, we can rely on an in-memory hash for 
testing.  Every time "create" is called, we store the created instance in a hash.  We look into this hash
whenever find is called.

####install_activeresource_fake

Monkey-patches several methods on ActiveResource::Base to introduce the desired isolation

####uninstall_activeresource_fake

Reverts all the monkey-patches from install_activeresource_fake to return ActiveResource behavior back to normal.

####Mixin vs. Module methods

You are welcome to use the module methods (eg. ActiveResourceFake.install_activeresource_fake), but you are
responsible for your own cleanup when your test is complete.  If you do not cleanup, your test's run leaks into
subsequent tests in the suite, which is not desirable.

Alternatively, you can use the mixin by "include ActiveResourceFake" in the TestCase subclass.  This will call
install_activeresource_fake in the setup, and uninstall_activeresource_fake in the teardown.  Explicit is
better than implicit, but even better is making sure your tests are isolated from each other.

###TestHelpers::Request

##kobold_sinatra_test

###TestHelpers::Request
