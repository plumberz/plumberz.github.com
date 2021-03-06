

# Stream Programming libraries in haskell

* [Intro](#intro)
  * [Theoretical background](#theoretical-background)
* [The Libraries](#the-libraries)
  * [Pipes](#pipes)
  * [Tubes](#tubes)
  * [Comparison](#comparison)
  * [Windowed Wordcount with Pipes](#windowed-wordcount-with-pipes)
* [Conclusions](#conclusions)

# Intro
Research conducted by Luca Lodi and Philippe Scorsolini for the course of "Principles of Programming Languages" at Politecnico di Milano by Professor Pradella Matteo and with the supervision of Riccardo Tommasini.

In this post we'll try to examine two haskell "stream processing" libraries, [Pipes](https://hackage.haskell.org/package/pipes) and[Tubes](https://hackage.haskell.org/package/tubes), investigating whether or not they can be used and/or adapted to perform "stream processing", to be meant as in systems as Flink, Spark Streaming, Kafka and others.

## Theoretical background

If you are not familiar with haskell's theoretical concepts this chapter offer a brief recap of the main ones to grasp in order to understand the components of these libraries.
The concepts are presented in an intuitive rather than formal way. For a more formal explanation you can read the referred resources.

#### Kinds
"In type theory, a kind is the type of a type constructor or, less commonly, the type of a higher-order type operator. A kind system is essentially a simply typed lambda calculus 'one level up,' endowed with a primitive type, denoted * and called 'type', which is the kind of any (monomorphic) data type" [cit.](https://wiki.haskell.org/Kind)
Ordinary types, also called monotypes or nullary type constructors, have kind \*. Higher order type constructors have kinds of the form P -> Q, where P and Q are kinds.
```haskell
Int :: *
Maybe :: * -> *
Maybe Bool :: *
a -> a :: *
[] :: * -> *
(->) :: * -> * -> *
```
In Haskell 98, * is the only inhabited kind, that is, all values have types of kind "\*".

In shorter terms kinds define the signatures of type constructors, indicating what they need to build a "\*" (aka concrete) kind, very much in the same way as functions signatures show what the functions can take and what it will produce. Kinds can be curried too: ```Either :: * -> * -> *``` can be partially applied ```Either String :: * -> *``` and then again to obtain a concrete kind ```Either String Int :: *```.

#### Monad
Monads represents computations that can wrap values and potentially have side effects.
For example reading a character from a file is a computation with side effect (```IO Char``` monad).
Computations (monads) can be composed to obtain new more complex computations.
Many types of monads exists, to represent particular properties of computations. For example computation with input/output side-effects are represented as ```IO``` monads, while computations that wrap a state are based on the ```State``` monad.

[Monads in pictures](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html)

#### Monad transformer
Different monad types correspond to different computation's properties (or behaviour).
But many times programmer would like a computation to display properties from multiple monad types at once, without have to declare a new monad type.
Monad transformers solve this issue by providing a way to compose monad behaviours, to create complex ones.
Monad transformers extend a monad's behaviour by adding on top the one related to the transformer's monad.
For example we can apply the ```StateT``` transformer (```State``` monad transformer) to a ```IO``` monad, to obtain a computation that wraps a state but also have I/O side effects.

[Monad transformers](http://book.realworldhaskell.org/read/monad-transformers.html)

#### Lifting
Lifting is a concept that help generalize a function to work with monads, or in other settings.
```haskell
plus :: [Int] -> [Int] -> [Int]
plus = liftM2 (+)
-- plus [1,2,3] [3,6,9] ---> [4,7,10, 5,8,11, 6,9,12]
```
For example ```liftM2 (+)``` maps the ```+``` function to a new one, able to work with monads that are instances of ```Liftable```.
Lists represent non deterministic computations, and thus the lift of ```+``` is a function that return the list of all possible sums of the 2 lists.

The ```lift``` concept comes handy also in the case of monad transformers.
In fact, in addition to map pure functions to work with monads, ```lift``` can also map function between inner monads to work with the transformed (outer) monad.
For example ```liftIO``` map a function that returns an ```IO``` monad to return an IO monad encapsulated in other monad transformers.

[Haskell lift](https://wiki.haskell.org/Lifting)

#### TL;DR (Spoiler alert!)
After **playing with the two libraries** (Pipes and Tubes) and **understanding their structure**, we implemented a simple **word count example** to **compare the two libraries**.
This example is meaningful to highlight the **many similarities** and the **few differences** between the two libraries.

Then we tried to modify the two word count codes to support a **Flink-style time windowing**.
However, we soon found out that the **stream programming model** implemented by the libraries is **perfectly synchronous**, and so **can't be exploited to deal with time**. In fact Pipes and Tubes consider streaming as a way to break down a complex computation into simpler composable stages (**dataflow paradigm**), that form a pipeline able to work on a **huge amount of data with a small memory footprint**.

For these reasons, we implemented the time-window Flink-style word count only in a Pipes-derived library called **pipes-concurrency**, that **extends Pipes** with an **asynchronous dataflow model** (inpired by the actor model). This let us easily implement the use case.

All these **examples are available** [here](https://github.com/plumberz/plumberz.github.io/tree/master/code) with the relative **guide** on [how to run](code/HOW_TO_RUN.md) them.

Finally we came to the **conclusion** that the dataflow stream programming model of these haskell libraries is **not suitable to implement stream processing**, due to the **asynchronous and concurrent nature** of **stream processing** tasks.
In addition, Pipes and Tubes can **only model an almost linear dataflow**, while stream processing frameworks usually handle a **generic computation graph** (directed acyclic graph).


# The Libraries

## Pipes

Pipes *“is a clean and powerful stream processing library that lets you build and connect reusable streaming components”*. Its main focus is clearly to offer the simplest building blocks possible, that can then be used to build more sophisticated streaming abstractions, as can be seen in the rich ecosystem of libraries surrounding pipes.

### Types

#### Proxy

The main component of the library is the monad transformer Proxy. A **monad transformer** is a type constructor which takes a monad as an argument and returns a monad. Through **lifting** one is able to use functions defined in the base monad also in the monad obtained from the monad transformer application.
```haskell
-- Defined in ‘Control.Monad.Trans.Class’
lift :: (Monad m, MonadTrans t) => m a -> t m a

-- Defined in ‘Pipes.Internal’
data Proxy a' a b' b m r
    = Request a' (a  -> Proxy a' a b' b m r )
    | Respond b  (b' -> Proxy a' a b' b m r )
    | M          (m    (Proxy a' a b' b m r))
    | Pure r

```
As we can see Proxy is defined as a **sum type** with 4 different possible data constructors: Request, Respond, M and Pure. Not much is said in the documentation about them, but they are used mainly as a mean to decide which behavior to apply in the various instance definition of Proxy and are transparent to  the end  user of the library otherwise.

Proxy is defined as an instance of many type classes, as can be seen here:

```haskell

instance Monad m => Applicative (Proxy a' a b' b m)
instance Monad m => Functor (Proxy a' a b' b m)
instance Monad m => Monad (Proxy a' a b' b m)
instance (Monad m, Monoid r, Data.Semigroup.Semigroup r) =>
         Monoid (Proxy a' a b' b m r)
instance MFunctor (Proxy a' a b' b) -- Defined in ‘Pipes.Internal’
instance MMonad (Proxy a' a b' b) -- Defined in ‘Pipes.Internal’
instance MonadIO m => MonadIO (Proxy a' a b' b m)
instance MonadTrans (Proxy a' a b' b)
```

Obviously it is an instance of Applicative, Functor, Monad and MonadIO, but these are all common in the haskell world. Much more exotic are the MFunctor, MMonad and MonadTrans instances definitions (monad transformers will be explained later). All of them comes from the [mmorph: Monad morphism](https://hackage.haskell.org/package/mmorph-1.0.0/docs/Control-Monad-Morph.html#t:MMonad) package which defines monad morphisms related utilities.
Monad morphisms commonly arise when manipulating existing monad transformer code for compatibility purposes. The MFunctor, MonadTrans and MMonad classes define standard ways to change monad transformer stacks through lift, squash and hoist.
MonadTrans definition can be also found in [transformers](https://hackage.haskell.org/package/transformers-0.3.0.0/docs/Control-Monad-Trans-Maybe.html#t:MaybeT) package.

A ’Proxy’ receives and sends information both upstream and downstream:

-   **upstream interface**: receives *a* and send *a’*

-   **downstream interface**: send *b* and receives *b’*

-   **m**: the base monad

-   **r**: the return value

#### Concrete type synonyms

Pipes offers many concrete type synonyms for ’Proxy’ specializing further its more generic signature.

Defined X as the empty type, used to close output ends:
```haskell
type X = Void
```

##### Effect
```haskell
-- Defined in ‘Pipes.Core’
type Effect = Proxy X () () X :: (* -> *) -> * -> *
```

Which represents an effect in the base monad, modeling a non-streaming component, and can be *run*, converting it back to the base monad through:

```haskell
runEffect :: Monad m => Effect m r -> m r
```
##### Producer
```haskell
type Producer b = Proxy X () () b :: (* -> *) -> * -> *
```
Representing a Proxy producing *b* downstream, models a streaming source.

##### Consumer
```haskell
type Consumer a = Proxy () a () X :: (* -> *) -> * -> *
```
Representing a Proxy consuming *a* from upstream, models a streaming sink.

##### Pipe
```haskell
type Pipe a b = Proxy () a () b :: (* -> *) -> * -> *
```
Representing a Proxy consuming *a* from upstream and producing *b* downstream, models a stream transformation.

##### Polymorphic synonyms

For each of these types synonyms, except for Pipe, also a *polymorphic version* is defined, using the [Rank-N types](https://wiki.haskell.org/Rank-N_types) GHC extension:
```haskell
type Effect' (m :: * -> *) r = forall x' x y' y. Proxy x' x y' y m r
type Producer' b (m :: * -> *) r = forall x' x. Proxy x' x () b m r
type Consumer' a (m :: * -> *) r = forall y' y. Proxy () a y' y m r
```
which gives more freedom to some parameters (i.e. this way Pipe can both be seen as Producer’ and Consumer’ which allows to write more easily some of the following signatures)

### Communication

To enforce loose coupling, components can only communicate using two functions:
```haskell
-- Defined in ‘Pipes’
yield :: Monad m => a -> Producer' a m ()
await :: Monad m => Consumer' a m a
```
*yield* is used to send output data, *await* to receive inputs. Producers can only yield, Consumers only await, Pipes can do both and Effects neither yield nor await.

### Composition

Connection between components can be performed in different ways.
```haskell
for :: Monad m => Proxy x' x b' b m a' -> (b -> Proxy x' x c' c m b')
                 -> Proxy x' x c' c m a'              
```
A possible specialization of its signature can be:
```haskell
for :: Monad m => Producer a m r -> (a -> Effect m ()) -> Effect m r
```
Which explains better an example like (**for producer body**), where we loop over *producer* replacing each *yield* in it with *body*, which is a function from the output of the producer to an Effect. The **point-free counterpart** to *for* is (**∼>**) (pronounced *into*) such that:
```haskell
(f ~> g) x = for (f x) g  
```
Similarly we have “**feed**” (**>~**), that fills all the awaits this time with a given source.
```haskell
(>~) :: Monad m => Proxy a' a y' y m b -> Proxy () b y' y m c
               -> Proxy a' a y' y m c
```
e.g. (**draw >∼ p**): loops over *p* replacing each *await* with *draw*.

But the most used way to connect Proxies is surely **>->**:
```haskell
(>->) :: Monad m => Proxy a' a () b m r -> Proxy () b c' c m r
                 -> Proxy a' a c' c m r
```
Which can be used similarly to the Unix pipe operator.

Next we’ll see a basic example produced by mixing various examples in the Pipes’ tutorial , a simple main that gets strings from standard input, maps them to lower case and then prints them to standard output, showing the implementations of some parts of Pipes.Prelude.

The following example can be run by ```stack runghc code/pipes-simpleexample.hs``` and then passing inputs from keyboard, e.g. `ciao CIAO` -> `ciao ciao`

```haskell
import Pipes
import qualified Pipes.Prelude as P
import System.IO
import Data.Text (pack, unpack, toLower)
import Control.Monad (forever, unless)
import Control.Exception (try, throwIO)
import qualified GHC.IO.Exception as G

stdinLn :: Producer String IO ()            -- as defined in Pipes.Prelude
stdinLn = do
  eof <- lift isEOF        -- 'lift' an 'IO' action from the base monad
  unless eof $ do
      str <- lift getLine
      yield str            -- 'yield' the 'String'
      stdinLn              -- Loop

stdoutLn :: Consumer String IO ()           -- as defined in Pipes.Prelude
stdoutLn = do
  str <- await                     -- 'await' a 'String'
  x   <- lift $ try $ putStrLn str
  case x of
      -- Gracefully terminate if we got a broken pipe error
      Left e@(G.IOError { G.ioe_type = t}) ->
          lift $ unless (t == G.ResourceVanished) $ throwIO e
      -- Otherwise loop
      Right () -> stdoutLn
-- forever :: Applicative f => f a -> f b  
-- from Control.Monad

map' f = forever $ do                       -- as defined in Pipes.Prelude
  x <- await
  yield (f x)

```

When all the *awaits* and *yield* have been handled, the resulting Proxy can be run, removing the lifts and returning the underlying base monad.

```haskell
main :: IO ()
main = runEffect $ stdinLn >->
                    map' (unpack . toLower . pack) >->  -- pack :: String -> Text,
                              -- unpack :: Text -> String viceversa
                    stdoutLn
```
We have used 3 functions from the text package which many use to overload the String implementation as a list of Char with Text, a much more "A time and space-efficient implementation of Unicode text. Suitable for performance critical use, both in terms of large data quantities and high speed." (e.g. [A sensible starting Prelude template.](https://github.com/sdiehl/protolude), [OverloadedStrings ghc extension](https://www.schoolofhaskell.com/school/to-infinity-and-beyond/pick-of-the-week/guide-to-ghc-extensions/basic-syntax-extensions#overloadedstrings))
But here we are just using pack to convert a String to Text, toLower from the same package to convert the Text to lowercase and then unpack to convert it back to String. Using the ghc extension, packing and unpacking could be avoided thanks to the polymorphic definition of String it introduces.

Applying a monad transformer to a monad returns a monad, as we already said, so obviously results can be composed using the usual *bind* operator (>>=).

## Tubes

This report considers [tubes-2.1.1.0](https://hackage.haskell.org/package/tubes), a stream programming library based on the concept of duality, inspired by Pipes.

### Intuition

Tubes bring stream programming capabilities into haskell. In particular it respond to the need of process in a series of stages (pipelining) a possibly infinite stream of elements. This is useful when the sequence of elements can't be hold in memory, but must be processed in chunks. 

In the pipeline analogy, each stage is a Tube, and the composition of Tubes (itself a Tube) is the final pipeline. Tube has 3 specializations: Source, Channel, Sink. As suggested by names, a Tube usually is the composition of a Source (generate elements), optionally some Channels (process / transform elements) and a Sink (simply process the element).

For example a Source continuosly reads input from console, then a Channel maps those strings to numbers, another Channel filter those numbers by a predicate, and finally a Sink output those numbers to the console.

```haskell
runTube $  -- compose and execute the Tube computation
	sample prompt -- Source that continuosly read input from console
	>< map reverse -- Channel that transform by a function
	>< filter (odd . length) -- Channel that filter by a predicate
	>< pour display -- Sink that output strings to console
```

(```><```) is the Tube composition operator. It will be discussed in detail.
```map``` and ```filter``` are imported from ```Tubes.Util```.

### Types

The library is based on 2 main types: the **tube monad transformer**, and the dual **pump comonad transformer**.

#### Tube
A tube represent a computation that can **await** elements from upstream and **yield** elements to downstream.
```haskell
-- Defined in ‘Tubes.Core’ 
type Tube a b = FreeT (TubeF a b) -- Tube a b m r
yield :: Monad m => b -> Tube a b m () 
await :: Monad m => Tube a b m a
```
A general [tube](https://hackage.haskell.org/package/tubes-2.1.1.0/docs/Tubes-Core.html#t:Tube) awaits elements of type **a**, yields elements of type **b**, performing a computation **m** that return a result of type **r**.

The library provide 3 **subclasses** of tube type: **source**, **channel**, **sink**.
Each subclass has a different method to obtain the corresponding Tube:
- ```sample```: Tube from Source
- ```tune```: Tube from Channel
- ```pour```: Tube from Sink


![recap](tubes_recap.png)

Tubes can be composed using the (```><```) operator, to obtain a new tube.
```
-- Defined in `Tubes.Core`
(><)
    :: Monad m
    => Tube a b m r
    -> Tube b c m r
    -> Tube a c m r
-- E.g:
each [1..10] >< map (*2) >< pour display
-- 2 4 6 8 10 12 14 16 18 20
```
Note that the **composition works at the level of Tubes**, but not between its specialization. 
For example it is not possible to compose a Source and a Channel, but it is necessary to get the corresponding Tube (using ```sample``` and ```tune``` respectively), and only then apply the composition.

![reduce](tubes_reduce.png)

Is possible to obtain a monad ```m r``` from a ```Tube () () m r``` using the ```runTube``` function, or get a value from a tube that yield data with the ```reduce :: Monad m => (b -> a -> b) -> b -> Tube () a m () -> m b``` function.  

#### Source
```haskell
-- Defined in `Tubes.Source`
newtype Source m a = Source {
    sample :: Tube () a m ()
}
```
Sources are a specialization of Tube that can only ```yield``` elements downstream.
The ```sample``` function is used to get the ```Tube``` corresponding to a ```Source```.

![merge](tubes_merge.png)

```
merge :: Monad m => Source m a -> Source m a -> Source m a 
```
A source can be merged with another using the ```merge``` function. This merge can be seen as an alternated "zip" of the two sources. The resoulting Source will yield elements from the two Sources (alternating), untill they both have no elements left.

```haskell
charsFromFile :: MonadIO m => Handle -> Source m Char
charsFromFile handle = Source $ do
    eof <- liftIO . hIsEOF $ handle
    unless eof $ do
        c <- liftIO $ hGetChar handle
        yield c
        sample $ charsFromFile handle
```

This example shows a function ```charsFromFile``` that takes a [file handle](https://hackage.haskell.org/package/base-4.11.1.0/docs/System-IO.html#g:2) and return a Source that yields characters read from that handle.

#### Sink
```haskell
-- Defined in `Tubes.Sink`
newtype Sink m a = Sink {
    pour :: Tube a () m ()
}
```
In symmetry with the ```Source```, ```Sink``` is a specialization of ```Tube``` that can only await elements.
```Pour``` is used to obtain the corresponding ```Tube``` from a ```Sink```.

```
-- Defined in `Tubes.Sink`
instance Monad m => Contravariant (Sink m) where
    contramap f snk = Sink $ map f >< (pour snk)
instance Monad m => Semigroup (Sink m a) where
    s1 <> s2 = divide (\x -> (x,x)) s1 s2
```

```Sink``` is both a ```Contravariant``` functor and a ```Semigroup```. Thus it is possible to map transformations over its inputs, using Contravariant's ```contramap``` [method](https://hackage.haskell.org/package/contravariant-1.4.1/docs/Data-Functor-Contravariant.html#v:contramap). It is also possible to merged a Sink together beside another ```Sink```, using the Semigroup's ```<>``` [append operator](https://hackage.haskell.org/package/semigroups).

#### Channel
```haskell
-- Defined in `Tubes.Channel`
newtype Channel m a b = Channel {
    tune :: Tube a b m ()
}
```
```Channel``` is a Tube that can convert values, while performing a monadic computation.
While it can independently ```await``` and ```yield``` elements, it can be considered as an ```Arrow``` if it yields exactly once after awaiting.

```haskell
split_words :: Monad m => Char -> Channel m Char String
split_words separator = Channel $ go [] where
    go w_acc = do
        c <- await
        if c == separator then do
            unless (null w_acc) $ yield (reverse w_acc)
            go []
        else do
            let w_acc' = c : w_acc
            go w_acc'
```

In the example a custom Channel awaits characters from upstream and yields a word (String) only once a split character is received.

```haskell
validChar :: Char -> Bool
validChar c = (c==' ') || (isAlphaNum c)

word2KV w = (w,1) -- returns a (word:1) pair

myTube handle = sample (charsFromFile handle)
    >< map toLower
    >< filter validChar
    >< tune (split_words ' ')
    >< map word2KV
```

Notice that different Channels can be composed to form a complex Tube. In this case the tube ```myTube``` (that depends on a file handle) is the composition of one Source and 3 Cannels.
1. Source: read characters one by onefrom the handle.
2. Channel: map charachters to lowercase
3. Channel: filter out invalid (non alphanumerical) character.
4. Channel: accumulate characters and yield words to next stage.
5. Channel: transform incoming words (String) into (word:1) pairs.

This Tube can be thought to perform the preprocessing part of a map-reduce computation, as it yields key:value pairs.

#### Pump
```haskell
-- Defined in `Tubes.Core`
type Pump b a = CofreeT (PumpF b a) -- (Comonad w) => Pump b a w r 
send :: Comonad w => b -> Pump a b w r -> Pump a b w r 
recv :: Comonad w => Pump a b w r -> (a, Pump a b w r) 
```
Pumps are the dual of Tubes, they can ```send``` elements of type ```b``` and ```recv``` (receive) elements of type ```a```.
Pumps are internally used in order to run Tubes (```runTube```), since Pump's ```send``` and ```recv``` match with Tube's ```await``` and ```yield```.
Pumps can also be used in order to process streams (```lfold```, ```stream``` functions).

### Utilities

One particularly useful feature of the library is a set of function that ease the creation of Sources, Channels, and Sinks.

Among them:

- ```map```, ```filter```, ```drop```, ```take```, ```takewhile``` create a Tube that works like the corrisponding functions in Prelude, but on a stream rather than a list.
- ```prompt``` is a Source that continuosly yield strings read from the console input
- ```display``` is a Sink that continuosly output the awaited element to the console.
- ```each``` yield each element in a foldable
- ```every``` is like each, but return the element wrapped in a ```Maybe```.

## Comparison

#### Batch use case

In order to show the similarities in the concepts and the small differences in these two libraries, we implemented a Map-Reduce flavoured batch word count using both of them, taking advantage of the stream programming paradigm offered which allowed us to adopt a dataflow approach to the problem, pipelining the steps of the computation.

##### Pipes
Code can be run by ```stack runghc code/wordcount_mapReduce.hs```  that will read input from [tets-text.txt](code/test-text.txt) and will output something like  ```[("per",2),("sociis",1),("vivamus",2),("mus",1),("montes",1),("torquent",1),("augue",11),("integer",2),("f ...```.
```haskell
import qualified Pipes.Prelude as P
import Data.List.Split
import Pipes
import Control.Monad (forever)
import Data.HashMap.Strict
import Data.Char (isAlphaNum)
import Data.Text (pack, unpack, toLower)
import System.IO
--
-- wordcount mapReduce-style, only benefit is to reduce memory consumption
--

main :: IO ()
main =  withFile "test-text.txt" ReadMode $
    \file -> runEffect (P.fold (\x a -> insertWith (+) a 1 x)      -- insert into the state map, adding one each time
                        empty  -- the empty map
                        (show . toList)   -- at the end it produces a String
                        (P.fromHandle file >->
                            P.map (splitOn " ") >->  -- splits the strings received and     produces a list of the words                           
                            forever (await >>= each) >->     -- each unpacks the elements of the list received by awaiting
                            P.map (filter isAlphaNum) >->     -- strip all non alphanumeric characters
                            P.map (unpack . toLower . pack)   -- use toLower from Text to convert a string to lowercase
                            )
                        )
        >>= putStrLn

```

##### Tubes

Code can be run by ```stack runghc code/tubes_FastWordCount.hs```  that will read input from [tets-text.txt](code/test-text.txt) and will output something like  ```"fromList [(\"per\",2),(\"sociis\",1),(\"vivamus\",2),(\"mus\",1),(\"montes\",1),(\"torquent\",1),(\"augue\",11),(\"integer\",2),(\"f ...```


```haskell
import Prelude hiding (map, filter, reduce)
import qualified Prelude as P
import Control.Monad (unless, liftM)
import Control.Monad.IO.Class (liftIO, MonadIO)
import Tubes
import System.IO
import Data.Char (toLower, isAlphaNum)
import qualified Data.HashMap.Strict as Map
--
-- wordcount mapReduce-style, only benefit is to reduce memory consumption
--

-- read a word from a file handle,
-- a word is a non-empty sequence of alphanumeric characters.
hGetWord :: Handle -> IO String
hGetWord handle = go [] where
    go :: String -> IO String
    go word = do
        eof <- hIsEOF $ handle
        if not eof then do
            c <- hGetChar handle
            if not $ isAlphaNum c then do -- any not alphanum is separator
                if not (null word) then do
                    return word
                else go word
            else do
                let word' = word ++ [c]
                go word'
        else do
            return word

-- a Source that continuosly yields words  read from a file handle.
wordsFromFile :: MonadIO m => Handle -> Source m String
wordsFromFile handle = Source $ do
    eof <- liftIO $ hIsEOF handle                                 
    unless eof $ do   
        w <- liftIO $ hGetWord handle
        yield w
        sample $ wordsFromFile handle

-- Open a test file and print to console a map (word : count).
main :: IO ()
main = do
    handle <- openFile "test-text.txt" ReadMode
    let words = (sample $ wordsFromFile handle) >< (map $ liftM toLower) -- read words and convert to lowercase
    wcount <- reduce countWords Map.empty words
    print $ show wcount
    where
        countWords m word = Map.insertWith (+) word 1 m -- increment the map value for the corresponding word, or initialize word ounter to 1
```

Notice that:
- Both the programs read a textual test file, pass that to a Produce / Source that read chucks of data, get the words, map the words to lowercase, then reduce the outputs of the pipeline incrementing the value of a map corresponding to that word.

- Both the programs use the ```map``` and the ```fold```/```reduce``` utils, that has a similar syntax and semantic with respect to the Prelude version.

- The composition follows a similar linear structure: ```... >-> ... >-> ...``` (Pipes) vs ```... >< ... >< ...``` (Tubes).

Is possible to grasp some correspondences between the two libraries. These covers syntax, types, communication primitives and helper functions. However some differences start to emerge when comparing the composition mechanism of these two libraries.
All these aspects are now analyzed. 

#### Types

Since Tubes is inspired by Pipes, many base types and functions of the library have a corresponding type in the other library.

| *Pipes*  	| *Tubes*                                            	|
|----------	|----------------------------------------------------	|
| ```Proxy```    	| ```Tube``` 	|
| ```Producer``` 	| ```Source```                                             	|
| ```Pipe```     	| ```Channel```                                            	|
| ```Consumer``` 	| ```Sink```                                               	|
| ```Effect```   	| ```Tube () () m r```                                     	|

Note that unlike ```Proxy```, ```Tube``` is mono-directional.

#### Communication

The communication between Proxy / Tube use the very same primitives ```yield``` and ```await``` with the same semantics.

#### Prelude functions

Both the libraries reimplement the basic operations on lists (```map```, ```filter```, ...) containted in ```Prelude``` in terms of ```Proxy``` / ```Tube```. Pipes define these functions in ```Pipes.Prelude```, Tubes inside ```Tubes.Util```.

#### Composition

Composition in Pipes can be performed both between Proxies and between Producers/Pipes/Consumers thanks to Rank-N ghc extension, via the ```>->``` operator.

On the contrary, in Tubes the ```><``` operator can compose only matching Tubes. Thus is necessary to obtain the corresponding Tube from a Source/Channel/Sink via ```sample```/```tune```/```pour``` before applying the composition.

![tubes_recap](tubes_recap.png)

So the Pipes equivalent of this picture would not need the transformation between subclasses and Proxy, but would apply the composition operator directly between Producer/Pipe/Consumer.

This is the result of two different design choices. On one hand Pipes defines Producer/Pipes/Consumer in terms of Proxy, and this type is instance of many useful types. On the other hand Source/Channel/Sink just have a getter for the corresponding Tube, and each of them (Source/Channel/Sink) is instance of different types.

Nevertheless this semantic difference does not impact in the usage / syntax of the two programs. It is just a matter of adding a getter in front of every element in the composition, from the programmer's point of view. 


## Windowed Wordcount with Pipes
**How to run the code is explained [here](code/HOW_TO_RUN.md)**

We'll here describe the various attempt we made to try to create a simple timed "_wordcount_" example as the one that can be found on the examples of many stream processing engine (such as [Flink](https://ci.apache.org/projects/flink/flink-docs-release-1.4/quickstart/setup_quickstart.html)), given a tumbling window of 5 seconds and an input stream of lines of text, returns at every closing of the window a map {word: count, ...} where count is the number of times a certain word has been seen during the elapsed time and assuming the input data arriving in the correct order.

```haskell
module Wordcount (
    s2r,
    r2s,
    wordcount,
    timer) where
import qualified Pipes.Prelude as P
import System.IO
import Control.Concurrent (forkOS, threadDelay, takeMVar, putMVar, newMVar)
import System.Timeout
import Control.Monad (forever)
import Control.Concurrent.Async
import Pipes
import Pipes.Concurrent
import Data.Time.Clock
import Data.HashMap.Strict
import Data.List.Split

-- asynchronous version of wordcount, but relies on the fact that the timer allows to

s2r e = loop e where
            loop s' = do
                tic <- await
                case tic of
                    Nothing -> do
                        yield s'
                        loop e
                    Just x -> loop $! mappend s' $ return x

wordcount = forever $ await >>= (yield . toList . fromListWith (+))

r2s = forever $ await >>= each

timer :: Int -> Proxy x' x () (Maybe a) IO ()
timer t = do
    lift $ threadDelay t
    yield Nothing


main :: IO ()
main = do
    (output, input) <- spawn unbounded
    t1 <- async $ do
                runEffect $ P.stdinLn >->
                            P.map (splitOn " ") >->
                            r2s >->
                            P.map (\x -> (x,1)) >->
                            P.map Just >->
                            toOutput output
                performGC
    t2 <- async $ do
                runEffect $ forever (timer 5000000) >-> toOutput output
                performGC
    runEffect $ fromInput input >->
                        s2r [] >->
                        wordcount >->
                        r2s >->
                        P.map show >-> P.stdoutLn

```

Firstly in the large ecosystem of libraries surrounding Pipes we found [pipes-concurrent](https://hackage.haskell.org/package/pipes-concurrency-2.0.9/docs/Pipes-Concurrent.html#v:recv), which provides "_Asynchronous communication between pipes_" and makes possible the adoption of an actor model approach, through the spawning of mailboxes and pieces of pipes which communicate through these as separate actors.

Therefore the [first attempt](code/Wordcount.hs) was to try to implement our wordcount using **pipes-concurrent** the result was quite promising at first, by passing manually strings to standard input everything seemed to work properly, but after a while we noticed that the concurrent  access to the shared mailbox used to communicate between the two pieces of the pipe in certain cases didn't behave as expected, resulting in the **window never closing** if the input stream kept coming at a really **high rate*** (e.g. "**yes| stack runghc code/Wordcount.hs**"), breaking the chosen reporting policy, not reporting at all.

We were feeding into the mailbox a _Maybe Char_, so that when the downstream pipe received a _Nothing_ by a timer running on a different thread and was interpreted as a signal of the closing of the window.

At the same time we tried to adopt the approach of "[ The CQL continuous query language: semantic foundations and query execution](https://link.springer.com/article/10.1007%2Fs00778-004-0147-z)" by dividing the computation in **s2r** (stream to relation), **r2r** (relation to relation), **r2s** (relation to stream), forcing to start the execution of the part counting the words seen only after the window was triggered, accumulating all the input upstream and then passing them down. This approach due to the incremental nature of our task represented a **huge performance overhead**, for this reason in the following examples we looked for different approaches and opted for **merging the s2r and r2r-_ish_ part** into a single accumulator, that will be triggered by a timer and so yield downstream its result at the desired time (obviously more or less the desired time, due to the fact, that no guarantees are given on the upper bound of the _thread delay_ ). This choice as said was due to the nature of this specific example, whereas in cases where the whole window has to  be known at the moment of the r2r operation it could surely still be applied. It has to be said that only the performance were worsened by this approach, not the correctness of the results, obviously.

We have to recognize that the biggest issue we found was due to the introduction of time and asynchronicity, so how to practically trigger the window evaluation at the right time was the biggest question.

```haskell
import qualified Pipes.Prelude as P
import System.IO
import Data.Time.Clock
import Control.Concurrent (forkOS, threadDelay, takeMVar, putMVar, newMVar)
import Control.Concurrent.Async
import Pipes
import Pipes.Concurrent
import Data.HashMap.Strict
import Data.List.Split
import Wordcount

-- synchronous version, no output if no input after window passed
--
-- splitting windowing and counting in 2 steps brings to a major decrease in performance
-- the major problem is the synchronous execution of the windowing part and the wordcount part

main = do
    hSetBuffering stdout NoBuffering
    runEffect $ P.stdinLn >->
                    P.map (splitOn " ") >->
                    r2s >->
                    windowCount 5 >->
                    r2s >-> P.map show >-> P.stdoutLn


windowCount w = do
    t0 <- lift getCurrentTime
    loop (realToFrac w) t0 empty where
        loop w last s = do
            x <- lift getCurrentTime
            el <- await
            let diff = diffUTCTime x last
            if diff <= w
                then loop w last $! insertWith (+) el 1 s
                else do
                    yield $ toList s
                    loop w x empty
```
Therefore in the  **[second attempt](code/wordcount_flink_v1.hs)** we didn't use pipes-concurrent anymore, instead we tried a to **use the standard [clock library](https://hackage.haskell.org/package/time-1.9.1/docs/Data-Time-Clock.html) from haskell,** which obviously does not guarantees the exactness of the time returned by its **getCurrentTime** function, seen that it returns the system clock time, which can be modified by the user or adjusted by the system in any moment, but still, assuming an ideal situation, would be enough to prove what we are trying to show.

The result where good, it kept the pace of **yes**, but this time the **low rate inputs were the problem**.
The triggering of the window was achieved by taking the time before receiving a new input and checking after having received it, if the desired time from the last triggering had passed. Clearly this approach brought to the thread indefinitely waiting for a new input and **never be able to yield downstream** even if the window should have been triggered. This problem arose because we were checking the time at each new tuple and we were not able to trigger it from the outside, but we were still able to use the component as a Pipe and connect it to following Proxies.

These considerations brought to the **[third version](code/wordcount_flink_v2.hs)** and through the use of **[MVars](https://hackage.haskell.org/package/base-4.10.1.0/docs/Control-Concurrent-MVar.html)** we managed to **separate the timer from the counter** in two different threads. Doing so every time the timer is triggered, the timer prints the map contained in the shared MVar and resets it.

Being the main thread the one awaiting for inputs and the timer's secondary thread not a Pipe, we didn't manage to yield downstream the result of the counting allowing it to be used for further computation, **breaking this way the composability at the core of the library**. Has to be said that in the **same++ way as we did, the function **_fold_ in the Prelude of Pipes** does not produce a Pipe and cannot be further composed, so it seems to be accepted this sort of behavior, even if it doesn't fit well in the framework of the usual stream processing definition.

```haskell
import qualified Pipes.Prelude as P
import System.IO
import Control.Concurrent (threadDelay, takeMVar, putMVar, newMVar)
import Control.Concurrent.Async
import Pipes
import Data.HashMap.Strict
import Data.List.Split
import Wordcount (r2s)
-- Asynchronous version that does not output if the main thread has ended
-- By unifying the windowing and the counting we achieve a considerable performance boost,

main :: IO ()
main = runEffect $ P.stdinLn >-> P.map (splitOn " ") >-> r2s >-> timedWindow 5000000

timedWindow t = do
        v <- lift $ newMVar empty
        lift $ async $ timer' t v empty
        wordcount' empty v

timer' w v e = do
    threadDelay w
    c <- takeMVar v
    print $ toList c
    putMVar v e
    timer' w v e

wordcount' e v = do
    x <- await
    s' <- lift $ takeMVar v
    lift $ putMVar v $ insertWith (+) x 1 s'
    wordcount' e v
```
## Conclusions

In the end, with respect to the **initial question** we where trying to investigate, so whether or not the **stream programming** paradigm of the two libraries we have taken in consideration could easily be adapted to perform stream computing tasks, given their current implementations, the answer we can give is "not so easily". The **semantic of stream processing** as in the aforementioned distributed frameworks is **hard to implement** in these libraries and often a question of **tradeoffs** one is willing to accept. Hint of this difficulty is the great complexity of such systems that obviously surpass the complexity of a simple library as the ones we've taken in consideration. For sure the programming paradigm this kind of libraries offer is quite **similar** to the ones offered by the API of the various stream engine or big data framework nowadays, but the underlying architecture of these two libraries has been thought **much more for batch jobs** than for streaming ones and so the concept of **time is completely missing and difficult to plug in**, the main focus is on memory usage and single thread performance and **not concurrency**, even when specific libraries built on top of these exists as for pipes.
