Monads for functional programming
=================================

Takes an approach for a simple arithmetic language.

```
data Term = Con Int | Div Term Term
```

Have the following:

```
eval :: Term -> Int
eval (Con a) = a
eval (Div t1 t2) = eval t1 / eval t2
```

**Question**: How do we extend this to allow for certain other operations? Example, exceptions, counting number of divs.
In impure languages (i.e. Java, C) this is simple, just use mutable state.

Variation One: Exceptions
--------------------------

```
-- Either throws an exception or returns a type. Similar to the Either Monad.
data M a = Raise Exception
         | Return a
type Exception = String

-- Need to update this to return an M Int
eval :: Term -> M Int
eval (Con a) = Return a
eval (Div t1 t2) = case eval t1 of
                      Raise e  -> Raise e
                      Return a -> case eval t2 of
                                     Raise e  -> Raise e
                                     Return b -> if b == 0
                                                 then Raise "divide-by-zero"
                                                 else Return (a / b)
```

Clearly, this could be less gross. Note that this turns into a cascading set of `case` statements.

Variation Two: State
--------------------

Say we wish to track the number of divisions that occur, for profiling purposes.

```
-- Function that takes an initial state, returns a value plus the state.
type M a = State -> (a, State)
type State = Int

-- Now, eval takes both a term and an M Int, which takes in an initial state.
eval :: Term -> M Int
eval (Con a) s = (a, s)
eval (Div t1 t2) s = let (a,y) = eval t1 s
                         (b,z) = eval t2 y
                     in (a / b, z+1)
```

Again, this is a bit gross to have to write out by hand.


Monads To the Resuce
--------------------

Monads can help us to be briefer here. There are two elementary operations defined on a monad:

```
unit :: a -> M a
bind :: M a -> (a -> M b) -> M b
```

Variation One: Exceptions Monad
-------------------------------

```
type M a = Raise Exception
         | Return a
type Exception = String

unit :: a -> M a
unit a = Return a

bind :: M a -> (a -> M b) -> M b
m `bind` f = case m of
               Raise e  -> Raise e
               Return a -> f a

eval :: Term -> M Int
eval (Con a) = unit a
eval (Div t1 t2) = eval t1 `bind`
                    \a -> eval t2 `bind`
                      \b -> if b == 0 then Raise "divide by zero"
                                      else unit (a / b)
```

Variation Two: State Monad
--------------------------

Now, as before we wish to track the number of divisions performed, but we want to make the monad handle this for us.

```
type M a = State -> (a, State)
type State = Int

-- | unit takes in a value, and returns a function that takes an initial state and returns the initial state and value
unit :: a -> M a
unit a = \s -> (a, s)

-- Extra functions
tick :: M ()
tick \x -> ((), x+1)

-- Take a current state, run it through a function to update the state.
bind :: M a -> (a -> M b) -> M b
m `bind` f = \s -> let (a,y) = m s
                       (b,z) = f a y
                   in (b,z)

eval :: Term -> M Int
eval (Con a) = unit a
eval (Div t1 t2) = eval t1 `bind`
                    \a -> eval t2 `bind`
                      \b -> tick `bind`
                        \_ -> unit (a / b)
```

The tick function will go ahead and up the counter by one when a division is encountered, but then return `a/b` in an `M Int`

