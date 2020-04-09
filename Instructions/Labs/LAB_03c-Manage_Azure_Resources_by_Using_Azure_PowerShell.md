---
lab:
    title: '03c - Azure PowerShell を使用して Azure リソースを管理する'
    module: 'モジュール 03 - Azure 管理'
---

# ラボ 03c - Azure PowerShell を使用して Azure リソースを管理する
# 受講者用ラボ マニュアル

## ラボ シナリオ

Azure portal とAzure Resource Manager テンプレートを使用して、リソース のプロビジョニングとリソース調整に基づく構成に関連する基本的な Azure 管理機能について確認しました。次に Azure PowerShell を使って、同等のタスクを実行する必要があります。Azure PowerShell モジュールのインストールを避けるため、Azure Cloud Shellで使用できる PowerShell 環境を活用します。

## 目標

このラボでは、次の内容を学習します。

+ タスク 1: Azure Cloud Shell で PowerShell セッションを開始する
+ タスク 2: Azure PowerShell を使用してリソース グループと Azure マネージド ディスクを作成する
+ タスク 3: Azure PowerShell を使用してマネージド ディスクを構成する

## 指示

### 演習 1

#### タスク 1: Azure Cloud Shell で PowerShell セッションを開始する

このタスクでは、Cloud Shell で PowerShell セッションを開きます。 

1. このポータルでは、Azure Portal の右上にあるアイコンをクリックして **Azure Cloud Shell** を開きます。

1. **Bash** または **PowerShell** のいずれかを選択するように求められた場合、**PowerShell**を選択します。 

    >**注**: **Cloud Shell** を初めて起動し、「 **ストレージがマウントされていません**」というメッセージが表示されたら、このラボで使用しているサブスクリプションを選択し、[**ストレージの作成**] をクリックします。 

1. メッセージが表示されたら、**ストレージの作成**をクリックし、[Azure Cloud Shell] ウィンドウが表示されるまで待ちます。 

1. [Cloud Shell] ペインの左上隅にあるドロップダウン メニューに **PowerShell** が表示されていることを確認します。

#### タスク 2: Azure PowerShell を使用してリソース グループと Azure マネージド ディスクを作成する

このタスクでは、クラウド シェル内で Azure PowerShell セッションを使用して、リソース グループと Azure マネージド ディスクを作成します。

1. 前のラボで作成した **az104-03b-rg1 **リソース グループと同じ Azure リージョンにリソース グループを作成するには、Cloud Shell 内の PowerShell セッションから次を実行します。

   ```pwsh
   $location = (Get-AzResourceGroup -Name az104-03b-rg1).Location

   $rgName = 'az104-03c-rg1'

   New-AzResourceGroup -Name $rgName -Location $location
   ```
1. 新しく作成されたリソース グループのプロパティを取得するには、次の手順を実行します。

   ```pwsh
   Get-AzResourceGroup -Name $rgName
   ```
1. このモジュールの前のラボで作成したものと同じ特性を持つ新しいマネージド ディスクを作成するには、次の手順を実行します。

   ```pwsh
   $diskConfig = New-AzDiskConfig `
    -Location $location `
    -CreateOption Empty `
    -DiskSizeGB 32 `
    -Sku Standard_LRS

   $diskName = 'az104-03c-disk1'

   New-AzDisk `
    -ResourceGroupName $rgName `
    -DiskName $diskName `
    -Disk $diskConfig
   ```

1. 新しく作成されたディスクのプロパティを取得するには、次の手順を実行します。

   ```pwsh
   Get-AzDisk -ResourceGroupName $rgName -Name $diskName
   ```

#### タスク 3: Azure PowerShell を使用してマネージド ディスクを構成する

このタスクでは、クラウド シェル内で Azure PowerShell セッションを使用して、Azure マネージド ディスクの構成を管理します。 

1. Azure マネージド ディスクのサイズを **64 GB**に増やすには、Cloud Shell 内の PowerShell セッションから次のコマンドを実行します。

   ```pwsh
   New-AzDiskUpdateConfig -DiskSizeGB 64 | Update-AzDisk -ResourceGroupName $rgName -DiskName $diskName
   ```

1. 変更が有効になっていることを確認するには、次の手順を実行します。

   ```pwsh
   Get-AzDisk -ResourceGroupName $rgName -Name $diskName
   ```

1. ディスク パフォーマンス SKU を **Premium_LRS** に変更するには 、Cloud Shell 内の PowerShell セッションで次の手順を実行します。

   ```pwsh
   New-AzDiskUpdateConfig -Sku Premium_LRS | Update-AzDisk -ResourceGroupName $rgName -DiskName $diskName
   ```

1. 変更が有効になっていることを確認するには、次の手順を実行します。

   ```pwsh
   (Get-AzDisk -ResourceGroupName $rgName -Name $diskName).Sku
   ```

#### リソースのクリーンアップ

   >**注**: この課題で展開したリソースは、このモジュールの次のラボで使用するため削除しないでください。

#### レビュー

このラボでは次の内容を学習しました。

- Azure Cloud Shell で PowerShell セッションの開始
- Azure PowerShell を使用してリソース グループと Azure マネージド ディスクの作成
- Azure PowerShell を使用してマネージド ディスクの構成