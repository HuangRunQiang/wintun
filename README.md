# [Wintun网络适配器](https://www.wintun.net/)
### 用于Windows的TUN设备驱动程序

这是一个为Windows 7、8、8.1、10和11设计的第3层TUN驱动程序。最初为[WireGuard](https://www.wireguard.com/)创建，旨在为在用户空间主要实现的各种需要第3层隧道设备的项目提供帮助。

## 安装

Wintun作为特定于平台的 `wintun.dll` 文件进行部署。将 `wintun.dll` 文件与您的应用程序一起放置在同一目录下。从[wintun.net](https://www.wintun.net/)下载dll文件，以及下面描述的应用程序的头文件。

## 使用

通过简单地将[`wintun.h`文件](https://git.zx2c4.com/wintun/tree/api/wintun.h)复制到您的项目中并使用[`LoadLibraryEx()`](https://docs.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-loadlibraryexa)和[`GetProcAddress()`](https://docs.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-getprocaddress)来动态加载 `wintun.dll`，并使用头文件中提供的typedef来解析每个函数。 [example.c代码中的InitializeWintun函数](https://git.zx2c4.com/wintun/tree/example/example.c)提供了一个函数，您可以简单地复制和粘贴。

设置好库之后，Wintun可以通过首先创建一个适配器，配置它，然后将其状态设置为“up”来使用。适配器具有名称（例如“OfficeNet”）和类型（例如“Wintun”）。

```C
WINTUN_ADAPTER_HANDLE Adapter1 = WintunCreateAdapter(L"OfficeNet", L"Wintun", &SomeFixedGUID1);
WINTUN_ADAPTER_HANDLE Adapter2 = WintunCreateAdapter(L"HomeNet", L"Wintun", &SomeFixedGUID2);
WINTUN_ADAPTER_HANDLE Adapter3 = WintunCreateAdapter(L"Data Center", L"Wintun", &SomeFixedGUID3);
```

创建适配器后，我们可以通过启动会话来使用它：

```C
WINTUN_SESSION_HANDLE Session = WintunStartSession(Adapter2, 0x400000);
```

然后，`WintunAllocateSendPacket`和`WintunSendPacket`函数可用于发送数据包（[在example.c代码中使用`SendPackets`](https://git.zx2c4.com/wintun/tree/example/example.c)）：

```C
BYTE *OutgoingPacket = WintunAllocateSendPacket(Session, PacketDataSize);
if (OutgoingPacket)
{
    memcpy(OutgoingPacket, PacketData, PacketDataSize);
    WintunSendPacket(Session, OutgoingPacket);
}
else if (GetLastError() != ERROR_BUFFER_OVERFLOW) // 如果环形缓冲区已满，则悄悄丢弃数据包
    Log(L"Packet write failed");
```

`WintunReceivePacket`和`WintunReleaseReceivePacket`函数可用于接收数据包（[在example.c代码中使用`ReceivePackets`](https://git.zx2c4.com/wintun/tree/example/example.c)）：

```C
for (;;)
{
    DWORD IncomingPacketSize;
    BYTE *IncomingPacket = WintunReceivePacket(Session, &IncomingPacketSize);
    if (IncomingPacket)
    {
        DoSomethingWithPacket(IncomingPacket, IncomingPacketSize);
        WintunReleaseReceivePacket(Session, IncomingPacket);
    }
    else if (GetLastError() == ERROR_NO_MORE_ITEMS)
        WaitForSingleObject(WintunGetReadWaitEvent(Session), INFINITE);
    else
    {
        Log(L"Packet read failed");
        break;
    }
}
```

在某些高性能使用情况下，可能希望在转回等待读取等待事件之前，在`WintunReceivePackets`上旋转多个周期。

**强烈建议**阅读[**example.c简短示例**](https://git.zx2c4.com/wintun/tree/example/example.c)，了解如何组合一个简单的用户空间网络隧道。

下面的参考文档中记录了各种函数和定义。

## 参考

### 宏定义

#### WINTUN\_MAX\_POOL

`#define WINTUN_MAX_POOL   256`

包括零终止符在内的最大池名称长度

#### WINTUN\_MIN\_RING\_CAPACITY

`#define WINTUN_MIN_RING_CAPACITY   0x20000 /* 128kiB */`

最小环容量。

#### WINTUN\_MAX\_RING\_CAPACITY

`#define WINTUN_MAX_RING_CAPACITY   0x4000000 /* 64MiB */`

最大环容量。

#### WINTUN\_MAX\_IP\_PACKET\_SIZE

`#define WINTUN_MAX_IP_PACKET_SIZE   0xFFFF`

最大IP数据包大小

### Typedefs

#### WINTUN\_ADAPTER\_HANDLE

`typedef void* WINTUN_ADAPTER_HANDLE`

表示Wintun适配器的句柄

#### WINTUN\_ENUM\_CALLBACK

`typedef BOOL(* WINTUN_ENUM_CALLBACK) (WINTUN_ADAPTER_HANDLE Adapter, LPARAM Param)`

由WintunEnumAdapters为池中的每个适配器调用。

**参数**

- *Adapter*：适配器句柄，在此函数返回时将被释放。
- *Param*：传递给WintunEnumAdapters的应用程序定义的值。

**返回值**

非零以继续迭代适配器；零以停止。

#### WINTUN\_LOGGER\_CALLBACK

`typedef void(* WINTUN_LOGGER_CALLBACK) (WINTUN_LOGGER_LEVEL Level, DWORD64 Timestamp, const WCHAR *Message)`

由内部记录器调用以报告诊断消息

**参数**

- *Level*：消息级别。
- *Timestamp*：自1601-01-01 UTC以来的100ns间隔的消息时间戳。
- *Message*：消息文本。

#### WINTUN\_SESSION\_HANDLE

`typedef void* WINTUN_SESSION_HANDLE`

表示Wintun会话的句柄

### 枚举类型

#### WINTUN\_LOGGER\_LEVEL

`enum WINTUN_LOGGER_LEVEL`

确定记录级别，传递给WINTUN\_LOGGER\_CALLBACK。

- *WINTUN\_LOG\_INFO*：信息
- *WINTUN\_LOG\_WARN*：警告
- *WINTUN\_LOG\_ERR*：错误

枚举器

### 函数

#### WintunCreateAdapter()

`WINTUN_ADAPTER_HANDLE WintunCreateAdapter (const WCHAR * Name, const WCHAR * TunnelType, const GUID * RequestedGUID)`

创建新的Wintun适配器。

**参数**

- *Name*：适配器的请求名称。最多MAX\_ADAPTER\_NAME-1个字符的零终止字符串。
- *TunnelType*：适配器隧道类型的名称。最多MAX\_ADAPTER\_NAME-1个字符的零终止字符串。
- *RequestedGUID*：创建的网络适配器的GUID，然后以确定性地影响NLA生成。如果设置为NULL，则系统将随机选择GUID，因此每个新适配器都将创建一个新的NLA条目。它被称为“请求”GUID，因为它使用的API是完全未记录的，因此其使用可能会出现一些有趣的小问题。

**返回值**

如果函数成功，则返回值为适配器句柄。必须使用WintunCloseAdapter进行释放。如果函数失败，则返回值为NULL。要获取扩展的错误信息，请调用GetLastError。

#### WintunOpenAdapter()

`WINTUN_ADAPTER_HANDLE WintunOpenAdapter (const WCHAR * Name)`

打开现有的Wintun适配器。

**参数**

- *Name*：适配器的请求名称。最多MAX\_ADAPTER\_NAME-1个字符的零终止字符串。

**返回值**

如果函数成功，则返回值为适配器句柄。必须使用WintunCloseAdapter进行释放。如果函数失败，则返回值为NULL。要获取扩展的错误信息，请调用GetLastError。

#### WintunCloseAdapter()

`void WintunCloseAdapter (WINTUN_ADAPTER_HANDLE Adapter)`

释放Wintun适配器资源，并且如果适配器是使用WintunCreateAdapter创建的，则移除适配器。

**参数**

- *Adapter*：使用WintunCreateAdapter或WintunOpenAdapter获取的适配器句柄。

#### WintunDeleteDriver()

`BOOL WintunDeleteDriver()`

如果没有更多的适配器在使用，则删除Wintun驱动程序。

**返回值**

如果函数成功，返回值为非零。如果函数失败，返回值为零。要获取扩展的错误信息，请调用GetLastError。

#### WintunGetAdapterLuid()

`void WintunGetAdapterLuid(WINTUN_ADAPTER_HANDLE Adapter, NET_LUID *Luid)`

返回适配器的LUID。

**参数**

- *Adapter*：使用WintunOpenAdapter或WintunCreateAdapter获取的适配器句柄
- *Luid*：指向LUID的指针，用于接收适配器的LUID。

#### WintunGetRunningDriverVersion()

`DWORD WintunGetRunningDriverVersion(void)`

确定当前加载的Wintun驱动程序的版本。

**返回值**

如果函数成功，返回值为版本号。如果函数失败，返回值为零。要获取扩展的错误信息，请调用GetLastError。可能的错误包括以下内容：ERROR\_FILE\_NOT\_FOUND Wintun未加载

#### WintunSetLogger()

`void WintunSetLogger(WINTUN_LOGGER_CALLBACK NewLogger)`

设置记录器回调函数。

**参数**

- *NewLogger*：要用作新全局记录器的回调函数指针。NewLogger可能会同时从各种线程调用。如果记录需要序列化，则必须在NewLogger中处理序列化。设置为NULL以禁用。

#### WintunStartSession()

`WINTUN_SESSION_HANDLE WintunStartSession(WINTUN_ADAPTER_HANDLE Adapter, DWORD Capacity)`

启动Wintun会话。

**参数**

- *Adapter*：使用WintunOpenAdapter或WintunCreateAdapter获取的适配器句柄
- *Capacity*：环容量。必须在WINTUN\_MIN\_RING\_CAPACITY和WINTUN\_MAX\_RING\_CAPACITY（包括）之间。必须是2的幂。

**返回值**

Wintun会话句柄。必须使用WintunEndSession进行释放。如果函数失败，返回值为NULL。要获取扩展的错误信息，请调用GetLastError。

#### WintunEndSession()

`void WintunEndSession(WINTUN_SESSION_HANDLE Session)`

结束Wintun会话。

**参数**

- *Session*：使用WintunStartSession获取的Wintun会话句柄

#### WintunGetReadWaitEvent()

`HANDLE WintunGetReadWaitEvent(WINTUN_SESSION_HANDLE Session)`

获取Wintun会话的读取等待事件句柄。

**参数**

- *Session*：使用WintunStartSession获取的Wintun会话句柄

**返回值**

指针，用于在读取时等待可用数据的事件句柄。如果WintunReceivePackets返回ERROR\_NO\_MORE\_ITEMS（在重载下一段时间后），请在重试WintunReceivePackets之前等待此事件变为已发出。不要在此事件上调用CloseHandle - 它由会话管理。

#### WintunReceivePacket()

`BYTE* WintunReceivePacket(WINTUN_SESSION_HANDLE Session, DWORD *PacketSize)`

检索一个或一个数据包。在使用完数据包内容后，调用WintunReleaseReceivePacket以释放从此函数返回的数据包的内部缓冲区。此函数是线程安全的。

**参数**

- *Session*：使用WintunStartSession获取的Wintun会话句柄
- *PacketSize*：指针，用于接收数据包大小。

**返回值**

指向第3层IPv4或IPv6数据包的指针。客户端可以随意修改其内容。如果函数失败，返回值为NULL。要获取扩展的错误信息，请调用GetLastError。可能的错误包括以下内容：ERROR\_HANDLE\_EOF Wintun适配器正在终止；ERROR\_NO\_MORE\_ITEMS Wintun缓冲区已耗尽；ERROR\_INVALID\_DATA Wintun缓冲区损坏

#### WintunReleaseReceivePacket()

`void WintunReleaseReceivePacket(WINTUN_SESSION_HANDLE Session, const BYTE *Packet)`

在客户端处理接收到的数据包后释放内部缓冲区。此函数是线程安全的。

**参数**

- *Session*：使用WintunStartSession获取的Wintun会话句柄
- *Packet*：使用WintunReceivePacket获取的数据包

#### WintunAllocateSendPacket()

`BYTE* WintunAllocateSendPacket(WINTUN_SESSION_HANDLE Session, DWORD PacketSize)`

为要发送的数据包分配内存。在填充内存与数据包数据后，调用WintunSendPacket发送并释放内部缓冲区。WintunAllocateSendPacket是线程安全的，而WintunAllocateSendPacket的调用顺序定义了数据包的发送顺序。

**参数**

- *Session*：使用WintunStartSession获取的Wintun会话句柄
- *PacketSize*：确切的数据包大小。必须小于或等于WINTUN\_MAX\_IP\_PACKET\_SIZE。

**返回值**

返回指向内存的指针，用于准备第3层IPv4或IPv6数据包以便发送。如果函数失败，返回值为NULL。要获取扩展的错误信息，请调用GetLastError。可能的错误包括以下内容：ERROR\_HANDLE\_EOF Wintun适配器正在终止；ERROR\_BUFFER\_OVERFLOW Wintun缓冲区已满；

#### WintunSendPacket()

`void WintunSendPacket(WINTUN_SESSION_HANDLE Session, const BYTE* Packet)`

发送数据包并释放内部缓冲区。WintunSendPacket是线程安全的，但是WintunAllocateSendPacket的调用顺序定义了数据包的发送顺序。这意味着数据包在调用WintunSendPacket时并不一定会被发送。

**参数**

- *Session*：使用WintunStartSession获取的Wintun会话句柄
- *Packet*：使用WintunAllocateSendPacket获取的数据包

## 构建

**不要分发名为"Wintun"的驱动程序或文件，因为它们很可能会与官方部署发生冲突。而是分发[`wintun.dll`从wintun.net下载](https://www.wintun.net)。**

一般要求：

- [Visual Studio 2019](https://visualstudio.microsoft.com/downloads/)与Windows SDK
- [Windows Driver Kit](https://docs.microsoft.com/en-us/windows-hardware/drivers/download-the-wdk)

可以在Visual Studio中打开`wintun.sln`进行开发和构建。在启用未签名驱动程序加载之前，请确保运行`bcdedit /set testsigning on`然后重新启动。在Visual Studio中默认的运行序列（F5）将构建示例项目及其依赖项。

## 许可证

整个[存储库](https://git.zx2c4.com/wintun/)的内容，包括所有文档和示例代码，均为"版权所有©2018-2021 WireGuard LLC。保留所有权利。"源代码根据[GPLv2](COPYING)许可。从[wintun.net](https://www.wintun.net/)预构建的二进制文件则以更宽松的许可证发布，适用于分发到.zip文件中的更多形式的软件内。