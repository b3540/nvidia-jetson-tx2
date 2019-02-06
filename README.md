# nvidia-jetson-tx2
JETSON TX2のサンプルの、[jetson-inference](https://github.com/dusty-nv/jetson-inference)のimagenet-cameraの画像認識結果をAzure IoT Hubに送信する改造手順を説明します。 

## 必要なもの
[NVIDIA JETSON TX2開発者キット](https://www.nvidia.com/ja-jp/autonomous-machines/embedded-systems-dev-kits-modules/) 
Microsoft Azure サブスクリプション ※[無料お試し](https://azure.microsoft.com/ja-jp/) で可
## 事前準備 
- JETSON TX2にJetpackを[https://qiita.com/akrian/items/54c321bfa95d8eaef17d](https://qiita.com/akrian/items/54c321bfa95d8eaef17d)等を参考にインストール 
- IoT ALGYANメンバー武藤先生作成の[ドキュメント](https://docs.google.com/document/d/1-1Puk72SiXX1sQra-2dGf5clSbMn4eUiFpOpQc6AsDY/)を参考に、etson-inferenceをインストールし、imagenet-cameraの動作を確認  
- [IoT Kit HoL V4](http://aka.ms/iotkitholv4)の[第一章](https://1drv.ms/p/s!Aihe6QsTtyqct5NQBGA32Y7fOV06hA)を参考に、Azure IoT Hubを作成 
## 改造作業
### Azure IoT Device SDK for Cのインストール
認識結果をIoT Hubに送信するのに必要なSDKライブラリをインストールします。jetson-inference/imagenet-cameraはC言語でプログラミングされているので、C言語向けのSDKを使います。 
JETSON TX2のシェルで以下のコマンドを実行していきます。
```shell
sudo add-apt-repository ppa:azureiotsdklinux/ppa-azureiot
sudo apt-get update
sudo apt-get install azure-iot-sdk-c-dev
``` 
以上を実行することにより、/usr/include、/usr/libにAzure IoT SDK for Cのヘッダーファイル、ライブラリがインストールされます。 

### ビルドルールの変更
Azure IoT Device SDKをコンパイル、リンクするのに必要な改造をCMakeList.txtに加えます。 
jetson-inrerence/imagenet-cameraのCMakeList.txtを、このリポジトリの[CMakeLists.txt](./CMakeLists.txt)の内容で全て上書きします。 
### ソースコードの変更 
imagenet-camera/imagenet-camera.cppの173行目付近 
```C
			if( display != NULL )
			{
				char str[256];
				sprintf(str, "TensorRT build %i.%i.%i | %s | %s | %04.1f FPS", NV_TENSORRT_MAJOR, NV_TENSORRT_MINOR, NV_TENSORRT_PATCH, net->GetNetworkName(), net->HasFP16() ? "FP16" : "FP32", display->GetFPS());
				//sprintf(str, "TensorRT build %x | %s | %04.1f FPS | %05.2f%% %s", NV_GIE_VERSION, net->GetNetworkName(), display->GetFPS(), confidence * 100.0f, net->GetClassDesc(img_class));
				display->SetTitle(str);	
			}	
		}	
```
の、display->SetTitle(str);の直下の '}'の次に、[imagenet-camera.cpp.fragment3](./imagenet-camera.cpp.fragment3)の中身を挿入します。 以下の様になります。
```C
				display->SetTitle(str);	
			}	
			if (azureIoTHubReady)
			{
				char send_message[1024];
				struct tm *tm;
				struct timeval myTime;
				gettimeofday(&myTime, NULL);
				tm = localtime(&myTime.tv_sec);
				char current_time_str[32];
				sprintf(current_time_str, "%04d-%02d-%02dT%02d:%02d:%02d.%03ld", tm->tm_year + 1900, tm->tm_mon + 1, tm->tm_mday, tm->tm_hour, tm->tm_min, tm->tm_sec, myTime.tv_sec);
				sprintf(send_message, "{\"timestamp\":\"%s\",\"confidence\":%5.3f,\"imageclass\":\"%s\"}", current_time_str, confidence, net->GetClassDesc(img_class));
				azureIoTHubSend(send_message);
			}
```
このコードは、画像認識の結果を認識した時間（msec単位）Azure IoT Hubに送信するためのJSON形式に変更しています。 
次に、55行目付近の 
```C
int main( int argc, char** argv )
{
	printf("imagenet-camera\n  args (%i):  ", argc);
```
のprintf文の前に、[imagenet-camera.cpp.fragment2](./imagenet-camera.cpp.fragment2)を挿入します。 
挿入後、以下の様になります。
```C
int main(int argc, char **argv)
{
	bool azureIoTHubReady = false;
	azureIoTHubReady = azureIoTHubSetup();

	printf("imagenet-camera\n  args (%i):  ", argc);
```
このコードは、Azure IoT Hubへの接続を初期化するものです。接続が成功すれば、azureIoTHubReadyにtrueが代入され、画像認識結果を送信するコードが実行されます。 
次に、上の2つの改造に出てきた関数や変数を挿入します。 imagenet-camera.cppの35行目付近の
```C
nclude "cudaNormalize.h"
#include "cudaFont.h"
#include "imageNet.h"


#define DEFAULT_CAMERA -1	// -1 for onboard camera, or change to index of /dev/video V4L2 camera (>=0)	
```
#include "imageNet.h"の後に、[imagenet-camera.cpp.fragment1](./imagenet-camera.cpp.fragment1) を挿入します。 
この改造で、Azure IoT Hubに認識結果を送信するために必要な関数群が実装されます。画像認識は動画のフレームごとに行われ、そのままIoT Hubに送信すると、IoT Hubのスケール設定がF1の場合は、あっという間に一日の上限に達してしまうので、一旦認識結果をバッファリングし、メッセージが7172バイトを超える、または、前回送ったときから10秒間たった時に送信するという処理を行っています。最大メッセージ長と送信間隔を変えたい場合は、それぞれ、AZURE_MESSAGE_LENGTH_MAXと、g_intervalの値を変えてください。 
### Azure IoT Hubへの接続情報設定
[IoT Kit HoL V4](http://aka.ms/iotkitholv4)の[第一章](https://1drv.ms/p/s!Aihe6QsTtyqct5NQBGA32Y7fOV06hA)の説明を参考に、新しくデバイスを一つ登録して、接続文字列をコピペします。 
imagenet-camera.cppの最後に挿入したコードの50秒目の
```C
/* Paste in your device connection string  */
static const char *connectionString = "<< Connection String for Azure IoT Hub >>";
```
コピペした接続文字列で<< Connection String for Azure IoT Hub >>の部分を上書きします。以下の様な形式になります。
```C
/* Paste in your device connection string  */
static const char *connectionString = ">HostName=XXX.azure-devices.net;DeviceId=jetson;SharedAccessKey=xxxx";
```
以上で、imagenet-camera.cppの改造は終わりです。 
※この改造は、[Azure IoT Device SDK for Cのサンプル](https://github.com/Azure/azure-iot-sdk-c/tree/master/iothub_client/samples/iothub_convenience_sample)をベースにしています。imagenet-camera.cpp.fragment1の中には、Azure IoT Hubからデバイスへのコマンド送信、メソッドコール、Device TwinのDesired Properties更新の受信なども含まれています。皆さんのアイデアでこれらの機能を活用してみてくださいね。 

### ビルドと実行 
jetson-inferenceの通常のビルドに従って変更したソースをビルドします。 
```shell
cd jetson-inference/build
make
```
ビルド終了後
```shell
cd aarch64
./imagenet-camera
```
で実行開始です。Device ExplorerやVS CodeのAzure IoT Hub Toolkit等で、IoT Hubへの受信内容を確認してください。 
※本サンプルはAMQPを使っています。実行しているネット環境がAMQPに対応していない場合は、HTTPやMQTTに変えてください。 
## 2019/2/6現在の不具合への対応
apt-get installでインストール可能な公開されているAzure IoT Device SDKライブラリに不備があり、ビルド中のリンクで失敗します。リンクエラーになった場合は、以下の手順で対応してください。  
※ 基本はhttps://github.com/azure/azure-iot-sdk-cをビルドしてライブラリを上書きします。 
### SDKのビルド 
SDKをビルドするのに必要な基本ツール、ライブラリをインストールします。
```shell
sudo apt-get update
sudo apt-get install -y git cmake build-essential curl libcurl4-openssl-dev libssl-dev uuid-dev
```
Githubからクローンしてビルドします。
```shell
git clone --recursive https://github.com/Azure/azure-iot-sdk-c/tree/master/iothub_client/samples/iothub_convenience_sample
cd azure-iot-sdk-c
mkdir cmake
cd cmake
```
ビルドが完了すると、cmake/iothub_clinetの下にライブラリができるので、
```shell
cd iothub_client
sudo cp -r lib*.a /usr/lib
```
この後、jetson-inferenceを再ビルドしてください。 

## Stream Analytics での処理
JETSON TX2を接続したIoT HubのバックエンドとしてStream Analyticsを利用する場合には、クエリーは、このサイトの[asa.sql](./asa.sql)を使ってください。 
Stream Analyticsの使い方は、[IoT Kit HoL V4](http://aka.ms/iotkitholv4)の[第三章](https://1drv.ms/p/s!Aihe6QsTtyqct5NPsPTykYx8VQ6aNw)を参照してください。　
## IoT Centralとの接続
IoT HubやStream Analyticsとかめんどくせーって方は、IoTのSaaSである、IoT Centralの利用をお勧めします。 
ただし、IoT Centralは、セキュアにデバイスの接続を自動化するDevice Provisioning Service対応なので、接続文字列の取得には一工夫が必要です。 
### IoT Centralの作成
Matsujirushiさんの[Rebuttonに関するブログ](http://matsujirushi.hatenablog.jp/entry/2019/01/23/171257)を参考に、IoT Centralを作成してください。 
JETSONをつなぐために、一つ新しくデバイスを登録し、スコープID、デバイスID、キーを用意します。
### 接続文字列
接続文字列は、JETSON上のコマンドで生成します。コマンドはNode.js上で動くので、先ずは、JETSONにNode.jsをインストールします。 
※  [インストールの参考資料](https://github.com/nodesource/distributions/blob/master/README.md#debinstall)
```shell
curl -sL https://deb.nodesource.com/setup_10.x | sudo -E bash -
sudo apt-get install -y nodej
```
次に、接続文字列生成用のコマンドを入力します。
```shell
sudo npm i -g dps-keygen
```
これで必要なツールが入るので、
```shell
dps-keygen <key> <deviceId> <scope id>
```
と、IoT Centralから取得したキー（PrimaryでもSecondaryでも可）、デバイスID、スコープIDを引数にして実行すると、接続文字列が表示されます。 
表示された接続文字列で、imagenet-camera.cppの<< Connection String for Azure IoT Hub >>を修正して、ビルド＆実行します。
