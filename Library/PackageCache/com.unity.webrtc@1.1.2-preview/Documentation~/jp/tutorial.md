# チュートリアル

WebRTC パッケージの基本的な利用方法について説明します。


### 名前空間の追加

名前空間は `Unity.WebRTC` を指定します。

```CSharp
using UnityEngine;
using Unity.WebRTC;
```

### 初期化

`WebRTC` の利用時には初期化を行うために、 `WebRTC.Initialize()`を呼び出します。終了時には `WebRTC.Finalize()` を呼び出します。

```CSharp
public class MyPlayerScript : MonoBehaviour
{
    private void Awake()
    {
        // Initialize WebRTC
        WebRTC.Initialize();
    }
}
```

デフォルトではエンコーダーはハードウェアを使用します。ハードウェアエンコーダーではなく、ソフトウェアエンコーダーを使用する場合、初期化時に `EncoderType.Software` を指定します。
初期化後にエンコーダーを変更することはできません。

```CSharp
public class MyPlayerScript : MonoBehaviour
{
    private void Awake()
    {
        // Initialize WebRTC with software encoder
        WebRTC.Initialize(EncoderType.Software);
    }
}
```

### ローカルピアの作成

ローカルピアを作成し `RTCDataChannel` を取得します。`RTCDataChannel` を利用してバイナリデータの送受信を行うことができます。`RTCDataChannel` の初期化時もしくは終了時に処理を行う場合は、 `OnOpen` と `OnClose` のコールバックを登録してください。また、メッセージの受信を行う場合は `OnMessage` にコールバックを指定します。

```CSharp
    // Create local peer
    var localConnection = new RTCPeerConnection();
    var sendChannel = localConnection.CreateDataChannel("sendChannel");
    channel.OnOpen = handleSendChannelStatusChange;
    channel.OnClose = handleSendChannelStatusChange;
```

### リモートピアの作成

リモートピアを作成し、`OnDataChannel` にコールバックを設定します。

```CSharp
    // Create remote peer
    var remoteConnection = new RTCPeerConnection();
    remoteConnection.OnDataChannel = ReceiveChannelCallback;
```

### 通信経路候補の登録

ピアの接続を行うために、ICE (Interactive Connectivity Establishment) の交換が必要になります。各ピアで通信経路候補が発見されたら、 `OnIceCandidate` が呼び出されます。各ピアのコールバックで `AddIceCandidate` を呼び出して経路候補の登録を行います。


```CSharp
localConnection.OnIceCandidate = e => { !string.IsNullOrEmpty(e.candidate)
        || remoteConnection.AddIceCandidate(ref e); }

remoteConnection.OnIceCandidate = e => { !string.IsNullOrEmpty(e.candidate)
        || localConnection.AddIceCandidate(ref e); }

```

### シグナリング処理の流れ

ピア間で SDP の交換を行います。最初に `CreateOffer` でオファー SDP を作成します。オファー SDP を取得後、ローカルピアとリモートピアの双方に SDP を設定します。このとき `SetLocalDescription` と `SetRemoteDescription` を取り違えないよう気をつけてください。

オファー SDP の設定が完了したら、 `CreateAnswer` を呼び出してアンサー SDP を作成します。オファー SDP と同様に、アンサー SDP もローカルピアとリモートピアの双方に設定します。

```csharp
var op1 = localConnection.CreateOffer();
yield return op1;
var op2 = localConnection.SetLocalDescription(ref op1.desc);
yield return op2;
var op3 = remoteConnection.SetRemoteDescription(ref op1.desc);
yield return op3;
var op4 = remoteConnection.CreateAnswer();
yield return op4;
var op5 = remoteConnection.setLocalDescription(op4.desc);
yield return op5;
var op6 = localConnection.setRemoteDescription(op4.desc);
yield return op6;
```

### ICE の接続状態の確認

ピア間で SDP の交換を行うと、ICE の交換が開始されます。ICE の接続状態を確認するには `OnIceConnectionChange` のコールバックを利用します。

```CSharp
localConnection.OnIceConnectionChange = state => {
    Debug.Log(state);
}
```

### データチャネルの接続

ICE の交換が完了すると、`OnDataChannel` が呼び出され、他方のピアのデータチャネルが生成されます。
`OnMessage` コールバックを登録して、メッセージ受信時の処理を記述します。

```CSharp
RTCDataChannel receiveChannel;
void ReceiveChannelCallback(RTCDataChannel channel) 
{
    receiveChannel = channel;
    receiveChannel.OnMessage = HandleReceiveMessage;  
}
```

### メッセージの送信

ピア双方の `RTCDataChannel` がオープンすると、メッセージのやり取りが可能になります。送信可能なメッセージの形式は `string` もしくは `byte[]` を利用できます。

```csharp
void SendMessage(string message)
{
  sendChannel.Send(message);
}

void SendBinary(byte[] bytes)
{
  sendChannel.Send(bytes);
}
```

### メッセージの受信

メッセージ受信時には `OnMessage` に登録したコールバックが呼ばれます。`byte[]` 形式で取得することができ、文字列として扱う場合は以下のように変換します。

```csharp
void HandleReceiveMessage(byte[] bytes)
{
  var message = System.Text.Encoding.UTF8.GetString(bytes);
  Debug.Log(message);
}
```

### 終了処理

利用を終了するときには、`RTCDataChannel`と `RTCPeerConnection`の`Close()`を呼び出す必要があります。最終的に、オブジェクトを破棄したあと `WebRTC.Finalize()` を呼び出してください。

```csharp
private void OnDestroy()
{
  sendChannel.Close();
  receiveChannel.Close();
  
  localConnection.Close();
  remoteConnection.Close();
  
  WebRTC.Finalize();
}
```

### オーディオストリーミング

オーディオをストリーミングするためには、はじめにストリームのインスタンスを取得します。`Audio.CaptureStream()` を呼び出してください。

```csharp
audioStream = Audio.CaptureStream();
```

ピアにメディアトラックを追加します。`RTCRtpSender` のインスタンスは、メディアを破棄する際に利用します。

```csharp
    var senders = new List<RTCRtpSender>();
    foreach (var track in audioStream.GetTracks())
    {
        var sender = localConnection.AddTrack(track);
        senders.Add(sender);
    }
```

メディアの破棄は、 `RemoveTrack` メソッドを使用します。

```csharp
    foreach(var sender in senders)
    {
        localConnection.RemoveTrack(sender);
    }
```

`MonoBehaviour` の `OnAudioFilterRead` メソッド内で、`Audio` の `Update` メソッドを呼び出してください。

```csharp
    private void OnAudioFilterRead(float[] data, int channels)
    {
        Audio.Update(data, data.Length);
    }
```

> [!NOTE]
> `OnAudioFilterRead` メソッドを利用する場合は、 `AudioListener` コンポーネントと同じ GameObject に関連付ける必要があります。

あるいは、`AudioRenderer` を利用する方法もあります。

```csharp

    private void Start()
    {
        AudioRenderer.Start();
    }

    private void Update()
    {
        var sampleCountFrame = AudioRenderer.GetSampleCountForCaptureFrame();
        var channelCount = 2; // AudioSettings.speakerMode == Stereo
        var length = sampleCountFrame * channelCount;
        var buffer = new NativeArray<float>(length, Allocator.Temp);
        AudioRenderer.Render(buffer);
        Audio.Update(buffer.ToArray(), buffer.Length);
        buffer.Dispose();
    }

```

### ビデオストリーミング

`MediaStream` を使ってビデオストリームをキャプチャするために、`Camera` の `CaptureStream()` を呼び出します。




