# String Interner

It's very expensive to use strings as symbols, so we safe all symbols in one place
and give out an unique id.


```rust
use std::collections::HashMap;
use std::u32::MAX;
```


The strings are stored in a vector, while we hold an index from symbol to
it's position in the vector, which we treat as id.
```rust
pub struct Interner<'a> {
    storage: Vec<String>,
    lookup: HashMap<&'a str, usize>,
}

impl<'a> Interner<'a> {
    pub fn new() -> Interner<'a> {
        Interner {
            storage: Vec::new(),
            lookup: HashMap::new(),
        }
    }

    // panics on insertion of more than u32::MAX strings
    pub fn insert(&mut self, string: &'a str) -> u32 {
        if let Some(index) = self.lookup.get(string) {
            // since we only store trough this method and the max size is tested, we know
            // that we can safely cast to u32
            return *index as u32;
        }
        self.storage.push(string.to_string());
        let index = self.storage.len() - 1;
        self.lookup.insert(string, index);
        assert!(index <= MAX as usize);
        index as u32
    }

    pub fn get_index(&self, string: &str) -> Option<u32> {
        // cast is safe because insertion allows only indexes < u32:MAX
        self.lookup.get(string).map(|x| *x as u32)
    }

    pub fn get_str(&self, index: u32) -> Option<&str> {
        self.storage.get(index as usize).map(|x| x as &str)
    }
}

#[cfg(test)]
mod tests {
    use super::Interner;

    #[test]
    fn creation_and_insertion() {
        let mut interner = Interner::new();
        assert_eq!(interner.insert("foo"), 0);
        assert_eq!(interner.insert("bar"), 1);
        assert_eq!(interner.insert("baz"), 2);
        assert_eq!(interner.insert("foo"), 0);
    }

    #[test]
    fn lookup_from_string() {
        let mut interner = Interner::new();
        interner.insert("foo");
        interner.insert("bar");
        assert_eq!(interner.get_index("foo"), Some(0));
        assert!(interner.get_index("foo") != interner.get_index("bar"));
        assert_eq!(interner.get_index("foo"), interner.get_index("foo"));
        assert_eq!(interner.get_index("bar"), interner.get_index("bar"));
        assert_eq!(interner.get_index("unkown"), None);
    }

    #[test]
    fn lookup_from_index() {
        let mut interner = Interner::new();
        interner.insert("foo");
        interner.insert("bar");
        assert!(interner.get_str(0) != interner.get_str(1));
        assert_eq!(interner.get_str(0), interner.get_str(0));
        assert_eq!(interner.get_str(12), None);
        assert_eq!(interner.get_str(0), Some("foo"));
    }
}
```
