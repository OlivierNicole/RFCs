# List and array comprehensions

The object of this proposal is to add list and array comprehensions to OCaml,
that is, a syntactic form for cleanly building lists and arrays based on
mathematical set-builder notation. It presents the design currently implemented
at Jane Street.

```ocaml
# (* Pythagorean triples with components from 1 to 10, no duplicate triples *)
  [ a, b, c for a = 1 to 10 for b = a to 10 for c = b to 10 when a * a + b * b = c * c ];;
- : (int * int * int) list = [(3, 4, 5); (6, 8, 10)]

# (* Let's describe some objects *)
  [| Printf.sprintf "a %s %s" adjective noun
     for noun in [| "light"; "pepper" |]
     and adjective in [| "red"; "yellow"; "green" |]
  |];;
- : string array =
[|"a red light"; "a yellow light"; "a green light"; "a red pepper";
  "a yellow pepper"; "a green pepper"|]

# (* Compute a list of reciprocals in increasing order *)
  [ 1. /. Float.of_int x for x = 5 downto 0 when x <> 0 ];;
- : float list = [0.2; 0.25; 0.333333333333333315; 0.5; 1.]

# (* Flatten a nested array *)
  let sentences =
    [| [| "hello"; "world" |]
    ;  [| "how"; "are"; "you"; "doing" |]
    ;  [| "please"; "enjoy"; "these"; "comprehensions" |]
    |]
  in
  [| word for sentence in sentences for word in sentence |];;
- : string array =
[|"hello"; "world"; "how"; "are"; "you"; "doing"; "please"; "enjoy"; "these";
  "comprehensions"|]

# (* We could use comprehensions to reimplement map... *)
  let map' f l = [ f x for x in l ];;
val map' : ('a -> 'b) -> 'a list -> 'b list = <fun>

# (* ...and filter *)
  let filter' f l = [| x for x in l when f x |];;
val filter' : ('a -> bool) -> 'a array -> 'a array = <fun>
```

## Syntax

The BNF grammar for comprehensions is:

```
expr +::=
  | `[` comprehension `]`
  | `[|` comprehension `|]`

comprehension ::=
  expr { comprehension_clause }+

comprehension_clause ::=
  | `for` comprehension_iterator { `and` comprehension_iterator }*
  | `when` expr

comprehension_iterator ::=
  | pattern `in` expr
  | value-name `=` expr ( `to` | `downto` ) expr
```

A comprehension consists in an expression, followed by one or more _clauses_,
each clause being either of the form `for ... [and ...]` comprising one or more
_iterators_, or of the form `when e`, where `e` is an expression that acts as a
filtering predicate.

## Semantics

Evaluating a comprehension happens in the following order:

1. Clauses are evaluated from left to right. Clauses further to the right will
   be evaluated once for each of the values from the surrounding iterators.
   * To evaluate a `for ... and ...` clause:
     1. Before performing any iteration, each iterator’s _source of values_ is
        evaluated exactly once from left to right, that is:
        * For a `PAT in SEQ` iterator, the expression `SEQ` is evaluated.
        * For a `VAR = START to/downto STOP iterator`, the expression `START` is
          evaluated before the expression `STOP`.
     2. Then, the iterators are iterated over, from left to right; the iterators
        further to the right “vary faster”. Each time a value is drawn from an
        iterator, the next iterator will be iterated over in its entirety before
        the “outer” (more leftwards) iterator moves onto the next value.
   * To evaluate a `when COND` clause, the expression `COND` is evaluated; if it
     evaluates to `false`, the current iteration is terminated and the innermost
     surrounding iterator advances to its next value. No clauses further to the
     right are evaluated, and nor is the body. At each iteration step, once of
     all the clauses have been evaluated (and all the `when` clauses have
     evaluated to `true`), the body is evaluated, and the result is the next
     element of the resulting sequence.

One may ask: **What is the difference between `for x in s1 for y in s2`, and
`for x in s1 and y in s2`?** The difference is that in the `for ... for ...`
case, iterators are allowed to refer to variables defined by iterators to their
left, which is not the case with the `for ... and ...` syntax. To express it in
categorical terms, `for ... for ...` behaves like nested bindings in the usual
list monad, whereas `for ... and ...` results in the “parallel” bindings that
one can perform in an applicative functor.

This difference also has performance implications, as we will in a moment.

## Implementation

List and array comprehensions are compiled completely differently; the former
are compiled in terms of some internal pre-provided functions, and the latter
are compiled as a series of nested loops.

### Compiling list comprehensions

List comprehensions are compiled in terms of reversed difference lists. A
difference list in general is a function from lists to lists; by “reversed”, we
mean that these lists are stored backwards, and need to be reversed at the end.
We make both these choices for the usual efficiency reasons: difference lists
allow for efficient concatenation; they can also be viewed as based on passing
around accumulators, which allows us to make our functions tail-recursive, at
the cost of building our lists up backwards. This is implemented in
the internal module `CamlinternalComprehension`:

```ocaml
type 'a rev_list =
  | Nil
  | Snoc of { init : 'a rev_list; last : 'a }

type 'a rev_dlist = 'a rev_list -> 'a rev_list
```

We then work exclusively in terms of `'a rev_dlist` values, reversing them into
a list only at the very end.

We desugar each iterator of a list comprehension into the application of a
tail-recursive higher-order function analogous to `concat_map`, whose type is of
the following form:

```
  ...iterator arguments... ->
  ('elt -> 'res rev_dlist) ->
  'res rev_dlist
```

Here, the `...iterator arguments...` define the sequence of values to be iterated
over (the `seq` of a `for pat in seq` iterator, or the `start` and `end` of a
`for x = start to/downto end` iterator); the function argument is then to be
called once for each item. What goes in the function? It will be the next
iterator, desugared in the same way. At any time, a `when` clause might intervene,
which is desugared into a conditional that gates entering the next phase of the
translation.

Eventually, we reach the body, which is placed into the body of the innermost
translated function; it produces the single-item reversed difference list
(alternatively, snocs its generated value onto the accumulator).
The whole thing is then passed into a reversal function, building the final
list.

For example, consider the following list comprehension:

```ocaml
[x+y for x = 1 to 3 when x <> 2 for y in [10*x; 100*x]]
(* = [11; 101; 33; 303] *)
```

This translates to the (Lambda equivalent of the) following:

```ocaml
(* Convert the result to a normal list *)
CamlinternalComprehension.rev_list_to_list (
  (* for x = 1 to 3 *)
  let start = 1 in
  let stop  = 3 in
  CamlinternalComprehension.rev_dlist_concat_iterate_up
    start stop
    (fun x acc_x ->
      (* when x <> 2 *)
      if x <> 2
      then
        (* for y in [10*x; 100*x] *)
        let iter_list = [10*x; 100*x] in
        CamlinternalComprehension.rev_dlist_concat_map
          iter_list
          (fun y acc_y ->
            (* The body: x+y *)
            Snoc { init = acc_y; last = x*y })
          acc_x
      else
        acc_x)
    Nil)
```

### Compiling array comprehensions

Array comprehensions are compiled completely differently from list
comprehensions: they turn into a nested series of loops that mutably update an
array. This is simple to say, but slightly tricky to do. One complexity is that
we want to apply an optimization to certain array comprehensions: if an array
comprehension contains exactly one clause, and it’s a `for ... and ...` clause,
then we can allocate an array of exactly the right size up front (instead of
having to grow the generated array dynamically, as we usually do). We call this
the _fixed-size array comprehension optimization_. We cannot do this with nested
`for`s, as the sizes of iterators further to the right could depend on the
values generated by those on the left; indeed, this is one of the reasons we
have `for ... and ...` instead of just allowing the user to nest fors.

In the general case, the structure is: we allocate an array and a mutable index
counter that starts at 0; each iterator becomes a loop; `when` clauses become an
`if` expression, same as with lists; and in the body, every time we generate an
array element, we set it and increment the index counter by one. If we’re not in
the fixed-size array case, then we also need the array to be growable. This is
the first source of extra complexity: we keep track of the array size, and if we
would ever exceed it, we double the size of the array. This means that at the
end, we have to use a subarray operation to cut it down to the right size.

In the fixed-size array case, we have to first compute the size of every
iterator and multiply them together; for both of these operations, we have to
check for overflow, in which case we fail. We also check to see if any of the
iterators would be empty (have size 0), in which case we can shortcut this whole
process and return an empty array. Once we do that, though, the loop body is
lighter as there’s no need to double the array size, and we don’t need to cut
the array down to size at the end.

To see some examples of what this translation looks like, consider the following
array comprehension, the same as the list comprehension we had before:

```ocaml
[| x+y for x = 1 to 3 when x <> 2 for y in [| 10*x; 100*x |] |]
(* = [| 11; 101; 33; 303 |] *)
```

This translates to (the Lambda equivalent of) the following:

```ocaml
(* Allocate the (resizable) array *)
let array_size = ref 8 in
let array      = ref [|0; 0; 0; 0; 0; 0; 0; 0|] in
(* Next element to be generated *)
let index = ref 0 in
(* for x = 1 to 3 *)
let start = 1 in
let stop  = 3 in
for x = start to stop do
  (* when x <> 2 *)
  if x <> 2 then
    (* for y in [|10*x; 100*x|] *)
    let iter_arr = [|10*x; 100*x|] in
    for iter_ix = 0 to Array.length iter_arr - 1 do
      let y = iter_arr.(iter_ix) in
      (* Resize the array if necessary *)
      if not (!index < !array_size) then begin
        array_size := 2 * !array_size;
        array := Array.append !array !array
      end;
      (* The body: x + y *)
      !array.(!index) <- x + y;
      index := !index + 1
    done
done;
(* Cut the array back down to size *)
Array.sub !array 0 !index
``

On the other hand, consider this array comprehension, which is subject to the fixed-size array comprehension optimization:

```ocaml
[|x*y for x = 1 to 3 and y = 10 downto 8|]
(* = [|10; 9; 8; 20; 18; 16; 30; 27; 24|] *)
```

This translates to (the Lambda equivalent of) the following rather different OCaml:

```ocaml
(* ... = 1 to 3 *)
let start_x = 1  in
let stop_x  = 3  in
(* ... = 10 downto 8 *)
let start_y = 10 in
let stop_y  = 8  in
(* Check if any iterators are empty *)
if start_x > stop_x || start_y < stop_y
then
  (* If so, return the empty array *)
  [||]
else
  (* Precompute the array size *)
  let array_size =
    (* Compute the size of the range [1 to 3], failing on overflow (the case
       where the range is correctly size 0 is handled by the emptiness check) *)
    let x_size =
      let range_size = (stop_x - start_x) + 1 in
      if range_size > 0
      then range_size
      else raise (Invalid_argument "integer overflow when precomputing \
                                    the size of an array comprehension")
    in
    (* Compute the size of the range [10 downto 8], failing on overflow (the
       case where the range is correctly size 0 is handled by the emptiness
       check) *)
    let y_size =
      let range_size = (start_y - stop_y) + 1 in
      if range_size > 0
      then range_size
      else raise (Invalid_argument "integer overflow when precomputing \
                                    the size of an array comprehension")
    in
    (* Multiplication that checks for overflow ([y_size] can't be [0] because we
       checked that above *)
    let product = x_size * y_size in
    if product / y_size = x_size
    then product
    else raise (Invalid_argument "integer overflow when precomputing \
                                  the size of an array comprehension")
  in
  (* Allocate the (nonresizable) array *)
  let array = Array.make array_size 0 in
  (* Next element to be generated *)
  let index = ref 0 in
  (* for x = 1 to 3 *)
  for x = start_x to stop_x do
    (* for y = 10 downto 8 *)
    for y = start_y downto stop_y do
      (* The body: x*y *)
      array.(!index) <- x*y;
      index := !index + 1
    done
  done;
  array
```

You can see that the loop body is tighter, but there’s more up-front size checking work to be done.
