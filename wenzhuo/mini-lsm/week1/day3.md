# Block

## 磁盘结构
磁盘结构的基本单元是块。块的大小通常为4KB（大小可能因存储介质而异），   

这相当于操作系统中的页面大小和SSD上的页面大小。

块存储有序的键值对。一个SST由多个Block组成。

当memtable数量超过系统限制时，它会将memtable刷新为SST。

接下来要实现一个块的解码和编码  

```
----------------------------------------------------------------------------------------------------
|             Data Section             |              Offset Section             |      Extra      |
 ----------------------------------------------------------------------------------------------------
| Entry #1 | Entry #2 | ... | Entry #N | Offset #1 | Offset #2 | ... | Offset #N | num_of_elements |
 ----------------------------------------------------------------------------------------------------
``` 
每个条目都是一个键值对。
```
----------------------------------------------------------------------------------------------------
|             Data Section             |              Offset Section             |      Extra      |
 ----------------------------------------------------------------------------------------------------
| Entry #1 | Entry #2 | ... | Entry #N | Offset #1 | Offset #2 | ... | Offset #N | num_of_elements |
 ----------------------------------------------------------------------------------------------------
``` 
键和值对长度都是2个字节，这意味着它们的最大长度都是65535.（内部存储为u16）  

我们假设键永远不会为空，而值可以为空。  

空值意味着相应的键在系统的其他部分的视图中已被删除。    

对于BlockBuilder和Blockiterator，我们只需按原样处理空值。   

在每个块的末尾，我们将存储每个条目的偏移量和条目的总数。  

例如，如果第一个条目位于块的第0个位置，而第二个条目位于块的第12个位置。 

```
-------------------------------
|offset|offset|num_of_elements|
-------------------------------
|   0  |  12  |       2       |
-------------------------------
``` 

块的页脚如上，每个数字存储为u16。块有大小限制，即target_size。      

除非第一个键值对超过目标块大小，否则应确保编码后的块大小始终小于或等于target_size。 

（在提供的代码中，这里的target_size本质上就是block_size）   

调用构建时，BlockBuilder将生成数据部分和未编码的条目偏移量。    

这些信息将存储在Block结构中。由于键值条目以原始格式存储，偏移量存储在单独的数组中， 

这减少了解码数据时不必要的内存分配和处理开销——      

您需要做的是简单地将原始块数据复制到数据数组中，并每隔2个字节解码条目偏移量，   

而不是创建类似Vec<(Vec)Vec>将所有的键值对存储在内存中的一个块中。这种紧凑的内存布局非常高效。

在Block::coding和Block::decode中，您需要按照上述格式对块进行编码/解码。

```rust
// encode
pub fn encode(&self) -> Bytes {
    let mut encode_buf = self.data.clone();
    let num_of_elements = self.offsets.len();
    for x in &self.offsets {
        encode_buf.put_u16(*x);
    }
    encode_buf.put_u16(num_of_elements as u16);
    encode_buf.into()
}

// decode
pub fn decode(data: &[u8]) -> Self {
    let num_of_elements = (&data[data.len() - 2..]).get_u16() as usize;
    let offsets_vec = &data[data.len() - 2 - num_of_elements * 2..data.len() - 2];
    let offsets = offsets_vec.chunks(2).map(|mut x| x.get_u16()).collect();
    let data = data[..data.len() - 2 - num_of_elements * 2].to_vec();
    Self { offsets, data }
}


//
fn seek_to_index(&mut self, index: usize) {
    self.idx = index;
    // 获取偏移量，并找到数据的位置
    let offset = self.block.offsets[index] as usize;
    let mut entry = &self.block.data[offset..];
    // 解码key的长度，get_u16会自动向后移动2个字节
    let key_length = entry.get_u16() as usize;
    // 解码key，复制
    let key = &entry[..key_length];
    self.key.clear();
    self.key.append(key);
    entry.advance(key_length);
    // 解码value的长度
    let value_length = entry.get_u16() as usize;
    let value_offset_begin = offset + 2 + key_length + 2;
    let value_offset_end = value_offset_begin + value_length;
    // 将value的起始位置与结束位置记录
    self.value_range = (value_offset_begin, value_offset_end);
}
``` 

