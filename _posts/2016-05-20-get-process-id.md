---
layout: post
title: "获取进程ID"
categories: Windows c++
---

# 获取进程PID



```c++
void GetProcPid(){
    autp pid{GetProcessPid()};
	auto proc_handle{OpenProcess(pid)};
      
    
}
```

