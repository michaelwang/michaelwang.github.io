---
layout: post
title:  "Java Stream Lambda Learning Notes"
date:   2024-07-20 11:01:11 +0900
categories: java
---

# Summary

1. In lambda we should not dependent on  the outside virable to finish some logic,(but why?)
When using lambda expression, we need dependent on the upper layer output, evey lambda expression should keep as stateless as possible. 

2. There is java.util.Function package, all the interfaces in the package only has a method, these interfaces are designed for lambda.

3. Stream are stands for operations, if there are not termination operations, these stream operations will not be executed, the termination operations includes .collect(), forEach(), by the way forEach is the worst termination operation, we should as much as possible to use .collect() as termination operation. 

4. The termination operation .collect() can be used as collect stream into list or map, when the list or map has duplicate item, we can use the third
paramter to do some extra logic , such as we can keep the latest item in the list or we can do some merge operation if multiple items have same key.

5. Function object is more succint than lambda expression, for example, if we want to insert a key into a map, if the key is already exist in the map, we need to increment the value, but if the key does not exist, we just need to insert the value into the map. Below is the code snipe to use lambda to achive this. 

   ```
   map.merge(key, 1, (cnt, increment) -> cnt + increment)

   ```
You can see the lambda is a little difficulte to understand. But if we use function reference like below code snipet, the code is more descriptive.

    ```
    map.merge(key, 1, Integer::sum) 

   ```


6. Function reference is not always be the best options when we coding in lambda, this only happens when the function reference is called in the same class, for the below example:

   ```
   service.execute(GoshThiClassNameIsHumongous::action)
   ```
but the lambda equivalent looks like this:

    ```    
    service.execute(() -> action());
    ```
7. There are lots of function interfaces in java.util.Function, it is not possible to remember them all, but if we can learn six basic function interfaces, other function type can be derived.
   ```
   UnaryOperator<T>     T apply(T t)         String::toLowerCase
   BinaryOperator<T>    T apply(T t1, T t2)  BigInteger::add
   Predicate<T>         boolean  test(T t)   Collection::isEmpty
   Function<T,R>        R apply(T t)         Arrays::asList
   Supplier<T>          T get()              Instant::now
   Consumer<T>          void accept(T t)     System.out::println
   ```