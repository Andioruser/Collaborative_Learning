##  Memtable迭代器

### 使用第三方库ouroboros来实现迭代器    

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

### 范围边界，迭代范围   

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

## 合并迭代器   

**用二叉堆（优先队列、小根堆）实现迭代器合并**  

### 比较规则    

MergeIterator内部维护一个二叉堆。您将看到二叉堆的排序是这样的：具有最低头键值的迭代器排在最前面。   

当多个迭代器具有相同的头键值时，最新的迭代器排在最前面。请注意，您需要处理错误（即，当迭代器无效时）并确保出现最新版本的键值对。    

例如，如果我们有以下数据：  

```
iter1: b->del, c->4, d->5
iter2: a->1, b->2, c->3
iter3: e->4
```

合并迭代器输出的序列应该是：    

```
a->1, b->del, c->4, d->5, e->4
```

### 项目中的实际运用    
迭代器内部封装了一个二叉堆【并且实现自己定义的排序规则】使得可以拿到我们想要的Key值最小且最新的数据。
```rust
/// Merge multiple iterators of the same type. If the same key occurs multiple times in some
/// iterators, prefer the one with smaller index.
struct HeapWrapper<I: StorageIterator>(pub usize, pub Box<I>);

impl<I: StorageIterator> PartialEq for HeapWrapper<I> {
    fn eq(&self, other: &Self) -> bool {
        self.cmp(other) == cmp::Ordering::Equal
    }
}

impl<I: StorageIterator> Eq for HeapWrapper<I> {}

impl<I: StorageIterator> PartialOrd for HeapWrapper<I> {
    fn partial_cmp(&self, other: &Self) -> Option<cmp::Ordering> {
        Some(self.cmp(other))
    }
}

impl<I: StorageIterator> Ord for HeapWrapper<I> {
    // 定义 cmp 方法，用于比较两个 HeapWrapper 实例的顺序
    fn cmp(&self, other: &Self) -> cmp::Ordering {
        // 首先比较两个实例的 key 字段，如果 key 相等，则比较第一个字段
        self.1
            .key()
            .cmp(&other.1.key())
            .then(self.0.cmp(&other.0))
            // 反转比较结果，使得较大的实例排在较小的实例前面
            .reverse()
    }
}

pub struct MergeIterator<I: StorageIterator> {
    iters: BinaryHeap<HeapWrapper<I>>,
    current: Option<HeapWrapper<I>>,
}

impl<I: StorageIterator> MergeIterator<I> {
    pub fn create(iters: Vec<Box<I>>) -> Self {
        let mut iter = MergeIterator {
            iters: BinaryHeap::new(),
            current: None,
        };
        if iters.iter().all(|x| !x.is_valid()) && !iters.is_empty() {
            let mut iters = iters;
            iter.current = Some(HeapWrapper(0, iters.pop().unwrap()));
            return iter;
        }
        for (index, storage_iter) in iters.into_iter().enumerate() {
            if storage_iter.is_valid() {
                iter.iters.push(HeapWrapper(index, storage_iter));
            }
        }
        if !iter.iters.is_empty() {
            iter.current = Some(iter.iters.pop().unwrap())
        }
        iter
    }
}

impl<I: 'static + for<'a> StorageIterator<KeyType<'a> = KeySlice<'a>>> StorageIterator
    for MergeIterator<I>
{
    type KeyType<'a> = KeySlice<'a>;

    fn key(&self) -> KeySlice {
        self.current.as_ref().unwrap().1.key()
    }

    fn value(&self) -> &[u8] {
        self.current.as_ref().unwrap().1.value()
    }

    fn is_valid(&self) -> bool {
        if let None = self.current {
            return false;
        }
        self.current.as_ref().unwrap().1.is_valid()
    }

    fn next(&mut self) -> Result<()> {
        let current = self.current.as_mut().unwrap();
        while let Some(mut inner_iter) = self.iters.peek_mut() {
            if inner_iter.1.key() != current.1.key() {
                break;
            }
            if let e @ Err(_) = inner_iter.1.next() {
                PeekMut::pop(inner_iter);
                return e;
            }
            if !inner_iter.1.is_valid() {
                PeekMut::pop(inner_iter);
            }
        }

        current.1.next()?;

        if !current.1.is_valid() {
            if let Some(iter) = self.iters.pop() {
                *current = iter;
            }
            return Ok(());
        }

        if let Some(mut inner_iter) = self.iters.peek_mut() {
            if *current < *inner_iter {
                std::mem::swap(&mut *inner_iter, current);
            }
        }

        Ok(())
    }
}

```

## LSM迭代器 + 融合迭代器   

我们使用LsmIterator结构来表示内部的LSM迭代器。当系统中添加了更多的迭代器时，您将需要在整个教程中多次修改此结构。目前，因为我们只有多个memtable，所以它应该定义为：  

类型LsmIteratorInner=MergeIterator<MemTableIterator>；  

您可以继续实现LsmIterator结构，它调用相应的内部迭代器，并且也跳过删除的键。 

我们不在此任务中测试LsmIterator。在任务4中，将有一个集成测试。  

然后，我们希望在迭代器上提供额外的安全性，以避免用户误用它们。当迭代器无效时，不应调用key、value或next。同时，如果next返回错误，则不应该再使用迭代器。FusedIterator是一个围绕迭代器的包装器，用于规范化所有迭代器的行为。你可以自己去实现它。   

```rust
//主要还是直接调用inner,就是在空value的时候需要挑到下一个键值对
impl LsmIterator {
    pub(crate) fn new(iter: LsmIteratorInner) -> Result<Self> {
        let mut lsm = Self { inner: iter };
        if lsm.is_valid() && lsm.value().is_empty() {
            lsm.next();
        }
        Ok(lsm)
    }
}

impl StorageIterator for LsmIterator {
    type KeyType<'a> = &'a [u8];

    fn is_valid(&self) -> bool {
        self.inner.is_valid()
    }

    fn key(&self) -> &[u8] {
        self.inner.key().raw_ref()
    }

    fn value(&self) -> &[u8] {
        self.inner.value()
    }

    fn next(&mut self) -> Result<()> {
        self.inner.next();
        if self.inner.is_valid() && self.inner.value().is_empty() {
            return self.next();
        }
        Ok(())
    }
}

//判断是否有异常、是否合法，否则报错
impl<I: StorageIterator> StorageIterator for FusedIterator<I> {
    type KeyType<'a> = I::KeyType<'a> where Self: 'a;

    fn is_valid(&self) -> bool {
        !self.has_errored && self.iter.is_valid()
    }

    fn key(&self) -> Self::KeyType<'_> {
        if !self.is_valid() {
            panic!("invalid access to the underlying iterator");
        }
        self.iter.key()
    }

    fn value(&self) -> &[u8] {
        if !self.is_valid() {
            panic!("invalid access to the underlying iterator");
        }
        self.iter.value()
    }

    fn next(&mut self) -> Result<()> {
        if self.has_errored {
            bail!("the iterator is tainted");
        }
        if self.iter.is_valid() {
            if let Err(e) = self.iter.next() {
                self.has_errored = true;
                return Err(e);
            }
        }
        Ok(())
    }
}
```

## 扫描路径-Scan

```rust
let snapshot = {
    let guard = self.state.read();
    Arc::clone(&guard)
};
let mut memtable_iters = Vec::with_capacity(snapshot.imm_memtables.len() + 1);
memtable_iters.push(Box::new(snapshot.memtable.scan(_lower, _upper)));
for memtable in snapshot.imm_memtables.iter() {
    memtable_iters.push(Box::new(memtable.scan(_lower, _upper)));
}
Ok(FusedIterator::new(LsmIterator::new(
    MergeIterator::create(memtable_iters),
)?))

```