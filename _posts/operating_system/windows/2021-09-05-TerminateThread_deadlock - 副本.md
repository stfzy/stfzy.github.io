---

layout: post 

title:  "TerminateThread 导致LoadLibary 死锁" 

---

运行下边的代码，线程会死锁，因为在`LoadLibrary`过程中会有一个临界区被占用，如果正好是这个时候此线程被`TerminateThread`终止，那么所有的`LoadLibrary`及其他使用此锁的线程都会进入一个无限等待这个锁释放的状态。所以不光是自己实现的锁要小心，系统某些API也要小心，因为其内部也有同步的逻辑。

```c++
#include <windows.h>
#include <stdio.h>

DWORD WINAPI Thread1(LPVOID lp) {
   do {
     HMODULE h = LoadLibraryExW(L"Wtsapi32.dll",0,0);
     Sleep(1);
     FreeLibrary(h);
   } while (1);
  return 0;
}

DWORD WINAPI Thread2(LPVOID lp) {
   int i = 0;
   do {
     HMODULE h = LoadLibraryExW(L"Wtsapi32.dll",0,0);
     Sleep(1);
     FreeLibrary(h);

     printf("\r%d", i++);
   } while (1);
  return 0;
}

DWORD WINAPI Thread3(LPVOID lp) {
   do {
     HANDLE h = CreateThread(0, 0, a1, 0, 0, 0);
     Sleep(200);
     TerminateThread(h,0);
   } while (1);
  return 0;
}

int main(int argc ,char *argv[]) {
   CloseHandle(CreateThread(0, 0, Thread1, 0, 0, 0));
   CloseHandle(CreateThread(0, 0, Thread2, 0, 0, 0));
   CloseHandle(CreateThread(0, 0, Thread3, 0, 0, 0));

   do {
     Sleep(10000);
   } while (1);
  return 0;
}
```



原因是（引自MSDN）：

> **TerminateThread** is a dangerous function that should only be used in the most extreme cases. You should call **TerminateThread** only if you know exactly what the target thread is doing, and you control all of the code that the target thread could possibly be running at the time of the termination. For example, **TerminateThread** can result in the following problems:
>
> - If the target thread owns a critical section, the critical section will not be released.
> - If the target thread is allocating memory from the heap, the heap lock will not be released.
> - If the target thread is executing certain kernel32 calls when it is terminated, the kernel32 state for the thread's process could be inconsistent.
> - If the target thread is manipulating the global state of a shared DLL, the state of the DLL could be destroyed, affecting other users of the DLL.