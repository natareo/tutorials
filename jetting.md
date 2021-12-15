Supporting Urbit:  Composing Jets
=================================

~lagrev-nocfep / Sigilante / Neal Davis

Objectives
----------

The goals of this tutorial are for you to be able to:

1. Read existing jet code.
2. Produce a jet matching a Hoon gate with a single argument.
3. Produce a more complex jet involving multiple values and floating-point arithmetic.

A fully-implemented version of the `++factorial` jet is available at [`natareo/urbit`](https://github.com/natareo/urbit/tree/master).


## Additional Resources

- [~timluc-miptev, “Jets”](https://github.com/timlucmiptev/docs-runtime/blob/master/jets.md) (recommended to start here first)
- [Tlon Corporation, “u3: Land of Nouns”](https://urbit.org/docs/vere/nouns/) (recommended as supplement to this document)
- [Tlon Corporation, “API overview by prefix”](https://urbit.org/docs/vere/api/) (recommended as supplement after this document)
- [Tlon Corporation, “Writing Jets”](https://urbit.org/docs/vere/jetting/) (largely superseded herein)


## Jet Walkthrough:  `++add`

Given a Hoon gate, how can a developer produce a matching C jet? Let us
illustrate the process using a simple `|%` core.
We assume the reader has achieved facility with both Hoon code and C
code.  This tutorial aims to communicate the practical process of
producing a jet, and many
[`u3` noun concepts](https://urbit.org/docs/vere/nouns/) are only
briefly discussed or alluded to.

To this end, we begin by examining the Hoon `++add` gate, which accepts two values in its sample.

The Hoon code for `++add` decrements one of these values and adds one to the other for each decrement until zero is reached.  This is because all atoms in Hoon are unsigned integers and Nock has no simple addition operation.  The source code for `++add` is located in `hoon.hoon`:

```hoon
|%
+|  %math
++  add
  ~/  %add
  ::  unsigned addition
  ::
  ::  a: augend
  ::  b: addend
  |=  [a=@ b=@]
  ::  sum
  ^-  @
  ?:  =(0 a)  b
  $(a (dec a), b +(b))
```

or in a more compact form (omitting the parent core and chapter label)

```hoon
++  add
  ~/  %add
  |=  [a=@ b=@]  ^-  @
  ?:  =(0 a)  b
  $(a (dec a), b +(b))
```

The jet hint `%add` allows Hoon to hint to the runtime that a jet _may_ exist.  By convention, the jet hint name matches the gate label.  Jets must be registered elsewhere in the runtime source code for the Vere binary to know where to connect the hint; we elide that discussion until we take a look at jet implementation below.

The following C code implements `++add` as a significantly faster operation including handling of >31-bit atoms.  It may be found in `urbit/pkg/urbit/jets/a/add.c`:

```c
u3_noun
u3qa_add(u3_atom a,
         u3_atom b)
{
  if ( _(u3a_is_cat(a)) && _(u3a_is_cat(b)) ) {
    c3_w c = a + b;

    return u3i_words(1, &c);
  }
  else if ( 0 == a ) {
    return u3k(b);
  }
  else {
    mpz_t a_mp, b_mp;

    u3r_mp(a_mp, a);
    u3r_mp(b_mp, b);

    mpz_add(a_mp, a_mp, b_mp);
    mpz_clear(b_mp);

    return u3i_mp(a_mp);
  }
}
u3_noun
u3wa_add(u3_noun cor)
{
  u3_noun a, b;

  if ( (c3n == u3r_mean(cor, u3x_sam_2, &a, u3x_sam_3, &b, 0)) ||
       (c3n == u3ud(a)) ||
       (c3n == u3ud(b) && a != 0) )
  {
    return u3m_bail(c3__exit);
  } else {
    return u3qa_add(a, b);
  }
}
```

The main entry point for a call into the function is `u3wa_add`.  `u3w` functions are translator functions which accept the entire sample as a `u3_noun` (or Nock noun).  `u3q` functions take custom combinations of nouns and atoms and generally correspond to unpacked samples.

`u3wa_add` defines two nouns `a` and `b` which will hold the unpacked arguments from the sample.  The sample elements are copied out by reference into `a` from sample address 2 (`u3x_sam_2`) and into `b` from sample address 3 (`u3x_sam_3`).  A couple of consistency checks are made; if these fail, `u3m_bail` yields a runtime error.  Else `u3qa_add` is invoked on the C-style arguments.

`u3qa_add` has the task of adding two Urbit atoms.  There is a catch, however!  An atom may be a _direct_ atom (meaning the value as an unsigned integer fits into 31 bits) or an _indirect_ atom (anything higher than that).  Direct atoms, called `cat`s, are indicated by the first bit being zero.

```
0ZZZ.ZZZZ.ZZZZ.ZZZZ.ZZZZ.ZZZZ.ZZZZ.ZZZZ
```

Any atom value which may be represented as $2^{31}-1 = 2.147.483.647$ or less is a direct atom.  The `Z` bits simply contain the value.

```hoon
> `@ub`2.147.483.647
0b111.1111.1111.1111.1111.1111.1111.1111
> `@ux`2.147.483.647
0x7fff.ffff
```

However, any atom with a value _greater_ than this (including many cords, floating-point values, etc.) is an indirect atom (or `dog`) marked with a prefixed bit of one.

```
11YX.XXXX.XXXX.XXXX.XXXX.XXXX.XXXX.XXXX
```

where bit 31 indicates indirectness, bit 30 is always set, and bit 29 (`Y`) indicates if the value is an atom or a cell.  An indirect atom contains a pointer into the loom from bits 0–28 (bits `X`).

What does this mean for `u3qa_add`?  It means that if the atoms are both direct atoms (`cat`s), the addition is straightforward and simply carried out in C.  When converted back into an atom, a helper function `u3i_words` deals with the possibility of overflow and the concomitant transformation to a `dog`.

```c
c3_w c = a + b;               # c3_w is a 32-bit C word.

return u3i_words(1, &c);
```

There's a second trivial case to handle one of the values being zero.  (It is unclear to the author of this tutorial why both cases as-zero are not being handled; the speed change may be too trivial to matter.)

Finally, the general case of adding the values at two loom addresses is dealt with.  This requires general pointer-based arithmetic with [GMP multi-precision integer operations](https://gnu.huihoo.org/gmp-3.1.1/html_chapter/gmp_4.html).

```c
mpz_t a_mp, b_mp;             # mpz_t is a GMP multi-precision integer type

u3r_mp(a_mp, a);              # read the atoms out of the loom into the MP type
u3r_mp(b_mp, b);

mpz_add(a_mp, a_mp, b_mp);    # carry out MP-correct addition
mpz_clear(b_mp);              # clear the now-unnecessary `b` value from memory

return u3i_mp(a_mp);          # write the value back into the loom and return it
```


The procedure to solve the problem in the C jet does not need to follow the same algorithm as the Hoon code.  (In fact, it is preferred to use native C implementations where possible to avoid memory leaks in the `u3` noun system.)

In general, jet code feels a bit heavy and formal.  Jet code may call other jet code, however, so much as with Hoon layers of complexity can be appropriately encapsulated.  Once you are used to the conventions of the u3 library, you will be in a good position to produce working and secure jet code.


## Jet Composition:  Integer `++factorial`

[Similar to how we encountered recursion way back in Hoon School](https://urbit.org/docs/hoon/hoon-school/recursion/) to talk about gate mechanics, let us implement a C jet of the `++factorial` example code.  We will call this library `trig` in a gesture to some subsequent functions you should implement as an exercise.  Create a file `lib/trig.hoon` with the following contents:

```hoon {caption="\\texttt{lib/trig.hoon}"}
~%  %trig  ..part  ~
|%
:: Factorial, $x!$
::
++  factorial
  ~/  %factorial
  |=  x=@ud  ^-  @ud
  =/  t=@ud  1
  |-  ^-  @rs
  ?:  =(x 0)  t
  ?:  =(x 1)  t
  $(x (sub x 1), t (mul t x))
--
```

We will create a generator `gen/trig.hoon` which will help us quickly check the library's behavior.

``` {caption="\\texttt{gen/trig.hoon}"}
/+  *trig
!:
:-  %say
|=  [[* eny=@uv *] [x=@rs n=@rs ~] ~]
::
~&  (factorial n)
~&  (absolute x)
~&  (exp x)
~&  (pow-n x n)
[%verb ~]
```

We will further define a few unit tests as checks on arm behavior in `tests/lib/trig.hoon`:

```hoon {caption="\\texttt{tests/lib/trig.hoon}"}
/+  *test, *trig
::
::::
  ::
|%
++  test-factorial  ^-  tang
  ;:  weld
    %+  expect-eq
      !>  1
      !>  (factorial 0)
    %+  expect-eq
      !>  1
      !>  (factorial 1)
    %+  expect-eq
      !>  120
      !>  (factorial 5)
    %+  expect-eq
      !>  720
      !>  (factorial 6)
  ==
--
```

### *Mise en place*

Jet development can require frequent production of new Urbit binaries
and , so let us pay attention to our working affordances. We will
primarily work in the fakezod on the two files just mentioned, and in
the `pkg/urbit` directory of the main Urbit repository, so we need a
development process that allows us to quickly access each of these, move
them into the appropriate location, and build necessary components. The
basic development cycle will look like this:

1.  Compose correct Hoon code.

2.  Hint the Hoon code.

3.  Register the jets in the Vere C code.

4.  Compose the jets.

5.  Compile and troubleshoot.

6.  Repeat as necessary.

You should consider using a terminal utility like `tmux` or `screen`
which allows you to work in several locations on your file system
simultaneously: one for file system operations (copying files in and out
of the `home` directory), one for running the fakezod, and one for
editing the files, or an IDE or text editor if preferred.

![A recommended screen layout.](https://raw.githubusercontent.com/sigilante/pixiesticks/master/layout.png)

To start off, you should obtain a clean up-to-date copy of the Urbit
repository, available on GitHub at
[`github.com/urbit/urbit`](https://github.com/urbit/urbit). Make a
working directory `~/tlon`. Create a new branch within the repo named
`trigjet`:

``` {.bash caption="Terminal 1 (Bash)" language="bash" style="nonumbers"}
cd
mkdir tlon
cd tlon
git clone https://github.com/urbit/urbit.git
cd urbit
git branch trigjet
```

Test your build process to produce a local executable binary of Vere:

``` {.bash caption="Terminal 1 (Bash)" language="bash" style="nonumbers"}
make
```

This invokes Nix to build the Urbit binary. Take note of where that
binary is located (typically in `/tmp` on your main file system) and
create a new fakezod using a downloaded pill.

``` {.bash caption="Terminal 1 (Bash)" language="bash" style="nonumbers"}
cd ..
wget https://bootstrap.urbit.org/urbit-v1.5.pill
<Nix build path>/bin/urbit -B urbit-v1.5.pill -F zod
```

Inside of that fakezod, sync `%clay` to Unix,

``` {.bash caption="Terminal 2 (Urbit)" language="bash" style="nonumbers"}
|mount %
```

Then copy the entire `home` desk out so that you can work with it and
copy it back in as necessary.

``` {.bash caption="Terminal 1 (Bash)" language="bash" style="nonumbers"}
cp -r zod/home .
```

Save the foregoing library code in `home/lib` and the generator code in
`home/gen`; also, don't forget the unit tests!  Whenever you work in your preferred editor, you should work on the `home` copies, then move them back into the fakezod and synchronize before execution.

``` {.bash caption="Terminal 1 (Bash)" language="bash" style="nonumbers"}
cp -r home zod
```

``` {.bash caption="Terminal 2 (Urbit)" language="bash" style="nonumbers"}
|commit %home
```

### Jet construction

Now that you have a developer cycle in place, let's examine what's
necessary to produce a jet. A jet is a C function which replicates the
behavior of a Hoon (Nock) gate. Jets have to be able to manipulate Urbit
quantities within the binary, which requires both the proper affordances
within the Hoon code (the interpreter hints) and support for
manipulating Urbit nouns (atoms and cells) within C.

Jet hints must provide a trail of symbols for the interpreter to know
how to match the Hoon arms to the corresponding C code. Think of these
as breadcrumbs. The [current jetting
documentation](https://urbit.org/docs/vere/jetting/) demonstrates a
three-deep jet; here we have a two-deep scenario. Specifically, we mark
the outermost arm with `~%` and an explicit reference to the Arvo core
(the parent of `part`). We mark the inner arms with `~/` because their
parent symbol can be determined from the context. The `@tas` token will
tell Vere which C code matches the arm. All symbols in the nesting
hierarchy must be included.

```hoon {caption="Jet hints in \\texttt{lib/trig.hoon}" style="nonumbers"}
~%  %trig  ..part  ~
|%
++  factorial
  ~/  %factorial
  |=  x=@ud  ^-  @ud
  ...
--
```

We also need to add appropriate handles for the C code. This consists of
several steps:

1.  Register the jet symbols and function names in `tree.c`.

2.  Declare function prototypes in headers `q.h` and `w.h`.

3.  Produce functions for compilation and linking in the
    `pkg/urbit/jets/e` directory.

The first two steps are fairly mechanical and straightforward.

**Register the jet symbols and function names.** A jet registration may
be carried out at in point in `tree.c`. The registration consists of
marking the core

``` {.objectivec language="C" caption="Additions to \\texttt{pkg/urbit/jets/tree.c}"}
/* Jet registration of ++factorial arm under trig */
static u3j_harm _140_hex__trig_factorial_a[] = {{".2", u3we_trig_factorial, c3y}, {}};
/* Associated hash */
static c3_c* _140_hex__trig_factorial_ha[] = {
  "903dbafb8e59427eced0b35379ad617c2eb6083a235075e9cdd9dd80e732efa4",
  0
};

static u3j_core _140_hex__trig_d[] =
  { { "factorial", 7, _140_hex__trig_factorial_a, 0, _140_hex__trig_factorial_ha },
  {}
  };
static c3_c* _140_hex__trig_ha[] = {
  "0bac9c3c43634bb86f6721bbcc444f69c83395f204ff69d3175f3821b1f679ba",
  0
};

/* Core registration by token for trig */
static u3j_core _140_hex_d[] =
{ /* ... pre-existing jet registrations ... */
  { "trig",   31, 0, _140_hex__trig_d, _140_hex__trig_ha  },
  {}
};
```

The numeric component of the title, `140`, indicates the Hoon Kelvin
version. Library jets of this nature are registered as `hex` jets,
meaning they live within the Arvo core. Other, more inner layers of
`%zuse` and `%lull` utilize `pen` and other three-letter jet tokens. The
core is conventionally included here, then either a `d` suffix for the
function association or a `ha` suffix for a jet hash.  (Jet hashes are a way of “signing” code.  They are not as of this writing actively used by the binary runtimes.)

The particular flavor of C mandated by the Vere kernel is quite
lapidary, particularly when shorthand functions (such as `u3z`) are
employed. In this code, we see the following `u3` elements:

1.  `c3_c`, the platform C 8-bit `char` type

2.  `c3y`, loobean true, `%.y` (similarly `c3n`, loobean false, `%.n`)

3.  `u3j_core`, C representation of Hoon/Nock cores

4.  `u3j_harm`, an actual C jet ("Hoon arm")

The numbers `7` and `31` refer to relative core addresses. In most
cases---unless you're building a particularly complicated jet or
modifying `%zuse` or `%lull`---you can follow the pattern laid out here.
`".2"` is a label for the axis in the core `[battery sample]`, so just
the battery. The text labels for the `|%` core and the arm are included
at their appropriate points. Finally, the jet function entry point
`u3we_trig_factorial` is registered.

For more information on u3, please check out the u3 summary below or the official documentation at [“u3: Land of Nouns”](https://urbit.org/docs/vere/nouns/).

**Declare function prototypes in headers.**

A `u3w` function is always the entry point for a jet. Every `u3w`
function accepts a `u3noun` (a Hoon/Nock noun), validates it, and
invokes the `u3q` function that implements the actual logic. The `u3q`
function needs to accept the same number of atoms as the defining arm
(since these same values will be extricated by the `u3w` function and
passed to it).

In this case, we have cited `u3we_trig_factorial` in `tree.c` and now
must declare both it and `u3qe_trig_factorial`:

In `w.h`:

``` {.objectivec language="C" caption="Additions to \\texttt{pkg/urbit/include/w.h}"}
u3_noun u3we_trig_factorial(u3_noun);
```

In `q.h`:

``` {.objectivec language="C" caption="Additions to \\texttt{pkg/urbit/include/q.h}"}
u3_noun u3qe_trig_factorial(u3_atom);
```

**Produce functions for compilation and linking.**

Given these function prototype declarations, all that remains is the
actual definition of the function. Both functions will live in their own
file; we find it the best convention to associate all arms of a core in
a single file. In this case, create a file `pkg/urbit/jets/e/trig.c` and
define all of your `trig` jets therein.  (Here we show `++factorial` only.)

As with `++add`, we have to worry about direct and indirect atoms when carrying out arithmetic operations, prompting the use of GMP `mpz` operations.

``` {.objectivec language="C" caption="\\texttt{pkg/urbit/jets/e/trig.c}"}
/* jets/e/trig.c
**
*/
#include "all.h"
#include <stdio.h>      // helpful for debugging, removable after development

/* factorial of @ud integer
*/
  u3_noun
  u3qe_trig_factorial(u3_atom a)  /* @ud */
  {
    fprintf(stderr, "u3qe_trig_factorial\n\r");  // DELETE THIS LINE LATER
    if (( 0 == a ) || ( 1 == a )) {
      return 1;
    }
    else if ( _(u3a_is_cat(a))) {
      c3_d c = ((c3_d) a) * ((c3_d) (a-1));

      return u3i_chubs(1, &c);
    }
    else {
      mpz_t a_mp, b_mp;

      u3r_mp(a_mp, a);
      mpz_sub(b_mp, a_mp, 1);
      u3_atom b = u3qe_trigrs_factorial(u3i_mp(b_mp));
      u3r_mp(b_mp, b);

      mpz_mul(a_mp, a_mp, b_mp);
      mpz_clear(b_mp);

      return u3i_mp(a_mp);
    }
  }

  u3_noun
  u3we_trig_factorial(u3_noun cor)
  {
    fprintf(stderr, "u3we_trig_factorial\n\r");  // DELETE THIS LINE LATER
    u3_noun a;

    if ( c3n == u3r_mean(cor, u3x_sam, &a, 0) ||
         c3n == u3ud(a) )
    {
      return u3m_bail(c3__exit);
    }
    else {
      return u3qe_trig_factorial(a);
    }
  }
```

This code merits ample discussion. Without focusing on the particular
types used, read through the logic and look for the skeleton of a
standard simple factorial algorithm.

`u3r` operations are used to extract Urbit-compatible types as C values.

`u3i` operations wrap C values back into Urbit-compatible types.


## u3 Overview

Before proceeding to compose a more complicated floating-point jet, we should step back and examine the zoo of u3 functions that jets use to formally structure atom access and manipulation.

#### `u3` functions.

`u3` defines a number of functions for extracting data from Urbit types
into C types for ready manipulation, then wrapping those same values
back up for Urbit to handle. These fall into several categories:

| Prefix | Mnemonic | Source File | Example of Function |
|--------|----------|-------------|---------------------|
| `u3a_` | Allocation | `allocate.c` | `u3a_malloc` |
| `u3e_` | Event (persistence) | `events.c` | `u3e_foul` |
| `u3h_` | Hash table | `hashtable.c` | `u3h_put` |
| `u3i_` | Imprisonment (noun construction) | `imprison.c` |  |
| `u3j_` | Jet control | `jets.c` | `u3j_boot` |
| `u3k_` | Jets (transfer semantics, C arguments) | `[a-g]/*.c` |  |
| `u3l_` | Logging | `log.c` | `u3l_log` |
| `u3m_` | System management | `manage.c` | `u3m_bail` |
| `u3n_` | Nock computation | `nock.c` | `u3nc` |
| `u3q_` | Jets (retain semantics, C arguments) | `[a-g]/*.c` |  |
| `u3r_` | Retrieval; returns on error | `retrieve.c` | `u3r_word` |
| `u3t_` | Profiling and tracing | `trace.c` | `u3t` |
| `u3v_` | Arvo operations | `vortex.c` | `u3v_reclaim` |
| `u3w_` | Jets (retain semantics, Nock core argument) | `[a-g]/*.c` |  |
| `u3x_` | Retrieval; crashes on error | `xtract.c` | `u3x_cell` |
| `u3z_` | Memoize | `zave.c` | `u3z_uniq` |

#### `u3` nouns.

The `u3` system allows you to extract Urbit nouns as atoms or cells.
Atoms may come in one of two forms: either they fit in 31 bits or less
of a 32-bit unsigned integer, or they require more space. In the former
case, you will use the singular functions such as `u3r_word` and
`u3a_word` to extract and store information. If the atom is larger than
this, however, you need to treat it a bit more like a C array, using the
plural functions `u3r_words` and `u3a_words`. (For native sizes larger
than 32 bits, such as double-precision floating-point numbers, replace
`word` with `chub` in these.)

An audit of the jet source code shows that the most commonly used `u3`
functions include:

1.  `u3a_free` frees memory allocated on the loom (Vere memory model).

2.  `u3a_malloc` allocates memory on the loom (Vere memory model).
    (Never use regular C `malloc` in `u3`.)

3.  `u3i_bytes` writes an array of bytes into an atom.

4.  `u3i_chub` is the ≥32-bit equivalent of `u3i_word`.

5.  `u3i_chubs` is the ≥32-bit equivalent of `u3i_words`.

6.  `u3i_word` writes a single 31-bit or smaller atom.

7.  `u3i_words` writes an array of 31-bit or smaller atoms.

8.  `u3m_bail` produces an error and crashes the process.

9.  `u3m_p` prints a message and a `u3` noun.

10. `u3r_at` retrieves data values stored at locations in the sample.

11. `u3r_byte` retrieves a byte from within an atom.

12. `u3r_bytes` retrieves multiple bytes from within an atom.

13. `u3r_cell` produces a cell `[a b]`.

14. `u3r_chub` is the \>32-bit equivalent of `u3r_word`.

15. `u3r_chubs` is the \>32-bit equivalent of `u3r_words`.

16. `u3r_mean` deconstructs a noun by axis address.

17. `u3r_met` reports the total size of an atom.

18. `u3r_trel` factors a noun into a three-element cell `[a b c]`.

19. `u3r_word` retrieves a value from an atom as a C `uint32_t`.

20. `u3r_words` is the multi-element (array) retriever like `u3r_word`.

#### u3 samples.

Defining jets which have a different sample size requires querying the correct nodes of the sample as binary tree:

    1.  1 argument → `u3x_sam`

    2.  2 arguments → `u3x_sam_2`, `u3x_sam_3`

    3.  3 arguments → `u3x_sam_2`, `u3x_sam_6`, `u3x_sam_7`

    4.  4 arguments → `u3x_sam_2`, `u3x_sam_6`, `u3x_sam_14`, `u3x_sam_15`

    5.  5 arguments → `u3x_sam_2`, `u3x_sam_6`, `u3x_sam_14`, `u3x_sam_30`,
        `u3x_sam_31`

    6.  6 arguments → `u3x_sam_2`, `u3x_sam_6`, `u3x_sam_14`, `u3x_sam_30`,
        `u3x_sam_62`, `u3x_sam_63`

A more complex argument structure requires grabbing other entries; e.g.,

```hoon
|=  [u=@lms [ia=@ud ib=@ud] [ja=@ud jb=@ud]]
```

requires

```c
u3x_sam_2, u3x_sam_12, u3x_sam_13, u3x_sam_14, u3x_sam_15
```


## Jet Composition:  Floating-Point `++factorial`

Let us examine jet composition using a more complicated floating-point operation.  The Urbit runtime uses [SoftFloat](http://www.jhauser.us/arithmetic/SoftFloat-3/doc/SoftFloat.html) to provide a reference software implementation of floating-point mathematics.  This is slower than hardware FP but more portable.

This library `lib/trig-rs.hoon` provides a few transcendental functions useful in many mathematical calculations. The `~%` "sigcen" rune registers the jets (with explicit arguments, necessary at the highest level of inclusion). The `~/` "sigfas" rune indicates which arms will be jetted.

```hoon {caption="\\texttt{lib/trig-rs.hoon}"}
::  Transcendental functions library, compatible with @rs
::
=/  tau  .6.28318530717
=/  pi   .3.14159265358
=/  e    .2.718281828
=/  rtol  .1e-5
~%  %trig  ..part  ~
|%
:: Factorial, $x!$
::
++  factorial
  ~/  %factorial
  |=  x=@rs  ^-  @rs
  =/  t=@rs  .1
  |-  ^-  @rs
  ?:  =(x .0)  t
  ?:  =(x .1)  t
  $(x (sub:rs x .1), t (mul:rs t x))
:: Absolute value, $|x|$
::
++  absolute
  |=  x=@rs  ^-  @rs
  ?:  (gth:rs x .0)
    x
  (sub:rs .0 x)
:: Exponential function, $\exp(x)$
::
++  exp
  ~/  %exp
  |=  x=@rs  ^-  @rs
  =/  rtol  .1e-5
  =/  p   .1
  =/  po  .-1
  =/  i   .1
  |-  ^-  @rs
  ?:  (lth:rs (absolute (sub:rs po p)) rtol)
    p
  $(i (add:rs i .1), p (add:rs p (div:rs (pow-n x i) (factorial i))), po p)
:: Integer power, $x^n$
::
++  pow-n
  ~/  %pow-n
  |=  [x=@rs n=@rs]  ^-  @rs
  ?:  =(n .0)  .1
  =/  p  x
  |-  ^-  @rs
  ?:  (lth:rs n .2)
    p
  ::~&  [n p]
  $(n (sub:rs n .1), p (mul:rs p x))
--
```

We will create a generator which will pull the arms and slam each gate
such that we can assess the library's behavior. Later on we will create
unit tests to validate the behavior of both the unjetted and jetted
code.

``` {caption="\\texttt{gen/trig-rs.hoon}"}
/+  *trig-rs
!:
:-  %say
|=  [[* eny=@uv *] [x=@rs n=@rs ~] ~]
::
~&  (factorial n)
~&  (absolute x)
~&  (exp x)
~&  (pow-n x n)
[%verb ~]
```

We will further define a few unit tests as checks on arm behavior:

``` {caption="\\texttt{tests/lib/trig-rs.hoon}"}
/+  *test, *trig-rs
::
::::
  ::
|%
++  test-factorial  ^-  tang
  ;:  weld
    %+  expect-eq
      !>  .1
      !>  (factorial .0)
    %+  expect-eq
      !>  .1
      !>  (factorial .1)
    %+  expect-eq
      !>  .120
      !>  (factorial .5)
    %+  expect-eq
      !>  .720
      !>  (factorial .6)
  ==
--
```



##### Jet composition.

As before, the jet hints must provide a trail of symbols for the interpreter to know how to match the Hoon arms to the corresponding C code.

``` {caption="Jet hints in \\texttt{lib/trig-rs.hoon}" style="nonumbers"}
~%  %trig-rs  ..part  ~
|%
++  factorial
  ~/  %factorial
  |=  x=@rs  ^-  @rs
  ...
++  exp
  ~/  %exp
  |=  x=@rs  ^-  @rs
  ...
++  pow-n
  ~/  %pow-n
  |=  [x=@rs n=@rs]  ^-  @rs
  ...
--
```

As before:

1.  Register the jet symbols and function names in `tree.c`.

2.  Declare function prototypes in headers `q.h` and `w.h`.

3.  Produce functions for compilation and linking in the
    `pkg/urbit/jets/e` directory.

**Register the jet symbols and function names.** A jet registration may
be carried out at in point in `tree.c`. The registration consists of
marking the core

``` {.objectivec language="C" caption="Additions to \\texttt{pkg/urbit/jets/tree.c}"}
/* Jet registration of ++factorial arm under trig-rs */
static u3j_harm _140_hex__trigrs_factorial_a[] = {{".2", u3we_trigrs_factorial, c3y}, {}};
/* Associated hash */
static c3_c* _140_hex__trigrs_factorial_ha[] = {
  "903dbafb8e59427eced0b35379ad617c2eb6083a235075e9cdd9dd80e732efa4",
  0
};

static u3j_core _140_hex__trigrs_d[] =
  { { "factorial", 7, _140_hex__trigrs_factorial_a, 0, _140_hex__trigrs_factorial_ha },
  {}
  };
static c3_c* _140_hex__trigrs_ha[] = {
  "0bac9c3c43634bb86f6721bbcc444f69c83395f204ff69d3175f3821b1f679ba",
  0
};

/* Core registration by token for trigrs */
static u3j_core _140_hex_d[] =
{ /* ... pre-existing jet registrations ... */
  { "trig-rs",   31, 0, _140_hex__trigrs_d, _140_hex__trigrs_ha  },
  {}
};
```

**Declare function prototypes in headers.**

We must declare `u3we_trigrs_factorial`  and `u3qe_trigrs_factorial`:

In `w.h`:

``` {.objectivec language="C" caption="Additions to \\texttt{pkg/urbit/include/w.h}"}
u3_noun u3we_trigrs_factorial(u3_noun);
```

In `q.h`:

``` {.objectivec language="C" caption="Additions to \\texttt{pkg/urbit/include/q.h}"}
u3_noun u3qe_trigrs_factorial(u3_atom);
```

**Produce functions for compilation and linking.**

Given these function prototype declarations, all that remains is the
actual definition of the function. Both functions will live in their own
file; we find it the best convention to associate all arms of a core in
a single file. In this case, create a file `pkg/urbit/jets/e/trig-rs.c` and
define all of your `trig-rs` jets therein.  (Here we show `++factorial` only.)

``` {.objectivec language="C" caption="\\texttt{pkg/urbit/jets/e/trig-rs.c}"}
/* jets/e/trig-rs.c
**
*/
#include "all.h"
#include <softfloat.h>  // necessary for working with software-defined floats
#include <stdio.h>      // helpful for debugging, removable after development
#include <math.h>       // provides library fabs() and ceil()

  union sing {
    float32_t s;    //struct containing v, uint_32
    c3_w c;         //uint_32
    float b;        //float_32, compiler-native, useful for debugging printfs
  };

/* ancillary functions
*/
  bool isclose(float a,
               float b)
  {
    float atol = 1e-6;
    return ((float)fabs(a - b) <= atol);
  }

/* factorial of @rs single-precision floating-point value
*/
  u3_noun
  u3qe_trigrs_factorial(u3_atom u)  /* @rs */
  {
    fprintf(stderr, "u3qe_trigrs_factorial\n\r");  // DELETE THIS LINE LATER
    union sing a, b, c, e;
    u3_atom bb;
    a.c = u3r_word(0, u);  // extricate value from atom as 32-bit word

    if (ceil(a.b) != a.b) {
      // raise an error if the float has a nonzero fractional part
      return u3m_bail(c3__exit);
    }

    if (isclose(a.b, 0.0)) {
      a.b = (float)1.0;
      return u3i_words(1, &a.c);
    }
    else if (isclose(a.b, 1.0)) {
      a.b = (float)1.0;
      return u3i_words(1, &a.c);
    }
    else {
      // naive recursive algorithm
      b.b = a.b - 1.0;
      bb = u3i_words(1, &b.c);
      c.c = u3r_word(0, u3qe_trig_factorial(bb));
      e.s = f32_mul(a.s, c.s);
      u3m_p("result", u3i_words(1, &e.c));  // DELETE THIS LINE LATER
      return u3i_words(1, &e.c);
    }
  }

  u3_noun
  u3we_trigrs_factorial(u3_noun cor)
  {
    fprintf(stderr, "u3we_trigrs_factorial\n\r");  // DELETE THIS LINE LATER
    u3_noun a;

    if ( c3n == u3r_mean(cor, u3x_sam, &a, 0) ||
         c3n == u3ud(a) )
    {
      return u3m_bail(c3__exit);
    }
    else {
      return u3qe_trigrs_factorial(a);
    }
  }
```

This code deviates from the integer implementation in two ways:  because all `@rs` atoms are guaranteed to be 32-bits, we can assume that `c3_w` can always contain them; and we are using software-defined floating-point operations with SoftFloat.

We have made use of `u3r_word` to convert a
32-bit (really, 31-bit or smaller) Hoon atom (`@ud`) into a C `uint32_t`
or `c3_w`. This unsigned integer may be interpreted as a floating-point
value (similar to a cast to `@rs`) by the expedient of a C `union`,
which allows multiple interpretations of the same bit pattern of data;
in this case, as an unsigned integer, as a SoftFloat `struct`, and as a
C single-precision `float`.

`f32_mul` and its sisters (`f32_add`, `f64_mul`, `f128_div`, etc.) are
floating-point operations defined in software. These are not as
efficient as native hardware operations would be, but allow Urbit to
guarantee cross-platform compatibility of operations and not rely on
hardware-specific implementations. Currently all Urbit floating-point
operations involving `@r` values use SoftFloat.

### Compiling and using the jet.

With this one jet for `++factorial` in place, compile the jet and take
note of where Nix produces the binary.

``` {.bash caption="Terminal 1 (Bash)" language="bash" style="nonumbers"}
make
```

Copy the affected files back into the ship's pier:

``` {.bash caption="Terminal 1 (Bash)" language="bash" style="nonumbers"}
cp home/lib/trig-rs.hoon zod/home/lib
cp home/gen/trig-rs.hoon zod/home/gen
```

Restart your fakezod using the new Urbit binary and synchronize these to
the `%home` desk:

``` {.bash caption="Terminal 2 (Urbit)" language="bash" style="nonumbers"}
|commit %home
```

If all has gone well to this point, you are prepared to test the jet
using the `%say` generator from earlier:

``` {.bash caption="Terminal 2 (Urbit)" language="bash" style="nonumbers"}
+trig 5
```

Among the other output values, you should observe the `stderr` messages
emitted by the jet functions each time they are called.

3.  The type union remains necessary to easily convert the
    floating-point result back into an unsigned integer atom.

``` {.objectivec language="C" caption="\\texttt{pkg/urbit/jets/e/trig.c}"}
/* integer power of @rs single-precision floating-point value
*/
  u3_noun
  u3qe_trigrs_pow_n(u3_atom x,  /* @rs */
                  u3_atom n)  /* @rs */
  {
    fprintf(stderr, "u3qe_trig_pow_n\n\r");
    union sing x_, n_, f_;
    x_.c = u3r_word(0, x);  // extricate value from atom as 32-bit word
    n_.c = u3r_word(0, n);

    f_.b = (float)pow(x_, n_);

    return u3i_words(1, &f_.c);
  }

  u3_noun
  u3w_trigrs_pow_n(u3_noun cor)
  {
    fprintf(stderr, "u3w_trig_pow_n\n\r");
    u3_noun a, b;

    if ( c3n == u3r_mean(cor, u3x_sam_2, &a,
                              u3x_sam_3, &b, 0) ||
         c3n == u3ud(a) || c3n == u3ud(b) )
    {
      return u3m_bail(c3__exit);
    }
    else {
      return u3q_trigrs_pow_n(a, b);
    }
  }
```

We leave the implementation of the other jets to the reader as an
exercise. (Please do not skip this: the exercise will both solidify your
understanding and raise new important situational questions.)

Again, the C jet code need not follow the same logic as the Hoon source
    code; in this case, we simply use the built-in `math.h` `pow`
    function. (We could—arguably should—have used SoftFloat
    functions.)


## Pills.

A *pill* is a Nock "binary blob", really a parsed Hoon abstract syntax
tree. Pills are used to bypass the bootstrapping procedure for a new
ship, and are particularly helpful when jetting code in `hoon.hoon`,
`%zuse`, `%lull`, or the main Arvo vanes.

The legacy jetting tutorial contains [specific
instructions](https://urbit.org/docs/vere/jetting/) on how to compile a
pill and work with a more rapid setup for jetting with fakezods.

You don't strictly need to use pills in producing jets, but it can speed up your development cycle.


## Unit testing.

All nontrivial code should be thoroughly tested to ensure software
quality. To rigorously verify the jet's behavior and performance, we
will combine live testing in a single Urbit session, comparative
behavior between a reference Urbit binary and our modified binary, and
unit testing.

1.  Live spot checks rely on you modifying the generator `trig-rs.hoon` and
    observing whether the jet works as expected.

    When producing a library, one may use the `-build-file` thread to
    build and load a library core through a face. Two fakezods can be
    operated side-by-side in order to verify consistency between the
    Hoon and C code.

    ``` {.bash language="bash" style="nonumbers"}
    > =trig-rs -build-file %/lib/trig-rs/hoon
    > (exp:trig-rs .5)
    ```

2.  Comparison to the reference Urbit binary can be done with a second
    fakezod and the same Hoon library and generator.

3.  Unit tests rely on using the `-test` thread and will be covered
    subsequently.

    ```
    > -test %/tests/lib/trig-rs ~
    ```


## Et cetera.

We omit from the current discussion a few salient points:

1.  Reference counting with transfer and retain semantics. (For
    everything the new developer does outside of real Hoon shovel work,
    one will use transfer semantics.)

2.  The structure of memory: the loom, with outer and inner roads.

3.  Many details of C-side atom declaration and manipulation from the
    `u3` library.

We commend to the reader the exercise of selecting particular
Hoon-language library functions provided with the system, such as
[`++cut`](https://github.com/urbit/urbit/blob/ceed4b78d068d7cb70350b3cd04e7525df1c7e2d/pkg/arvo/sys/hoon.hoon#L854), locating the corresponding jet code

- [`tree.c`](https://github.com/urbit/urbit/blob/cd400dfa69059e211dc88f4ce5d53479b9da7542/pkg/urbit/jets/tree.c#L1575)
- [`w.h`](https://github.com/urbit/urbit/blob/cd400dfa69059e211dc88f4ce5d53479b9da7542/pkg/urbit/include/jets/w.h#L53)
- [`q.h`](https://github.com/urbit/urbit/blob/cd400dfa69059e211dc88f4ce5d53479b9da7542/pkg/urbit/include/jets/q.h#L51)
- [`cut.c`](https://github.com/urbit/urbit/blob/master/pkg/urbit/jets/c/cut.c)

and learning in detail how
particular operations are realized in `u3` C. Note in particular that
jets do not need to follow the same solution algorithm and logic as the
Hoon code; they merely need to reliably produce the same result. At the
current time, the feasibility of automatic jet verification is an open
research question.

A jet can also solve certain cases efficiently but leave others to the Hoon implementation.  Per ~master-morzod:

> Jets can be partial; a `u3w_*` jet interface function takes the entire core as one noun argument and returns a `u3_weak` result. If the return value is `u3_none` [distinct from `u3_nul`, `~`], the core is evaluated; otherwise the resulting noun is produced in place of the nock.
