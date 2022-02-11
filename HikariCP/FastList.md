- [FastList是什么](#fastlist是什么)
- [源码剖析](#源码剖析)
  - [初始化](#初始化)
  - [添加元素](#添加元素)
  - [查询元素](#查询元素)
  - [删除元素](#删除元素)
  - [其他方法](#其他方法)
  
# FastList是什么

HikariCP中实现的一个List，底层基于数组实现，目的是提高List操作的性能，主要用于HikariCP中缓存`Statement`实例和链接。

与JDK自带的ArrayList的主要优化：

1. 去掉了`add`、`get`、`remove`等操作时的范围检查。源码中FastList的注释为：*`Fast list without range checking`*
2. `remove`方法里取值时从后向前遍历

可以发现其实并没有太高级的操作，但这就是HikariCP快的原因之一，将细节做到极致。

# 源码剖析

## 初始化

初始化时默认数组长度为32。

`ArrayList`中默认数组长度为10。

```java
public FastList(Class<?> clazz)
   {
      //默认32个元素
      this.elementData = (T[]) Array.newInstance(clazz, 32);
      this.clazz = clazz;
   }
```

## 添加元素

1. 新元素放在数组的尾端
2. 每次扩容时长度为旧数组的两倍

```java
   /**
    * Add an element to the tail of the FastList.
    *
    * @param element the element to add
    */
   @Override
   public boolean add(T element)
   {
      try {
         //不检查数组空间是否充足，直接赋值，如果数据越界，再执行数组扩容
         elementData[size++] = element;
      }
      catch (ArrayIndexOutOfBoundsException e) {
         final int oldCapacity = elementData.length;
         //每次扩容为旧数组的两倍
         final int newCapacity = oldCapacity << 1;
         @SuppressWarnings("unchecked")
         final T[] newElementData = (T[]) Array.newInstance(clazz, newCapacity);
         System.arraycopy(elementData, 0, newElementData, 0, oldCapacity);
         newElementData[size - 1] = element;
         elementData = newElementData;
      }

      return true;
   }
```

作为对比，看下`ArrayList`中的`add`方法：

1. 新元素放在数组的尾端
2. 每次扩容时长度为旧数组的1.5倍

```java
public boolean add(E e) {
    //检查数组空间是否充足，若空间不足，执行扩容操作，新数据是旧数组的1.5倍
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    //同样是尾插
    elementData[size++] = e;
    return true;
}
```

## 查询元素

不做范围检查，直接返回数组下标。

在HikariCP中，`FastList`用于保存Statement和链接，程序可以保证`FastList`的元素不会越界，这样可以省去范围检查的耗时。

```java
   /**
    * Get the element at the specified index.
    *
    * @param index the index of the element to get
    * @return the element, or ArrayIndexOutOfBounds is thrown if the index is invalid
    */
   @Override
   public T get(int index)
   {
      return elementData[index];
   }
```

同样看下ArrayList里的get方法

```java
public E get(int index) {
      //先做范围检查，如果数组越界，抛出IndexOutOfBoundsException
      rangeCheck(index);

      return elementData(index);
   }

   private void rangeCheck(int index) {
      if (index >= size)
         throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
   }

   E elementData(int index) {
      return (E) elementData[index];
   }
```

## 删除元素

`remove`里在查找元素时是从后向前遍历的，原因还是FastList用来保存`Statement`，而Statement通过是后创建出来的先被Close掉，这样可以提高查询效率。

```java
/**
 * This remove method is most efficient when the element being removed
 * is the last element.  Equality is identity based, not equals() based.
 * Only the first matching element is removed.
 *
 *@param element the element to remove
 */
@Override
public boolean remove(Object element)
{
   //从后往前遍历
   for (int index = size - 1; index >= 0; index--) {
      //基于==做比较而不是equals
      if (element == elementData[index]) {
         final int numMoved = size - index - 1;
         if (numMoved > 0) {
            System.arraycopy(elementData, index + 1, elementData, index, numMoved);
         }
         elementData[--size] = null;
         return true;
      }
   }

   return false;
}
```

同样看下ArrayList中的Remove方法：

```java
/**
    * Removes the first occurrence of the specified element from this list,
    * if it is present.  If the list does not contain the element, it is
    * unchanged.  More formally, removes the element with the lowest index
    * <tt>i</tt> such that
    * <tt>(o==null&nbsp;?&nbsp;get(i)==null&nbsp;:&nbsp;o.equals(get(i)))</tt>
    * (if such an element exists).  Returns <tt>true</tt> if this list
    * contained the specified element (or equivalently, if this list
    * changed as a result of the call).
    *
    * @param o element to be removed from this list, if present
    * @return <tt>true</tt> if this list contained the specified element
    */
   public boolean remove(Object o) {
      if (o == null) {
         //从前向后遍历，找到第一个匹配的数据就结束
         for (int index = 0; index < size; index++)
            //null使用==判断
            if (elementData[index] == null) {
               fastRemove(index);
               return true;
            }
      } else {
         for (int index = 0; index < size; index++)
            //非null使用equals判断
            if (o.equals(elementData[index])) {
               fastRemove(index);
               return true;
            }
      }
      return false;
   }
```

## 其他方法

`FastList`实现了`List`接口，但并没有将所有方法都实现出来，对HikariCP中用不到的方法直接抛出了`UnsupportedOperationException`

```
@Override
public <E> E[] toArray(E[] a)
{
   throw new UnsupportedOperationException();
}

@Override
public boolean containsAll(Collection<?> c)
{
   throw new UnsupportedOperationException();
}

@Override
public boolean addAll(Collection<? extends T> c)
{
   throw new UnsupportedOperationException();
}
```