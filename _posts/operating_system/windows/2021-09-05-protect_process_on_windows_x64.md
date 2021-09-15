---

layout: post 

title:  "Windows x64 实现进程保护" 

---

以前没有PG的时候，相信绝大部分人都是用的SSDT HOOK 来进行进程保护的。等有了PG该怎么办呢？答案就是用微软提供的 ObRegisterCallbacks 函数。

```c++
 NTSTATUS
    ObRegisterCallbacks (
        _In_ POB_CALLBACK_REGISTRATION CallbackRegistration,
        _Outptr_ PVOID *RegistrationHandle
        );
```



上边这是函数定义 。 第一个参数是注册回调的一些信息。第二个参数返回此回调的指针，可以去跟进程句柄去类比，创建一个进程会返回一个进程句柄，类似的创建一个回调会返回一个跟此回调相关的指针。

 

第一个参数 POB_CALLBACK_REGISTRATION 类型的定义是：

 

```cpp
    typedef struct _OB_CALLBACK_REGISTRATION {

    _In_ USHORT                     Version;

    _In_ USHORT                     OperationRegistrationCount;

    _In_ UNICODE_STRING             Altitude;

    _In_ PVOID                      RegistrationContext;

    _In_ OB_OPERATION_REGISTRATION  *OperationRegistration;

} OB_CALLBACK_REGISTRATION, *POB_CALLBACK_REGISTRATION;
```


**第一个成员，总是填写 OB_FLT_REGISTRATION_VERSION。

第二个成员，The number of entries in the OperationRegistration array.这个跟最后一个成员 OperationRegistration有关。是告诉系统你注册了几个回调函数。

第三个成员，可以认为是与优先级相关。此值需要向微软申请，网络上多用"321000"来填写。

第四个成员，就是传给回调函数的参数。

第五个是一个结构体，定义如下：**

 

```cpp
    typedef struct _OB_OPERATION_REGISTRATION {

    _In_ POBJECT_TYPE                *ObjectType;

    _In_ OB_OPERATION                Operations;

    _In_ POB_PRE_OPERATION_CALLBACK  PreOperation;

    _In_ POB_POST_OPERATION_CALLBACK PostOperation;

} OB_OPERATION_REGISTRATION, *POB_OPERATION_REGISTRATION;
```



第一个参数可选 PsProcessType for process handle operations, or PsThreadType for thread handle operations.

第二个参数可选

OB_OPERATION_HANDLE_CREATEA
new process handle or thread handle was or will be opened.
OB_OPERATION_HANDLE_DUPLICATEA
process handle or thread handle was or will be duplicated

  

第三个参数就是Pre回调，第四个就是post回调。比如用户层CreateProcess了，返回前调用Pre的，返回后调用Post的。

 

Pre回调定义如下：

 

```cpp
    OB_PREOP_CALLBACK_STATUS 
      ObjectPreCallback(
        __in PVOID  RegistrationContext,
        __in POB_PRE_OPERATION_INFORMATION  OperationInformation
        );
```

  

Post回调定义如下：

 

```cpp
    VOID 
      ObjectPostCallback(
        __in PVOID  RegistrationContext,
        __in POB_POST_OPERATION_INFORMATION  OperationInformation
        );
```

 

 

 

 

 

于是我们这么调用（以下代码用VS2015+WDK8.1在win7x86下测试通过，如果想用到x64上需要修改_LDR_DATA_TABLE_ENTRY结构体，将LIST_ENTRY改为LIST_ENTRY64即可）：

 

```cpp
    #include <ntddk.h>
    #include <wdm.h>
     
    #define PROCESS_TERMINATE         0x0001  
    #define PROCESS_VM_OPERATION      0x0008  
    #define PROCESS_VM_READ           0x0010  
    #define PROCESS_VM_WRITE          0x0020  
    extern
    UCHAR *
    PsGetProcessImageFileName(
    	__in PEPROCESS Process
    );
    typedef struct _LDR_DATA_TABLE_ENTRY
    {
    	LIST_ENTRY    InLoadOrderLinks;
    	LIST_ENTRY    InMemoryOrderLinks;
    	LIST_ENTRY    InInitializationOrderLinks;
    	PVOID            DllBase;
    	PVOID            EntryPoint;
    	ULONG            SizeOfImage;
    	UNICODE_STRING    FullDllName;
    	UNICODE_STRING     BaseDllName;
    	ULONG            Flags;
    	USHORT            LoadCount;
    	USHORT            TlsIndex;
    	PVOID            SectionPointer;
    	ULONG            CheckSum;
    	PVOID            LoadedImports;
    	PVOID            EntryPointActivationContext;
    	PVOID            PatchInformation;
    	LIST_ENTRY    ForwarderLinks;
    	LIST_ENTRY    ServiceTagLinks;
    	LIST_ENTRY    StaticLinks;
    	PVOID            ContextInformation;
    	ULONG            OriginalBase;
    	LARGE_INTEGER    LoadTime;
    } LDR_DATA_TABLE_ENTRY, *PLDR_DATA_TABLE_ENTRY;
     
     
    PVOID g_pRegiHandle = NULL;
    OB_PREOP_CALLBACK_STATUS MyObjectPreCallback
    (
    	__in PVOID  RegistrationContext,
    	__in POB_PRE_OPERATION_INFORMATION  OperationInformation
    )
    {
    	if (OperationInformation->KernelHandle)
    		return OB_PREOP_SUCCESS;
    	DbgPrint(PsGetProcessImageFileName((PEPROCESS)OperationInformation->Object));
    	if (OperationInformation->Operation == OB_OPERATION_HANDLE_CREATE)//这是把所有的进程Create操作都过滤了。禁止了终止、内存读写的权限，于是就实现了进程保护啦
    	{
    		if ((OperationInformation->Parameters->CreateHandleInformation.OriginalDesiredAccess & PROCESS_TERMINATE) == PROCESS_TERMINATE)
    		{
    			//Terminate the process, such as by calling the user-mode TerminateProcess routine..
    			OperationInformation->Parameters->CreateHandleInformation.DesiredAccess &= ~PROCESS_TERMINATE;
    		}
    		if ((OperationInformation->Parameters->CreateHandleInformation.OriginalDesiredAccess & PROCESS_VM_OPERATION) == PROCESS_VM_OPERATION)
    		{
    			//Modify the address space of the process, such as by calling the user-mode WriteProcessMemory and VirtualProtectEx routines.
    			OperationInformation->Parameters->CreateHandleInformation.DesiredAccess &= ~PROCESS_VM_OPERATION;
    		}
    		if ((OperationInformation->Parameters->CreateHandleInformation.OriginalDesiredAccess & PROCESS_VM_READ) == PROCESS_VM_READ)
    		{
    			//Read to the address space of the process, such as by calling the user-mode ReadProcessMemory routine.
    			OperationInformation->Parameters->CreateHandleInformation.DesiredAccess &= ~PROCESS_VM_READ;
    		}
    		if ((OperationInformation->Parameters->CreateHandleInformation.OriginalDesiredAccess & PROCESS_VM_WRITE) == PROCESS_VM_WRITE)
    		{
    			//Write to the address space of the process, such as by calling the user-mode WriteProcessMemory routine.
    			OperationInformation->Parameters->CreateHandleInformation.DesiredAccess &= ~PROCESS_VM_WRITE;
    		}
    		 
    	  }
    	return OB_PREOP_SUCCESS;
    }
     
    VOID DriverUnload(_In_ struct _DRIVER_OBJECT *DriverObject)
    {
    	ObUnRegisterCallbacks(g_pRegiHandle);
    }
     
    NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
    {
    	OB_OPERATION_REGISTRATION oor;
    	OB_CALLBACK_REGISTRATION ob;
    	
    	PLDR_DATA_TABLE_ENTRY ldr;
    	ldr = (PLDR_DATA_TABLE_ENTRY)DriverObject->DriverSection;
    	ldr->Flags |= 0x20;//加载驱动的时候会判断此值。必须有特殊签名才行，增加0x20即可。否则将调用失败 
    	DriverObject->DriverUnload = DriverUnload;
    	
    	oor.ObjectType = PsProcessType;
    	oor.Operations = OB_OPERATION_HANDLE_CREATE;
    	oor.PreOperation = MyObjectPreCallback;
    	oor.PostOperation = NULL;
    	
    	ob.Version = OB_FLT_REGISTRATION_VERSION;
    	ob.OperationRegistrationCount = 1;
    	ob.OperationRegistration = &oor;
    	RtlInitUnicodeString(&ob.Altitude, L"321000"); 
    	ob.RegistrationContext = NULL;
    	 
    	return ObRegisterCallbacks(&ob, &g_pRegiHandle);
    }
```

