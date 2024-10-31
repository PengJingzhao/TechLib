# ArrayList源码分析

## 字段

````java
    transient Object[] elementData;
    private int size;
````

## 常量

````java
    private static final int DEFAULT_CAPACITY = 10;
    private static final Object[] EMPTY_ELEMENTDATA = new Object[0];
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = new Object[0];
    private static final int MAX_ARRAY_SIZE = 2147483639;
````

## 初始化

````java
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else {
            if (initialCapacity != 0) {
                throw new IllegalArgumentException("Illegal Capacity: " + initialCapacity);
            }

            this.elementData = EMPTY_ELEMENTDATA;
        }

    }
````

## 新增

````java
    private void add(E e, Object[] elementData, int s) {
        if (s == elementData.length) {
            elementData = this.grow();
        }

        elementData[s] = e;
        this.size = s + 1;
    }
````

## 删除

````java
public boolean remove(Object o) {
        Object[] es = this.elementData;
        int size = this.size;
        int i = 0;
        if (o == null) {
            while(true) {
                if (i >= size) {
                    return false;
                }

                if (es[i] == null) {
                    break;
                }

                ++i;
            }
        } else {
            while(true) {
                if (i >= size) {
                    return false;
                }

                if (o.equals(es[i])) {
                    break;
                }

                ++i;
            }
        }

        this.fastRemove(es, i);
        return true;
    }
````

````java
private void fastRemove(Object[] es, int i) {
        ++this.modCount;
        int newSize;
        if ((newSize = this.size - 1) > i) {
            System.arraycopy(es, i + 1, es, i, newSize - i);
        }

        es[this.size = newSize] = null;
    }
````

## 查询

````java
E elementData(int index) {
        return this.elementData[index];
    }
````

## 扩容

````java
    private Object[] grow(int minCapacity) {
        return this.elementData = Arrays.copyOf(this.elementData, this.newCapacity(minCapacity));
    }
````

````java
private int newCapacity(int minCapacity) {
        int oldCapacity = this.elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity <= 0) {
            if (this.elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
                return Math.max(10, minCapacity);
            } else if (minCapacity < 0) {
                throw new OutOfMemoryError();
            } else {
                return minCapacity;
            }
        } else {
            return newCapacity - 2147483639 <= 0 ? newCapacity : hugeCapacity(minCapacity);
        }
    }
````

