The `path` module represents the paths that we extract by following
adjacent characters on a grid. These paths can take the form of closed
polygons, or lines, which are all the fragments of polygons that we
could not close).

Paths can carry an optional identifier (denoted in the grid itself by
a markdown-style `[name]` marking, where the open square bracket
character `[` lies somewhere either:

 1. immediately to the right of a vertical line on the path:

    ```
     .-------.
     |       |
     |[ex1]  |
     |       |
     '-------'
    ```

    or,

 2. immediately beneath a horizontal line on the path

    ```
    -.
     |
     |
    -'
    [ex2]
    ```

If we find more than one `[ident]` marker that would qualify to
identify a path, we issue a warning and choose one of them
"arbitrarily." In practice one should follow the practice used
in [a2s][]: put the `[ident]` marker in the upper-left corner
of the object in question, as demonstrated here:

```
 .-------.
 |[ex3]  |
 |       |
 |       |
 '-------'
```

The identifier can be used to attach attributes to the
path, via an end-note notation of the form

o```
[ident]: { "name": "value", "name2": "value2" }
```

One can also attach these attributes to paths lacking
a symbolic identifier, using instead a "row,column"
format to identify the path:

```
[row,col]: { "name": "value", "name2": "value2" }
```

```rust
use grid::{Grid, Elem, Pt};

#[derive(PartialEq, Eq, Copy, Clone, Debug, Hash)]
pub enum Closed { Closed, Open }

#[derive(PartialEq, Eq, Clone, Debug, Hash)]
pub struct Path {
    pub (crate) steps: Vec<(Pt, char)>,
    pub (crate) closed: Closed,
    pub (crate) id: Option<String>,
    pub (crate) attrs: Option<Vec<(String, String)>>,
}

impl Path {
    /// If this is a closed path that follows the border of a
    /// rectangle with non-zero areaq, and the path has no interesting
    /// characters on the edges (i.e. they are all either `|` or `-`
    /// as appropirate), then returns an array holding its four
    /// corners.
    pub (crate) fn is_rectangular(&self) -> Option<[(Pt, char); 4]> {
        let steps = &self.steps;
        if steps.is_empty() { return None; }
        if self.closed == Closed::Open { return None; }

        let [mut ul, mut ur, mut bl, mut br]: [(Pt, char); 4] = [steps[0], steps[0], steps[0], steps[0]];

        // first, determine what the extremities are in the first place.
        for &(s, c) in steps {
            if s.col() <= ul.0.col() && s.row() <= ul.0.row() { ul = (s, c); }
            if s.col() <= bl.0.col() && s.row() >= bl.0.row() { bl = (s, c); }
            if s.col() >= ur.0.col() && s.row() <= ur.0.row() { ur = (s, c); }
            if s.col() >= br.0.col() && s.row() >= br.0.row() { br = (s, c); }
        }

        // next, ensure that we have a rectangle:
        if ul.0.row() != ur.0.row() { return None; }
        if bl.0.row() != br.0.row() { return None; }
        if ul.0.col() != bl.0.col() { return None; }
        if ur.0.col() != br.0.col() { return None; }

        // extract the defining boundary of the rectange.
        let lft_col = ul.0.col();
        let rgt_col = ur.0.col();
        let top_row = ul.0.row();
        let bot_row = br.0.row();

        // next, ensure that the area is non-zero.
        if lft_col >= rgt_col || top_row >= bot_row { return None; }

        // next, ensure edges outside the corners are uninteresting
        for &(s, c) in steps {
            // if its a corner, it can hold whatever content it likes
            if s == ul.0 || s == ur.0 || s == bl.0 || s == br.0 { continue; }

            // if its not on an edge, we don't have a rectangle
            if s.row() == top_row || s.row() == bot_row {
                if c != '-' && c != '+' { return None; }
            } else if s.col() == lft_col || s.col() == rgt_col {
                if c != '|' && c != '+' { return None; }
            } else {
                return None; 
            }
        }

        // at this point, we have ensured that the extracted corners
        // effectively describe the rectangle.
        return Some([ul, ur, br, bl]);
    }
}

#[derive(PartialEq, Eq, Debug)]
pub enum Remove {
    Leave,
    Clear,
    Mark(char),
}

impl Remove {
    fn cat(c: char) -> Remove {
        match c {
            '+' | '*' => Remove::Mark(c),
            _ => Remove::Clear,
        }
    }
}

impl Grid {
    fn remove_steps(&mut self, steps: &[(Pt, char)])
    {
        for &(pt, c) in steps {
            assert!(self[pt] == Elem::C(c) || self[pt] == Elem::Used(c));
        }
        for &(pt, c) in steps.iter() {
            match Remove::cat(c) {
                Remove::Clear => {
                    self.rows[pt.row_idx()][pt.col_idx()] = Elem::Clear;
                }
                Remove::Leave => {
                    // do nothing
                }
                Remove::Mark(c) => {
                    self.rows[pt.row_idx()][pt.col_idx()] = Elem::Used(c);
                }
            }
        }
    }

    pub fn remove_path(&mut self, p: &Path) {
        self.remove_steps(&p.steps);
    }
}
```
