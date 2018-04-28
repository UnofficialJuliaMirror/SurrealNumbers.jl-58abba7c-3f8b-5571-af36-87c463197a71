# SurrealNumbers

[![Build Status](https://travis-ci.org/mroughan/SurrealNumbers.jl.svg?branch=master)](https://travis-ci.org/mroughan/SurrealNumbers.jl)

[![Coverage Status](https://coveralls.io/repos/mroughan/SurrealNumbers.jl/badge.svg?branch=master&service=github)](https://coveralls.io/github/mroughan/SurrealNumbers.jl?branch=master)

[![codecov.io](http://codecov.io/github/mroughan/SurrealNumbers.jl/coverage.svg?branch=master)](http://codecov.io/github/mroughan/SurrealNumbers.jl?branch=master)


## Intro

This is a slightly crazy package implementing some parts of the
Surreal Number system invented by John Horton Conway, and explained by
Knuth in "Surreal Numbers: How Two Ex-Students Turned on to Pure Mathematics and Found Total Happiness."

It isn't intended to be useful, so much as educational. It was
educational for me to write, in terms of learning Julia. I wanted a
task that would be painful to do in Matlab but "easy" to do in
Julia. It might also be helpful, I hope, for someone trying to learn
about Surreal numbers. 

## Background: the Surreal Numbers

Surreal numbers aren't numbers as we are taught, but they have many of
the same properties. The tricky thing is that they are defined
recursively from the very start. The nice part is that they use only
set operations.

The definition is as follows: a surreal number $x$ is an ordered pair of
sets of surreal numbers (call them $X_L$ and $X_R$) such that every
member of the left set is $\leq$ all of the members of the right set.

There is a starting point -- we can always use empty sets -- as so the
first surreal number (usually denoted zero, because it will turn out
to be the additive identity) is $<\{\} | \{\}>$, where I will use this
angle bracket and pipe notation to denote $x = < X_L | X_R>$.

Then on the "second day" a new generation of surreals can be created in terms of the initial case. On the third day we create a new generation and so on. Each has a meaning corresponding to traditional numbers in order to place a consistent interpretation on standard mathematical operators defined on the surreals.

Just to reiterate, the tricky things is that everything is
recursive. Even comparitors like $\leq$, and hence, we can't even know
if something is a valid surreal in its own right, but only through
recursively investigating its component sets. 

In any case, this is not a survey or tutorial on surreal
numbers. There are many out there better than I can write, e.g.,

+ https://en.wikipedia.org/wiki/Surreal_number
+ https://conservancy.umn.edu/bitstream/handle/11299/174778/Hebert_umn_0130M_15912.pdf?sequence=1
+ https://libarynth.org/_media/surreal_numbers.pdf
+ https://math.stackexchange.com/questions/816540/proof-of-conways-simplicity-rule-for-surreal-numbers
+
+ 

The purpose here is simply to introduce the key reasons for
implementing this in Julia -- it enables users to define
high-performance types of their own, and those types can be
recursively defined.

I know you can do this in other languages, and in some cases also
achieve high performance. But it's *so* easy in Julia. 

On the other hand, while surreals use sets, and Julia has a Set type,
I didn't find that easy to work with. As a type, it doesn't seem to
have all the bells and whistles I would expect from sets. However,
there is an easy out. Although the surreals use sets, i.e., order of
the elements is not important, almost all texts do write these sets in
order. So I felt justified in putting these elements into sorted
arrays. It works nicely, as long as you make sure to only put unique
entries into the array, and this is a little more tricky for surreals
for reasons we will get to in a moment.

...



## Icky bits

The surreals include all real numbers (and infinity and epsilon and
others). However, many of these require infinitely large sets $X_L$
and/or $X_R$. I have some ideas about how to do that (using lazy
evaluation), but they aren't fleshed out yet so for the moment, I will
restrict myself to finite surreals, i.e., surreals with finite sets
$X_L$ and $X_R$. 

This is a pretty rich set by itself, but it doesn't cover even the
entire set of Rationals; only the *dyadic* rationals. So a few words
on them seem in order.


### Dyadic numbers

The dyadic rational numbers are those that have a denumerator that is
an exact power of 2, that is, numbers of the form

\[ x = \frac{ n }{ 2^k } \]

It turns out that every dyadic has a finite representation as a
surreal, and every finite surreal represents a dyadic.

It might seem a little limiting to be restricted to this set, but
remember that floating point numbers are dyadics. They are a (binary)
integer (the *significand*) multiplied by 2 to the power of a (binary)
exponent (just called the *exponent*). Thus all (finite) floating
point numbers have a finite surreal representation.




### Converting a number to a surreal

The description of a surreal given above generates a *form*. The forms
satisfy the rules of arithmetic ($+$, $-$, $\times$ and $/$), and so
we can identify these with the real numbers. However, there are many
forms that equate to the same real number. Thus there are sets of
forms that are equivalent in the sense that $x \equiv y$ if and only
if $x \leq y$ and $y \leq x$.

I think of this loosely by analogy to the rationals, e.g., we can have

    2 // 4 == 1 // 2

but it seemed to distinguish equality from equivalence here. It allows us to test when two surreal forms are the same thing, not just equivalent to the same number. Maybe later we should reduce this to an equality relation. 

BTW, here we hit one of the weirdnesses of Julia; 99\% of the time,
you can redefine operators and comparators to do whatever you like on
your new type. But you can't redefine $===$ or $\equiv$. The blog I
read suggests that this is because this is a core operation, that
might cause problems if a user broke it.

\url{}

Anyway, I have defined *congruence* $\cong$ to do the same thing, check for
equivalence. However, as there are many possible surreal forms we
could create to represent any given real number, we have to chose
one. Call that the *canonical* form. We could define it in several
ways, but the standard is

+ zero => 0 = <{} | {} >

+ integers n => n+1 = <{n} | {} >

+ dyadic fraction => x = \frac{ n }{ 2^k } becomes
     < { x - 1/2^{k-1} } | (x + 1/2^{k-1} } >

+ negative number => use the identity that $-x = <-X_R | -X_L>$

This is the set of conversions implemented, and it includes all
dyadics, and hence floats. Note the recursion inherent in the
construction/conversion. So some care (particularly) with floats
should be taken not to create a variable that exhausts the stack. The
current code isn't very clever in checking for this.

Examples:




### Converting surreal back to a real number

The conversion of canonical forms is relatively easy, but remember
that non-canonical forms are possible, and can be quite
counter-intuitive. 

For instance, naively, you might expect that the form $<\{3\} |
\{17\}>$ could be mapped to the mean of the two sets, i.e., $10$.
However, $x = <X_L|X_R>$ is the simplest number such that $X_L < x
< X_R$, so, in fact, this form is equivalent to $4.

Note, often in texts it is written $X \not \leq Y$ whereas I am writing $X > Y$. The distinction takes care of the case where one or the other is the empty set, but we can equally define $>$ to be synonomous with $\not \leq$.

  check this

The secret of the conversion is again to use recursion, but that is not quite enough in this case. We use several tricks along the way: 

+ If $x$ is equivalent to a known surreal such as 0 or 1, we convert directly
+ If $x$ is negative, we use the identity that $-x = <-X_R | -X_L>$
+ And, most importantly, if $0 < x < 1$ we know $x$ will be the "simplest" dyadic rational number such that  $x_L < x < x_R$. In the interval $(0,1)$ simplest means having the denominator with the lowest power, i.e., in order of preference we would like the denomator to the $1,2,3, \ldots$. We can find this though a simple modification of the standard binary search shown below.

	Pseudo-code

Now that we have these rules, we can convert any number $x \in
[-1,1]$. To convert numbers into this range, we substract 1 (the
surreal additive identity), convert the result (recuseively), and then
add back 1 (this time a real).

The result is not the world's most beautiful code -- I'm sure it can
be improved, but there are so many other inefficiency's here, I am not
sure it warrants it.


### Implementations of operators and standard functions

Not to much to report here. Most of the operators follow standard
surreal definitions and defining them in Julia is easy. There are all
recursive, as you might guess, and so very inefficient -- I wouldn't
want to do demanding computations this way, but they are easy to
understand, for instance

     +(x::SurrealFinite, y::SurrealFinite) = SurrealFinite([x.X_L .+ y; x .+ y.X_L], [x.X_R .+ y; x .+ y.X_R] )

Notice that we are exploiting here Julia's natural extension of
operators from scalar to vectors (this is one of the reasons that
using arrays instead of sets for $X_L$ and $X_R$ is appealling). Thus we can write

     x .+ [y_1, y_2]

once addition is defined on the scalar surreal type, without any additional definitions, and this is particular appealling here as the scalar addition operator is recursively defined in terms of the vector+scalar addition `.+`. 

The one interesting thing to note is that the definitions often
generate non-canonical forms. Part of the aim of this package was to
let people experiment and see such things. It is otherwise far to
laborious to calculate anything but the very simplest cases (none of
the above texts do any but the simplest). 

...

The other pieces are the standard things you expect to be able to do
with numbers, e.g. round, sign, isinteger, ...

I probably haven't implemented all of these, but hopefully enough that
any others can be easily added.

There are two approaches: one is to use intrinsic surreal arithmetic,
e.g. `sign` and `abs` are implemented using

    sign(x::SurrealFinite) = x<zero(x) ? -one(x) : x>zero(x) ? one(x) : zero(x)

Sometimes this is so simple that the function looks almost exactly like it would for any other number, e.g., `sign` above or `abs`

    abs(x::SurrealFinite) = x<zero(x) ? -x : x

The other is somewhat of a cheat. It involves converting the number to
a real, and then using the appropriate operation on that field. I have
tried to avoid that approach when possible. 



The `show` command is designed to show the full structure unless there is a "shorthand" defined for a surreal (most of the simple conversions will set this up). This aids in viewing the surreals succintly, but sometimes we want to see deeper. In the case where shorthand is defined we can use the command `pf` to see deeper, but it will stop at the first layer below with a shorthand. The command `pff` will plot the entire thing, but that is almost impossible to understand for non-trivial surreals, so we also provide a "tree-view" using ?????



###  Uniquely surreal functions

There are two pieces that are unique to surreals:

 + generation: the generation of a surreal is 1 + the generation of
               the surreals used to construct it. Again this is easy
               to implement recursively.
 
 + simplify: convert a surreal into its equivalent canonical form. The
             easiest way to implement this was to use a similar cheat
             to that above, i.e., convert to a real, and then convert
             back to the equivalent surreal in canonical form.

	     This should probably be call canonicalise

Generation comes from Knuth's story (also called birthday)

Important as $x = <X_L | X_R>$ is the simplest number such that $X_L \leq x \leq X_R$.


## Other comments

Don't bother to tell me that this is horribly inefficient. It will
never be anything but. Surreals were not created with numerical
computing in mind. They are about as inefficient a way to do
calculations as I can reasonably think of (excluding the addition of
nullops everywhere).

And note that this isn't really exploring an entire chunk of the
surreals, i.e., the transfinite numbers that can be represented this
way. I'll get to that one day, time gods willing.

There are other implementations of the surreals. For instance
 
+ Coq https://dl.acm.org/citation.cfm?id=2150520
+ Coq https://github.com/pirapira/surreal
+ Haskell https://github.com/Lacaranian/surreal (only integers)
+ Haskell https://github.com/serialhex/Surreal-Numbers (not quite working)
+ Haskell https://github.com/elfakyn/Haskell-surreals (only integers)
+ Mathematica http://demonstrations.wolfram.com/GeneratingTheSurrealNumbers/
              (only generation)
+ Python https://github.com/codeinthehole/python-surreal (not much
         implemented)
+ Python https://github.com/314eter/surreal-numbers
+ Ruby http://raganwald.com/2009/03/07/elegance-and-the-surreals.html
+ 

And some of these languages might be more appropriate in some ways for
this task. But I wanted to learn Julia, and see how far I could take
it here. Moreover, most of these are at least as incomplete as the
code here.




### More information about Surreals

+ https://www.ics.uci.edu/~eppstein/cgt/surreal.html


### Final notes

The package comes with a set of examples of its use in the tests directory.


