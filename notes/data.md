# [#0001] - Data structures

|             |      |
| ----------- | ---- |
| **Authors** | Quil |
| **Last updated** | 2nd December 2018 |

- [ ] Discussion
- [ ] Implementation

## Summary

Programs primarily manipulate data, so a programming language needs good support not only for modelling processes, but also the data these processes manipulate.

For Purr, we're interested in modelling data structures that:

- Support evolution. That is, we must support defining and extending data structure definitions **without** breaking existing code. This rules out positional values and tagged unions (as commonly implemented).

- Support precise modelling. So we must support scalar types, aggregate types, and one-of types.

- Support encapsulation with fine-grained capabilities. It should be possible to provide a read-only access to a single field in a data structure to someone without giving them access to the whole data structure. Same for write-only accesses.

This document describes how Purr achieves these goals.

## References

- [Extensible records with scoped labels](http://www.cs.ioc.ee/tfp-icfp-gpce05/tfp-proc/21num.pdf)  
  -- Dan Leijen, 2005

- [Cap'n Proto language specification](https://capnproto.org/language.html)  
  -- Kenton Varda

- [Maybe not](https://www.youtube.com/watch?v=YR5WdGrpoug)  
  -- Rich Hickey, 2018

## Core language

The short story:

```
x in Variables
s in Symbols
v in Values

Label l := x | s

Values += {}

Expression e :=
  | { e with l_1: v_1, ..., l_n: v_n }      -- introduction
  | e.l                                     -- projection
  | symbol x                                -- introducing symbols
  | x | v
```

Records are a collection of labelled values. A label may be either a variable name (a regular name in a flat namespace), or a symbol (an unforgeable, unique identifier).

They may be constructed either with the special empty record value, or by extending an existing record with new labelled values.

Values may be accessed by projecting them through its label.

The following semantics describe this less informally:

```
E[{ { l1_1: v1_1, ..., l1_n: v1_n } with { l2_1: v2_1, ..., l2_m: v2_m }}]
--> { l1_1: v1_1, ..., l1_n: v1_n, l2_1: v2_1, ..., l2_m: v2_m }

E[{ l: v, ... }.l]
--> v

E[symbol x]
--> s, where `s` is an unique symbol with a textual description `x`
```

## Structuring data

There are two primary forms of data that applications manipulate:

- Scalar data: a single thing like a piece of text, or a number;
- Aggregate data: a collection of different kinds of data. For exemple, information about a person including their name, age, and address;

Applications also manipulate these pieces of information differently depending on the context. A context may expect data that may be a Person information or a Company information. It may expect data that may exist or not. We separate the _usage_ of data from the _modelling_ of data.

For modelling data, Purr provides records. Records are a collection of key/value pairs, such as:

```
let alice = { name: "Alice", age: 12 };
```

Records give us a way of bringing different pieces of information together, and passing them around as a single thing.

We may use this data in different ways. The common ones are by projecting a field from the record:

```
alice.name; // => "Alice"
```

Which works as long as the record defines such field. Or using pattern matching as a way of projecting many fields at once:

```
match alice with {
  case { name, age } -> // handle name and age here
}
```

Which works as long as the record defines all required fields.

## Records

As previously said, a record is a collection of labelled values. The key (or label) for these values can be a simple name (for example, `name` or `age`), as in our previous `alice` example. But it can also be an unique symbol. We'll see unique symbols in more details later when we cover encapsulation, capabilities, and security.

For now, let's focus on records.

We may construct a record by listing their labelled values:

```
let point2d = { x: 1, y: 2 };
let point3d = { x: 1, y: 2, z: 0 };
```

We may also construct a record by extending an existing record:

```
// this is equivalent to our previous `point3d` definition
let point3d' = { point2d with z: 0 };
```

In fact, our short `{ l: v, ... }` form is sugar for extending the empty record, so the following are strictly equivalent:

```
let p1 = { x: 1, y: 2 };
let p2 = { {} with x: 1, y: 2 };
```

Separating how data is defined and constructed from hwo data is used, and placing the data constraints on the latter allows data to evolve without breaking programs. As long as the data continues to provide the same labelled values with the same semantics, all users of the data will continue to work. The producers of the data are free to provide any additional labelled values they want, at any point in time.

### Extending records

Records can be extended with the `with` syntax. Conceptually `{ e with ... }` means "construct a record that has the same labelled values in `e`, and these labelled values". Like in extensible records, a new value with the same label as an existing labelled value overrides that value.

The literal interpretation of these semantics isn't efficient, however. It would require allocating memory for the entire set of labelled values, rather than just the ones that are different. Since records are effectively immutable, however, this structure can be safely shared between different records, reducing the costs of constructing new records (both in time and memory) to the differences between them.

While extending records, new labels may be added, or old labels may be assigned a new value, but labels cannot be removed from the record (although they may be assigned a value that suggests their non-existence, such as `Nothing`).

### Names and symbols

Labels may be a plain name or a symbol. We've seen labels as plain names so far:

```
let point2d = { x: 1, y: 2 };
```

In this expression, `x` and `y` are plain names. They're available for everyone to use, anytime, without needing to do anything special to access them. Their _names_ are already known.

But values may also be labelled by symbols. A symbol is a special form of label whose name is _unique_. You may construct a symbol like so:

```
symbol tag ("The type of a value.");
```

The symbol is referred by the variable `tag`, and has the (optional) description "The type of a value".

We may then use this symbol to label values and project them:

```
let point2d = { (tag): "point2d", x: 1, y: 2 };
point2d.(tag) --> "point2d";
```

The parenthesised form where a label is expected allows one to use an expression that resolves to a symbol. And that symbol is then used to label information in the record, or project information from it.

It's important to note that the **only** way of accessing and associating information with a symbol is by using a reference to it. Such reference is unforgeable: you can only get your hands in one if the code that created it passes a reference to you.

Further, labels that use symbols are not shown when inspecting an object, serialising it, etc. Unless the code has an explicit reference that allows such value to be exposed.

### Methods

As in prototype-based OOP, records double as objects. Here the labelled values play the same role as prototype-based OOP's slots, and function-valued labels are treated as methods. A simple method could be defined as follows:

```
let counter = (value) => {
  value: value,
  next: () => counter(value + 1)
};

let c1 = counter(0);
let c2 = c1.next();
let c3 = c2.next();
c3.value --> 2;
```

To support open recursion, Purr provides a special method syntax. So counter could be expressed like this:

```
let counter = {
  value: 0,
  member self.next() = { self with value: self.value + 1 }
};

counter.next().next() --> 2;
```

Here the parameter `self` refers to the receiver parameter of the method (like in F#, you can name it anything). New counters are constructed by extending the receiver with a new value, incrementing the receiver's value.

Projecting a method partially specifies the method's receiver, resulting in a regular function. You may think of methods as a sort-of curried function, where one of the parameters is specified on the left side of the dot. This way:

```
let next = counter.next;
next().next() --> 2;
```

The following table lists all special syntaxes that Purr defines for methods.

| **Syntax**             | **Description**                              | **Label**             |
| ---------------------- | -------------------------------------------- | --------------------- |
| `member self.x`        | A proxy for the projection of `x`            | `x`                   |
| `member self.f(...)`   | A regular method                             | `f`                   |
| `member self <- y`     | Updates the value of the record              | `op$update`           |
|                        |                                              |                       |
| **Collections**        |                                              |                       |
| `member self[k]`       | Retrieves the value indexed by `k` in `self` | `op$at`               |
| `member value in self` | Membership of `value` in `self`              | `op$in`               |
| `member self ++ that`  | Concatenation of collections                 | `op$concat`           |
|                        |                                              |                       |
| **Logical**            |                                              |                       |
| `member not self`      | Logical negation                             | `op$not`              |
| `member self and that` | Logical conjunction                          | `op$and`              |
| `member self or that`  | Logical disjunction                          | `op$or`               |
|                        |                                              |                       |
| **Relational**         |                                              |                       |
| `member self == that`  | Structural (value) equality                  | `op$equal`            |
| `member self /= that`  | Structural (value) inequality                | `op$not_equal`        |
| `member self > that`   | Greater than                                 | `op$greater_than`     |
| `member self >= that`  | Greater than or equal to                     | `op$greater_or_equal` |
| `member self < that`   | Less than                                    | `op$less_than`        |
| `member self <= that`  | Less than or equal to                        | `op$less_or_equal`    |
|                        |                                              |                       |
| **Arithmetic**         |                                              |                       |
| `member self + that`   | Arithmetic addition                          | `op$plus`             |
| `member self - that`   | Arithmetic subtraction                       | `op$minus`            |
| `member self * that`   | Arithmetic multiplication                    | `op$times`            |
| `member self / that`   | Arithmetic division                          | `op$divide`           |
| `member self ** that`  | Arithmetic exponentiation                    | `op$power`            |
|                        |                                              |                       |
| **Categories**         |                                              |                       |
| `member self >> that`  | Composition of morphisms (left-to-right)     | `op$compose_right`    |
| `member self << that`  | Composition of morphisms (right-to-left)     | `op$compose_left`     |

> **TODO**: operators do need some love right now. Will arithmetic ones be based on abstract algebra?

### The prototype pattern

The common prototype pattern seen in Self (and JavaScript), where common methods are extracted to a separate object, and new objects extend it through delegation, is also supported in Purr.

A point2d could be declared as:

```
let point2d = (x, y) => {
  x: x,
  y: y,

  member self.distance(aPoint) = match [self, aPoint] {
    case [{x1, y1}, { x2, y2 }] ->
      (((x2 - x1) ** 2) + ((y2 - y1) ** 2)).square_root();
  }
}
```

But this is inefficient, with the common method being allocated many times in memory. A memory/time-efficient way to express this would be:

```
let point2d_prototype = {
  member self.distance(aPoint) = match [self, aPoint] {
    case [{x1, y1}, { x2, y2 }] ->
      (((x2 - x1) ** 2) + ((y2 - y1) ** 2)).square_root();
  }
};

let point2d = (x, y) => { point2d_prototype with x: x, y: y };
```

## Data types

Records don't need any structuring, or prior definition. They can be constructed in any manner, as we only care about the set of labels they have. However, there's value in defining the shape of structures for reasoning about code and enforcing constraints. This, in turn, happens to make capabilities more practical as well, as we'll see shortly.

### Defining data types

A record data type may be introduced with the `data` form:

```
data Point2d = {
  public x;
  public y;

  member self.distance(aPoint) = match [self, aPoint] {
    case [{x1, y1}, { x2, y2 }] ->
      (((x2 - x1) ** 2) + ((y2 - y1) ** 2)).square_root();
  }
}
```

This is not a new primitive in the language, but rather a sugared form for the following:

```
define Point2d_prototype = {
  member self.distance(aPoint) = match [self, aPoint] {
    case [{x1, y1}, { x2, y2 }] ->
      (((x2 - x1) ** 2) + ((y2 - y1) ** 2)).square_root();
  }
};

define Point2d = {
  /* Transforms a regular record into a Point2d */
  member self.lift(point) =
    { Point2d_prototype with x: point.x, y: point.y };

  /* Used for pattern matching */
  member self.unapply(value, keys) {
    let x = if "x" in keys then { x: value.x } else {};
    let y = if "y" in keys then { y: value.y } else {};
    { ...x, ...y };
  }

  /* Used for contract matching */
  member value is self = match value {
    case { x, y } -> true;
    default -> false;
  }

  /* Transforms a Point2d into a regular record. */
  member self.to_public_record(value) = { x: value.x, y: value.y };
}
```

There's also a sugar for constructing these records:

```
let p = Point2d { x: 1, y: 2 };
// Equivalent to:
let p = Point2d.lift({ x: 1, y: 2 });
```

Now, so far most of these methods seem unecessary. Sure `Point2d { x: 1, y: 2 }` is slightly less to type than `{ Point2d_prototype with x: 1, y: 2 }`. But even with the questionable value of lift, the other methods don't even seem remotely useful -- we could just use projection and pattern matching as usual.

### Access control

Consider the following record:

```
let person = {
  username: "alice",
  password: "a bad idea"
};
```

We may want to give people access to the `name` of this person, but we don't just want to give them access to `password`--that'd be very insecure. And sure, we could construct public records with the information we want to pass around any time we want to, but we'd have to remember to do this--and that's how security issues happen.

A better idea would be to model password as a symbol, instead of a public name. This way only people who have a reference to the symbol could read it, regardless of who we pass the record to. That's a much better default, because now security issues can't happen unless we explicitly pass the key to accessing the private fields around as well (and that's a lot of work!):

```
symbol person_password ("The password of someone.");

let person = {
  username: "alice",
  (person_password): "a bad idea"
};
```

Everyone who gets their hands on `person` has access to `username`, but access to password is only granted to those who've got their hands on `person_password` as well. If we had more sensitive information, we could just make more symbols:

```
symbol person_password ("The password of someone.");
symbol social_number ("The person's social number.");

let person = {
  username: "alice",
  (person_password): "a bad idea",
  (social_number): "11111111"
};
```

But this gets out of hand pretty quickly, and if making things secure is difficult, people will just not put in the effort. Not a great look for a language that wants to be safe. So, instead, Purr provides sugar for this:

```
data Person {
  public username;
  private password, social_number;
}

let person = Person {
  username: "alice",
  password: "a bad idea",
  social_number: "11111111"
};
```

While the record used to construct this person doesn't use symbols, that's exactly what the `Person` structure does for you. Here's the desugared code:

```
symbol Person_password;
symbol Person_social_number;

define Person = {
  password: Person_password,
  social_number: Person_social_number,

  member self.lift(point) =
    match point {
      case { username, password, social_number }
        -> {
              username: username,
              (Person_password): password,
              (Person_social_number): social_number
            }
    };

  member self.unapply(value, keys) {
    let username = if "username" in keys then
                    { username: value.username }
                   else {};
    let password = if "password" in keys then
                    { password: value.(Person_password) }
                   else {};
    let social = if "social_number" in keys then
                  { social_number: value.(Person_social_number) }
                 else {};
    { ...username, ...password, ...social };
  }

  member value is self = match value {
    case { username: _, (Person_password): _, (Person_social_number): _ } -> true;
    default -> false;
  };

  /* Transforms a Point2d into a regular record. */
  member self.to_public_record(value) =
    {
      username: value.username,
      password: value.(Person_password),
      social_number: value.(Person_social_number)
    };
}

let person = Person.lift({
  username: "alice",
  password: "a bad idea",
  social_number: "11111111"
});

// Projecting values
let password = person.(Person.password);

// Or with pattern matching:
let Person { password } = person;
```

This makes writing safe code much more reasonable and practical. But we still have a problem: passing `Person` around gives people access to _everything_. Sometimes we want to give access to only a couple of things. Purr solves this with Views.

### Views

A view is basically a subset of a data type. We can make views by choosing what to expose (everything is hidden by default). Constructing a view is an expression, so you may or may not name them--this makes constructing views on-the-fly practical.

For example, if we want to give access to the social number of someone, we can give them a view:

```
define Social = Person view { expose social_number };
```

Now we can use the `Social` in the same way we'd use `Person`, but we only get access to the public fields and `social_number`:

```
// This is ok:
let Social { social_number } = person;

// This fails because `Social` doesn't give access to `password`:
let Social { password } = person;
```

It's important to note that views do not give people access to constructing records by default, only using existing ones. We can add a capability to construct the record as well:

```
define Social' = Person view {
  expose social_number;
  expose lift;
}
```

It's also possible to _hide_ features, although public labels and projection methods are exposed by default. Hiding public labels is kind of useless since they can be projected through the `.` operator anyway:

```
let OnlySocial = Person view {
  expose social_number;
  hide username;
  hide to_public_record;
}

// this fails
let OnlySocial { username } = person;

// but this succeeds
let username = person.username;

// this always fails
OnlySocial.to_public_record(person);

// but this succeeds (and results in the same record)
{
  username: person.username,
  social_number: person.(OnlySocial.social_number)
};
```

## Precise modelling

One of the goals of Purr is allow data to be modelled precisely. Part of this modelling is contextual--and that's solved with contracts. Part of this modelling is concerned about the structure, and that's something we need to solve in the facilities for constructing data in the language.

Remember we have three primary use cases for structure:

- Scalars, which are a single, stand-alone value (like numbers);
- Aggregates, which are a collection of pieces of data (like records);
- And choice types, which allow us to decide between one of many possibilities of data.

We've already got good tools for scalars and aggregates, but choice types are trickier. For example, consider the tree for a simple programming language. Each node in the tree has a different set of requirements, and we need to distinguish between these nodes to get any work done--because work depends exclusively on what a node "is".

In functional languages, this is often solved by tagged unions:

```
type Expr =
  | If of Expr * Expr * Expr
  | Let of string * Expr
  | Load of string
  | Num of int
```

This specifies that an Expression node may be one of: `If, with three expressions`, `Let, with a string and an expression`, `Load, with a string`, or `Num, with an int`. But it can _never_ be more than one of these at the same time. It has to be exactly one of them.

Tagged unions are naturally representable as a record with a tag label:

```
define If = (t, c, a) => { tag: "If", test: t, consequent: c, alternate: a };
define Let = (n, e) => { tag: "Let", name: n, value: e };
define Load = (n) => { tag: "Load", name: n };
define Num = (n) => { tag: "Num", value: n };
```

And we can use it with pattern matching as usual:

```
match node {
  case { tag: "If", test, consequent, alternate } ->
    // handle if

  case { tag: "Let", name, value } ->
    // handle let

  case { tag: "Load", name: n, value: e } ->
    // handle load

  case { tag: "Num", name: n } ->
    // handle num
}
```

But this is a bit cumbersome, and the flat-namespaced tags can cause conflicts. For example, if two different modules define a "tag" field of `"If"`, then the data in them could be very different from our expectations. Contracts help us at least catching this mistake, but they can't help us if we actually _want_ to match the different records and do different things for each of them!

So, instead, we have more sugar on top of symbols--which are unique names, perfect for this situation. An `union` introduces a set of records with the tags, ways of constructing them, and ways of pattern matching them. These follow the same patterns seen with capabilities previously.

With unions, we could declare this as:

```
union Expr {
  data If { test, consequent, alternate }
  data Let { name, value }
  data Load { name }
  data Num { value }
}
```

Which desugars to:

```
symbol Expr_If_tag ("Expr.If")
symbol Expr_Let_tag ("Expr.Let")
symbol Expr_Load_tag ("Expr.Load")
symbol Expr_Num_tag ("Expr.Num")

data Expr_If { tag = Expr_If_tag, test, consequent, alternate }
data Expr_Let { tag = Expr_Let_tag, name, value }
data Expr_Load { tag = Expr_Load_tag, name }
data Expr_Num { tag = Expr_Num, value }

define Expr = {
  If: Expr_If,
  Let: Expr_Let,
  Load: Expr_Load,
  Num: Expr_Num
}
```

(and some additional contract declarations).

You can use them in the same way you'd use any other record:

```
match expr {
  case Expr.If { test, consequent, alternate } -> ...
  case Expr.Let { name, value } -> ...
  case Expr.Load { name } -> ...
  case Expr.Num { value } -> ...
}
```

Capabilities still work in the same way described in the previous section, since they're not fundamentally different.

The two things this formulation of unions bring to the table are: ad-hoc unions, and extensible fields.

### Extensible fields

For the second one, the union syntax allows defining common fields for every record in the union. For example, AST nodes usually include information about the original location of the node in the source code for diagnostics. In most tagged union formulations you'd have to repeat this information everywhere, and if you decided to add it _after_ someone's started using it, all users would break.

With this formulation of records you just add an additional field that interested parties may want to start using, but they can safely ignore as well. Existing code continues working as always:

```
union Range {
  data Known { line, column, filename };
  data Unknown {}
}

union Expr {
  data If { test, consequent, alternate }
  data Let { name, value }
  data Load { name }
  data Num { value }

  public range = Range.Unknown;
}
```

This is strictly equivalent to the more troublesome work of adding the field to every data definition in the union, like so:

```
union Expr {
  data If { public range = Range.Unknown, test, consequent, alternate }
  data Let { public range = Range.Unknown, name, value }
  data Load { public range = Range.Unknown, name }
  data Num { public range = Range.Unknown, value }
}
```

### Ad-hoc unions

Because every data definition in an union uses an _universally-unique_ tag, it's possible to combine any record in an union. This makes things like nanopass' languages, where you can define new unions by refining existing ones, possible without any special support from the language.

For example, if we want to define a separate expression union that includes additional nodes and exclude existing ones, we can do so:

```
union ExprOpt' {
  data LoadLocal { name }
  data LoadFree { name }

  public range = Range.Unknown
}

define ExprOpt = {
  If: Expr.If,
  Let: Expr.Let,
  LoadLocal: ExprOpt'.LoadLocal,
  LoadFree: ExprOpt'.LoadFree,
  Num: Expr.Num
};
```

## Implementation in JS

JavaScript's objects are almost the same as the records described in this document, so an efficient translation is not difficult.

| **Purr concept** | **JS concept**              | **Description** |
| ------------------- | --------------------------- | --------------- |
| `{}`                | `Object.create(null)`       | Empty records   |
| `{ e with ... }`    | `Object.create(e, { ... })` | Extensions      |
| `x`                 | `"x"`                       | Public names    |
| `s`                 | `Symbol(s)`                 | Symbols         |
| `e.l`               | `$project(e, l)`            | Projections     |

Empty records map to the precise empty object in JS (which has to be `Object.create(null)` since `{}` comes with a lot of methods from `Object.prototype` that are not safe). Extending records uses JavaScript's object inheritance. Names and symbols map directly to the same concepts in JS.

Projections are a bit trickier because we guarantee that all methods being projected _will_ be partially applied to the receiver, and JavaScript does not do this by default. And we have to make sure that the label being accessed actually exists in the record (the program halts otherwise), JavaScript once again does a different thing here where it resolves the projection to `undefined` and keeps executing the (obviously wrong) program.

## Future work

Sharing behaviour in a coherent manner, and adding behaviour to existing records safely is not possible at the moment. But the system _can_ be extended to support this without breaking any of the existing guarantees or simplicity.

### Traits

Traits are pretty much parameterisable records, which can be mixed into other records to provide common behaviour, guaranteeing that there can be no conflicts between the behaviours that can't be resolved at the _inclusion_ site.

Symbols and records provide a good foundation for traits. For example:

```
union Ordering {
  data Less {}
  data Equal {}
  data Greater {}
}

trait Equality {
  require self.compare_to(that) -> Boolean;

  member self.equals(that) = self.(Equality.compare_to)(that) is Ordering.Equal;
  member self.not_equals(that) = not self.(Equality.equals)(that);
}
```

`compare_to`, `equals`, and `not_equals` are all symbols, so they can't possibly conflict with any record label. When including a trait in a record, the user must select which names they'll take at their new context (defaulting to the names specified in the declaration), and provide an implementation to all required members:

```
let integer = {
  value: 0,

  member self.compare_to(that) = Integer.compare(self, that);

  include Equality {
    member self.compare_to(that) = self.compare_to(that);
    member self.equals(that) as self == that;
    member self.not_equals(that) as self /= that;
  }
}
```

### Layers and contexts

Extending existing objects in a capability-secure language is tricky. You can't just mutate things because everyone that has a reference to the object would see that, and you might accidentally provide untrusted parts of the code with powerful capabilities.

Symbols once again provide a good way of solving this. No code can see values labelled with symbols unless they have a reference to the symbol itself. This means that extensions with a symbol are always "safe" in this sense.

We may also want to support multiple implementations of methods named in the same way, so we can choose which is the most appropriate implementation depending on where we're using it.

This all has to still be modular, since we can't allow code to depend on some global configuration (it wouldn't be safe, and would break most of the guarantees and requirements of Purr's module system -- order would become important, code'd have to be executed during instantiation, etc).

To solve this we can bring some ideas from Context-Oriented Programming, Implicits, (Modular) Type Classes, and Protocols.

We define requirements as interfaces:

```
interface Equality {
  require self == that;
  member self /= that = not (self == that);
}
```

Implementations need a context:

```
context Default;

let counter = {
  value: 0,

  interface Equality @ Default {
    member self == that = self.value == that.value;
  }
}
```

Regular uses of counter can't see the method:

```
counter == counter;
// [Panic] == is not defined for `counter`
```

So you need to view the value in a context:

```
(counter as Equality @ Default) == counter --> true;
```

Contexts are first-class values so they can be passed around, combined (conflicting implementations must be resolved), etc.

Existing values can be extended to implement interfaces in a context safely, since these values are only accessible if you can get a capability for a particular interface in a context:

```
extend SomeRecord {
  interface Equality @ Default { ... }
}
```

(There's a problem of how to make this late-bound, however...)
