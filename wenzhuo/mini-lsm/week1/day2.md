## 使用第三方库ouroboros来实现迭代器    

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