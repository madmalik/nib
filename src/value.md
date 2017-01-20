# Value representation

This chapter is about the value representation. Nib is dynamically typed -
or, to be more precise - unityped. That means that every value has the same
type at compile time, but carries all necessary information about it's
content, so that it can be handled at runtime.

Another description would be that type information is always bound to the
value itself and not to the type, in contrast to languages like rust, which
can do both.

A natural and simple way to implement such a type in rust would be an `enum`.
But since a f64 is one of the possible types of our value, the enum needs
some space for a tag and everything will get aligned to word size, our value
will need 128 bits in memory, which is quite a lot when multiplied with all
uses of the primitive value type.

So a neat little hack called NaN-tagging is used.

f64 floating point values follow the IEEE 754 specification. The 64 bits
are segmented into three parts:

| sign  | exponent | mantissa |
| ------| -------- | -------- |
| 1 bit | 11 bits  | 52 bits  |

[TODO] NaN


so lets get to in, shall we?


Some useful constants we need throughout the implementation to construct
and deconstruct NaN-tagged values.

```rust
use std::mem::transmute;

const TAG_MASK: u64 = 0xFFFF_0000_0000_0000;
const NAN_BASE: u64 = 0x7ff8_0000_0000_0000;

const TAG_INT: u64 = 0x_____1_0000_0000_0000 | NAN_BASE;
const TAG_SYMBOL: u64 = 0x__2_0000_0000_0000 | NAN_BASE;
const TAG_REF: u64 = 0x_____3_0000_0000_0000 | NAN_BASE;
const TAG_FAST_REF: u64 = 0x4_0000_0000_0000 | NAN_BASE;
const TAG_BOOL: u64 = 0x____5_0000_0000_0000 | NAN_BASE;
const TAG_UNIT: u64 = 0x____6_0000_0000_0000 | NAN_BASE;
```


Just wrap an f64, all the other magic happens later.
```rust
#[derive(PartialEq, Debug, Copy, Clone)]
pub struct Value(f64);
```

This is an representation that is used for matching on the variants of `Value`.
Note that all variants that would not fit into 64 bits are just tags. This
is for performance reasons.
```rust
pub enum UnpackedValue {
    F64,
    Unit,
    Int(u32),
    Bool(bool),
    Ref,
    FastRef,
}
```


A `Key` is a strict subset of `Value`, used for variants that can be used for
indexing into collections.

```rust
#[derive(Debug, Copy, Clone, Hash, Eq, PartialEq)]
pub struct Key(u64);

impl Key {
    pub fn from_index(i: usize) -> Key {
        Key(i as u64 | TAG_INT)
    }
}
```


Now to everything that `Value` should be able to do. The first methods are
constructors from different rust types.
```rust
impl Value {
    pub fn from_u64(i: u64) -> Value {
        use std::u32::MAX;
        if i <= MAX as u64 {
            // fits into range
            Value::from_u32(i as u32)
        } else {
            Value::from_f64(i as f64)
        }
    }

    pub fn from_u32(i: u32) -> Value {
        unsafe {
            // we can safely bit-or it together because all upper bits that are used
            // in TAG_INT must be zero
            transmute(TAG_INT | i as u64)
        }
    }

    pub fn from_f64(f: f64) -> Value {
        // must be f64 or a non signaling NAN
        debug_assert!(!f.is_nan() || unsafe { transmute::<f64, u64>(f) == NAN_BASE });
        Value(f)
    }

    pub fn from_bool(b: bool) -> Value {
        Value(unsafe { transmute(TAG_BOOL | b as u64) })
    }

    pub fn unit() -> Value {
        Value(unsafe { transmute(TAG_UNIT) })
    }


    pub fn from_ref(to_node: u32, in_node: u16) -> Value {
        unsafe { transmute(TAG_REF | (to_node as u64) << 16 | in_node as u64) }
    }
```

As said earlier, `Key` is a strict subset of `Value`. If it's possible to convert
t a simple copy bit for bit is sufficient.

```rust
    pub fn to_key(self) -> Option<Key> {
        let bits: u64 = unsafe { transmute(self.0) };
        match bits & TAG_MASK {
            TAG_SYMBOL | TAG_INT => Some(Key(bits)),
            _ => None,
        }
    }
```

There are two methods to extract a reference into the heap. One is checked,
the other just yields the result. There is no memory unsafety, but the
result of the second one could potentily be garbage.
```rust
    pub fn to_ref(self) -> Option<(u32, u16)> {
        let bits: u64 = unsafe { transmute(self.0) };
        match bits & TAG_MASK {
            TAG_REF => Some(((bits >> 16) as u32, bits as u16)),
            _ => None,
        }
    }

    pub fn as_ref_i_know_what_im_doing(self) -> (u32, u16) {
        let bits: u64 = unsafe { transmute(self.0) };
        ((bits >> 16) as u32, bits as u16)
    }

    pub fn is_nan(self) -> bool {
        self.0.is_nan()
    }

    pub fn as_f64(self) -> f64 {
        self.0
    }

    pub fn unpack(self) -> UnpackedValue {
        use self::UnpackedValue::*;
        let bits: u64 = unsafe { transmute(self.0) };
        if !self.0.is_nan() || bits == NAN_BASE {
            return F64;
        }
        match bits & TAG_MASK {
            TAG_INT => Int(bits as u32),
            TAG_REF => Ref,
            TAG_FAST_REF => FastRef,
            TAG_BOOL => Bool(bits as u32 & 1 == 1),
            TAG_UNIT => Unit,
            _ => unreachable!(),
        }

    }
}


#[cfg(test)]
mod tests {

    use super::{Value, Key};
    use super::UnpackedValue::*;
    use std::f64;

    macro_rules! matches(
            ($e:expr, $p:pat) => (
                match $e {
                    $p => true,
                    _ => false
                }
            )
        );

    #[test]
    fn sizes() {
        use std::mem::{size_of, transmute};
        assert_eq!(size_of::<fn(i32) -> i32>(), 8);
        fn test(i: u32) -> u32 {
            i
        }

        let bits: u64 = unsafe { transmute(&test) };
        assert!(bits & 0xFFFF_0000_0000_0000 == 0);
    }

    #[test]
    fn nan() {
        assert!(Value::from_f64(f64::NAN).is_nan());
        assert!(!Value::from_f64(0.444).is_nan());
        assert!(Value::from_bool(true).is_nan());
        assert!(Value::from_bool(false).is_nan());
        assert!(Value::unit().is_nan());
    }

    #[test]
    fn references() {
        let r = Value::from_ref(23, 2);
        assert!(matches!(r.unpack(), Ref));
        if let Some((to_node, in_node)) = r.to_ref() {
            assert_eq!(to_node, 23);
            assert_eq!(in_node, 2);
        } else {
            panic!("None where none should be");
        }
        let (to_node, in_node) = r.as_ref_i_know_what_im_doing();
        assert_eq!(to_node, 23);
        assert_eq!(in_node, 2);
    }


    #[test]
    fn key() {
        use super::TAG_INT;
        assert_eq!(Value::from_u64(1).to_key(), Some(Key(TAG_INT | 1)));
    }

    #[test]
    fn packing_and_unpacking() {
        assert!(matches!(Value::from_f64(0.0).unpack(), F64));
        assert!(matches!(Value::from_f64(f64::NAN).unpack(), F64));
        assert!(matches!(Value::from_bool(true).unpack(), Bool(true)));
        assert!(matches!(Value::from_bool(false).unpack(), Bool(false)));
        assert!(matches!(Value::unit().unpack(), Unit));
    }

}
```
