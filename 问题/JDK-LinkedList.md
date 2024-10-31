# LinkedList

## 字段

````java
transient int size;
transient Node<E> first;
transient Node<E> last;
````

## 结点

````java
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
````



## 增加

```java
    private void linkFirst(E e) {
        Node<E> f = this.first;
        Node<E> newNode = new Node((Node)null, e, f);
        this.first = newNode;
        if (f == null) {
            this.last = newNode;
        } else {
            f.prev = newNode;
        }

        ++this.size;
        ++this.modCount;
    }
```

````java
    void linkBefore(E e, Node<E> succ) {
        Node<E> pred = succ.prev;
        Node<E> newNode = new Node(pred, e, succ);
        succ.prev = newNode;
        if (pred == null) {
            this.first = newNode;
        } else {
            pred.next = newNode;
        }

        ++this.size;
        ++this.modCount;
    }
````



## 删除

````java
    private E unlinkFirst(Node<E> f) {
        E element = f.item;
        Node<E> next = f.next;
        f.item = null;
        f.next = null;
        this.first = next;
        if (next == null) {
            this.last = null;
        } else {
            next.prev = null;
        }

        --this.size;
        ++this.modCount;
        return element;
    }
````

## 查询

````java
    Node<E> node(int index) {
        Node x;
        int i;
        if (index < this.size >> 1) {
            x = this.first;

            for(i = 0; i < index; ++i) {
                x = x.next;
            }

            return x;
        } else {
            x = this.last;

            for(i = this.size - 1; i > index; --i) {
                x = x.prev;
            }

            return x;
        }
    }
````

