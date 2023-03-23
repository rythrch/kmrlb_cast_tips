# castaliatips
[Castalia](https://github.com/boulis/Castalia)
に関するtipsを、簡単にまとめます。  
主に、以下の内容について説明します。
- MACレイヤプロトコルモジュールの追加・修正
- Wake-up Radio方式の実装


## インストール
- **Castalia - Installation.docx**を参照してください。**OMNeT++のバージョンに注意（4.3 to 4.6）**。
- シミュレーション等の実行に際して、**Python2**が必要です（Castalia/Castalia/bin/のスクリプトファイルがPython2で記述されているため）。それらスクリプトファイルのShebangも適宜変更してください。

## マニュアル
必ず**Castalia - User Manual.docx**に目を通しておいてください。シミュレーションの実施方法や、独自のプロトコルの実装方法について学べます。
以下の内容も、おおよそマニュアルに記載されています。
現状、Castaliaに関する日本語のリファレンスは存在しないので、頑張って英語を読んでください。

## シミュレーションの実施と結果の確認
- マニュアルの3章を参照のこと。  
- いわゆるシミュレーションシナリオは`.ini`ファイルに記述します。
- `-r`コマンドで同一シミュレーションの実施回数を指定できます。実装結果は1つのファイルに集約されます。筆者がNS-2からCastaliaに鞍替えした理由の最たるものです。
- `CastaliaResults`を用いて、各項目の結果を表示する際のコマンドについて。何も指定しないと、各ノードの結果の平均が表示されます。`--sum`を指定すると、各ノードの結果の合計が表示されます。`-n`を指定すると、ノードごとの結果が表示されます。
- `CastaliaResults`から`CastaliaPlot`にパイプすることで、リザルトファイルからgnuplotの図を直接生成する機能が紹介されていますが、使い勝手はあまり良くありません。それよりも、それらのスクリプトを参考にして、gnuplotに入力するためのデータファイルを自動生成するスクリプトを作成するのが賢明かと思われます。

## MACレイヤプロトコルの実装
### モジュールの追加・変更
マニュアルの5章に従って、モジュールならびにソースコードの追加・修正を行ってください。  

モジュールは`.ned`ファイルによって定義されます。
MACレイヤプロトコルモジュールを追加する場合、`Castalia/src/node/communication/mac/`にディレクトリを作成し、そこで`.ned`ファイルを作成します。

```ned
package node.communication.mac.myMac;

simple MyMAC like node.communication.mac.iMac {
 parameters:
	bool collectTraceInfo = default (false);
	int macMaxPacketSize = default (0);		
	int macPacketOverhead = default (8);	
	int macBufferSize = default (32);
	...
										   
 gates:
	output toNetworkModule;
	output toRadioModule;
	input fromNetworkModule;
	input fromRadioModule;
	input fromCommModuleResourceMgr;
}
```
同じディレクトリに`.cc`,`.h`ファイルを作成し、モジュールを実装します。
`.cc`ファイルでは、以下のようにしてモジュールの登録を明示します。  
```c++
Define_Module(MyMAC);
```

Castaliaをリビルドすることで、変更が反映されます。
```
Castalia $ make clean
Castalia $./makemake
Castalia $ make
```

### VirtualMac
MACレイヤプロトコルモジュールクラスは**VirtualMac**クラスを継承します。すなわち、**VirtualMac**クラスの作法に従って、MACレイヤプロトコルの動作を記述します。

### 主なメソッドの紹介
- `startup()`では、主に`.ned`ファイルで定義したパラメータの読み込みを実施します。
`.ned`ファイルにあるパラメータにはデフォルト値が設定されていますが、`.ini`ファイル内や、あるいはシミュレーションの実行時に、それらの値を変更することができます（デフォルト値が存在しないものについては、値を指定しないとエラーになる）。

```c++
void MyMAC::startup()
{
	...
	randomTxOffset = ((double)par("randomTxOffset")) / 1000.0; // convert msecs to secs
	...
}
```  
---
- ルーティングレイヤプロトコルが送出したパケットは`fromNetworkLayer(cPacket *pkt, int dstAddr)`に通されます。通常、パケットは即座には送出されず、一時的にバッファされます。
```c++
void MyMAC::fromNetworkLayer(cPacket * netPkt, int destination)
{
	// Create a new MAC frame from the received packet and buffer it (if possible)
	MyMacPacket *macPkt = new MyMacPacket("MyMAC data packet", MAC_LAYER_PACKET);
	encapsulatePacket(macPkt, netPkt);
	macPkt->setType(DATA_MYMAC_PACKET);
	macPkt->setSource(SELF_MAC_ADDRESS);
	macPkt->setDestination(destination);
	if (bufferPacket(macPkt)) {
		attemptTx();
	} else {
		//We could send a control message to upper layer to inform of full buffer
	}
}
```  
---
- 受信したパケットは、Radioレイヤを経て`fromRadioLayer(cPacket *pkt, double RSSI, double LQI)`に通されます。パケットタイプに応じて、処理を切り替えるようにします。
自身に宛てられたアプリケーションデータパケットについては、`toNetworkLayer()`を介して上位レイヤに引き継ぎます。
```c++
void MyMAC::fromRadioLayer(cPacket * pkt, double RSSI, double LQI)
{
	MyMacPacket *macPkt = dynamic_cast < MyMacPacket * >(pkt);
	if (macPkt == NULL)
		return;

	int source = macPkt->getSource();
	int destination = macPkt->getDestination();

	switch (macPkt->getType()) {

		/* received a RTS frame */
		case RTS_MYMAC_PACKET:{
			...
			break;
		}

		/* received a CTS frame */
		case CTS_MYMAC_PACKET:{
			...
			break;
		}
		
		case DATA_MYMAC_PACKET:{
		// Forward the frame to upper layer first
		if (isNotDuplicatePacket(macPkt))
			toNetworkLayer(decapsulatePacket(macPkt));
		...

```
---

- `toRadioLayer(cMessage *msg)`によって、パケットならびにコマンド（制御信号）をRadioレイヤに引き渡します。下のコードは、実質的に「MACレイヤにおける送信」を意味します。
```c++
toRadioLayer(packet);
toRadioLayer(createRadioCommand(SET_STATE, TX));

```
---

- MACレイヤプロトコルの動作（例えば、CSMA/CA）を実現する上で、タイマ機能は欠かせません。詳細については、マニュアル5.1.2項を参照のこと。  
タイマのセットは`setTimer(int index, simtime_t time)`で行います。
指定した時間が経過すると、`timerFiredCallback(int index)`が呼び出されます。
タイマのキャンセルは`cancelTimer(int index)`で行います。
```c++
void MyMAC::timerFiredCallback(int timerIndex)
{
	switch (timerIndex) {

		case PERFORM_CCA: {
			if (macState == MAC_STATE_GTS || macState == MAC_STATE_SLEEP) break;
			CCA_result CCAcode = radioModule->isChannelClear();
			if (CCAcode == CLEAR) {
			...

```
---

## パケットの実装
新たなパケットクラスを作成する場合、それを一からコーディングする必要はありません。
マニュアル5.2.2項やデフォルトで実装されているMACレイヤプロトコルを参考にして、新たな`.msg`ファイルを作成してください。
`.msg`ファイル内で、パケットの各フィールドを定義します。その上で、リビルド
```
Castalia $ make clean
Castalia $./makemake
Castalia $ make
```
すると、パケットクラスのソースコードが自動生成されます。  

例えば、`MyPacket.msg`を作成しておくと、リビルドによって`MyPacket_m.cc`と`MyPacket_mc.h`が自動生成されます。  
リビルドに際して、あらかじめモジュールのヘッダ等で
```
#include "MyPacket_m.h"
```
を記述しておく必要があります。  

自動生成されるパケットクラスには、ゲッタ・セッタ関数等が含まれます。
基本的に、パケットの生成時や送信時にフィールドに値をセットし、受信時にそれらをゲットすることになるでしょう。
なお、パケットのサイズは、
```c++
ackPacket->setByteLength(ACK_PKT_SIZE);
```
のように明示的に指定する必要があります。このサイズは、パケットクラスにおけるフィールドの数や型とは関係ありません。

## ロギング
ログに残したい内容は、適宜、`trace()`に書き出します。
```c++
trace() << "CSMA/CA random backoff value: " << rnd << ", in " << CCAtime << " seconds";
```
シミュレーション実施時にログファイルを出力するには、`.ini`ファイル内で
```
SN.node[*].Communication.MAC.collectTraceInfo = true
```
を記述します。デフォルトでは、この値はfalseです。


## アウトプット
シミュレーション結果として出力したい項目を、独自に定義することができます。
詳細については、マニュアル5.1.1項を確認してください。
`declareOutput(const char *)`で項目を定義し、`collectOutput(const char *descr, const char *label, double amt)`で任意の項目の値をインクリメントします。
シミュレーション期間中に値が都度更新される場合を除いて、これらの設定は、主にシミュレーション終了時に呼び出される`finishSpecific()`関数内で行われます。  

MACレイヤプロトコルモジュールインスタンスは各ノードに属しているため、ここでのアウトプットは、当然ながら各ノードに関する情報のみを問題とします。
各ノードのアウトプットの平均は、先述のように、「何もコマンドを指定しない」ことで得られます。
しかしながら、例えば「各ノードのアウトプットの分散」を直接に得ようとする場合、その算出に際して、各ノードのモジュールにアクセスする必要があるでしょう。
以下は、`void MyMAC::finishSpecific()`関数内で、各ノードの消費電力量の分散を`collectOutput()`するためのコードです。
具体的には、ある一つのノード（`self == 0`、ここではシンクノードのこと）が、各ノードの`ResourceManger`モジュールに順番にアクセスし、消費電力量の情報を取得していきます。
```c++
void MyMAC::finishSpecific()
{	
	if (self == 0) {
		int numNodes = getParentModule()->getParentModule()->getParentModule()->par("numNodes");;
		double sumSpentEnergy = 0.0;
		double sumSpentEnergySquare = 0.0;
		
		cTopology *topo;
		topo = new cTopology("topo");
		topo->extractByNedTypeName(cStringTokenizer("node.Node").asVector());
		
		for (int i = 0; i < numNodes; i++) {
			ResourceManager *resModule = dynamic_cast<ResourceManager*>
				(topo->getNode(i)->getModule()->getSubmodule("ResourceManager"));
			double spentEnergy = resModule->getSpentEnergy();
			sumSpentEnergy += spentEnergy;
			sumSpentEnergySquare += pow(spentEnergy, 2.0);
		}
		delete(topo);
		
		double spentEnergyVariance = sumSpentEnergySquare/numNodes - pow(sumSpentEnergy, 2.0);
		declareOutput("Variance of Spent Energy in Nodes");
		collectOutput("Variance of Spent Energy in Nodes", "", spentEnergyVariance);
	}
}
```
実のところ、MACレイヤプロトコルモジュールにおいて、このアウトプットの計算を行う必要性はないと思います。`ResourceManger`モジュールの`finishSpecific()`内で、同様の処理を実施する方が適当でしょう。







## Wake-up Radio方式の実装のヒント
CastaliaはWake-up Radio方式の実装をサポートしていないので、何か上手いやり方を考えなければなりません。
筆者は、楽をしたいので、最も安直な方法を思いつきました。
すなわち、Wake-up TransceiverとMain Transceiverの切り替えを、
- 搬送周波数
- 受信モード
- 送信レベル
- （CCAの閾値）  
の変更によって表現する、というものです。こんなもので果たして大丈夫なのかどうかは、わかりません。


### パラメータの切り替え
Radioレイヤにおけるパラメータの変更は、`toRadioLayer()`を介します。
```c++
void MyMAC::TransitionToWurx()
{
	toRadioLayer(createRadioCommand(SET_STATE, RX)); 
	toRadioLayer(createRadioCommand(SET_CARRIER_FREQ, wurCarrierFreq));
	toRadioLayer(createRadioCommand(SET_MODE, "WUR"));
	toRadioLayer(createRadioCommand(SET_TX_OUTPUT,wurTxDbm));
	toRadioLayer(createRadioCommand(SET_CCA_THRESHOLD,wurCcaTh));
}
```

```c++
void MyMAC::TransitionToMainrx()
{
	toRadioLayer(createRadioCommand(SET_STATE, RX)); 
	toRadioLayer(createRadioCommand(SET_CARRIER_FREQ, mainCarrierFreq));
	toRadioLayer(createRadioCommand(SET_MODE, "MAIN"));
	toRadioLayer(createRadioCommand(SET_TX_OUTPUT,mainTxDbm));
	toRadioLayer(createRadioCommand(SET_CCA_THRESHOLD,mainCcaTh));
}
```

ここで、Radioモード`MAIN`,`WUR`、また、送信出力dBmは、後述のパラメータファイルで、あらかじめ指定しておく必要があります。

### Radioパラメータファイル
Radioパラメータファイルは、以下の5つの項目を指定します。各項目の詳細については、マニュアル4.2.1項を参照のこと。

- RX MODES
- TX LEVELS
- DELAY TRANSITION MATRIX
- POWER TRANSITION MATRIX
- SLEEP LEVELS  

パラメータファイルは`Castalia/Simulations/Parameters/Radio/`に配置します。.iniファイル内で、
`
SN.node[*].Communication.Radio.RadioParametersFile = "(ファイルパス)" 
`
と指定することで、設定がシミュレーションに反映されます。  


`RX MODES`と`TX LEVELS`について。　

`RX MODES`には、それぞれ固有の名称を指定します。`SET_MODE`コマンドを送信する際、引数でこの名称を指定します。  
`TX LEVELS`は送信出力と消費電力の対応を表します。`SET_TX_OUTPUT`コマンドを送信する際、引数で指定できるdBm値は、この`TX LEVELS`に記載されているものに限られます。

筆者は、結局、次のようなパラメータファイルを使用しました。各種の消費電力については、先行研究で紹介されていた実験の諸元を拝借しました。  
このパラメータファイルにはいくつか問題点があります。
WURモードのmodulationTypeをIDEALとしていること（ASKは用意されていない？）や、実装を簡単にするためにRXとTXの切り替えにかかる時間を0にしていること（実際にはあり得ない）、などです。
```
RX MODES
# Name, dataRate(kbps), modulationType, bitsPerSymbol, bandwidth(MHz), noiseBandwidth(KHz), noiseFloor(dBm), sensitivity(dBm), powerConsumed(mW)
WUR, 10, IDEAL, 1, 20, 194, -100, -76, 0.001071
MAIN, 250, PSK, 4, 20, 194, -100, -95, 33.6

TX LEVELS
Tx_dBm 10 7
Tx_mW 90 31.2

DELAY TRANSITION MATRIX
#	RX	TX	SLEEP
RX	-	0.0	0.194
TX	0.0	-	0.194
SLEEP	0.05	0.05	-

POWER TRANSITION MATRIX
#       RX      TX      SLEEP
RX	-	0.0	0.0
TX	0.0	-	0.0
SLEEP	0.0	0.0	-

SLEEP LEVELS
idle 1.4, -, -, -, -
```


### 切り替えのタイミング
Wake-up Transceiverを用いてWake-up Call（WuC）パケットを送信したあと、トランスミッタノードはMain Transceiverでの動作に移行します。
この動作を、
```c++
toRadioLayer(wucPacket);
toRadioLayer(createRadioCommand(SET_STATE, TX));
TransitionToMainrx();
```
と記述した場合、Radioレイヤにおいて送信が完了する前に、`toRadioLayer()`によって搬送周波数等が変更されるため、WuCパケットが(Wake-up Receiverで待ち構えている)相手に届きません。
従って、`TransitionToMainrx()`は、Radioレイヤにおいて送信が完了するのを確認してから呼ぶ必要があります。  

Radioモジュール（`Radio.cc`）において、送信が完了したタイミングで、その旨を通知するコマンドをMACモジュールに送ることによって、この機構を実現できそうです。
```c++
/* Radio.cc */
void Radio::completeStateTransition()
{
	state = (BasicState_type) changingToState;
	...
	
	switch (state) {
		...

		case RX:{
			...
			
			RadioControlMessage *msg = new RadioControlMessage("Done Trans to RX", RADIO_CONTROL_MESSAGE);
			msg->setRadioControlMessageKind(DONE_TRANS_TO_RX);
			send(msg, "toMacModule");
			break;
		}
		...
```

```c++
/* MyMAC.cc */
int MyMAC::handleRadioControlMessage(cMessage * msg)
{
	RadioControlMessage *radioMsg = check_and_cast <RadioControlMessage*>(msg);
	if (radioMsg->getRadioControlMessageKind() == DONE_TRANS_TO_RX){
		TransitionToMainrx();
	}
	...
```

ACKを返送して、Main TransceiverからWake-up Transceiverに切り替わる場合についても、同様のことが言えるでしょう。


## その他
- C++ということもあるので、パケットオブジェクトの解放については、よく注意すること。解放が適切になされていないと、シミュレーション終了時に、ターミナル上でその旨を伝えるメッセージが吐き出されたような気がします。

## ライセンス
このREADMEファイルは、[Castalia](https://github.com/boulis/Castalia)の[Academic Public License](https://github.com/boulis/Castalia/blob/master/Castalia/LICENSE)に準拠します。  
このREADMEファイルには、[Academic Public License](https://github.com/boulis/Castalia/blob/master/Castalia/LICENSE)によってライセンスされた、[Castalia](https://github.com/boulis/Castalia)のソースコード（Copyright: National ICT Australia,  2007 - 2010）の一部を抜粋・改変したものが含まれます。

23/3/2023
