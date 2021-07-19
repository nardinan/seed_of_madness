# 0.0.3_rc1

## bugs fixed

- fix the tokeniser eating away `}` in an interpolation when skip chars occur right before `}`.

## new features

#### labels

Support has been added for **labels**. Labels can be used to mark certain choices, when there is need to track or modify their state.
Presently, the only tracked state for a choice is it having been chosen at least once.

Labels can be prepended to both kinds of choices –– *advance* (`=>`, also called *generic*), and *acquire* (`+>`). What follows are the currently accepted syntaxes for choices:

**normal form:**

- *code* `=>` *code*

derived forms:

- [`[` *name* `]`] [`(` *string* `)`] *string* `=>` *code*
- *code* `=>` [`(` *string* `)`] *name*
- [`[` *name* `]`] [`(` *string* `)`] *string* `=>` [`(` *string* `)`] *name*

and

- [`[` *name* `]`] [`(` *string* `)`] *string* `+>` [`(` *string* `)`] *name*
- [`[` *name* `]`] [`(` *string* `)`] *string* `+>` [`(` *string* `)`] *set of names*

The label part is the sigil between `[` and `]` as first optional element of a non-code choice precondition (left hand).

Whenever a choice is labelled, the choice selection is tracked in a set named `taken` (as in "the roads not taken..."). The following behaviour occurs:

- in the *choices collecting phase*, before checking the choice precondition (left hand of arrow), check whether its label is in `taken`. If so, skip this choice.
- in the *choice selected phase*, before executing the choice postcondition (right hand of arrow), put the choice label in `taken`.

of course, `taken` can be freely manipulated, as for `visited` and `inventory` (by the way, crash **will** occur when their value types change: they **must** be sets).

with the introduction of labels, the acquire arrow (`+>`) becomes a simpler derived form, that is, this form:

```
("a terra c'è un coltello") "prendo il coltello" +> ("lo metti in tasca") :coltello
```

is transformed into this form:

```
[<an internally generated unique label>] ("a terra c'è un coltello") "prendo il coltello" => { message "lo metti in tasca", let inventory += [:coltello] }
```

The _internally generated unique label_ is currently built as `:<scene name>_<choice number>`. This might create confusion when previously saved states are used on game files that have been updated in the meantime by adding new acquire choices in between the existing ones. For this and other reasons, the internal format of generated acquire labels will probably change later on, so **don't count on the acquire choice labels being built this way**: if you want to keep track and modify the availability of an acquire choice, label it:

```
[:coltello_preso] ("a terra c'è un coltello") "prendo il coltello" +> ("lo metti in tasca") :coltello
```

Lastly, a tip: since a choice label is put in `taken` _before_ executing the postconditions, you can reinstate a choice right away if so needed:

```
[:coltello_preso] ("a terra c'è un coltello") "prendo il coltello" => {
	if rnd 100 < 20 then {
		message "ti scappa tra le mani, clumsy you!",
		let taken -= [:coltello_preso]
	} else {
		message "lo metti in tasca",
		let inventory += [:coltello]
	}
}
```

(remember that when a postcondition returns *void*, the scene stays the same)

---

#### tuples

Support has been added for **tuples**, that is, fixed-length collections of two or more values.

There is an ongoing debate (between me and I) over how much value tuples bring to this language. Not much, right now. But planned future features (deconstruction, pattern assignment, maps, scene models) will probably benefit from tuples.

My advice, for the time being, is to just **avoid them**. Be aware of the side effects of their existence (i.e. overload of operator `/`), though.

- tuple syntax: `(`*value*`,`*value* [`,` *value* ... ] `)`
  
  ```
  let player_map = ((:inventory, []), (:sex, :male), (:name, "baddo"))
  ```
  
  ```
  let player = ([], :male, "baddo")
  ```

- tuple selector: *tuple* `/` *number*
  
  ```
  <<What do I bring to you? This: {player_map/1/2}>>
  ```
  
  ```
  <<Name's {player_map/3/2}, I like {"firetrucks" when player_map/2/2 = :male else "dolls"}, and I'm retarded.>>
  ```
  
  ```
  # dumb named accessor mode
  let inventory = 1, let sex = 2, let name = 3,
  let msg_1 = <<What do I bring to you? This: {player/inventory}.>>,
  let msg_2 = <<Name's {player/name}, I like {"dolls" when player/sex = :male else "firetrucks"}, and I'm nonbinary.>>
  ```

- tuple assignment: `let` *tuple* `/` *number* [ `/` *number* ... ] `=` *expression*

  ```
  message "You feel somewhat disturbed.",
  let player/sex = :male when player/sex = :female else :female,
  let player_map/2/2 = player/sex
  ```

---
#### new operators

- `pick` *set*

  picks a random item from a set.
  
  `let inventory -= pick inventory`
  
  `let inventory += pick (satchel - [:cursed_stone])`

- `rnd` *number*

  picks a random real number in [0, N) (i.e. 0 <= X < N).
    
  ```
  	message "the temperature is slightly raising...",
  	let temperature += rnd 2.4
  ```

- *number* `%` *number*

  returns the integer module of two numbers by converting them to integers internally.
  
  ```
  message "I'll throw a d20...",
  let result = 1 + rnd 20 % 19
  ```
