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


