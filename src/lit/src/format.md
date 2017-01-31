Overview:

Path search and rendering is table-driven. That means that there is a
table of rules that defines how we find paths, and then other
components of those same rules dictate how to render those paths we
have found.




We have various imports.

First, there is the compass directions abstraction.

```rust
use directions::{self, Direction, ToDirections};
```

Then there are various structures that the parsing code is responsible
for constructing, all of which come from the `grammar` module.

```rust
#[allow(unused_imports)]
use grammar::{self, Rendering, CharSet};
#[allow(unused_imports)]
use grammar::Dirs as GrammarDirs;
use grammar::Dir as GrammarDir;
use grammar::Match as GrammarMatch;
use grammar::Rule as GrammarRule;
```

Now, when this code was first written, there was no external text
format for the path search and rendering rules. Instead, I built up
the rule structures directly. This meant that I tried to make them
somewhat readable (but even then it was too painful to write out the
specifications by hand, which led me to make some macro wrappers).

Anyway, `Match` is one of the record structures I made from the
outset.  It is actually quite similar to the `CharSet` structure from
the `grammar` module.

```rust
#[derive(Clone, Debug)]
pub enum Match {
    One(char),

    Chars(Vec<char>),

    // matches any non-blank character
    Any,
}

impl Match {
    pub fn matches(&self, c: char) -> bool {
        match *self {
            Match::One(m) => m == c,
            Match::Chars(ref v) => v.contains(&c),
            Match::Any => !c.is_whitespace(),
        }
    }
}
```

As mentioned above, I tried to make inputting the rules as painless as
possible. To support this, I wanted to make it convenient to use various
syntax for matching things: a single character if that's the only thing,
or a vector of characters, or a string literal...

To support this, I made a trait that can convert any of the above into
the `Match` structure.

```rust
pub trait IntoMatch { fn into_match(self) -> Match; }

impl IntoMatch for Match { fn into_match(self) -> Match { self } }
impl IntoMatch for char { fn into_match(self) -> Match { Match::One(self) } }
impl IntoMatch for Vec<char> { fn into_match(self) -> Match { Match::Chars(self) } }
impl<'a> IntoMatch for &'a str {
    fn into_match(self) -> Match { Match::Chars(self.chars().collect()) }
}
impl IntoMatch for String {
    fn into_match(self) -> Match { (self[..]).into_match() }
}
```

Next I had ways to specify in the rules about what the neighbors have
to look like.

```rust
#[derive(Clone, Debug)]
pub enum Neighbor<T> {
    /// no neighbor allowed (i.e. pattern for some end of the path).
    Blank,
    /// must match some non-blank neighbor
    Must(T),
    /// may match some non-blank neighbor, but also matches an end of the path.
    May(T),
}
```

An `Entry` is a path search and rendering rule. (It is called an
"Entry" because the whole set of rules is called a "Table"; its a
table made up of many entries.)

```rust
/// Each Entry describes how to render a character along a path,
/// based on the context in which it appears.
#[derive(Clone, Debug)]
pub struct Entry {
    entry_text: &'static str,

    /// `loop_start` is true if this entry represents a starting point
    /// for a closed polygon, e.g. a corner `+` is one such character.
    ///
    /// FIXME: there are impossible states (like Blank
    /// incoming/outgoing + loop_start true).  would be better to
    /// revise representation, e.g. with an enum {
    /// Edge(in,curr,out,is_loop),
    pub(crate) loop_start: bool,

    /// `Blank` if the first step in path; otherwise, the set of
    /// previous characters matched by this entry and direction from
    /// the previous step into `curr`.
    incoming: Neighbor<(Match, Vec<Direction>)>,

    /// The set of current characters matched by this entry.
    curr: Match,

    /// `Blank` if the last step in path; otherwise, direction from
    /// `curr` into next step, and the set of characters for next step
    /// matched by this entry.
    outgoing: Neighbor<(Vec<Direction>, Match)>,

    /// The template to use when rendering `curr`.
    template: String,

    /// attribute(s) that should be present on element if this pattern
    /// is matched along the path.
    include_attributes: Vec<(String, String)>,

    /// If `instrumented` is true, then during rendering we will
    /// announce a message (i.e. invoke a callback) every time this
    /// entry is considered, including the `entry_text`, the actual
    /// inputs under consideration, and the returned result.
    pub(crate) instrumented: bool,
}

impl Entry {
    pub fn incoming(&self) -> Neighbor<(Match, Vec<Direction>)> {
        self.incoming.clone()
    }

    pub fn outgoing(&self) -> Neighbor<(Vec<Direction>, Match)> {
        self.outgoing.clone()
    }
}
```

The `Announce` type is a simple callback. The intention is that when
instrumentation is supported, you pass the instrumentation result (a
string) to the announce-callback.

```rust
pub(crate) type Announce<'a> = &'a Fn(String);
```

The `Entry` type supports a slew of methods.

```rust
impl Entry {
```

Note that `fn matches_curr` is not the same as `fn matches` (it is
calling `matches` on `self.curr`, not `self`). This basically just
asks: does this rule even match the character we are currently
pointing at on the grid.

```rust
    pub(crate) fn matches_curr(&self, _a: Announce, curr: char) -> bool {
        let ret = self.curr.matches(curr);
        // if self.instrumented {
        //     a(format!("matches_curr({:?}) on {} => {:?}",
        //               curr, self.entry_text, ret));
        // }
        ret
    }
```

`fn matches_incoming` checks if the current rule applies given the
"incoming character" (which is coupled with the direction that matches
the trajectory from the incoming character to the current character).
(The incoming character is `None` when this is the first step on a
hypothetical path.)

```rust
    fn matches_incoming(&self, _a: Announce, incoming: Option<(char, Direction)>) -> bool {
        use self::Neighbor::{Blank, Must, May};
        let ret = match (&self.incoming, &incoming) {
            (&Blank, &Some(_)) | (&Must(..), &None) => false,
            (&Blank, &None) | (&May(..), &None) => true,
            (&Must((ref m, ref dirs)), &Some((c, d))) |
            (&May((ref m, ref dirs)), &Some((c, d))) =>
                if !dirs.contains(&d) {
                    false
                } else if !m.matches(c) {
                    false
                } else {
                    true
                },
        };
        // if self.instrumented {
        //     a(format!("matches_incoming({:?}) on {} => {:?}",
        //               incoming, self.entry_text, ret));
        // }
        ret
    }
```

`fn matches_outgoing` checks if the current rule applies given the
"outgoing character" (which is coupled with the direction that matches
the trajectory from the current character to the outgoing character).
(The outgoing character is `None` when this is the last step on a
hypothetical non-loop path.)

```rust
    fn matches_outgoing(&self, _a: Announce, outgoing: Option<(Direction, char)>) -> bool {
        use self::Neighbor::{Blank, Must, May};
        let ret = match (&self.outgoing, &outgoing) {
            (&Blank, &Some(_)) | (&Must(..), &None) => false,
            (&Blank, &None) | (&May(..), &None) => true,
            (&May((ref dirs, ref m)), &Some((d, c))) |
            (&Must((ref dirs, ref m)), &Some((d, c))) =>
                if !dirs.contains(&d) {
                    false
                } else if !m.matches(c) {
                    false
                } else {
                    true
                },
        };
        // if self.instrumented {
        //     a(format!("matches_outgoing({:?}) on {} => {:?}",
        //               outgoing, self.entry_text, ret));
        // }
        ret
    }
```

`fn matches` gathers all three of the above checks:

  1. does the incoming trajectory match,
  2. does the current character match, and
  3. does the outgoing trajectory match?

```rust
    pub(crate) fn matches(&self,
                          a: Announce,
                          incoming: Option<(char, Direction)>,
                          curr: char,
                          outgoing: Option<(Direction, char)>) -> bool {
        let ret = if !self.matches_incoming(a, incoming) { false
        } else if !self.curr.matches(curr) {
            false
        } else if !self.matches_outgoing(a, outgoing) {
            false
        } else {
            true
        };
        if self.instrumented {
            a(format!("matches({:?}, {:?}, {:?}) on {} => {:?}",
                      incoming, curr, outgoing, self.entry_text, ret));
        }
        ret
    }
```

`fn matches_start` handles matching for *just* the start rule, which 
dictates how non-loop paths start.

```rust
    pub(crate) fn matches_start(&self,
                                a: Announce,
                                curr: char,
                                outgoing: Option<(Direction, char)>) -> bool {
        use self::Neighbor::{Blank, Must, May};
        let ret = match &self.incoming {
            &Blank | &May(..) => {
                if !self.curr.matches(curr) {
                    false
                } else if !self.matches_outgoing(a, outgoing) {
                    false
                } else {
                    true
                }
            }
            &Must(..) => false,
        };
        if self.instrumented {
            a(format!("matches_start({:?}, {:?}) on {} => {:?}",
                      curr, outgoing, self.entry_text, ret));
        }
        ret
    }
```

`fn matches_end` handles matching for *just* the end rule, which 
dictates how non-loop paths end.

```rust
    pub(crate) fn matches_end(&self,
                              a: Announce,
                              incoming: Option<(char, Direction)>,
                              curr: char) -> bool {
        use self::Neighbor::{Blank, Must, May};
        let ret = if !self.matches_incoming(a, incoming) {
            false
        } else if !self.curr.matches(curr) {
            false
        } else {
            match &self.outgoing {
                &Blank | &May(..) => true,
                &Must(..) => false,
            }
        };
        if self.instrumented {
            a(format!("matches_end({:?}, {:?}) on {} => {:?}",
                      incoming, curr, self.entry_text, ret));
        }
        ret
    }
}
```

When we are searching for a path and we loop all the way around to the
start again, then we double-check that there exist two distinct
neighbors that work for the incoming and outgoing parts of the loop
rule.

```rust
impl Entry {
    pub(crate) fn corner_incoming(&self) -> (Match, Vec<Direction>) {
        match self.incoming {
            Neighbor::Blank => panic!("A loop_start cannot require blank neighbor"),
            Neighbor::May(ref t) | Neighbor::Must(ref t) => t.clone(),
        }
    }

    pub(crate) fn corner_outgoing(&self) -> (Vec<Direction>, Match) {
        match self.outgoing {
            Neighbor::Blank => panic!("A loop_start cannot require blank neighbor"),
            Neighbor::May(ref t) | Neighbor::Must(ref t) => t.clone(),
        }
    }
}
```

The `IntoAttributes` and `IntoEntry` traits are hacks similar to
`IntoMatch` to ease direct input of the rule specification, prior to
my adding a proper grammar to the system.

```rust
pub trait IntoAttributes { fn into_attributes(self) -> Vec<(String, String)>; }
impl IntoAttributes for () { fn into_attributes(self) -> Vec<(String, String)> { vec![] } }
impl IntoAttributes for [(&'static str, &'static str); 1] {
    fn into_attributes(self) -> Vec<(String, String)> {
        self.into_iter().map(|&(a,b)|(a.to_string(), b.to_string())).collect()
    }
}

pub trait IntoEntry { fn into_entry(self, text: &'static str) -> Entry; }

pub trait IntoCurr: IntoMatch { fn is_loop(&self) -> bool { false } }
```

I defined the nullary `All` and unary `May`/`Loop` structures again as
ways to ease direct entry of rule specifications.

```rust
/// Use `All` to match either the end of the path or any non-blank character.
pub struct All;

/// Use `May` to match either the end of the path or a particular match
pub struct May<C>(C);

/// Use `Loop` to match a corner for a closed polygon.
pub struct Loop<C>(C);
```

Here we implement the various traits above to specify how to convert
the specification as written in source code into a rule structure that
the system can interpret.


```rust
impl<C:IntoMatch> IntoMatch for Loop<C> {
    fn into_match(self) -> Match { self.0.into_match() }

}
impl<C:IntoMatch> IntoCurr for Loop<C> {
    fn is_loop(&self) -> bool { true }
}

impl IntoCurr for Match { }
impl IntoCurr for char { }
impl IntoCurr for Vec<char> { }
impl<'a> IntoCurr for &'a str { }
impl IntoCurr for String { }

impl<'a, C0, D0, C1, D1, C2> IntoEntry for (C0, D0, C1, D1, C2, &'a str) where
    C0: IntoMatch, D0: ToDirections, C1: IntoCurr, D1: ToDirections, C2: IntoMatch
{
    fn into_entry(self, text: &'static str) -> Entry {
        use self::Neighbor::{Must};
        Entry {
            instrumented: false,
            entry_text: text,
            loop_start: self.2.is_loop(),
            incoming: Must((self.0.into_match(), self.1.to_directions())),
            curr: self.2.into_match(),
            outgoing: Must((self.3.to_directions(), self.4.into_match())),
            template: self.5.to_string(),
            include_attributes: vec![],
        }
    }
}

impl<'a, C0, D0, C1, D1, C2, A> IntoEntry for (C0, D0, C1, D1, C2, &'a str, A) where
    C0: IntoMatch, D0: ToDirections, C1: IntoCurr, D1: ToDirections, C2: IntoMatch, A: IntoAttributes
{
    fn into_entry(self, text: &'static str) -> Entry {
        use self::Neighbor::{Must};
        Entry {
            instrumented: false,
            entry_text: text,
            loop_start: self.2.is_loop(),
            incoming: Must((self.0.into_match(), self.1.to_directions())),
            curr: self.2.into_match(),
            outgoing: Must((self.3.to_directions(), self.4.into_match())),
            template: self.5.to_string(),
            include_attributes: self.6.into_attributes(),
        }
    }
}

impl<'a, C0, D0, C1, D1, C2> IntoEntry for (May<(C0, D0)>, C1, D1, C2, &'a str) where
    C0: IntoMatch, D0: ToDirections, C1: IntoCurr, D1: ToDirections, C2: IntoMatch
{
    fn into_entry(self, text: &'static str) -> Entry {
        use self::Neighbor::{Must, May};
        Entry {
            instrumented: false,
            entry_text: text,
            loop_start: self.1.is_loop(),
            incoming: May((((self.0).0).0.into_match(),
                           ((self.0).0).1.to_directions())),
            curr: self.1.into_match(),
            outgoing: Must((self.2.to_directions(), self.3.into_match())),
            template: self.4.to_string(),
            include_attributes: vec![],
        }
    }
}

impl<'a, C0, D0, C1, D1, C2, A> IntoEntry for (May<(C0, D0)>, C1, D1, C2, &'a str, A) where
    C0: IntoMatch, D0: ToDirections, C1: IntoCurr, D1: ToDirections, C2: IntoMatch, A: IntoAttributes
{
    fn into_entry(self, text: &'static str) -> Entry {
        use self::Neighbor::{Must, May};
        Entry {
            instrumented: false,
            entry_text: text,
            loop_start: self.1.is_loop(),
            incoming: May((((self.0).0).0.into_match(),
                           ((self.0).0).1.to_directions())),
            curr: self.1.into_match(),
            outgoing: Must((self.2.to_directions(), self.3.into_match())),
            template: self.4.to_string(),
            include_attributes: self.5.into_attributes(),
        }
    }
}

impl<'a, C0, D0, C1, D1, C2> IntoEntry for (C0, D0, C1, May<(D1, C2)>, &'a str) where
    C0: IntoMatch, D0: ToDirections, C1: IntoCurr, D1: ToDirections, C2: IntoMatch
{
    fn into_entry(self, text: &'static str) -> Entry {
        use self::Neighbor::{Must, May};
        Entry {
            instrumented: false,
            entry_text: text,
            loop_start: self.2.is_loop(),
            incoming: Must((self.0.into_match(), self.1.to_directions())),
            curr: self.2.into_match(),
            outgoing: May((((self.3).0).0.to_directions(),
                           ((self.3).0).1.into_match())),
            template: self.4.to_string(),
            include_attributes: vec![],
        }
    }
}

impl<'a, C0, D0, C1, D1, C2, A> IntoEntry for (C0, D0, C1, May<(D1, C2)>, &'a str, A) where
    C0: IntoMatch, D0: ToDirections, C1: IntoCurr, D1: ToDirections, C2: IntoMatch, A: IntoAttributes
{
    fn into_entry(self, text: &'static str) -> Entry {
        use self::Neighbor::{Must, May};
        Entry {
            instrumented: false,
            entry_text: text,
            loop_start: self.2.is_loop(),
            incoming: Must((self.0.into_match(), self.1.to_directions())),
            curr: self.2.into_match(),
            outgoing: May((((self.3).0).0.to_directions(),
                           ((self.3).0).1.into_match())),
            template: self.4.to_string(),
            include_attributes: self.5.into_attributes(),
        }
    }
}

impl<'a, C0, D0, C1, D1, C2> IntoEntry for (May<(C0, D0)>, C1, May<(D1, C2)>, &'a str)
    where
    C0: IntoMatch, D0: ToDirections, C1: IntoCurr, D1: ToDirections, C2: IntoMatch
{
    fn into_entry(self, text: &'static str) -> Entry {
        use self::Neighbor::{May};
        Entry {
            instrumented: false,
            entry_text: text,
            loop_start: self.1.is_loop(),
            incoming: May((((self.0).0).0.into_match(), ((self.0).0).1.to_directions())),
            curr: self.1.into_match(),
            outgoing: May((((self.2).0).0.to_directions(), ((self.2).0).1.into_match())),
            template: self.3.to_string(),
            include_attributes: vec![],
        }
    }
}

impl<'a, C0, D0, C1, D1, C2, A> IntoEntry for (May<(C0, D0)>, C1, May<(D1, C2)>, &'a str, A)
    where
    C0: IntoMatch, D0: ToDirections, C1: IntoCurr, D1: ToDirections, C2: IntoMatch, A: IntoAttributes,
{
    fn into_entry(self, text: &'static str) -> Entry {
        use self::Neighbor::{May};
        Entry {
            instrumented: false,
            entry_text: text,
            loop_start: self.1.is_loop(),
            incoming: May((((self.0).0).0.into_match(), ((self.0).0).1.to_directions())),
            curr: self.1.into_match(),
            outgoing: May((((self.2).0).0.to_directions(), ((self.2).0).1.into_match())),
            template: self.3.to_string(),
            include_attributes: self.4.into_attributes(),
        }
    }
}
```

The `struct Start` and `struct Finis` were my old ways of directly
encoding the "start" and "end" rules that are now in the grammar.

```rust
pub struct Start;
pub struct Finis;

impl<'a, C1, D1, C2> IntoEntry for (Start, C1, D1, C2, &'a str)
    where C1: IntoMatch, D1: ToDirections, C2: IntoMatch
{
    fn into_entry(self, text: &'static str) -> Entry {
        use self::Neighbor::{Blank, Must};
        Entry {
            instrumented: false,
            entry_text: text,
            loop_start: false,
            incoming: Blank,
            curr: self.1.into_match(),
            outgoing: Must((self.2.to_directions(), self.3.into_match())),
            template: self.4.to_string(),
            include_attributes: vec![]
        }
    }
}

impl<'a, C1, D1, C2, A> IntoEntry for (Start, C1, D1, C2, &'a str, A)
    where C1: IntoMatch, D1: ToDirections, C2: IntoMatch, A: IntoAttributes,
{
    fn into_entry(self, text: &'static str) -> Entry {
        use self::Neighbor::{Blank, Must};
        Entry {
            instrumented: false,
            entry_text: text,
            loop_start: false,
            incoming: Blank,
            curr: self.1.into_match(),
            outgoing: Must((self.2.to_directions(), self.3.into_match())),
            template: self.4.to_string(),
            include_attributes: self.5.into_attributes(),
        }
    }
}

impl<'a, C0, D0, C1> IntoEntry for (C0, D0, C1, Finis, &'a str)
    where C0: IntoMatch, D0: ToDirections, C1: IntoMatch
{
    fn into_entry(self, text: &'static str) -> Entry {
        use self::Neighbor::{Blank, Must};
        Entry {
            instrumented: false,
            entry_text: text,
            loop_start: false,
            incoming: Must((self.0.into_match(), self.1.to_directions())),
            curr: self.2.into_match(),
            outgoing: Blank,
            template: self.4.to_string(),
            include_attributes: vec![],
        }
    }
}

impl<'a, C0, D0, C1, A> IntoEntry for (C0, D0, C1, Finis, &'a str, A)
    where C0: IntoMatch, D0: ToDirections, C1: IntoMatch, A: IntoAttributes
{
    fn into_entry(self, text: &'static str) -> Entry {
        use self::Neighbor::{Blank, Must};
        Entry {
            instrumented: false,
            entry_text: text,
            loop_start: false,
            incoming: Must((self.0.into_match(), self.1.to_directions())),
            curr: self.2.into_match(),
            outgoing: Blank,
            template: self.4.to_string(),
            include_attributes: self.5.into_attributes(),
        }
    }
}
```

Originally I had expected to be writing a number of rules where the
rendering was the same for *any* preceding or succeeding character.
Thus you have `IntoEntry` impls like these two, where its saying
`(All, C, All, "<path data>")` for some `C`.

However, it seems in practice that these forms were never actually
used in the final tables that I end up employing; probably because
such rules are actually far too broad.

TODO: Maybe I should just remove the `struct All` entirely.

```rust
impl<'a, C1> IntoEntry for (All, C1, All, &'a str) where
    C1: IntoCurr,
{
    fn into_entry(self, text: &'static str) -> Entry {
        use self::Neighbor::{May};
        Entry {
            instrumented: false,
            entry_text: text,
            loop_start: self.1.is_loop(),
            incoming: May((Match::Any, directions::Any.to_directions())),
            curr: self.1.into_match(),
            outgoing: May((directions::Any.to_directions(), Match::Any)),
            template: self.3.to_string(),
            include_attributes: vec![],
        }
    }
}

impl<'a, C1, A> IntoEntry for (All, C1, All, &'a str, A) where
    C1: IntoCurr, A: IntoAttributes
{
    fn into_entry(self, text: &'static str) -> Entry {
        use self::Neighbor::{May};
        Entry {
            instrumented: false,
            entry_text: text,
            loop_start: self.1.is_loop(),
            incoming: May((Match::Any, directions::Any.to_directions())),
            curr: self.1.into_match(),
            outgoing: May((directions::Any.to_directions(), Match::Any)),
            template: self.3.to_string(),
            include_attributes: self.4.into_attributes(),
        }
    }
}
```

The `struct Loud` was a bit of a trick to embed instrumentation code
into the rule structure. It is what would set the `.instrumented` bit
on an entry.

```rust
#[allow(dead_code)]
struct Loud<X>(X) where X: IntoEntry;
impl<X: IntoEntry> IntoEntry for Loud<X> {
    fn into_entry(self, text: &'static str) -> Entry {
        Entry { instrumented: true, ..self.0.into_entry(text) }
    }
}
```

At this point, we *do* have a real grammar and a real parser, so even though
there's all that preceding support code for directly writing entries in
Rust source, we also want to support the front-end syntax.

The plan: use the existing `IntoEntry` infrastructure, and just
add a new impl that converts a `grammar::Rule` into a local `Entry`.

(I'm not yet sure how I am going to supply the source text; I know
LalrPop has ways to observe left- and right-hand cursors tracking
the span of the input that was parsed. Maybe I will just need to use
that in tandem with tracking the original input string slice. Though
none of that will work until `entry_text` is generalized to `Cow<'static, str>`
so that I will have the option to clone the input into a String
when it has less than static lifetime.)

```rust
trait IntoNeighbor<T> { fn into_neighbor(self) -> Neighbor<T>; }

fn conv_dirs(gds: GrammarDirs) -> Vec<Direction> {
    macro_rules! match_dirs {
        ($dir: expr, $($id:ident),*) => {
            match $dir {
                $(
                    GrammarDir::$id => directions::Direction::$id,
                )*
            }
        }
    }
    gds.0.into_iter()
        .map(|d| match_dirs!(d, N, NE, E, SE, S, SW, W, NW))
        .collect()
}

impl IntoNeighbor<(Match, Vec<Direction>)> for (CharSet, GrammarDirs) {
    fn into_neighbor(self) -> Neighbor<(Match, Vec<Direction>)> {
        let (chars, dirs) = self;
        Neighbor::Must((chars.into_match(), conv_dirs(dirs)))
    }
}

impl IntoNeighbor<(Vec<Direction>, Match)> for (GrammarDirs, CharSet) {
    fn into_neighbor(self) -> Neighbor<(Vec<Direction>, Match)> {
        let (dirs, chars) = self;
        Neighbor::Must((conv_dirs(dirs), chars.into_match()))
    }
}

impl IntoMatch for CharSet {
    fn into_match(self) -> Match {
        match self {
            CharSet::Char(c) => Match::One(c),
            CharSet::String(s) => Match::Chars(s.chars().collect()),
            CharSet::Any => Match::Any,
        }
    }
}

impl IntoEntry for GrammarRule {
    fn into_entry(self, text: &'static str) -> Entry {
        let GrammarRule { pat, render } = self;
        let mut loop_start: bool = false;

        let incoming: Neighbor<(Match, Vec<Direction>)>;
        let curr: Match;
        let outgoing: Neighbor<(Vec<Direction>, Match)>;
        let template: String = render.0;
        // FIXME: not yet supported
        let include_attributes: Vec<(String, String)> = vec![];
        // FIXME: not yet supported
        let instrumented: bool = false;

        match pat {
            GrammarMatch::Loop(prev, p_dirs, loop_curr, n_dirs, next) => {
                loop_start = true;
                incoming = (prev, p_dirs).into_neighbor();
                curr = loop_curr.into_match();
                outgoing = (n_dirs, next).into_neighbor();
            }
            GrammarMatch::Step(prev, p_dirs, step_curr, n_dirs, next) => {
                incoming = (prev, p_dirs).into_neighbor();
                curr = step_curr.into_match();
                outgoing = (n_dirs, next).into_neighbor();
            }
            GrammarMatch::Start(start_curr, dirs, next) => {
                incoming = Neighbor::Blank;
                curr = start_curr.into_match();
                outgoing = (dirs, next).into_neighbor();
            }
            GrammarMatch::End(prev, dirs, end_curr) => {
                incoming = (prev, dirs).into_neighbor();
                curr = end_curr.into_match();
                outgoing = Neighbor::Blank;
            }
        }
        Entry {
            entry_text: text,
            loop_start: loop_start,
            incoming: incoming,
            curr: curr,
            outgoing: outgoing,
            template: template,
            include_attributes: include_attributes,
            instrumented: instrumented,
        }
    }
}
```

Of course we still need to test that the above integration works.
As a most basic sanity check, lets at least run the conversion on the
sample grammar I included in the `mod grammar`.

```rust
#[test]
fn convert_sample_grammar() {
    let rules = grammar::parse_rules(grammar::SAMPLE_GRAMMAR).unwrap();
    for rule in rules {
        rule.into_entry(""); // FIXME make `fn into_entry` more flexible about its input.
    }
}
```

The `entries!` macro is a convenience wrapper that effectively maps a
slew of calls to into_entry on the inputs. It is also stringifying
those same inputs, so that the instrumentation can actually report
which original rule (using the syntax that was originally employed to
write the rule) is being processed at the moment.

```rust
macro_rules! entries {
    ($($e:expr),* $(,)*) => { vec![$($e.into_entry(stringify!($e)),)*] }
}
```

Our rules are just a `Table` of entries.

```rust
#[allow(dead_code)]
#[derive(Clone)]
pub struct Table {
    pub(crate) entries: Vec<Entry>,
}
```

Here is a demo version of the `Table`. I believe I used this for proof
of concept work that ended up being embedded into the PADL paper
(after I hand transcribed it into the grammar as specified in that
paper). That was I was actually able to run the demo table on a sample
input (where the input and output were both included in the PADL paper
as well).

```rust
impl Table {
    pub fn demo() -> Self {
        use directions::{N, S, E, W, NE, SE, SW, NW};
        use directions::Any as AnyDir;
        Table {
            entries: entries! {
                ("|-/\\", AnyDir, Loop('+'), (N,S), "|", "M {C}"),
                ("|-/\\", AnyDir, Loop('+'), (E,W), "-", "M {C}"),

                (Start,   '-', (E,W), "-+", "M {RO} L {O}"),
                (Start,   '|', (N,S), "|+", "M {RO} L {O}"),
                (Start,   '+', AnyDir, Match::Any, "M {C}"),

                (Match::Any, (E,NE,N,NW,W), Loop('.'), (E,SE,S,SW,W), "-|\\/", "M {I} Q {C} {O}"),
                (Match::Any, (E,SE,S,SW,W), Loop('\''), (E,NE,N,NW,W), "-|\\/", "Q {C} {O}"),

                ("+-.'", (E, W), '-', May(((E, W), "-+.'>")), "L {O}"),
                ("+|.'", (N, S), '|', May(((N, S), "|+.'")), "L {O}"),

                (Match::Any, (E,NE,N,NW,W), '.', (E,SE,S,SW,W), "-|\\/", "Q {C} {O}"),
                (Match::Any, (E,SE,S,SW,W), '\'', (E,NE,N,NW,W), "-|\\/", "Q {C} {O}"),

                ("|-/\\>", AnyDir, '+', May(((N,S), "|")), "L {C}"),
                ("|-/\\>", AnyDir, '+', May(((E,W), "-")), "L {C}"),
                ("|-/\\>", AnyDir, '+', (NE,SW), "/", "L {C}"),
                ("|-/\\>", AnyDir, '+', (NW,SE), "\\", "L {C}"),

                (Match::Any, (NE, SW), '/', May(((NE, SW), "/+.'oO")), "L {O}"),
                (Match::Any, (NW, SE), '\\', May(((NW, SE), "\\+.'oO")), "L {O}"),

                ('-', E, '>', Finis, "L {C} l 3,0 m -3,-3 l 3,3 l -3,3 m 0,-3"),
                ('-', E, '>', E, '+', "L {E} m -2,0 l 4,0 m -4,-3 l 4,3 l -4,3 m 0,-3 m  4,0"),
                ('+', W, '>', W, '-', "M {E} m -2,0 l 4,0 m -4,-3 l 4,3 l -4,3 m 0,-3 m  4,0  M {E} L {C}"),

            }
        }
    }
}
```

Some basic operation on `Table`: What are the entries? Can you find a
relevant entry for a given predecessor, current character, and
successor?  And (as a separate special case), can you find a loop for
a given predecessor, current character, and successor.

```rust
impl Table {
    pub fn entries(&self) -> ::std::slice::Iter<Entry> { self.entries.iter() }

    pub(crate) fn find(&self,
                a: Announce,
                incoming: Option<(char, Direction)>,
                curr: char,
                outgoing: Option<(Direction, char)>) -> Option<(&str, &[(String, String)])> {
        for e in &self.entries {
            if !e.loop_start && e.matches(a, incoming, curr, outgoing) {
                return Some((&e.template, &e.include_attributes[..]));
            }
        }

        return None;
    }

    pub(crate) fn find_loop(&self,
                     a: Announce,
                     incoming: (char, Direction),
                     curr: char,
                     outgoing: (Direction, char)) -> Option<(&str, &[(String, String)])> {
        for e in &self.entries {
            if e.loop_start && e.matches(a, Some(incoming), curr, Some(outgoing)) {
                return Some((&e.template, &e.include_attributes[..]));
            }
        }

        return None;
    }
}
```

`struct Table` implements the `Default` trait. This is what dictates
the default set of rules for our clients, so its important that they
be impressive! (It is also important that they be predictable and
sane, goals which I arguably have not spent as much time on at the
moment.)

```rust
impl Default for Table {
    fn default() -> Self {
        // FIXME: switch to grammars_default once it is done.
        Self::original_default()
    }
}

impl Table {
    fn grammars_default() -> Self {
        use super::default_input::DEFAULT_INPUT;
        // FIXME: switch to dynamically parsing based on formal grammar.
        let mut entries = Vec::new();
        for (j, line) in DEFAULT_INPUT.lines().enumerate() {
            let j = j + 1; // report lines as 1-indexed.
            let rule = match grammar::parse_rules(line) {
                Err(parse_err) => {
                    panic!("Error parsing line {} `{}`: {:?}", j, line, parse_err);
                }
                Ok(rules) => {
                    match rules.len() {
                        0 => continue,
                        1 => rules[0].clone(),
                        x => panic!("more than one rule found on line {} `{}`", j, line),
                    }
                }
            };
            entries.push(rule.into_entry(line));
        }

        Table { entries: entries }
    }
}
```

Here is the original default table. It uses the crazy encoding
and also encodes the crazy stuff that I think is interesting
but probably not worth other people trying to make sense of.

```rust
impl Table {
    fn original_default() -> Self {
        use directions::{N, S, E, W, NE, SE, SW, NW};
        use directions::Any as AnyDir;
        use directions::NonNorth;
        use directions::NonSouth;
        const JOINTS: &'static str = ".'+oO";
        const LINES: &'static str = "-|/\\:=";
        const LINES_AND_JOINTS: &'static str = r"-|/\:=.'+oO";
        const STRICT_LINES_AND_JOINTS: &'static str = r"-|/\:=+";
        const ZER_SLOPE: &'static str = r"-=.'+oO><";
        const INF_SLOPE: &'static str = r"|:.'+oO^v";
        const POS_SLOPE: &'static str =  r"/.'+oO";
        const NEG_SLOPE: &'static str =  r"\.'+oO";
        Table {
            entries: entries! {
                (Start, "│", AnyDir, Match::Any, "M {N} L {S}"),
                (Start, "─", AnyDir, Match::Any, "M {W} L {E}"),
                (Start, "┌", AnyDir, Match::Any, "M {S} L {C} L {E}"),
                (Start, "┐", AnyDir, Match::Any, "M {W} L {C} L {S}"),
                (Start, "└", AnyDir, Match::Any, "M {N} L {C} L {E}"),
                (Start, "┘", AnyDir, Match::Any, "M {W} L {C} L {N}"),
                (Start, "├", AnyDir, Match::Any, "M {N} L {C} L {E} L {C} L {S}"),
                (Start, "┤", AnyDir, Match::Any, "M {N} L {C} L {W} L {C} L {S}"),
                (Start, "┬", AnyDir, Match::Any, "M {W} L {C} L {S} L {C} L {E}"),
                (Start, "┴", AnyDir, Match::Any, "M {W} L {C} L {N} L {C} L {E}"),
                (Start, "┼", AnyDir, Match::Any, "M {W} L {C} L {N} L {C} L {E} L {C} L {S}"),

                (Match::Any, AnyDir, "│", May((AnyDir, Match::Any)), "M {N} L {S}"),
                (Match::Any, AnyDir, "─", May((AnyDir, Match::Any)), "M {W} L {E}"),
                (Match::Any, AnyDir, "┌", May((AnyDir, Match::Any)), "M {S} L {C} L {E}"),
                (Match::Any, AnyDir, "┐", May((AnyDir, Match::Any)), "M {W} L {C} L {S}"),
                (Match::Any, AnyDir, "└", May((AnyDir, Match::Any)), "M {N} L {C} L {E}"),
                (Match::Any, AnyDir, "┘", May((AnyDir, Match::Any)), "M {W} L {C} L {N}"),
                (Match::Any, AnyDir, "├", May((AnyDir, Match::Any)), "M {N} L {C} L {E} L {C} L {S}"),
                (Match::Any, AnyDir, "┤", May((AnyDir, Match::Any)), "M {N} L {C} L {W} L {C} L {S}"),
                (Match::Any, AnyDir, "┬", May((AnyDir, Match::Any)), "M {W} L {C} L {S} L {C} L {E}"),
                (Match::Any, AnyDir, "┴", May((AnyDir, Match::Any)), "M {W} L {C} L {N} L {C} L {E}"),
                (Match::Any, AnyDir, "┼", May((AnyDir, Match::Any)), "M {W} L {C} L {N} L {C} L {E} L {C} L {S}"),

/*
                ('-', W, '-',  W, '-', ""),
                ('-', E, '-',  E, '-', ""),
                ('/', NE, '/', NE, '/', ""),
                ('/', SW, '/', SW, '/', ""),
                ('\\', NW, '\\', NW, '\\', ""),
                ('\\', SE, '\\', SE, '\\', ""),
                ('|', N, '|', N, '|', ""),
                ('|', S, '|', S, '|', ""),
*/
                (Start, '-', E, Match::Any, "M {W} L {E}"),

                (Start, '-', W, Match::Any, "M {E} L {W}"),
                (Start, '|', N, Match::Any, "M {S} L {N}"),
                (Start, '|', S, Match::Any, "M {N} L {S}"),
                (Start, '/', (SW,S,W), Match::Any, "M {NE} L {SW}"),
                (Start, '/', (NE,N,E), Match::Any, "M {SW} L {NE}"),
                (Start,'\\', (SE,S,E), Match::Any, "M {NW} L {SE}"),
                (Start,'\\', (NW,N,W), Match::Any, "M {SE} L {NW}"),
                (Start, '.', (W,E), ZER_SLOPE, "M {S} Q {C} {O}"),
                (Start, '.', E, POS_SLOPE, "M {S}"),
                (Start, '.', W, NEG_SLOPE, "M {S}"),
                (Start, "'", (W,E), ZER_SLOPE, "M {N} Q {C} {O}"),
                (Start, "'", E, NEG_SLOPE, "M {N}"),
                (Start, "'", W, POS_SLOPE, "M {N}"),
```

This block adds support for little circles along a line,
via the elliptical arc command `A`.

```rust
                (STRICT_LINES_AND_JOINTS, AnyDir, 'o', Finis,
                 "L {I/o} A 2,2 0 1 0 {RI/o}  A 2,2 0 0 0 {I/o} A 2,2 0 1 0 {RI/o}"),
```

Commented out code below is the same mistake I have
made elsewhere: there
are "natural" directions for characters like `/` and
`\`, which I have encoded in the SLOPE classes above.
But that means you cannot just match willy-nilly
against all LINES or LINES_AND_JOINTS in the
`next` component of the tuple; you need to put in
a stricter filter.

```rust
                // Loud((LINES_AND_JOINTS, AnyDir, 'o', AnyDir, LINES_AND_JOINTS,
                //      "L {I} A 4,4 360 1 0  {O}  A 4,4 180 0 0 {I} M {O}")),
                (LINES_AND_JOINTS, AnyDir, 'o', (W,E), r"-=+",
                      "L {I/o} A 2,2  0 1 0  {O/o}  A 2,2  0 0 0 {I/o} A 2,2 0 1 0 {O/o}"),
                (LINES_AND_JOINTS, AnyDir, 'o', (N,S), r"|:+",
                      "L {I/o} A 2,2  0 1 0  {O/o}  A 2,2  0 0 0 {I/o} A 2,2 0 1 0 {O/o}"),
                (LINES_AND_JOINTS, AnyDir, 'o', (NE,SW), r"/+",
                      "L {I/o} A 2,2  0 1 0  {O/o}  A 2,2  0 0 0 {I/o} A 2,2 0 1 0 {O/o}"),
                (LINES_AND_JOINTS, AnyDir, 'o', (NW,SE), r"\+",
                      "L {I/o} A 2,2  0 1 0  {O/o}  A 2,2  0 0 0 {I/o} A 2,2 0 1 0 {O/o}"),

                (LINES_AND_JOINTS, AnyDir, Loop('o'), (W,E), r"-=+",
                      "M {I} L {I/o} A 2,2 0 1 0  {O/o}  A 2,2 0 0 0 {I/o} A 2,2 0 1 0 {O/o}"),
                (LINES_AND_JOINTS, AnyDir, Loop('o'), (N,S), r"|:+",
                      "M {I} L {I/o} A 2,2 0 1 0  {O/o}  A 2,2 0 0 0 {I/o} A 2,2 0 1 0 {O/o}"),
                (LINES_AND_JOINTS, AnyDir, Loop('o'), (NE,SW), r"/+",
                      "M {I} L {I/o} A 2,2 0 1 0  {O/o}  A 2,2 0 0 0 {I/o} A 2,2 0 1 0 {O/o}"),
                (LINES_AND_JOINTS, AnyDir, Loop('o'), (NW,SE), r"\+",
                      "M {I} L {I/o} A 2,2 0 1 0  {O/o}  A 2,2 0 0 0 {I/o} A 2,2 0 1 0 {O/o}"),
```

These are similar to all the circle rules above, but the above rules will sometimes
yield small circles when joining pair of lines at a thin angle. Sometimes that's
the right thing, but when you want the circle sizes to be regular, you can use this
instead.

```rust
                (STRICT_LINES_AND_JOINTS, AnyDir, 'O', Finis,
                 "L {I/o} A 4,4 0 1 0 {RI/o}  A 4,4 0 0 0 {I/o} A 4,4 0 1 0 {RI/o}"),

                (LINES_AND_JOINTS, AnyDir, 'O', (W,E), r"-=+",
                      "L {I/o} A 4,4  0 1 0  {O/o}  A 4,4  0 0 0 {I/o} A 4,4 0 1 0 {O/o}"),
                (LINES_AND_JOINTS, AnyDir, 'O', (N,S), r"|:+",
                      "L {I/o} A 4,4  0 1 0  {O/o}  A 4,4  0 0 0 {I/o} A 4,4 0 1 0 {O/o}"),
                (LINES_AND_JOINTS, AnyDir, 'O', (NE,SW), r"/+",
                      "L {I/o} A 4,4  0 1 0  {O/o}  A 4,4  0 0 0 {I/o} A 4,4 0 1 0 {O/o}"),
                (LINES_AND_JOINTS, AnyDir, 'O', (NW,SE), r"\+",
                      "L {I/o} A 4,4  0 1 0  {O/o}  A 4,4  0 0 0 {I/o} A 4,4 0 1 0 {O/o}"),

                (LINES_AND_JOINTS, AnyDir, Loop('O'), (W,E), r"-=+",
                      "M {I} L {I/o} A 4,4 0 1 0  {O/o}  A 4,4 0 0 0 {I/o} A 4,4 0 1 0 {O/o}"),
                (LINES_AND_JOINTS, AnyDir, Loop('O'), (N,S), r"|:+",
                      "M {I} L {I/o} A 4,4 0 1 0  {O/o}  A 4,4 0 0 0 {I/o} A 4,4 0 1 0 {O/o}"),
                (LINES_AND_JOINTS, AnyDir, Loop('O'), (NE,SW), r"/+",
                      "M {I} L {I/o} A 4,4 0 1 0  {O/o}  A 4,4 0 0 0 {I/o} A 4,4 0 1 0 {O/o}"),
                (LINES_AND_JOINTS, AnyDir, Loop('O'), (NW,SE), r"\+",
                      "M {I} L {I/o} A 4,4 0 1 0  {O/o}  A 4,4 0 0 0 {I/o} A 4,4 0 1 0 {O/o}"),
```

This block is made of special cases for rendering horizontal
lines with curve characters in "interesting" ways.
They are not necessarily consistent nor do they exhibit symmetry,
but it seems better to do *something* rather than fall through
to default handlers that often show nothing special at all
along the path.
```rust
                (      r"\", E, '.', May((E, LINES)),   "Q {SW} {S}"),
                (      r"/", W, '.', May((W, LINES)),   "Q {SE} {S}"),
                (      r"/", E, "'", May((E, LINES)),   "Q {NW} {N}"),
                (      r"\", W, "'", May((W, LINES)),   "Q {NE} {N}"),
                (ZER_SLOPE, E, '.', May((E, LINES)),   "Q {C} {S}"),
                (ZER_SLOPE, W, '.', May((W, LINES)),   "Q {C} {S}"),
                (ZER_SLOPE, E, "'", May((E, LINES)),   "Q {C} {N}"),
                (ZER_SLOPE, W, "'", May((W, LINES)),   "Q {C} {N}"),
                (     ".'", E, '-', May((E, Match::Any)), "Q {W} {E}"),
                (     ".'", W, '-', May((W, Match::Any)), "Q {E} {W}"),
                (     ".",  E,  '/', May((E, r"'-\")), "Q {SW} {NE}"),
                (     ".",  W, r"\", May((W, r"'-/")), "Q {SE} {NW}"),
                (     "'",  E, r"\", May((E, r".-/")), "Q {NW} {SE}"),
                (     "'",  W, '/', May((W, r".-\")), "Q {NE} {SW}"),
```

These bits for `(` are another set of special cases for handling the
sides of a diamond when I don't want to use `+`.

By "diamond" I mean something like this:

```
  +    <-- `.` also acceptable here
 / \
(   )
 \ /
  +    <-- likewise `'` works here.
```

I don't want to use `+` here because I only want it to connect to the
diamond
and not to other neighboring lines (which is what `+` and other generic
joints would imply).

```rust
                // FIXME below cases seems like they are not always matching for some reason
                (     r"/", SW, '(', SE, r"\", "Q {C} {SE}"),
                (     r"/", NE, ')', NW, r"\", "Q {C} {NW}"),
                (     r"\", SE, ')', SW, r"/", "Q {C} {SW}"),
                (     r"\", NW, '(', NE, r"/", "Q {C} {NE}"),
                (Match::Any, AnyDir, r"/", SW, '(', "L {SW}"),
                (Match::Any, AnyDir, r"/", NE, ')', "L {NE}"),
                (Match::Any, AnyDir, r"\", SE, ')', "L {SE}"),
                (Match::Any, AnyDir, r"\", NW, '(', "L {NW}"),

                (Match::Any, E, '-', May((E, ZER_SLOPE)), "L {E}"),
                (Match::Any, W, '-', May((W, ZER_SLOPE)), "L {W}"),
                (Match::Any, N, '|', May((N, INF_SLOPE)), "L {N}"),
                (Match::Any, S, '|', May((S, INF_SLOPE)), "L {S}"),

                (Start, '=', E, ZER_SLOPE, "M {W} L {E}", [("stroke-dasharray", "5,2")]),
                (Start, '=', W, ZER_SLOPE, "M {E} L {W}", [("stroke-dasharray", "5,2")]),
                (Start, ':', N, INF_SLOPE, "M {S} L {N}", [("stroke-dasharray", "5,2")]),
                (Start, ':', S, INF_SLOPE, "M {N} L {S}", [("stroke-dasharray", "5,2")]),
                (Match::Any, E, '=', May((E, ZER_SLOPE)), "L {E}", [("stroke-dasharray", "5,2")]),
                (Match::Any, W, '=', May((W, ZER_SLOPE)), "L {W}", [("stroke-dasharray", "5,2")]),
                (Match::Any, N, ':', May((N, INF_SLOPE)), "L {N}", [("stroke-dasharray", "5,2")]),
                (Match::Any, S, ':', May((S, INF_SLOPE)), "L {S}", [("stroke-dasharray", "5,2")]),

                (Start, '+', AnyDir, Match::Any, "M {C}"),
                (Match::Any, AnyDir, '+', Finis, "L {C}"),
                // Below is riskier than I actually want to take
                // on right now.
                // (LINES_AND_JOINTS, AnyDir, '+', May((AnyDir, JOINTS)), "L {C}"),

                (Match::Any, NE, '/', May((NE, POS_SLOPE)), "L {NE}"),
                (Match::Any, SW, '/', May((SW, POS_SLOPE)), "L {SW}"),
                (Match::Any, SE, '\\', May((SE, NEG_SLOPE)), "L {SE}"),
                (Match::Any, NW, '\\', May((NW, NEG_SLOPE)), "L {NW}"),
                (Match::Any, NE, '/',  E, JOINTS, "L {NE}"),
                (Match::Any, SW, '/',  E, JOINTS, "L {NE}"),
                (Match::Any, SE, '\\', E, JOINTS, "L {SE}"),
                (Match::Any, NW, '\\', E, JOINTS, "L {SE}"),
                (Match::Any, NW, '\\', W, JOINTS, "L {NW}"),
                (Match::Any, SE, '\\', W, JOINTS, "L {NW}"),
                (Match::Any, NE, '/',  W, JOINTS, "L {SE}"),
                (Match::Any, SW, '/',  W, JOINTS, "L {SE}"),

                ('>', E, '+', May((AnyDir, LINES_AND_JOINTS)), "M {C}"),
                ('<', W, '+', May((AnyDir, LINES_AND_JOINTS)), "M {C}"),
                ('^', N, '+', May((AnyDir, LINES_AND_JOINTS)), "M {C}"),
                ('v', S, '+', May((AnyDir, LINES_AND_JOINTS)), "M {C}"),
                ("-=", (E, W), '+', May(((E, W), ZER_SLOPE)), "L {C}"),

                (LINES, AnyDir, Loop('+'), (N,S), INF_SLOPE, "M {C}"),
                (LINES, AnyDir, Loop('+'), (E,W), ZER_SLOPE, "M {C}"),
                (LINES, AnyDir, Loop('+'), (NE,SW), POS_SLOPE, "M {C}"),
                (LINES, AnyDir, Loop('+'), (NW,SE), NEG_SLOPE, "M {C}"),

                (LINES, AnyDir, '+', (N,S), INF_SLOPE, "L {C}"),
                (LINES, AnyDir, '+', (E,W), ZER_SLOPE, "L {C}"),
                (LINES, AnyDir, '+', (NE,SW), POS_SLOPE, "L {C}"),
                (LINES, AnyDir, '+', (NW,SE), NEG_SLOPE, "L {C}"),

                // The curves!  .-   .-  .-   .
                // part 1:      |   /     \  /| et cetera
                (Match::Any, NonSouth,      '.',  NonNorth, LINES, "Q {C} {O}"),
                (Match::Any, NonSouth, Loop('.'), NonNorth, LINES, "M {I} Q {C} {O}"),

                // curves       |   \/   /
                // part 2:      '-  '   '-   et cetera
                (Match::Any, NonNorth,      '\'',  NonSouth, LINES, "Q {C} {O}"),
                (Match::Any, NonNorth, Loop('\''), NonSouth, LINES, "M {I} Q {C} {O}"),

                // Arrow Heads!
                //
                // Perhaps more importantly, this code builds in an
                // assumption that each grid cell is 9x12 (or at least
                // WxH for W>9 and H>12).
                //
                // An assumption along these lines is perhaps
                // inevitable (I think its probably better to make
                // such an assumption up front rather than pretend
                // that the cell is a NxN square and thus have the
                // user be surprised when it turns out to be
                // non-square).
                //
                // But the question remains: is building in the
                // numbers 9 and 12 a good idea?  Or should they be
                // other numbers, like 3 and 4 (i.e. reduced form) or
                // 36 and 48 (which are both immediately divisible by
                // 2,3,4, and 6, which may be preferable to dealing in
                // fractions).
                //
                // horizontal arrow heads
                ('-', E, '>', Finis, "L {C} l 3,0 m -3,-3 l 3,3 l -3,3 m 0,-3"),
                (Start, '>', W, '-', "M {C} l 3,0 m -3,-3 l 3,3 l -3,3 m 0,-3"),
                ('-', W, '<', Finis, "L {C} l -3,0 m 3,-3 l -3,3 l 3,3 m 0,-3"),
                (Start, '<', E, '-', "M {C} l -3,0 m 3,-3 l -3,3 l 3,3 m 0,-3"),
                // vertical arrow heads
                (Start,  '^', S, '|', "M {C} l 0,-5 m -3,5 l 3,-5 l 3, 5 m -3,0"),
                (Start,  '^', S, ':', "M {C} l 0,-5 m -3,5 l 3,-5 l 3, 5 m -3,0", [("stroke-dasharray", "5,2")]),
                (Start,  'v', N, '|', "M {C} l 0,5 m -3,-5 l 3, 5 l 3,-5 m -3,0"),
                (Match::Any, S, 'v', Finis, "L {C} l 0,5 m -3,-5 l 3, 5 l 3,-5 m -3,0"),

                // arrow heads that join with other paths
                ('|', N, '^', N, '+', "L {N} l 0,-5 m -3,5 l 3,-5 l 3, 5 m -3,0 m 0,-5"),
                ('+', S, '^', S, '|', "M {N} l 0,-5 m -3,5 l 3,-5 l 3, 5 m -3,0 M {N} L {C}"),
                ('|', S, 'v', S, '+', "L {S} l 0,5 m -3,-5 l 3, 5 l 3,-5 m -3,0 m 0, 5"),
                ('+', N, 'v', N, '|', "L {S} l 0,5 m -3,-5 l 3, 5 l 3,-5 m -3,0 m 0, 5 M {S} L {C}"),
                ('-', E, '>', E, '+', "L {E} m -2,0 l 4,0 m -4,-3 l 4,3 l -4,3 m 0,-3 m  4,0"),
                ('+', W, '>', W, '-', "M {E} m -2,0 l 4,0 m -4,-3 l 4,3 l -4,3 m 0,-3 m  4,0  M {E} L {C}"),
                ('-', W, '<', W, '+', "L {W} m 2,0 l -4,0 m 4,-3 l -4,3 l 4,3 m 0,-3 m -4,0"),
                ('+', E, '<', E, '-', "M {W} m 2,0 l -4,0 m 4,-3 l -4,3 l 4,3 m 0,-3 m -4,0  M {W} L {C}"),

                (Start, '.', E, '-', "M {S} Q {C} {E}"),
                (Start, '.', W, '-', "M {S} Q {C} {W}"),
                (Start, '\'', E, '-', "M {N} Q {C} {E}"),
                (Start, '\'', W, '-', "M {N} Q {C} {W}"),
            }
        }
    }
}
```
