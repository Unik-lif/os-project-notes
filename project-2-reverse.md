---
description: 不得不说chatgpt真的太好用了....有两个卡顿的问题分分钟帮我解决了。
---

# Project 2: reverse

需要牢记的例子：两个文件是否是相同的真正判别方法。

```clike
// Some code
#include <stdbool.h>
#include <sys/stat.h>

bool are_files_different(const char* file1, const char* file2) {
    struct stat stat1, stat2;
    if (stat(file1, &stat1) != 0 || stat(file2, &stat2) != 0) {
        // Error occurred while reading file stats
        return true;
    }
    if (stat1.st_ino != stat2.st_ino || stat1.st_dev != stat2.st_dev || stat1.st_size != stat2.st_size) {
        // Files are different
        return true;
    }
    // Files are the same
    return false;
}
```

Vim的复制：修改一下状态。

:set paste，完成后 :set nopaste就好了
