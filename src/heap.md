
```rust
use value::{Key, Value};
use std::collections::HashMap;
use std::u32;

use std::slice::Iter;

#[derive(Debug, Clone)]
pub struct Heap {
    pub data: Vec<Node>,
    pub freelist: Vec<u32>,
    pub classes: Vec<HashMap<Key, usize>>,
}

const HEAP_START_SIZE: usize = 0x400; // random value for now

impl Heap {
    pub fn new() -> Heap {
        Heap::with_capacity(HEAP_START_SIZE)
    }

    pub fn with_capacity(capacity: usize) -> Heap {
        Heap {
            data: vec![Node::empty(); capacity],
            freelist: (0..capacity).map(|i| i as u32).collect::<Vec<_>>(),
            classes: Vec::new(),
        }
    }

    pub fn create_node(&mut self, values: Vec<Value>, class: ClassRef) -> u32 {
        if let Some(addr) = self.freelist.pop() {
            // todo: actual look for the class, just do lists right now
            let mut node = Node::empty();
            node.inner = values;
            self.data[addr as usize] = Node::empty();

            return addr;
        }
        // heap full
        unimplemented!();
    } 

}



#[derive(Debug, Clone)]
pub struct Node {
    gc_data: u32,
    class: ClassRef,
    // possible optimization is to store shorter Nodes as a array of Values
    inner: Vec<Value>,
}

impl Node {
    pub fn empty() -> Node {
        Node {
            gc_data: 0,
            class: ClassRef::new_list(),
            inner: Vec::new(),
        }
    }
}

#[derive(Debug, Copy, Clone)]
pub struct ClassRef(u32);

impl ClassRef {
    pub fn new_list() -> ClassRef {
        ClassRef(u32::MAX)
    }
}


#[cfg(test)]
mod tests {
    use super::Node;

    #[test]
    fn sizes() {
        use std::mem::size_of;
        assert_eq!(size_of::<Node>(), 32);
    }
}
```
