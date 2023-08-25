# Using-UniTask-instead-of-Thread-in-Socket

## Overview

The `Thread` class has been available since .NET Framework 1.0 and remains supported up to .NET 5.0. 
The `Task` class was introduced starting from .NET Framework 4.0 and continues to be supported in subsequent versions. 
Task received enhancements with the introduction of .NET 4.5 and further development in .NET Core and .NET 5.0, as well as subsequent versions. 
`UniTask` is a dedicated asynchronous programming library tailored for Unity game development. It holds an advantage in handling asynchronous operations within the Unity engine, providing a more lightweight implementation along with additional features.

Using the `Thread` class necessitates manual management of thread lifecycle, synchronization, and exception handling, 
which can lead to intricate and `error-prone` code. There is also a higher overhead associated with Thread, given the expense of creating and destroying threads. 
Moreover, excessive threads can result in `resource contention` and `performance` issues.

`Task` serves as a higher-level abstraction over threads, capable of `automatically` utilizing a thread pool to manage thread resources, 
reducing the need for manual thread creation and destruction. Task is better suited for `asynchronous programming`. It seamlessly employs the async/await 
keywords to facilitate handling asynchronous operations, avoiding callback hell. Task supports task organization and management, 
including waiting for a group of tasks to complete. Additionally, Task can be used in conjunction with the `Parallel` class to enable more convenient parallel operations.

In contemporary versions of C#, Task has become the more commonly used and recommended approach for multithreaded programming. It offers better utilization of system resources, higher levels of abstraction, convenience, and helps avoid pitfalls associated with multithreaded programming.

`UniTask` is a specialized asynchronous programming library designed explicitly for Unity game development. It holds an advantage in managing asynchronous operations within the Unity engine, offering a lighter-weight implementation and supplementary features. If you are engaged in asynchronous programming within a Unity game project, UniTask might be the more suitable option. However, in other domains, C# Task remains the more versatile approach for asynchronous programming.

I'm using a project that employs raw Threads in conjunction with UDP for communication with a lower-level device as an example to demonstrate how to replace Threads with UniTask.

In this demonstration, I will showcase how to substitute the conventional Thread approach with UniTask, showcasing the benefits of enhanced asynchronous communication in a Unity environment. This transition offers improved efficiency, seamless integration with Unity's asynchronous operations, and additional functionalities tailored for game development scenarios.
## Implement

### Connect
`Thread`
```csharp
public void Connect()
{
    ipSend = new IPEndPoint(IPAddress.Parse("192.168.1.252"), 1030);
    socket = new Socket(AddressFamily.InterNetwork, SocketType.Dgram, ProtocolType.Udp);
    var ipEndPoint = new IPEndPoint(IPAddress.Parse("192.168.1.3"), 1031);
    socket.Bind(ipEndPoint);
    isRuning = true;

    connectThread = new Thread(SocketReceive);
    connectThread.Start();
}
```
`UniTask`
```csharp
protected async UniTask ConnectAsync()
{
    _ipSend = new IPEndPoint(IPAddress.Parse("192.168.1.252"), 1030);
    _udpClient = new UdpClient(new IPEndPoint(IPAddress.Parse("192.168.1.3"), 1031));
    _isRunning = true;
    await UniTask.RunOnThreadPool(SocketReceiveAsync);
}
```
### Receive Data
`Thread`
```csharp
void SocketReceive()
{
    while (isRuning)
    {
        // There's some problem,but I don't wanna fix it because the author is not me.
        recvData = new byte[20];
        EndPoint clientEnd = new IPEndPoint(IPAddress.Any, 0);
        socket.ReceiveFrom(recvData, ref clientEnd);
        HandleReceiveData(ref recvData);
    }
}
```
`UniTask`
```csharp
private async UniTask SocketReceiveAsync()
{
    while (_isRunning)
    {
        var receiveResult = await _udpClient.ReceiveAsync();
        HandleReceiveData(receiveResult.Buffer);
    }
}
```
### Send Data
`Thread`
```csharp
protected void SendDataToPortByBinary(string writeData, bool isXOR = false)
{
    byte[] bytes = HexStringToBinary(writeData, isXOR);
    if (isRuning)
        int lenght = socket.SendTo(bytes, bytes.Length, SocketFlags.None, ipSend);
}

private void Function()
{
    SendDataToPortByBinary(data, true);
}
```
`UniTask`
```csharp
protected async UniTask SendDataToPortByBinaryAsync(string writeData, bool isXor = false)
{
    if (!_isRunning) return;
    await UniTask.RunOnThreadPool(() =>
    {
        var bytes = HexStringToBinary(writeData, isXor);
        try
        {
            _udpClient.Send(bytes, bytes.Length, _ipSend);
        }
        catch (Exception e)
        {
            Log.Log.Lower.Fatal($"SendDataToPortByBinaryAsync Failed + {e}");
        }
    });
}

private async UniTask Function()
{
    await SendDataToPortByBinaryAsync(data, true);
}
```
### Handle Data
`Thread`
```csharp
public virtual void HandleReceiveData(ref byte[] byteData)
{
}

public override void HandleReceiveData(ref byte[] byteData)
{
    // Do something
}
```
`UniTask`
```csharp
protected virtual UniTask HandleReceiveData(byte[] byteData)
{
    return UniTask.CompletedTask;
}

protected override async UniTask HandleReceiveData(byte[] byteData)
{
    // Do something
    // await UniTask.Delay(1000);
}
```
### Call
`Thread`
```csharp
Connect();
```
`UniTask`
```csharp
ConnectAsync().Forget();
```
### Destroy
`Thread`
```csharp
isRuning = false;

if (connectThread != null)
{
    connectThread.Interrupt();
    connectThread.Abort();
}

if (socket != null)
{
    socket.Close();
    socket.Dispose();
}
```
`UniTask`
```csharp
_isRunning = false;
_udpClient?.Close();
```
