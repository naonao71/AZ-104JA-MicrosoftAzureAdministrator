---
lab:
    title: '09c - Azure Kubernetes Service を実装する'
    module: 'モジュール 09 - サーバーレス コンピューティング'
---

# ラボ 09c - Azure Kubernetes Service を実装する
# 受講者用ラボ マニュアル

## ラボ シナリオ

Contoso には、Azure Container Instances を使用して実行するのに適していない多層アプリケーションが多数あります。コンテナー化されたワークロードとして実行できるかどうかを判断するために、Kubernetes をコンテナー オーケストレーターとして使用して評価します。また、簡単化されたデプロイの確認やスケーリング機能など、Azure Kubernetes Service をテストして、さらなる管理上のオーバーヘッドの削減を図ります。

## 目標

このラボでは、次の内容を学習します。

+ タスク 1: Azure Kubernetes Service クラスターをデプロイする
+ タスク 2: Azure Kubernetes Service クラスターにポッドをデプロイする
+ タスク 3: Azure Kubernetes Service クラスターでコンテナー化されたワークロードをスケーリングする

## 予想時間: 40分間

## 手順

### エクササイズ 1

#### タスク 1: Microsoft.Kubernetes および Microsoft.KubernetesConfiguration リソース プロバイダーを登録します。

このタスクでは、Azure Kubernetes サービス クラスターをデプロイするために必要なリソース プロバイダーを登録します。

1. [Azure Portal](https://portal.azure.com) にサインインします。

1. Azure portal の右上にあるアイコンをクリックして **Azure Cloud Shell** を開きます。

1. **Bash** や **PowerShell** のどちらかを選択するためのプロンプトが表示されたら、**PowerShell** を選択します。 

    >**注**: **Cloud Shell** の初回起動時に「**ストレージがマウントされていません**」というメッセージが表示された場合は、このラボで使用しているサブスクリプションを選択し、「**ストレージの作成**」 を選択します。 

1. 「Cloud Shell」 ペインから、次のコマンドを実行して、Microsoft.Insights および Microsoft.AlertsManagement リソース プロバイダーを登録します。

   ```pwsh
   Register-AzResourceProvider -ProviderNamespace Microsoft.Kubernetes
   
   Register-AzResourceProvider -ProviderNamespace Microsoft.KubernetesConfiguration
   ```

1. 「Cloud Shell」 ペインを閉じます。   


#### タスク 2: Azure Kubernetes Service クラスターをデプロイする

このタスクでは、Azure portal を使用して Azure Kubernetes Services クラスターをデプロイします。

1. Azure portalで「**Kubernetes サービス**」を検索してから、「**Kubernetes サービス**」 ブレードで 「**+ 追加**」、「**+ Kubernetes クラスターの追加**」 の順にクリックします。 

1. 「**Kubernetes クラスターを作成**」 ブレードの 「**基本**」 タブで、次の設定を指定します。その他の設定は既定値のままにします。

    | 設定 | 値 |
    | ---- | ---- |
    | サブスクリプション | このラボで使用している Azure サブスクリプションの名前 |
    | リソース グループ | 新しいリソース グループを **az104-09c-rg1** の名前で作成 |
    | Kubernetes クラスター名 | **az104-9c-aks1** |
    | リージョン | Kubernetes クラスターをプロビジョニングできるリージョンの名前 |
    | Kubernetes バージョン | 既定値 |
    | ノード サイズ | 既定値 |
    | ノード数 | **1** |

1. 「**次: ノード プール >**」 をクリックし、「**Kubernetes クラスターを作成**」 ブレードの 「**ノード プール**」 タブで 、次の設定を指定します (その他の設定は既定値のままにします)。

    | 設定 | 値 |
    | ---- | ---- |
    | 仮想ノードを有効にする | **(チェックしない)** |
    | 仮想マシン スケール セットを有効にする | **(チェックされており、変更できません)** |
	
1. 「**次: 認証 >**」 をクリックし、「**Kubernetes クラスターを作成**」 ブレードの 「**認証**」 タブで、次の設定を指定します (その他は既定値のままにします)。

    | 設定 | 値 |
    | ---- | ---- |
    | サービス プリンシパル | **(新規)既定のサービス プリンシパル** |
    | ロールベースのアクセス制御 (RBAC) | **有効** |


1. 「**次: ネットワーク >**」 をクリックし、「**Kubernetes クラスターを作成**」 ブレードの 「**ネットワーク**」 タブで次の設定を指定します (その他の設定は既定値のままにします)。

    | 設定 | 値 |
    | ---- | ---- |
    | ネットワーク構成 | **kubenet** |
    | DNS 名のプレフィックス | グローバルに一意の有効な DNS ホスト名 |

1. 「**次: 統合 >**」 をクリックし、「**Kubernetes クラスターを作成**」 ブレードの 「**統合**」 タブで、「**コンテナーの監視**」 を 「**無効**」 に設定し、「**確認および作成**」 をクリックして、「**作成**」 をクリックします。 

    >**注**: 本番運用のシナリオでは、監視を有効にします。監視は、このラボではカバーしていないので無効にします。 

    >**注**: デプロイが完了するまで待ちます。これにはおよそ10 分かかる場合があります。


#### タスク 3: Azure Kubernetes Service クラスターにポッドをデプロイする

このタスクでは、ポッドを Azure Kubernetes Service クラスターにデプロイします。

1. デプロイ ブレードで、「**リソースに移動**」をクリックします。

1. 「**az104-9c-aks1** Kubernetes サービス」 ブレードの 「**設定**」 セクションで、「**ノード プール**」 をクリック します。

1. 「**az104-9c-aks1 | ノード プール**」 ブレードで、クラスターが 1 つのノードを持つ単一のプールで構成されていることを確認します。

1. Azure portal で右上にあるアイコンをクリックして **Azure Cloud Shell** を開きます。

1. **Azure Cloud Shell** を **Bash** (黒の背景) に切り替えます。

1. 「Cloud Shell」 ペインから、次のコマンドを実行して、AKS クラスターの資格情報を取得します。

    ```sh
    RESOURCE_GROUP='az104-09c-rg1'

    AKS_CLUSTER='az104-9c-aks1'

    az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER
    ``` 

1. 「**Cloud Shell**」 ペインで次のコマンドを実行して、AKS クラスターへの接続性を確認します。

    ```sh
    kubectl get nodes
    ```

1. 「**Cloud Shell**」 ペインで出力を確認し、このポイントでクラスターを構成する 1 つのノードが **Ready** の状態を報告していることを確認します。 

1. 「**Cloud Shell**」 ペインで次のコマンドを実行して、Docker Hub から 「**nginx**」 イメージをデプロイします。

    ```sh
    kubectl create deployment nginx-deployment --image=nginx
    ```

    > **注意**: デプロイの名前「nginx-deployment」を入力するときは、必ず小文字を使用してください

1. 「**Cloud Shell**」 ペインで次のコマンドを実行して、Kubernetes ポッドが作成されたことを確認します。

    ```sh
    kubectl get pods
    ```

1. 「**Cloud Shell**」 ペインで次のコマンドを実行して、デプロイの状態を識別します。

    ```sh
    kubectl get deployment
    ```

1. 「**Cloud Shell**」 ペインで次のコマンドを実行して、インターネットからポッドを使用できるようにします。

    ```sh
    kubectl expose deployment nginx-deployment --port=80 --type=LoadBalancer
    ```

1. 「**Cloud Shell**」 ペインで次を実行して、パブリック IP アドレスがプロビジョニングされているか確認します。

    ```sh
    kubectl get service
    ```

1. 「**nginx-deployment**」 エントリの 「**EXTERNAL-IP**」 列の値が **\<pending\>** からパブリック IP アドレスに変わるまで、このコマンドを再実行します。**nginx-deployment** の **EXTERNAL-IP** 列のパブリック IP アドレスを書き留めます。

1. ブラウザー ウィンドウを開いて、前の手順で取得した IP アドレスに接続します。ブラウザー ページに 「**Welcome to nginx!**」 というメッセージが表示されることを確認します。

#### タスク 4: Azure Kubernetes Service クラスターでコンテナー化されたワークロードをスケーリングする

このタスクでは、ポッドの数とクラスター ノードの数を水平方向にスケーリングします。

1. 「**Cloud Shell**」 ペインで次のコマンドを実行し、ポッドの数を 2 に増やしてデプロイを拡張します。

    ```sh
    kubectl scale --replicas=2 deployment/nginx-deployment
    ```

1. 「**Cloud Shell**」 ペインで次のコマンドを実行して、デプロイのスケーリング結果を確認します。

    ```sh
    kubectl get pods
    ```

    > **注意**: コマンドの出力を確認し、ポッドの数が 2 個に増加したことを確認します。

1. 「**Cloud Shell**」 ペインで次のコマンドを実行し、ノード数を 2 に増やしてクラスターをスケールアウトします。

    ```sh
    az aks scale --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER --node-count 2
    ```

    > **注意**: 追加ノードのプロビジョニングが完了するまで待ちます。これにはおよそ3 分かかる場合があります。失敗した場合は、`az aks scale` コマンドを再実行してください

1. 「**Cloud Shell**」 ペインで次のコマンドを実行して、クラスターのスケーリング結果を確認します。

    ```sh
    kubectl get nodes
    ```

    > **注意**: コマンドの出力を確認し、ノード数が 2 個に増加したことを確認します。

1. 「**Cloud Shell**」 ペインで次のコマンドを実行して、デプロイをスケーリングします。

    ```
    kubectl scale --replicas=10 deployment/nginx-deployment
    ```

1. 「**Cloud Shell**」 ペインで次のコマンドを実行して、デプロイのスケーリング結果を確認します。

    ```
    kubectl get pods
    ```

    > **注意**: コマンドの出力を確認し、ポッドの数が 10 個に増加したことを確認します。

1. 「**Cloud Shell**」 ペインで次のコマンドを実行して、クラスター ノードにボッドが分散されたことを確認します。

    ```
    kubectl get pod -o=custom-columns=NODE:.spec.nodeName,POD:.metadata.name
    ```

    > **注意**: コマンドの出力を確認し、ポッドが両方のノードに分散されていることを確認します。

1. 「**Cloud Shell**」 ペインで次のコマンドを実行して、デプロイを削除します。

    ```
    kubectl delete deployment nginx-deployment
    ```

1. 「**Cloud Shell**」 ペインを閉じます。


#### リソースのクリーンアップ

   >**注**: 新しく作成された Azure リソースのうち、使用しないリソースは必ず削除してください。使用していないリソースを削除することで、予期しないコストが発生しなくなります。

1. Azure portal で、 「**Cloud Shell**」 ペインから **Bash** セッションを開始します。

1. 次のコマンドを実行して、このモジュールのラボ全体で作成されたすべてのリソース グループを一覧表示します。

   ```sh
   az group list --query "[?starts_with(name,'az104-09c')].name" --output tsv
   ```

1. 次のコマンドを実行して、このモジュールのラボ全体で作成したすべてのリソース グループを削除します。

   ```sh
   az group list --query "[?starts_with(name,'az104-09c')].[name]" --output tsv | xargs -L1 bash -c 'az group delete --name $0 --no-wait --yes'
   ```

    >**注意**: このコマンドは非同期処理で実行されるため (--nowait パラメーターによって決まります)、同じ Bash セッション内で、すぐに別の Azure CLI コマンドを実行できますが、リソース グループが実際に削除されるまでに数分かかります。

#### レビュー

このラボでは、次の内容を学習しました。

- Azure Kubernetes Service クラスターの作成
- Azure Kubernetes Service クラスターでのポッドの展開
- コンテナー化されたワークロードの Azure Kubernetes Service クラスターでのスケーリング
