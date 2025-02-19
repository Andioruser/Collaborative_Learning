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


```