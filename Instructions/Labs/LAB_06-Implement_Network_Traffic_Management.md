---
lab:
    title: '06 - トラフィック管理を実装する'
    module: 'モジュール 06 - ネットワーク トラフィック管理'
---

# ラボ 06 - トラフィック管理を実装する
# 学生用ラボ マニュアル

## ラボ シナリオ

ハブおよびスポーク ネットワーク トポロジの Azure 仮想マシンを対象としたネットワーク トラフィックの管理テストを担当していましたが、Contoso は Azure 環境での実装を検討しています (前のラボでテストしたメッシュ トポロジを作成する代わりに)。このテストでは、ハブを経由してトラフィックを強制的に流すユーザー定義ルートに依存してスポーク間の接続を実装する必要があり、また、レイヤ 4 およびレイヤ 7 ロード バランサーを使用して仮想マシン間でのトラフィック分散を行う必要があります。この目的のために、Azure Load Balancer (レイヤー 4) と Azure Application Gateway (レイヤー 7) を使用する予定です。

>**注**: このラボでは、デフォルトで、Standard_D2s_v3 SKU の Azure VM のデプロイが 4 つ含まれるため、デプロイ用に選択したリージョンの Standard_Dsv3 シリーズで使用可能な vCPU が合計 8 個必要です。学生が試用版アカウントを使用しており、vCPU が 4 つに制限されている場合は、1 つの vCPU のみを必要とする VM サイズ (Standard_B1s など) を使用できます。

> このラボのすべてのリソースについて、**米国東部** リージョンを使用しています。クラスに使用する地域であることを講師に確認します。 

## 目標

このラボでは次の内容を学習します。

+ タスク 1: ラボ環境をプロビジョニングする
+ タスク 2: ハブとスポークのネットワーク トポロジを構成する
+ タスク 3: 仮想ネットワーク ピアリングの推移性をテストする
+ タスク 4: ハブとスポークのトポロジのルーティングを構成する
+ タスク 5: Azure Load Balancer を実装する
+ タスク 6: Azure Application Gateway を実装する（オプション）

## 予想時間: 60 分

## アーキテクチャの図

![image](../media/lab06.png)

## 手順

### 演習 1

#### タスク 1: ラボ環境をプロビジョニングする

このタスクでは、同じ Azure リージョンに 4 つの仮想マシンをデプロイします。最初の 2 つはハブバーチャル ネットワークに存在し、残りはそれぞれ別個のスポークバーチャル ネットワークに存在します。

1. [Azure portal](https://portal.azure.com) にサインインします。

1. Azure portal の右上にあるアイコンをクリックして **Azure Cloud Shell** を開きます。

1. **Bash** や **PowerShell** のどちらかを選択するためのプロンプトが表示されたら、**PowerShell** を選択します。

    >**注**: **Cloud Shell** の初回起動時に **「ストレージがマウントされていません」** というメッセージが表示された場合は、このラボで使用しているサブスクリプションを選択し、**「ストレージの作成」** を選択します。

1. Cloud Shell ウィンドウのツールバーで、**「ファイルのアップロード/ダウンロード」** アイコンをクリックし、ドロップダウン メニューで **「アップロード」** をクリックして、ファイル **\\Allfiles\\Labs\\06\\az104-06-vms-loop-template.json** と **\\Allfiles\\Labs\\06\\az104-06-vms-loop-parameters.json** を Cloud Shell ホーム ディレクトリにアップロードします。

    >**注**: parameters ファイルの adminPassword の値を変更し、**独自パスワード** にすることができます。

1. 「Cloud Shell」 ペインから次を実行して、ラボ環境をホストする最初のリソース グループを作成します(「(Get-AzLocation).Location」 コマンドレットを使用して、リージョン一覧を取得できます)。

   ```powershell
   $location = 'eastus'

   $rgName = 'az104-06-rg1'

   New-AzResourceGroup -Name $rgName -Location $location
   ```

1. 「Cloud Shell」 ウィンドウで、次のコマンドを実行して 3 つのバーチャル ネットワークを作成し、アップロードしたテンプレートとパラメーター ファイルを使用して、4 つの Azure VM を作成します。

   ```powershell
   New-AzResourceGroupDeployment `
      -ResourceGroupName $rgName `
      -TemplateFile $HOME/az104-06-vms-loop-template.json `
      -TemplateParameterFile $HOME/az104-06-vms-loop-parameters.json
   ```

    >**注**: デプロイが完了するまで待ってから次の手順に進んでください。これにはおよそ 5 分かかります。

1. Cloud Shell ウィンドウから、以下を実行して、前の手順でデプロイされた Azure VM に Network Watcher 拡張機能をインストールします。

   ```powershell
   $rgName = 'az104-06-rg1'
   $location = (Get-AzResourceGroup -ResourceGroupName $rgName).location
   $vmNames = (Get-AzVM -ResourceGroupName $rgName).Name

   foreach ($vmName in $vmNames) {
     Set-AzVMExtension `
     -ResourceGroupName $rgName `
     -Location $location `
     -VMName $vmName `
     -Name 'networkWatcherAgent' `
     -Publisher 'Microsoft.Azure.NetworkWatcher' `
     -Type 'NetworkWatcherAgentWindows' `
     -TypeHandlerVersion '1.4'
   }
   ```

    >**注**: デプロイが完了するまで待ってから次の手順に進んでください。これにはおよそ 5 分かかります。

1. 「Cloud Shell」 ウィンドウを閉じます。

#### タスク 2: ハブとスポークのネットワーク トポロジを構成する

このタスクでは、前のタスクで展開した仮想ネットワーク間のローカル ピアリングを構成して、ハブとスポークのネットワーク トポロジを作成します。

1. Azure portal で、 **「vnet」** を検索し「**仮想ネットワーク」** を選択します。

1. 前のタスクで作成した仮想ネットワークを確認します。 

    > **注**: 3 つの仮想ネットワークのデプロイに使用したテンプレートに、3 つの仮想ネットワークの IP アドレス範囲が重複しないようにします。

1. バーチャル ネットワークのリストで、**「az104-06-vnet2」** を選択します。

1. 「**az104-06-vnet2**」 ブレードで 「**プロパティ**」 を選択します。 

1. 「**az104-06-vnet2 \| プロパティ**」 ブレードで、「**リソース ID**」 プロパティの値を記録します。

1. バーチャル ネットワークのリストに戻り、**「az104-06-vnet3」** を選択します。

1. 「**az104-06-vnet3**」 ブレードで 「**プロパティ**」 を選択します。 

1. 「**az104-06-vnet3 \| プロパティ**」 ブレードで、「**リソース ID**」 プロパティの値を記録します。

    >**注**: このタスクの後半で、両方のバーチャル ネットワークの ResourceID プロパティの値が必要になります。

    >**注**: これは、バーチャル ネットワーク ピアリングを作成するときに、Azure Portal が新しくプロビジョニングされたバーチャル ネットワークを表示しないことがあるという問題に対処する回避策です。

1. 仮想ネットワークの一覧で、**「az104-06-vnet01」** をクリックします。

1. 「**az104-06-vnet01** 仮想ネットワーク」 ブレードの **「設定」** セクションで **「ピアリング」** をクリックしてから、**「+ 追加」** をクリックします。

1. 次の設定でピアリングを追加し (他のユーザーには既定値を残します)、「追加」 をクリックします。

    | 設定 | 値 |
    | --- | --- |
    | この仮想ネットワーク：ピアリング リンク名 | **az104-06-vnet01_to_az104-06-vnet2** |
    | この仮想ネットワーク：リモート仮想ネットワークへのトラフィック | **許可（規定）** |
    | この仮想ネットワーク：リモート仮想ネットワークから転送されるトラフィック | **この仮想ネットワークの外部から来ているトラフィックをブロックする** |
    | この仮想ネットワーク：仮想ネットワーク ゲートウェイまたはルートサーバー | **なし（規定）** |
    | リモート仮想ネットワーク：ピアリング リンク名 | **az104-06-vnet2_to_az104-06-vnet01** |
    | リモート仮想ネットワーク：仮想ネットワークのデプロイ モデル | **Resource Manager** |
    | リモート仮想ネットワーク：リソース ID を知っている | チェックする |
    | リソース ID | このタスクの前半で記録した **az104-06-vnet2** の resourceID パラメーターの値 |
    | リモート仮想ネットワークから転送されたトラフィック | **許可（規定）** |
    | リモート仮想ネットワークから転送されるトラフィック | **許可（規定）** |
    | 仮想ネットワーク ゲートウェイまたはルートサーバー | **なし（規定）** |

    >**注**: 操作が完了するまでお待ちください。

    >**注**: この手順では、az104-06-vnet01 から az104-06-vnet2、az104-06-vnet2 から az104-06-vnet01 までの 2 つのローカル ピアリングを確立します。

    > **注**: このラボで後ほど実装するスポーク仮想ネットワーク間のルーティングを容易にするために、**「転送トラフィックを許可する」** を有効にする必要があります。

1. 「**az104-06-vnet01** 仮想ネットワーク」 ブレードの **「設定」** セクションで **「ピアリング」** をクリックしてから、**「+ 追加」** をクリックします。

1. 次の設定でピアリングを追加し (他のユーザーには既定値を残します)、「追加」 をクリックします。

    | 設定 | 値 |
    | --- | --- |
    | この仮想ネットワーク：ピアリング リンク名 | **az104-06-vnet01_to_az104-06-vnet3** |
    | この仮想ネットワーク：リモート仮想ネットワークへのトラフィック | **許可（規定）** |
    | この仮想ネットワーク：リモート仮想ネットワークから転送されるトラフィック | **この仮想ネットワークの外部から来ているトラフィックをブロックする** |
    | この仮想ネットワーク：仮想ネットワーク ゲートウェイまたはルートサーバー | **なし（規定）** |
    | リモート仮想ネットワーク：ピアリング リンク名 | **az104-06-vnet3_to_az104-06-vnet01** |
    | リモート仮想ネットワーク：仮想ネットワークのデプロイ モデル | **Resource Manager** |
    | リモート仮想ネットワーク：リソース ID を知っている | チェックする |
    | リソース ID | このタスクの前半で記録した **az104-06-vnet3** の resourceID パラメーターの値 |
    | リモート仮想ネットワークから転送されたトラフィック | **許可（規定）** |
    | リモート仮想ネットワークから転送されるトラフィック | **許可（規定）** |
    | 仮想ネットワーク ゲートウェイまたはルートサーバー | **なし（規定）** |

    > **注**: この手順では、az104-06-vnet01 から az104-06-vnet3、az104-06-vnet3 から az104-06-vnet01 までの 2 つのローカル ピアリングを確立します。これで、ハブとスポーク トポロジの設定が完了します (2 つのスポーク仮想ネットワークを使用)。

    > **注**: このラボで後ほど実装するスポーク仮想ネットワーク間のルーティングを容易にするために、**「転送トラフィックを許可する」** を有効にする必要があります。

#### タスク 3: 仮想ネットワーク ピアリングの推移性をテストする

このタスクでは、Network Watcher を使用して仮想ネットワーク ピアリングの推移性をテストします。

1. Azure portal で、**「Network Watcher」** を検索して選択します。

1. **「Network Watcher」** ブレードで、Azure リージョンのリストを展開し、このラボの最初のタスクでリソースをデプロイした Azure でサービスが有効になっていることを確認します。

1. **「Network Watcher」** ブレードで、**「接続のトラブルシューティング」** に移動します。

1. **「Network Watcher - 接続のトラブルシューティング」** ブレードで、次の設定でチェックを開始します (その他の設定は既定値のままにします)。

    | 設定 | 値 |
    | --- | --- |
    | サブスクリプション | このラボで使用する Azure サブスクリプションの名前 |
    | リソース グループ | **az104-06-rg1** |
    | ソース タイプ | **仮想マシン** |
    | 仮想マシン | **az104-06-vm0** |
    | 宛先 | **手動で指定** |
    | URI、FQDN、IP アドレスのいずれか | **10.62.0.4** |
    | プロトコル | **TCP** |
    | 宛先のポート | **3389** |

    > **注**: **10.62.0.4** は、プライベート IP アドレス **az104-06-vm2** を表します。

1. **「チェック」** をクリックし、接続チェックの結果が返されるまで待ちます。状態が **「到達可能」** であることを確認します。ネットワーク パスを確認し、VM 間に中間ホップのない直接接続であったことに注意します。

    > **注**: ハブ仮想ネットワークは最初のスポーク仮想ネットワークと直接ピアリングされるため、これは予期されます。
   
    > Network Watcher がうまく動作しない場合のワークアラウンドとしては、仮想マシンの実行コマンドから **RunPowerShellScript** を選択して **「Test-NetConnection 10.62.0.4 -port 3389」** コマンドで確認します。

1. **「Network Watcher - 接続のトラブルシューティング」** ブレードで、次の設定でチェックを開始します (その他の設定は既定値のままにします)。

    | 設定 | 値 |
    | --- | --- |
    | サブスクリプション | このラボで使用する Azure サブスクリプションの名前 |
    | リソース グループ | **az104-06-rg1** |
    | ソース タイプ | **仮想マシン** |
    | 仮想マシン | **az104-06-vm0** |
    | 宛先 | **手動で指定** |
    | URI、FQDN、IP アドレスのいずれか | **10.63.0.4** |
    | プロトコル | **TCP** |
    | 宛先のポート | **3389** |

    > **注**: **10.63.0.4** は、プライベート IP アドレス **az104-06-vm3** を表します。

1. **「チェック」** をクリックし、接続チェックの結果が返されるまで待ちます。状態が **「到達可能」** であることを確認します。ネットワーク パスを確認し、VM 間に中間ホップのない直接接続であったことに注意します。

    > **注**: ハブ仮想ネットワークは 2 番目のスポーク仮想ネットワークと直接ピアリングされるため、これは予期されます。

1. **「Network Watcher - 接続のトラブルシューティング」** ブレードで、次の設定でチェックを開始します (その他の設定は既定値のままにします)。

    | 設定 | 値 |
    | --- | --- |
    | サブスクリプション | このラボで使用する Azure サブスクリプションの名前 |
    | リソース グループ | **az104-06-rg1** |
    | ソース タイプ | **仮想マシン** |
    | 仮想マシン | **az104-06-vm2** |
    | 宛先 | **手動で指定** |
    | URI、FQDN、IP アドレスのいずれか | **10.63.0.4** |
    | プロトコル | **TCP** |
    | 宛先のポート | **3389** |

1. **「チェック」** をクリックし、接続チェックの結果が返されるまで待ちます。ステータスが **「到達不可能」** であることに注意してください。

    > **注**: これは、2 つのスポーク仮想ネットワークが互いにピアリングされないので、予想されます (仮想ネットワーク ピアリングは推移的ではありません)。

#### タスク 4: ハブとスポークのトポロジのルーティングを構成する

このタスクでは、**az104-06-vm0** 仮想マシンのネットワーク インターフェイスで IP 転送を有効にし、オペレーティング システム内でルーティングを有効にし、スポーク仮想ネットワーク上でユーザー定義のルートを構成することにより、2 つのスポーク仮想ネットワーク間のルーティングを構成およびテストします。

1. Azure portal で、**「Virtual Machines」** を検索して選択します。

1. **「Virtual Machines」** ブレードの仮想マシンのリストで、**「az104-06-vm0」** をクリックします。

1. 「**az104-06-vm0** 仮想マシン」 ブレードの **「設定」** セクションで、**「ネットワーク」** をクリックします。

1. **「ネットワーク インターフェイス」** ラベルの横の **「az104-06-nic0」** リンクをクリックし、**「az104-06-nic0」** ネットワーク インターフェイス ブレードの **「設定」** セクションで **「IP 構成」** をクリックします。

1. **「IP 転送」** を **「有効」** に設定し、変更を保存します。

<details><summary>この後の手順でサブネットが表示されるまで時間がかかる場合があります。その際の回避策として PowerShell コマンドで対応します。</summary>
PowerShell で「IP 転送」を「有効」に設定します。
<div>
    
   ```powershell
## Place the network interface configuration into a variable. ##
$nic = Get-AzNetworkInterface -Name az104-06-nic0 -ResourceGroupName az104-06-rg1

## Set the IP forwarding setting to enabled. ##
$nic.EnableIPForwarding = 1

## Apply the new configuration to the network interface. ##
$nic | Set-AzNetworkInterface
   ```

</div>
</details>

   > **注**: この設定は、2 つのスポーク仮想ネットワーク間でトラフィックをルーティングするルーターとして **az104-06-vm0** が機能するために必要です。

   > **注**: ここで、ルーティングをサポートするように **az104-06-vm0** 仮想マシンのオペレーティング システムを構成する必要があります。

1. Azure portal で、「**az104-06-vm0** Azure 仮想マシン」 ブレードに戻り、**「概要」** をクリックします。

1. **az104-06-vm0** ブレードの 「**操作**」 セクションで、「**実行コマンド**」 をクリックし、コマンドのリストで **RunPowerShellScript** を選択します。

1. **「コマンド スクリプトの実行」** ブレードで、次のコマンドを入力し、**「実行」** をクリックしてリモート アクセス Windows サーバー ロールをインストールします。

   ```powershell
   Install-WindowsFeature RemoteAccess -IncludeManagementTools
   ```

   > **注**: コマンドが正常に完了したことを確認します。

1. **「コマンド スクリプトの実行」** ブレードで、次のコマンドを入力し、**「実行」** をクリックしてルーティング ロール サービスをインストールします。

   ```powershell
   Install-WindowsFeature -Name Routing -IncludeManagementTools -IncludeAllSubFeature
   Install-WindowsFeature -Name "RSAT-RemoteAccess-Powershell"
   Install-RemoteAccess -VpnType RoutingOnly
   Get-NetAdapter | Set-NetIPInterface -Forwarding Enabled
   ```

   > **注**: コマンドが正常に完了したことを確認します。

   > **注**: 次に、スポーク仮想ネットワーク上でユーザー定義のルートを作成および構成する必要があります。

1. Azure portal で **「ルート テーブル」** を検索して選択 し、**「ルート テーブル」** ブレードで **「+ 作成」** をクリックします。

<details><summary>この後の手順でサブネットが表示されるまで時間がかかる場合があります。その際の回避策として PowerShell コマンドで対応します。</summary>
手動で作成したルートテーブルを削除して以下のコマンドを実行します。
<div>
    
   ```powershell
   $Route = New-AzRouteConfig -Name "az104-06-route-vnet2-to-vnet3" -AddressPrefix 10.63.0.0/20 -NextHopType "VirtualAppliance" -NextHopIpAddress 10.60.0.4
   $rt=New-AzRouteTable -Name "az104-06-rt23" -ResourceGroupName "az104-06-rg1" -Location "EASTUS" -DisableBgpRoutePropagation -Route $Route
   $vnet=Get-AzVirtualNetwork -Name az104-06-vnet2
   Set-AzVirtualNetworkSubnetConfig -name subnet0 -VirtualNetwork $vnet -AddressPrefix "10.62.0.0/24" -RouteTable $rt | Set-AzVirtualNetwork
   $Route1 = New-AzRouteConfig -Name "az104-06-route-vnet3-to-vnet2" -AddressPrefix 10.62.0.0/20 -NextHopType "VirtualAppliance" -NextHopIpAddress 10.60.0.4
   $rt1=New-AzRouteTable -Name "az104-06-rt32" -ResourceGroupName "az104-06-rg1" -Location "EASTUS" -DisableBgpRoutePropagation -Route $Route1
   $vnet1=Get-AzVirtualNetwork -Name az104-06-vnet3
   Set-AzVirtualNetworkSubnetConfig -name subnet0 -VirtualNetwork $vnet1 -AddressPrefix "10.63.0.0/24" -RouteTable $rt1 | Set-AzVirtualNetwork
   ```

</div>
</details>
       
1. 次の設定でルート テーブルを作成します (その他の設定は既定値のままにします)。

    | 設定 | 値 |
    | --- | --- |
    | サブスクリプション | このラボで使用する Azure サブスクリプションの名前 |
    | リソース グループ | **az104-06-rg1** |
    | 場所 | 米国東部 |
    | 名前 | **az104-06-rt23** |
    | ゲートウェイのルートを伝達する | **No** |

1. **「確認および作成」** をクリックします。検証を実行し、**「作成」** をクリックしてデプロイを送信します。

   > **注**: 新しいルーティング テーブルが作成されるのを待ちます。これにはおよそ 3 分かかります。

1. **「ルート テーブル」** ブレードに戻り、**「最新の情報に更新」** をクリックし、**「az104-06-rt23」** をクリックします。

1. 「**az104-06-rt23** ルート テーブル」 ブレードの **「設定」** セクションで **「ルート」** をクリックしてから、**「+ 追加」** をクリックします。

1. 次の設定で新しいルートを追加します (その他の設定は既定値のままにします)。

    | 設定 | 値 |
    | --- | --- |
    | ルート名 | **az104-06-route-vnet2-to-vnet3** |
    | アドレス プレフィックス | **10.63.0.0/20** |
    | 次のホップのタイプ | **仮想アプライアンス** |
    | 次のホップのアドレス | **10.60.0.4** |

1. 「**OK**」 をクリックします

1. 「**az104-06-rt23** ルート テーブル」 ブレードの **「設定」** セクションで **「サブネット」** をクリックしてから、**「+ 関連付け」** をクリックします。

1. ルート テーブル **「az104-06-rt23」** を次のサブネットに関連付けます。

    | 設定 | 値 |
    | --- | --- |
    | 仮想ネットワーク | **az104-06-vnet2** |
    | サブネット | **subnet0** |

1. 「**OK**」 をクリックします

1. **「ルート テーブル」** ブレードに戻り、**「+ 作成」** をクリックします。

1. 次の設定でルート テーブルを作成します (その他の設定は既定値のままにします)。

    | 設定 | 値 |
    | --- | --- |
    | サブスクリプション | このラボで使用する Azure サブスクリプションの名前 |
    | リソース グループ | **az104-06-rg1** |
    | リージョン | 米国東部 |
    | 名前 | **az104-06-rt32** |
    | ゲートウェイのルートを伝達する | **No** |

1. **「確認および作成」** をクリックします。検証を実行し、**「作成」** をクリックしてデプロイを送信します。

   > **注**: 新しいルーティング テーブルが作成されるのを待ちます。これにはおよそ 3 分かかります。

1. **「ルート テーブル」** ブレードに戻り、**「最新の情報に更新」** をクリックし、**「az104-06-rt32」** をクリックします。

1. 「**az104-06-rt32** ルート テーブル」 ブレードの **「設定」** セクションで **「ルート」** をクリックしてから、**「+ 追加」** をクリックします。

1. 次の設定で新しいルートを追加する:

    | 設定 | 値 |
    | --- | --- |
    | ルート名 | **az104-06-route-vnet3-to-vnet2** |
    | アドレス プレフィックス | **10.62.0.0/20** |
    | 次のホップのタイプ | **仮想アプライアンス** |
    | 次のホップのアドレス | **10.60.0.4** |

1. 「**OK**」 をクリックします

1. 「**az104-06-rt32** ルート テーブル」 ブレードの **「設定」** セクションで **「サブネット」** をクリックしてから、**「+ 関連付け」** をクリックします。

1. ルート テーブル **「az104-06-rt32」** を次のサブネットに関連付けます。

    | 設定 | 値 |
    | --- | --- |
    | バーチャル ネットワーク | **az104-06-vnet3** |
    | サブネット | **subnet0** |

1. **「OK」** をクリックします

1. Azure portal で、**「Network Watcher - 接続トラブルシューティング」** ブレードに戻ります。

1. **「Network Watcher - 接続のトラブルシューティング」** ブレードで、次の設定でチェックを開始します (その他の設定は既定値のままにします)。

    | 設定 | 値 |
    | --- | --- |
    | サブスクリプション | このラボで使用する Azure サブスクリプションの名前 |
    | リソース グループ | **az104-06-rg1** |
    | ソース タイプ | **仮想マシン** |
    | 仮想マシン | **az104-06-vm2** |
    | 宛先 | **手動で指定する** |
    | URI、FQDN、IP アドレスのいずれか | **10.63.0.4** |
    | プロトコル | **TCP** |
    | 宛先ポート | **3389** |
    
    
   > Network Watcher がうまく動作しない場合のワークアラウンドとしては、仮想マシンの実行コマンドから **RunPowerShellScript** を選択して **「Test-NetConnection 10.63.0.4 -port 3389」** コマンドで確認します。
 
1. **「チェック」** をクリックし、接続チェックの結果が返されるまで待ちます。状態が **「到達可能」** であることを確認します。ネットワーク パスを確認し、トラフィックが **az104-06-nic0** ネットワーク アダプターに割り当てられた **10.60.0.4** を経由してルーティングされたことを確認します。ステータスが**到達不能**の場合は、az104-06-vm0 を再起動する必要があります。

    > **注**: これは想定どおりの結果です。スポーク仮想ネットワーク間のトラフィックは、ルーターとして機能するハブ仮想ネットワークにある仮想マシンを経由してルーティングされるためです。 

    > **注**: **Network Watcher** を使用して、ネットワークのトポロジを表示できます。

#### タスク 5: Azure Load Balancer を実装する

このタスクでは、ハブ仮想ネットワーク内の 2 つの Azure 仮想マシンの前に Azure Load Balancer を実装します。

<details><summary>ロードバランサーの設定時に仮想ネットワークが表示されるまで時間がかかる場合があります。その場合のワークアラウンドとしては PowerShell で下記コマンドを実行して作成します。</summary>
az104-06-rg4 のリソースグループを削除してから実行してください。その後、Azure Portalから作成されたロードバランサーを選択してバックエンドプールの設定でVMを追加します。
<div>

```powershell
$rgName='az104-06-rg4'
$location='eastus'

New-AzResourceGroup -Name $rgname -Location $location

$publicip = New-AzPublicIpAddress -ResourceGroupName $rgName `
-name "az104-06-pip4" `
-location $location `
-AllocationMethod Static `
-Sku Standard

## Create load balancer frontend configuration and place in variable. ##
$fip = @{
    Name = 'LoadBalancerFrontEnd'
    PublicIpAddress = $publicIp 
}
$feip = New-AzLoadBalancerFrontendIpConfig @fip

## Create backend address pool configuration and place in variable. ##
$bepool = New-AzLoadBalancerBackendAddressPoolConfig -Name 'az104-06-lb4-be1'

## Create the health probe and place in variable. ##
$probe = @{
    Name = 'az104-06-lb4-hp1'
    Protocol = 'tcp'
    Port = '80'
    IntervalInSeconds = '5'
    ProbeCount = '2'
}
$healthprobe = New-AzLoadBalancerProbeConfig @probe

## Create the load balancer rule and place in variable. ##
$lbrule = @{
    Name = 'az104-06-lb4-lbrule1'
    Protocol = 'tcp'
    FrontendPort = '80'
    BackendPort = '80'
    IdleTimeoutInMinutes = '15'
    FrontendIpConfiguration = $feip
    BackendAddressPool = $bePool
    probe = $healthprobe
}
$rule = New-AzLoadBalancerRuleConfig @lbrule -DisableOutboundSNAT

## Create the load balancer resource. ##
$loadbalancer = @{
    ResourceGroupName = $rgName
    Name = 'az104-06-lb4'
    Location = 'eastus'
    Sku = 'Standard'
    FrontendIpConfiguration = $feip
    BackendAddressPool = $bePool
    LoadBalancingRule = $rule
    Probe = $healthprobe
}
New-AzLoadBalancer @loadbalancer
```

</div>
</details>

1. Azure portal で **「ロード バランサー」** を検索して選択し、**「ロード バランサー」** ブレードで **「+ 作成」** をクリックします。

    > **注**: テナントによっては、以下の手順と異なる場合があります。その場合は、個別に各種設定を行う必要があります。

1. 「**基本**」タブで、次の設定を入力し「**次:フロントエンド IP 構成 >**」をクリックします。

    | 設定 | 値 |
    | --- | --- |
    | サブスクリプション | このラボで使用する Azure サブスクリプションの名前 |
    | リソース グループ | 新しいリソース グループ **az104-06-rg4** の名前 |
    | 名前 | **az104-06-lb4** |
    | リージョン| （US）米国東部 |
    | 種類 | **パブリック** |
    | SKU | **Standard** |
    | レベル | **地域** |

1. 「**フロントエンド IP 構成**」タブで、「**フロントエンドの追加**」をクリックします。

    | 設定 | 値 |
    | --- | --- |
    | 名前 | **LoadBalancerFrontEnd** |
    | パブリック IP アドレス | **新規作成** |
    | パブリック IP アドレス:名前 | **az104-06-pip4** |
    | パブリック IP アドレス:可用性ゾーン | **ゾーンなし** |

1. 「**次:バックエンドプール >**」 をクリックし、「**バックエンドプールの追加**」をクリックします。

1. 次の設定でバックエンド プールを追加します (その他は既定値のままにします)。

    | 設定 | 値 |
    | --- | --- |
    | 名前 | **az104-06-lb4-be1** |
    | 仮想ネットワーク | **az104-06-vnet01** |
    | バックエンドプールの構成 | **NIC** |
    | IP バージョン | **IPv4** |
    | 仮想マシン | **az104-06-vm0** |
    | 仮想マシン | **az104-06-vm1** |

1. 「**追加**」 をクリックします。

1. 「**次:インバウンド規則 >**」 をクリックし、「**負荷分散規則の追加**」をクリックします。

1. 「**負荷分散規則の追加**」ブレードで、次の設定で負荷分散規則を追加します (その他は既定値のままにします)。

    | 設定 | 値 |
    | --- | --- |
    | 名前 | **az104-06-lb4-lbrule1** |
    | IP バージョン | **IPv4** |
    | フロントエンド IP アドレス | **LoadBalancerFrontEnd** |
    | プロトコル | **TCP** |
    | ポート | **80** |
    | バックエンド ポート | **80** |
    | バックエンド プール | **az104-06-lb4-be1** |
    | 正常性プローブ | （新規作成）**az104-06-lb4-hp1** |
    | セッション永続化 | **なし** |
    | アイドル タイムアウト (分) | **4** |
    | TCP リセット | **無効** |
    | フローティング IP (Direct Server Return) | **無効** |

1. 次の設定で**正常性プローブ**を追加する:

    | 設定 | 値 |
    | --- | --- |
    | 名前 | **az104-06-lb4-hp1** |
    | プロトコル | **TCP** |
    | ポート | **80** |
    | 間隔 | **5** |
    | 異常しきい値 | **2** |

1. 「**追加**」 をクリックします

1. **「確認および作成」** をクリックし、**「作成」** をクリックします。

1. 負荷分散規則が作成されるのを待ち、**「概要」** をクリックして、**「パブリック IP アドレス」** の値を記録します。

1. 別のブラウザーのウィンドウを起動し、前の手順で識別した IP アドレスに移動します。

1. ブラウザー ウィンドウに、**「Hello World from az104-06-vm0」** または **「Hello World from az104-06-vm1」** というメッセージが表示されることを確認します。

1. 別のブラウザー ウィンドウを開きますが、今回は InPrivate モードを使用して、ターゲット VM が (メッセージで示されているように) 変更するかどうかを確認します。

    > **注**: ブラウザー ウィンドウを更新するか、InPrivate モードを使用して再度開く必要があります。

#### タスク 6: Azure Application Gateway を実装する（オプション）

このタスクでは、スポーク仮想ネットワーク内の 2 つの Azure 仮想マシンの前に Azure Application Gateway を実装します。

<details><summary>Azure Application Gateway の設定時に仮想ネットワークが表示されるまで時間がかかる場合があります。その場合のワークアラウンドとしては PowerShell で下記コマンドを実行して作成します。</summary>
<div>
    
```powershell
$rgName='az104-06-rg5'
$location='eastus'

New-AzResourceGroup -Name $rgname -Location $location

$agSubnetConfig = New-AzVirtualNetworkSubnetConfig `
  -Name subnet-appgw `
  -AddressPrefix 10.60.3.224/27
New-AzPublicIpAddress `
  -ResourceGroupName $rgName `
  -Location $location `
  -Name az104-06-pip5 `
  -AllocationMethod Static `
  -Sku Standard

$vnet   = Get-AzVirtualNetwork -ResourceGroupName az104-06-rg1 -Name az104-06-vnet01
$subnet = Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name subnet-appgw
$pip    = Get-AzPublicIPAddress -ResourceGroupName $rgName -Name az104-06-pip5
$gipconfig = New-AzApplicationGatewayIPConfiguration `
  -Name myAGIPConfig `
  -Subnet $subnet
$fipconfig = New-AzApplicationGatewayFrontendIPConfig `
  -Name myAGFrontendIPConfig `
  -PublicIPAddress $pip
$frontendport = New-AzApplicationGatewayFrontendPort `
  -Name myFrontendPort `
  -Port 80

$backendPool = New-AzApplicationGatewayBackendAddressPool `
  -Name az104-06-appgw5-be1 `
  -BackendIPAddresses "10.62.0.4", "10.63.0.4"
$poolSettings = New-AzApplicationGatewayBackendHttpSetting `
  -Name az104-06-appgw5-http1 `
  -Port 80 `
  -Protocol Http `
  -CookieBasedAffinity Disabled `
  -RequestTimeout 20

$defaultlistener = New-AzApplicationGatewayHttpListener `
  -Name az104-06-appgw5-rl1l1 `
  -Protocol Http `
  -FrontendIPConfiguration $fipconfig `
  -FrontendPort $frontendport
$frontendRule = New-AzApplicationGatewayRequestRoutingRule `
  -Name az104-06-appgw5-rl1 `
  -RuleType Basic `
  -Priority 1 `
  -HttpListener $defaultlistener `
  -BackendAddressPool $backendPool `
  -BackendHttpSettings $poolSettings

$sku = New-AzApplicationGatewaySku `
  -Name Standard_v2 `
  -Tier Standard_v2 `
  -Capacity 2

New-AzApplicationGateway `
  -Name az104-06-appgw5 `
  -ResourceGroupName $rgName `
  -Location $location `
  -BackendAddressPools $backendPool `
  -BackendHttpSettingsCollection $poolSettings `
  -FrontendIpConfigurations $fipconfig `
  -GatewayIpConfigurations $gipconfig `
  -FrontendPorts $frontendport `
  -HttpListeners $defaultlistener `
  -RequestRoutingRules $frontendRule `
  -Sku $sku
```

</div>
</details>

1. Azure portal で、**「VNET」** を検索して「**仮想ネットワーク**」選択します。

1. **「仮想ネットワーク」** ブレードで、**「az104-06-vnet01」** をクリックします。

1. 「**az104-06-vnet01** 仮想ネットワーク」 ブレードの 「**設定**」 セクションで、「**サブネット**」 をクリックし、「**+ サブネット**」 をクリックします。

1. 次の設定でサブネットを追加します (その他の設定は既定値のままにします)。

    | 設定 | 値 |
    | --- | --- |
    | 名前 | **subnet-appgw** |
    | サブネット アドレス範囲 | **10.60.3.224/27** |

1. 「**保存**」 をクリックする

    > **注**: このサブネットは、このタスクの後半でデプロイする Azure Application Gateway インスタンスで使用されます。Application Gateway には、/27 以上のサイズの専用サブネットが必要です。

1. Azure portal で **「アプリケーション ゲートウェイ」** を検索して選択し、**「アプリケーション ゲートウェイ」** ブレードで **「+ 作成」** をクリック します。

1. **「アプリケーション ゲートウェイ作成」** ブレードの **「基本」** タブで、次の設定を指定します (その他は既定値のままにします)。

    | 設定 | 値 |
    | --- | --- |
    | サブスクリプション | このラボで使用する Azure サブスクリプションの名前 |
    | リソース グループ | 新しいリソース グループ **az104-06-rg5** の名前 |
    | ゲートウェイ名 | **az104-06-appgw5** |
    | 地域 | 米国東部 |
    | レベル | **Standard V2** |
    | 自動スケール | **いいえ** |
    | 最小インスタンス数 | 規定値を受け入れる |
    | 最大インスタンス数 | 規定値を受け入れる |
    | 仮想性ゾーン | **なし** |
    | HTTP2 | **無効** |
    | 仮想ネットワーク | **az104-06-vnet01** |
    | サブネット | **subnet-appgw** |

1. **「次:フロントエンドの数 >」** をクリックし、**「フロントエンドの数」** タブで、次の設定を指定します (その他の設定は既定値のままにします)。

    | 設定 | 値 |
    | --- | --- |
    | フロントエンド IP の種類 | **パブリック** |
    | パブリック IP アドレス| 新しいパブリック IP アドレス **az104-06-pip5** の名前 |

1. **「次:バックエンド >」** をクリックし、**「バックエンド」** タブで、**「バックエンド プールの追加」** をクリックし、**「バックエンド プールの追加」** ブレードで、次の設定を指定します (その他は既定値のままにします)。

    | 設定 | 値 |
    | --- | --- |
    | 名前 | **az104-06-appgw5-be1** |
    | ターゲットを持たないバックエンド プールを追加します | **いいえ** |
    | ターゲットの種類 | **IP アドレスまたは FQDN** |
    | ターゲット | **10.62.0.4** |
    | ターゲットの種類 | **IP アドレスまたは FQDN** |
    | ターゲット | **10.63.0.4** |

    > **注**: ターゲットは、スポーク仮想ネットワーク **az104-06-vm2** および **az104-06-vm3** 内の仮想マシンのプライベート IP アドレスを表します。

1. **「追加」** をクリックし、**「次:構成 >**」 をクリックし、「**構成**」 タブで、「**+ ルーティング規則の追加**」 をクリックします。

1. **「ルーティング規則の追加」** ブレードの **「リスナー」** タブで、次の設定を指定します。

    | 設定 | 値 |
    | --- | --- |
    | ルール名 | **az104-06-appgw5-rl1** |
    | 優先度 | **1** |
    | リスナー名 | **az104-06-appgw5-rl1l1** |
    | フロントエンド IP | **パブリック** |
    | プロトコル | **HTTP** |
    | ポート | **80** |
    | リスナーの種類 | **Basic** |
    | エラー ページの URL | **いいえ** |

1. **「ルーティング規則の追加」** ブレードの **「バックエンド ターゲット」** タブに切り替え、次の設定を指定します (その他の設定は既定値のままにします)。

    | 設定 | 値 |
    | --- | --- |
    | ターゲット タイプ | **バックエンド プール** |
    | バックエンド ターゲット | **az104-06-appgw5-be1** |

1. **「バックエンド 設定」** テキスト ボックスの横にある **「新規追加」** をクリックし、**「バックエンド 設定の追加」** ブレードで次の設定を指定します (その他の設定は既定値のままにします)。

    | 設定 | 値 |
    | --- | --- |
    | バックエンド 設定名 | **az104-06-appgw5-http1** |
    | バックエンド プロトコル | **HTTP** |
    | バックエンド ポート | **80** |
    | Cookie ベースのアフィニティ | **無効化** |
    | 接続のドレイン | **無効化** |
    | 要求タイムアウト (秒) | **20** |

1. **「バックエンド設定の追加」** ブレードで **「追加」** をクリックし、「**ルーティング規則の追加**」 ブレードに戻って **「追加」** をクリックします。

1. 「**次:タグ >**」 をクリックし、続いて 「**次:確認および作成 >**」 をクリックし、**「作成」** をクリックします。

    > **注**: Application Gateway インスタンスが作成されるのを待ちます。8 分間程度かかる場合があります。

1. Azure portal で **「アプリケーション ゲートウェイ」** を検索して選択し、**「アプリケーション ゲートウェイ」** ブレードで **「az104-06-appgw5」** をクリックします。

1. **「az104-06-appgw5** アプリケーション ゲートウェイ」 ブレードで、**「フロントエンド パブリック IP アドレス」** の値を書き留めます。

1. 別のブラウザーのウィンドウを起動し、前の手順で識別した IP アドレスに移動します。

1. ブラウザー ウィンドウに、**「Hello World from az104-06-vm2」** または **「Hello World from az104-06-vm3」** というメッセージが表示されることを確認します。

1. 今度は InPrivate モードを使用して別のブラウザー ウィンドウを開き、Web ページに表示されるメッセージに基づいて、ターゲット VM が変更されたかどうかを確認します。

    > **注**: ブラウザー ウィンドウを更新するか、InPrivate モードを使用して再度開く必要があります。

    > **注**: 複数の仮想ネットワーク上の仮想マシンを対象とすることは一般的な構成ではありませんが、Application Gateway が複数の仮想ネットワーク上の仮想マシンや、他の Azureリージョンのエンドポイント、さらには Azure の外部のエンドポイントを対象とできる点を示すことを目的としています。同じ仮想ネットワーク内の仮想マシン間で負荷分散を行う Azure Load Balancer とは異なります。

#### リソースをクリーン アップする

   >**注**: 新しく作成した Azure リソースのうち、使用しないリソースは必ず削除してください。使用しないリソースを削除しないと、予期しないコストが発生する場合があります。

1. Azure portal の **「Cloud Shell」** ウィンドウで **「PowerShell」** セッションを開きます。

1. 次のコマンドを実行して、このモジュールのラボ全体で作成したすべてのリソース グループのリストを表示します。

   ```powershell
   Get-AzResourceGroup -Name 'az104-06*'
   ```

1. 次のコマンドを実行して、このモジュールのラボ全体で作成したすべてのリソース グループのリストを削除します。

   ```powershell
   Get-AzResourceGroup -Name 'az104-06*' | Remove-AzResourceGroup -Force -AsJob
   ```

    >**注**: コマンドは非同期で実行されるので (-AsJob パラメーターによって決定されます)、別の PowerShell コマンドを同一 PowerShell セッション内ですぐに実行できますが、リソース グループが実際に削除されるまでに数分かかります。

#### レビュー

このラボでは次の内容を学習しました。

+ ラボ環境をプロビジョニングしました
+ ハブとスポーク ネットワーク トポロジを構成しました
+ バーチャル ネットワーク ピアリングの推移性をテストしました
+ タスク 4: ハブとスポークのトポロジのルーティングを構成する
+ タスク 5: Azure Load Balancer を実装する
+ タスク 6: Azure Application Gateway を実装する
