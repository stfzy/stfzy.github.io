---
layout: post
categories: Windows c++
---

# 获取进程ID



```c++
void GetProcPid(){
    autp pid{GetProcessPid()};
	auto proc_handle{OpenProcess(pid)};
      
    
}
```

