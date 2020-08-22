---
title: "Python Cheatsheet"
date: 2020-08-20T00:49:17-04:00
draft: true
feature_image: /python
summary: "Python cheatsheet"
---

Someone sprang a surprise Python assessment on me, so here's a cheatsheet of the most common patterns and functions that are frequently used.



Flattening a list of lists

```
LoL = [[i for i in range(5)]]
result = [ele for L in LoL for ele in L]
```

### Collections library

`Counter`

```
from collections import Counter

c = Counter()
c = Counter("asdasdsa")                 # Creating directly from an iterable
c = Counter([1, 1, 2, 3, 4, 5, 5])      # Creating directly from an iterable
c = Counter({"k1": 4, "k2": 2})

```

`defaultdict`

```
from collections import defaultdict

d = defaultdict(lambda: set())
d["somekey"].add("random value")
```

### Generators

