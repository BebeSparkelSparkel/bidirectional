# Bidirectional - bidirectional parsing

This library is based on the [blog](https://blog.poisson.chat/posts/2017-01-01-monadic-profunctors.html) by Lysxia.

In it they define a bidirectional parser that can both generate and consume
values.

Imagine that you have a record `Person` and you want to serialize it into a
list and deserialize it back.

``` haskell
data Person = Person { name :: String, age :: Int }
```


The parser is parameterized over the parsing and generation contexts, these
needs to be selected carefully when implementing the parsers. It could be for
example a `RowParser` `Writer [SQLData]` combination for sqlite-simple, or a
simple state and writer for our running example.

So, let's figure out the context for our example. We want to encode into a list
and then decode the list back into a value.

``` haskell
type SimpleParser a = IParser (StateT [String] Maybe) (Writer [String]) a a
```

Then we create the parsers using the `parser` function.

``` haskell
int :: SimpleParser Int
int =
  parser
    (StateT $ \(x:xs) -> (,xs) <$> readMaybe x)
    (\x -> x <$ tell [show x])

string :: SimpleParser String
string =
  parser
    (StateT $ \(x:xs) -> Just (x,xs))
    (\x -> x <$ tell [x])
```

One small detail for creating records, is that the encoder gets the full record
structure by default, so you need to focus in on a specific part of the record
using the `(.=)` function.

``` haskell
person :: SimpleParser Person
person =
  Person <$> name .= string
         <*> age .= int
```

And now you can use your encoders and decoders

```
> execWriter (encode person (Person "foo" 30))
["foo", "30"]

> evalStateT (decode person ["foo", "30"])
Just (Person { name = "foo", age = 30 })
```
