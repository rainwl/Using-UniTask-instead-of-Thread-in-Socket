# Using-UniTask-instead-of-Thread-in-Socket

## Overview


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
