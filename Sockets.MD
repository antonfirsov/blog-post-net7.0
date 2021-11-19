## Socket-level improvements

### Handle `SocketsHttpHandler` port exhaustion by utilizing auto-reuse port range on Windows
When opening lots of HTTP/1.1 connections from the same machine simultaneously, you may face socket error 10055 `WSAENOBUFS`, typically around ~16K connections.
Normally, the network stack chooses a port that isn’t already bound to another socket, meaning that the number of simultaneusly open connections is limited by the [dynamic port range](https://docs.microsoft.com/en-us/windows/client-management/troubleshoot-tcpip-port-exhaust#default-dynamic-port-range-for-tcpip).
Since the port is a 16 bit number, the port range (and the number of connections) have a theoretical limit of ~64K.

Windows [introduced](https://support.microsoft.com/en-us/topic/reliability-and-scalability-improvements-in-tcp-ip-for-windows-8-1-and-windows-server-2012-r2-82b226f4-cadc-7676-67db-2195516b7956) an "auto-reuse port range" to address this problem, as long as the remote endpoints differ in IP addresses and/or ports.
[dotnet/runtime#48219](https://github.com/dotnet/runtime/issues/54903) added a change that auto-enables `SO_REUSE_UNICASTPORT`, so ports will be reused as long as:
- The connection’s full 4-tuple of (local port, local address, remote port, remote address) is unique
- The auto-reuse port range is configured on the machine. Because of strict backwards-compatibility requirements in Windows, this is an opt-in feature!

You can set the auto-reuse port range with the following PowerShell cmdlet:

```powershell
Set-NetTCPSetting -SettingName InternetCustom -AutoReusePortRangeStartPort <start port> -AutoReusePortRangeNumberOfPorts <num. of ports>
```

A reboot is required for the setting to take effect.

### An option to disable IPv6

Since .NET 5, we are using [DualMode](https://docs.microsoft.com/en-us/dotnet/api/system.net.sockets.socket.dualmode) sockets in `SocketsHttpHandler`.
This allows us to handle IPv4 traffic from an IPv6 socket, and considered to be a favorable practice by [RFC1933](https://tools.ietf.org/html/rfc1933).
On the other hand, we had several reports from users having trouble connecting through VPN tunnels which do not support IPv6 and/or dual-stack properly.
To mitigate these issues and other potential problems with IPv6,
[dotnet/runtime#55012](https://github.com/dotnet/runtime/pull/55012) implemented a switch which can disable IPv6 globally for the entire .NET 6 process.

You can now set the environment variable `DOTNET_SYSTEM_NET_DISABLEIPV6` or the `System.Net.DisableIPv6` [runtime configuration setting](https://docs.microsoft.com/en-us/dotnet/core/run-time-config/) to `true`
if you experience similar problems and decide to mitigate them by disabling IPv6.

### New Span- and Task-based overloads in `System.Net.Sockets`

With the help of the community, we managed to bring `Socket` and related types close to API-complete in terms of Span, Task, and cancellation support.
A complete API-diff is  way too long to include into this blog post, you can find it [here](https://github.com/dotnet/core/blob/main/release-notes/6.0/api-diff/.Net/6.0.0_System.Net.Sockets.md).
We would like to thank [@gfoidl](https://github.com/gfoidl), [@ovebastiansen](https://github.com/ovebastiansen) and [@PJB3005](https://github.com/PJB3005) for their contributions!
