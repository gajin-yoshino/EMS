
# Autopilot デバイス を Graph API で操作する

Autopilot はデバイス プロビジョニングの新しい形として有用だけど、運用観点ではまだ色々と物足りないところがあります。その一つが、ホスト名の指定が簡単な[テンプレート](https://learn.microsoft.com/ja-jp/mem/autopilot/profiles)でしか与えることができないところです。（ex. `Auto-%RAND:4%` 接頭辞 Auto- に続けて４桁の乱数）

別解として、Autopilot デバイス オブジェクト の プロパティ ”デバイス名” を指定することで任意にホスト名を設定することができるけれど、この指定は１レコードずつ行わなければならないという不便さがあり、一長一短というかんじです。

しかし、Graph API からこのプロパティを更新することができればバルク処理に道が開けるので、その実現性を検証してみましょう。

Autopilot に関する Graph API の公式ドキュメントはこちらになります。

https://learn.microsoft.com/ja-jp/graph/api/resources/intune-enrollment-windowsautopilotdeviceidentity?view=graph-rest-1.0

## Graph Powershell SDK のセットアップ

Graph API PowerShell SDK は Graph API のラッパーとして機能して、API の操作を PowerShell から簡便に実行させてくれる便利なツールセットです。 こちらを使って Autopilot デバイス情報を操作していきます。

### まずはインストール

https://learn.microsoft.com/en-us/powershell/microsoftgraph/installation?view=graph-powershell-1.0

```powershell
PS > Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
PS > Install-Module Microsoft.Graph -Scope CurrentUser
PS > Get-InstalledModule Microsoft.Graph

Version              Name                                Repository           Description
-------              ----                                ----------           -----------
1.19.0               Microsoft.Graph                     PSGallery            Microsoft Graph PowerShell module
```

最後のコマンドの結果としてモジュールの情報が出力されれば導入は成功しています。

この例では、Powershell を実行しているユーザーだけが利用できるように導入しているので、その端末上の全ユーザーに利用させたい場合は `-Scope` の引数を変更してください。

### Graph への接続

```powershell
PS > Connect-MgGraph -Scopes "DeviceManagementServiceConfig.ReadWrite.All"
Welcome To Microsoft Graph!
```

`-Scope` の引数は Graph API にアクセスする際に適用する権限になります。 必要な権限は API のドキュメント（例 [windowsAutopilotDeviceIdentities](https://learn.microsoft.com/ja-jp/graph/api/intune-enrollment-windowsautopilotdeviceidentity-list?view=graph-rest-1.0#prerequisites)）に記載があるので、最小権限の原則にそって必要なレベルを選択してください。 

コマンドを実行するとブラウザが立ち上がるので、以後のコマンドを実行させたいテナントのアカウントでサインオンを実行してください。 その際に、コマンドで指定した権限が画面上にリストされるで意図にあったものか改めて確認しましょう。

"`Welcome To Microsoft Graph!`" とメッセージが表示されれば接続完了です。

## デバイス情報の取得

[Graph Explorer](https://developer.microsoft.com/ja-jp/graph/graph-explorer) でまずは動きを確認してみましょう。

[エクスプローラで実行した画像を挿入]

実行結果を出力しているペインのタブの [コードスニペット] を選択すると、今実行したのと同じ内容となるスクリプトが言語ごとに表示されます。

[エクスプローラで実行した画像を挿入]

では、スニペットに表示されていたコマンドを実行してみましょう。

```powershell
PS > Get-MgDeviceManagementWindowAutopilotDeviceIdentity | fl
AddressableUserName          : 
AzureActiveDirectoryDeviceId : 9972c761-e7ff-4ac4-9e99-2ebddbb5----
DisplayName                  : JPAP0123
EnrollmentState              : notContacted
GroupTag                     : 
Id                           : 0e8e8a3a-b059-4a17-a3f5-f5d9a2e6----
LastContactedDateTime        : 0001/01/01 0:00:00
ManagedDeviceId              : 00000000-0000-0000-0000-000000000000
Manufacturer                 : Microsoft Corporation
Model                        : Surface Laptop Go
ProductKey                   : 8211cd3b-9a42-4256-a6fb-2e60e287----
PurchaseOrderIdentifier      : 
ResourceName                 : 
SerialNumber                 : 00665471----
SkuNumber                    : Surface_Laptop_Go_1943
SystemFamily                 : Surface
UserPrincipalName            : 
AdditionalProperties         : {}
```
期待通りに Autopilot デバイスの情報を取得することができました。（一部の値はマスクしています）

## デバイス名を更新する

Autopilot デバイスの [デバイス名] に該当する Graph API オブジェクトの属性は [DisplayName] です。 Microsoft Intune の表示言語を英語にしても [Device name]です。（名称を統一してほしい・・・）

[AAD Autopilot デバイスのプロパティ画面をキャプチャ]

デバイス名の更新は API [updateDeviceProperties](https://learn.microsoft.com/ja-jp/graph/api/intune-enrollment-windowsautopilotdeviceidentity-updatedeviceproperties?view=graph-rest-1.0) を使用します。

では、こちらも Graph Explorer で実行してみましょう。

[実行内容をキャプチャ]

変更が反映されたか確認してみます。

[実行内容をキャプチャ]

無事に反映されていました。

今回もコードスニペットに助けてもらいましょう。

```powershell
Import-Module Microsoft.Graph.DeviceManagement.Actions

$params = @{
	DisplayName = "Auto-1234"
}

Update-MgDeviceManagementWindowAutopilotDeviceIdentityDeviceProperty -WindowsAutopilotDeviceIdentityId $windowsAutopilotDeviceIdentityId -BodyParameter $params
```

なるほど [`Update-MgDeviceManagementWindowAutopilotDeviceIdentityDeviceProperty`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.devicemanagement.actions/update-mgdevicemanagementwindowautopilotdeviceidentitydeviceproperty?view=graph-powershell-1.0) コマンドが対応するのですね。

スニペットではパラメータ指定をハッシュで行なっていますが、ドキュメントをみると [displayName] を直接引数で指定することもできるようですので、試してみましょう。

```powershell
PS > Update-mgDeviceManagementWindowAutopilotDeviceIdentityDeviceProperty -WindowsAutopilotDeviceIdentityId $windowsAutopilotDeviceIdentityId -DisplayName 'Auto-0123'

PS > Get-MgDeviceManagementWindowAutopilotDeviceIdentity | fl id,displayName

Id          : 0e8e8a3a-b059-4a17-a3f5-f5d9a2e6----
DisplayName : Auto-1234

PS > Get-MgDeviceManagementWindowAutopilotDeviceIdentity | fl id,displayName

Id          : 0e8e8a3a-b059-4a17-a3f5-f5d9a2e6----
DisplayName : Auto-0123
```

`Update` を実行したすぐ後では反映を確認できませんでしたが、ちょと待って再実行すると更新が反映されたことが確認できました。 これで行けそうです。

## バルク更新に向けて

`Update` コマンドで指定するのは Azure Active Directory に登録された Autopilot デバイス オブジェクトの ID　になります。 そのため、まずはこの ID のリストを作って、そのリストの各IDに対応するホスト名を手作業で編集し、そのリストの各ラインを `Update` コマンドの処理対象にする、というプロセスで実行することにしましょう。

1. Autopilot デバイスのリストをCSVに出力
2. CSV にホスト名を追記（マニュアル作業）
3. CSV の各行を `Update` コマンドの引数にして実行

### Autopilot デバイスのリストをCSVに出力

`Export-Csv` コマンドレットを使用して CSV に出力します。 エントリーが一件だけだと繰り返し処理の確認ができないのでもう一つ Autopilot デバイスを事前に追加しました。

```powershell
PS > Get-MgDeviceManagementWindowAutopilotDeviceIdentity | Select id,displayname | Export-Csv -Encoding utf8 -Path c:\tmp\autopilot-devices.csv
PS > cat C:\tmp\autopilot-devices.csv
"Id","DisplayName"
"b56afdbc-d4f7-4efb-a1a3-14ffcee9----",""
"0e8e8a3a-b059-4a17-a3f5-f5d9a2e6----","Auto-0123"
```

### Update の実行

更新処理は出力した CSV ファイルにホスト名となる名前を加えて更新し、そのファイルを元に `Update` を繰り返し実行します。

```powershell
PS > cat C:\tmp\autopilot-devices.csv
"Id","DisplayName"
"b56afdbc-d4f7-4efb-a1a3-14ffcee9----","APD-0001"
"0e8e8a3a-b059-4a17-a3f5-f5d9a2e6----","APD-0002"

PS > $csv = import-csv C:\tmp\autopilot-devices.csv

PS > foreach ($line in $csv) {
     Update-mgDeviceManagementWindowAutopilotDeviceIdentityDeviceProperty `
         -WindowsAutopilotDeviceIdentityId $line.Id `
         -DisplayName $line.DisplayName
    }

PS > get-mgdevicemanagementWindowAutopilotDeviceIdentity | select id,displayname

Id                                   DisplayName
--                                   -----------
b56afdbc-d4f7-4efb-a1a3-14ffcee9---- APD-0001
0e8e8a3a-b059-4a17-a3f5-f5d9a2e6---- APD-0002
```

コマンド上では無事に更新されたことが確認できましたので、Microsoft Intune からも確認してみましょう。

[更新後のDisplayNameがわかるキャプチャ]

こちらでも更新が確認できました。

*注* スクリプトは動作を理解するためのサンプルにすぎず、エラー処理など一切考慮していないので適宜補ってください。

## まとめ
自由度のないテンプレートしか使えなかったホスト名の指定も、Graph API を活用することで組織の運用に沿ったものを指定することができました。Autopilot デバイスが登録された都度に作業をするのが面倒なので、ハード ベンダーさんが登録を代行する際に指定してくれるようになるといいなぁ。

[back to index](README.md)