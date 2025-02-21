## 使用第三方库ouroboros来实现迭代器    

```rust
use ouroboros::self_referencing;

#[self_referencing]
struct MyStruct {
    int_data: i32,
    float_data: f32,
    #[borrows(int_data)]
    // the 'this lifetime is created by the #[self_referencing] macro
    // and should be used on all references marked by the #[borrows] macro
    int_reference: &'this i32,
    #[borrows(mut float_data)]
    float_reference: &'this mut f32,
}

fn main() {
    // The builder is created by the #[self_referencing] macro
    // and is used to create the struct
    let mut my_value = MyStructBuilder {
        int_data: 42,
        float_data: 3.14,

        // Note that the name of the field in the builder
        // is the name of the field in the struct + `_builder`
        // ie: {field_name}_builder
        // the closure that assigns the value for the field will be passed
        // a reference to the field(s) defined in the #[borrows] macro

        int_reference_builder: |int_data: &i32| int_data,
        float_reference_builder: |float_data: &mut f32| float_data,
    }.build();

    // The fields in the original struct can not be accessed directly
    // The builder creates accessor methods which are called borrow_{field_name}()

    // Prints 42
    println!("{:?}", my_value.borrow_int_data());
    // Prints 3.14
    println!("{:?}", my_value.borrow_float_reference());
    // Sets the value of float_data to 84.0
    my_value.with_mut(|fields| {
        **fields.float_reference = (**fields.int_reference as f32) * 2.0;
    });

    // We can hold on to this reference...
    let int_ref = *my_value.borrow_int_reference();
    println!("{:?}", *int_ref);
    // As long as the struct is still alive.
    drop(my_value);
    // This will cause an error!
    // println!("{:?}", *int_ref);
}
```

项目实例代码：  

```rust

 /// Get an iterator over a range of keys.
    pub fn scan(&self, _lower: Bound<&[u8]>, _upper: Bound<&[u8]>) -> MemTableIterator {
        let (lower, upper) = (map_bound(_lower), map_bound(_upper));
        let mut iterator = MemTableIteratorBuilder {
            map: self.map.clone(),
            iter_builder: |map| map.range((lower, upper)),
            item: (Bytes::new(), Bytes::new()),
        }
        .build();
        iterator.next().unwrap();
        iterator
    }

#[self_referencing]
pub struct MemTableIterator {
    /// Stores a reference to the skipmap.
    map: Arc<SkipMap<Bytes, Bytes>>,
    /// Stores a skipmap iterator that refers to the lifetime of `MemTableIterator` itself.
    #[borrows(map)]
    #[not_covariant]
    iter: SkipMapRangeIter<'this>,
    /// Stores the current key-value pair.
    item: (Bytes, Bytes),
}

fn value(&self) -> &[u8] {
        &self.borrow_item().1[..]
    }

    fn key(&self) -> KeySlice {
        KeySlice::from_slice(&self.borrow_item().0[..])
    }

    fn is_valid(&self) -> bool {
        !self.borrow_item().0.is_empty()
    }

    fn next(&mut self) -> Result<()> {
        let entry = self.with_iter_mut(|iter| {
            let entry = iter.next();
            match entry {
                None => (Bytes::new(), Bytes::new()),
                Some(entry) => (entry.key().clone(), entry.value().clone()),
            }
        });
        self.with_item_mut(|item| {
            *item = entry;
        });
        Ok(())
    }
```

[ouroboros::self_referencing库链接](https://docs.rs/ouroboros/latest/ouroboros/attr.self_referencing.html#you-must-comply-with-these-limitations)   

## 范围边界，迭代范围   

Excluded(x), Included(x) ->  未包含x，包含x     

Unbounded -> 没有边界【负无穷或者正无穷】   

```rust
use std::collections::BTreeMap;
use std::ops::Bound::{Excluded, Included, Unbounded};

let mut map = BTreeMap::new();
map.insert(3, "a");
map.insert(5, "b");
map.insert(8, "c");

// (Excluded(3), Included(8)) -> (3,5] 未包含，包含 Unbounded -> 没有边界【负无穷或者正无穷】
for (key, value) in map.range((Excluded(3), Included(8))) {
    println!("{key}: {value}");
}

assert_eq!(Some((&3, &"a")), map.range((Unbounded, Included(5))).next());   

```
[std::ops::Bound](https://doc.rust-lang.org/beta/std/ops/enum.Bound.html)