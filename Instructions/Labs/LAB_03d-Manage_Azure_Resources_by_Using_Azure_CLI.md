---
lab:
    title: '03d - Azure CLI を使用して Azure リソースを管理する'
    module: 'モジュール 03 - Azure 管理'
---

# ラボ 03d - Azure CLI を使用して Azure リソースを管理する
# 受講者用ラボ マニュアル

## ラボ シナリオ

Azure portal 、Azure リソース マネージャー テンプレート、および Azure PowerShell を使用して、リソースのプロビジョニングに関連する基本的な Azure 管理機能を確認し、リソース グループに基づいてリソースを整理したら、次に、リソースを Azure CLI を使用して同等のタスクを実行します。Azure CLI のインストールを回避するには、Azure Cloud Shell で使用できる Bash 環境を活用します。

## 目標

このラボでは、次の内容を学習します。

+ タスク 1: Azure Cloud Shell で Bash セッションを開始する
+ タスク 2: Azure CLI を使用してリソース グループと Azure マネージド ディスクを作成する
+ タスク 3: Azure CLI を使用してマネージド ディスクを構成する

## 予想時間: 20分間

## 指示

### 演習 1

#### タスク 1: Azure Cloud Shell で Bash セッションを開始する

このタスクでは、Cloud Shell で Bash セッションを開きます。 

1. ポータルから、Azure portal の右上にあるアイコンをクリックして **Azure Cloud Shell** を開きます。

1. **Bash** または **PowerShell** のいずれかを選択するように求められた場合、[**Bash**] を選択します。      

    >**注**: **Cloud Shell** を初めて起動し、「**ストレージがマウントされていません**」というメッセージが表示されたら、このラボで使用しているサブスクリプションを選択し、[**ストレージの作成**] をクリックします。 

1. メッセージが表示されたら、**ストレージの作成**をクリックし、[Azure Cloud Shell] ウィンドウが表示されるまで待ちます。 

1. [Cloud Shell] ペインの左上隅にあるドロップダウンメニューに、**[Bash]** が表示されていることを確認します。

#### タスク 2: Azure CLI を使用してリソース グループと Azure マネージド ディスクを作成する

このタスクでは、Cloud Shell 内で Azure CLI セッションを使用して、リソース グループと Azure マネージド ディスクを作成します。

1. 前のラボで作成した **az104-03c-rg1**リソース グループと同じ Azure リージョンにリソース グループを作成するには、Cloud Shell 内の Bash セッションから次のコマンドを実行します。 

   ```sh
   LOCATION=$(az group show --name 'az104-03c-rg1' --query location --out tsv)

   RGNAME='az104-03d-rg1'

   az group create --name $RGNAME --location $LOCATION
   ```
1. 新しく作成されたリソース グループのプロパティを取得するには、次の手順を実行します。

   ```sh
   az group show --name $RGNAME
   ```
1. このモジュールの前のラボで作成したものと同じ特性を持つ新しいマネージド ディスクを作成するには、[Cloud Shell] 内の [Bash] セッションから次のコードを実行します。

   ```sh
   DISKNAME='az104-03d-disk1'

   az disk create \
   --resource-group $RGNAME \
   --name $DISKNAME \
   --sku 'Standard_LRS' \
   --size-gb 32
   ```
    >**Note**: 複数行の構文を使用する場合は、各行の末尾に後続スペースが入っていないバックスラッシュ (`\`) で終わり、各行の先頭に行間スペースが入らないようにしてください。

1. 新しく作成されたディスクのプロパティを取得するには、次の手順を実行します。

   ```sh
   az disk show --resource-group $RGNAME --name $DISKNAME
   ```

#### タスク 3: Azure CLI を使用してマネージド ディスクを構成する

このタスクでは、クラウド シェル内で Azure CLI セッションを使用して、Azure 管理ディスクの構成を管理します。 

1. Azure マネージド ディスクのサイズを**64 GB** に増やすには 、Cloud Shell 内の Bash セッションから次を実行します。 

   ```sh
   az disk update --resource-group $RGNAME --name $DISKNAME --size-gb 64
   ```

1. 変更が有効になっていることを確認するには、次の手順を実行します。

   ```sh
   az disk show --resource-group $RGNAME --name $DISKNAME --query diskSizeGb
   ```

1. ディスクパフォーマンス SKU を **Premium_LRS** に変更するには、Cloud Shell 内の Bash セッションで次の手順を実行します。 

   ```sh
   az disk update --resource-group $RGNAME --name $DISKNAME --sku 'Premium_LRS'
   ```

1. 変更が有効になっていることを確認するには、次の手順を実行します。

   ```sh
   az disk show --resource-group $RGNAME --name $DISKNAME --query sku
   ```

#### リソースのクリーンアップ

   >**注**: 使用しない新しく作成された Azure リソースは必ず削除してください。使用していないリソースを削除することで、予期しないコストが発生しなくなります。

1. Azure Portal で、 **[Cloud Shell]** ペインで **[Bash]** セッションを開始します。

1. 次のコマンドを実行して、このモジュールのラボ全体で作成されたすべてのリソース グループを一覧表示します。

   ```sh
   az group list --query "[?starts_with(name,'az104-03')].name" --output tsv
   ```

1. 次のコマンドを実行して、このモジュールのラボ全体で作成したすべてのリソース グループを削除します。

   ```sh
   az group list --query "[?starts_with(name,'az104-03')].[name]" --output tsv | xargs -L1 bash -c 'az group delete --name $0 --no-wait --yes'
   ```

    >**注記**: コマンドは非同期に実行されるので (--nowait パラメーターでの決定に従い)、同じ Bash セッション内で、すぐに別の Azure CLI コマンドを実行できますが、リソース グループが実際に削除されるまでに数分かかります。

#### レビュー

このラボでは次の内容を学習しました。

- Azure Cloud Shell で Bash セッションの開始。
- Azure CLI を使用してリソース グループと Azure マネージド ディスクの作成
- Azure CLI を使用してマネージド ディスクの構成