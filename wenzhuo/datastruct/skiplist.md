# 跳表实现  

![alt text](image.png)  

**跳表初始化**  

```go
package main

type Node struct {
    data int
    next *Node
}

type Element struct {
    data KV //真正的数据
    levels []*Element   //存放节点   
}


var ElemA *Element
var ElemB *Element
var ElemC *Element
var ElemE *Element
var ElemI *Element

func init() {
    ElemA.levels[0] = ElemB
    ElemA.levels[1] = ElemC
    ElemA.levels[2] = ElemE
    ElemA.levels[3] = ElemI
}

```

**有序单链表的查找**    

```go
curNode := nil
prevNode := nil

for curNode := head.next; curNode != nil; curNode = prevNode.next {
    if key > curNode.data {
        prevNode = curNode;
        continue;
    }
    if key <= curNode.data {
        if key == curNode.data {
            return curNode.data
        }
        break;
    }
}

return notFound
```

**查找整个跳表**   

```go
prevElem := list.header
i := len(list.header.levels) - 1

for i >= 0 {
    //在每一层执行有序单链表的查找，查找终止的条件是, nextElement > findKey
    //这个位置说明改值不存在，或者是该值应该被插入的位置
    for next := prevElem.levels[i]; next != nil; next = prevElem.levels[i] {
        if findKey <= next.key {
            return next.data
        }
        break
    }
    i--
}
return notFound

```

**加速查找**    

```go
// 把全长的key字符比较，转换成八位的比较，如果还是相等才进行真正的compare比较运算。
// 能在一定程度上加速比较key过程。
func compare(score float64, key []byte, next *Element) int {
    if score == next.score {
        return bytes.Compare(key, next.entry.Key)
    }

    if score < next.score {
        return -1
    } else {
        return 1
    }
}

func calcScore(key []byte) (score float64) {
    var hash uint64 
    l := len(key)

    if l > 8 {
       l = 8
    }
    
    for i := 0; i < l; i++ {
        shift := uint(64 - 8 - i*8)
        hash |= uint64(key[i]) << shift
    }

    score = float64(hash)
    return
}
```

```go
var newNode *Node

newNode.next = next
preNode.next = newNode

//----------------------------------------
prevElem := list.header
i := len(list.header.levels) - 1
prevElemList []*Element

for i >= 0 {
    
}


```