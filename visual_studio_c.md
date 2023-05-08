# 填写好名字，就可以开始编辑了：
## 我有一张图片![扫码](https://img.51miz.com/Element/00/86/19/08/225420b4_E861908_5a3143ca.png)
### fsssfsfsa asfa	  
* item1
  
  
```package com.example.test_rtl_1

import kotlin.contracts.ExperimentalContracts
import kotlin.contracts.contract

@OptIn(ExperimentalContracts::class)
inline fun <T> List<T>.fastAll(predicate: (T) -> Boolean): Boolean {
    contract { callsInPlace(predicate) }
    fastForEach { if (!predicate(it)) return false }
    return true
}
@OptIn(ExperimentalContracts::class)
inline fun <T> List<T>.fastForEach(action: (T) -> Unit) {
    contract { callsInPlace(action) }
    for (index in indices) {
        val item = get(index)
        action(item)
    }
}
@OptIn(ExperimentalContracts::class)
internal inline fun <T, R> List<T>.fastMapNotNull(transform: (T) -> R?): List<R> {
    contract { callsInPlace(transform) }
    val target = ArrayList<R>(size)
    fastForEach { e ->
        transform(e)?.let { target += it }
    }
    return target
}
@Suppress("BanInlineOptIn")
@OptIn(ExperimentalContracts::class)
inline fun <T> List<T>.fastAny(predicate: (T) -> Boolean): Boolean {
    contract { callsInPlace(predicate) }
    fastForEach { if (predicate(it)) return true }
    return false
}
