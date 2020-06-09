﻿---
lab:
    title: '07 - Azure Storage の管理'
   module: 'モジュール 07 - Azure Storage'
---

# ラボ 07 - Azure Storage を管理する
# 受講者用ラボ マニュアル

## ラボ シナリオ

現在オンプレミスのデータ ストアに存在するファイルを格納するための、Azure Storage の使用を評価する必要があります。これらのファイルの大半には頻繁にアクセスしませんが、例外の場合もあります。アクセス頻度の低いファイルを低価格のストレージ層に配置し、ストレージのコストを最小限に抑えたい場合。また、ネットワーク アクセス、認証、認可、レプリケーションなど、Azure Storage で提供されるさまざまな保護メカニズムについても検討する予定です。最後に、Azure Files サービスがオンプレミスのファイル共有をホストするのに、どの程度適しているかを判断します。

## 目標

このラボでは、次の内容を学習します。

+ タスク 1: ラボ環境をプロビジョニングする
+ タスク 2: Azure Storage アカウントを作成して構成する 
+ タスク 3: Blob Storage を管理する
+ タスク 4: Azure Storage の認証と認可を管理する
+ タスク 5: Azure Files 共有を作成して構成する
+ タスク 6: Azure Storage のネットワーク アクセスを管理する

## 予想時間: 40分間

## 手順

### 演習 1

#### タスク 1: ラボ環境をプロビジョニングする

このタスクでは、このラボの後半で使用する Azure 仮想マシンをデプロイします。 

1. [Azure portal](https://portal.azure.com) にログインします。

1. Azure portal で右上にあるアイコンをクリックして **Azure Cloud Shell** を開きます。

1. **Bash** または **PowerShell** のいずれかを選択するように求められた場合、**PowerShell** を選択します。 

    >**注**: **Cloud Shell** を初めて起動し、「**ストレージがマウントされていません**」というメッセージが表示されたら、このラボで使用しているサブスクリプションを選択し、「**ストレージの作成**」 をクリックします。 

1. 「Cloud Shell」 ペインのツールバーで 「**ファイルのアップロード/ダウンロード**」 アイコンをクリックし、ドロップダウン メニューで 「**アップロード**」 をクリックして、**\\Allfiles\\Module_07\\az104-07-vm-template.json** ファイルと **\\Allfiles\\Module_07\\az104-07-vm-parameters.json** ファイルを Cloud Shell のホーム ディレクトリにアップロードします。

1. 「Cloud Shell」 ペインから次のコマンドを実行して、仮想マシンをホストするリソース グループを作成します (`[Azure_region]` プレースホルダーを Azure 仮想マシンをデプロイする Azure リージョンの名前に置き換えます）。

   ```pwsh
   $location = '[Azure_region]'

   $rgName = 'az104-07-rg0'

   New-AzResourceGroup -Name $rgName -location $location
   ```
1. 「Cloud Shell」 ペインから次のコマンドを実行し、アップロードされたテンプレートとパラメーター ファイルを使用して、仮想マシンをデプロイします。

   ```pwsh
   New-AzResourceGroupDeployment `
      -ResourceGroupName $rgName `
      -TemplateFile $HOME/az104-07-vm-template.json `
      -TemplateParameterFile $HOME/az104-07-vm-parameters.json `
      -AsJob
   ```

    >**注**: デプロイが完了するのを待たずに、次のタスクに進みます。 

1. Cloud Shell ウインドウを閉じます。

#### タスク 2: Azure Storage アカウントを作成して構成する 

このタスクでは、Azure Storage アカウントを作成して構成します。 

1. Azure portal で、**ストレージ アカウント**を検索して選択し、「**+ 追加**」 をクリックします。 

1. 「**ストレージアカウントの作成**」 ブレードの 「**基本**」 タブで、次の設定を指定します (その他の設定は既定値のままにします)。

    | 設定 | 値 | 
    | --- | --- |
    | サブスクリプション | このラボで使用している Azure サブスクリプションの名前 |
    | リソース グループ | 新しいリソース グループを **az104-07-rg1** の名前で作成 |
    | ストレージ アカウント名 | グローバルに一意な、長さが 3 から 24 までの、文字と数字からなる任意の名前 |
    | 場所 | Azure ストレージ アカウントを作成できる Azure リージョンの名前  |
    | パフォーマンス | **Standard** |
    | アカウントのサブタイプ | **Storage (汎用 v1)** |
    | レプリケーション | **Read-access geo-redundant storage (RA-GRS)** |

1. 「**次:ネットワーク >**」 をクリックし、「**ストレージ アカウントの作成**」 ブレードの 「**ネットワーク**」 タブで、利用可能なオプションを確認し、既定のオプション 「**パブリック エンドポイント (すべてのネットワーク)**」 を指定して 、「**次:詳細 >**」 をクリックします。

1. 「**ストレージ アカウントの作成**」 ブレードの 「**詳細**」 タブで、利用可能なオプションを確認し、既定値のまま 「**確認および作成**」 をクリックし、検証プロセスが完了するのを待ってから、「**作成**」 をクリックします。

    >**注**: ストレージ アカウントが作成されるのを待ちます。これには約 2 分かかります。

1. 「デプロイ」 ブレードで、「**リソースに移動**」 をクリックして、「Azure ストレージ アカウント」 ブレードを表示します。 

1. Azure Storage アカウント ブレードの 「**設定**」 セクションで、「**構成**」 をクリックします。

1. 「**アップグレード**」 をクリックして、ストレージアカウントの種類を **Storage (汎用 v1)** から **StorageV2 (汎用 v2)** に変更します。 

1. 「**ストレージ アカウントのアップグレード**」 ブレードで、アップグレードは永続的であり、課金されるという警告を確認し、「**アップグレードの確認**」 テキスト ボックスにストレージ アカウントの名前を入力し、「**アップグレード**」 をクリックします。 

    > **注意**: プロビジョニング時にアカウントの種類を **StorageV2 (汎用 v2)** に設定するオプションがあります。前の 2 つの手順は、既存の汎用 v1 アカウントをアップグレードするオプションがあることを示すために用意されています。

    > **注意**: **StorageV2 (汎用 v2)** には、アクセス階層化など、汎用 v1 アカウントでは使用できない機能が多数用意されています。

    > **注意**: 他の構成オプションを確認します (**アクセス層 (既定)** が現在 「Hot」 に設定されていて変更可能であること。**パフォーマンス**が現在 「**標準**」 に設定されていて、アカウント プロビジョニング中にのみ設定できること。 **ファイル共有の ID ベースのアクセス** では Azure Active Directory Domain Services が必要であること)

1. 「ストレージ アカウント」 ブレードの 「**設定**」 セクションで、「**geo レプリケーション**」 をクリックし、セカンダリの場所をメモします。「**ストレージ エンドポイント**」 ラベルの下の 「**すべての表示**」 リンクをクリックし、「**ストレージ アカウント エンドポイント**」 ブレードをレビューします。  

    > **注意**: 設定どおり、「**ストレージ アカウントのエンドポイント**」 ブレードに、プライマリ エンドポイントとセカンダリ エンドポイントの両方が含まれています。

1. ストレージ アカウントの 「構成」 ブレードに切り替え、「**レプリケーション**」 ドロップダウン リストで 「**geo 冗長ストレージ (GRS)**」 を選択し、変更を保存します。

1. 「**geo レプリケーション**」 ブレードに戻り、セカンダリの場所が指定されていることを確認します。「**ストレージ エンドポイント**」 ラベルの下の 「**すべての表示**」 リンクをクリックし、「**ストレージ アカウント エンドポイント**」 ブレードをレビューします。  

    > **注意**: 設定どおり、「**ストレージ アカウントのエンドポイント**」 ブレードにはプライマリ エンドポイントだけが含まれています。

1. ストレージ アカウントの 「**構成**」 ブレードをもう一度表示し 、「**レプリケーション**」 ドロップダウン リストで**ローカル冗長ストレージ (LRS)** を選択し、変更を保存します。

1. 「**geo レプリケーション**」 ブレードに戻り、この時点では、ストレージ アカウントにプライマリの場所だけがあることを確認します。

1. ストレージ アカウントの 「**構成**」 ブレードをもう一度表示し、**アクセス層 (既定)** を**クール**に設定します。

    > **注意**: クール アクセス層は、たまにしかアクセスされないデータに適しています。

#### タスク 3: Blob Storage を管理する

このタスクでは、BLOB コンテナーを作成し、そのコンテナーに BLOB をアップロードします。 

1. 「ストレージ アカウント」 ブレードの 「**BLOB service**」 セクションで、「**コンテナー**」 をクリックします。

1. 「**+ コンテナー**」 をクリックして、次の設定を使用してコンテナーを作成します。 

    | 設定 | 値 |
    | --- | --- |
    | 名前 | **az104-07-container**  |
    | パブリック アクセス レベル | **プライベート (匿名アクセスはありません)** |

1. コンテナーの一覧で 「**az104-07-container**」 をクリックし、「**アップロード**」 をクリックします。

1. ラボ コンピューターで **\\Allfiles\\Module_07\\LICENSE** を参照し、「**開く**」 をクリックします。

1. 「**BLOB の アップロード**」 ブレードで、「**詳細設定**」 セクションを展開し、次の設定を指定します。その他の設定は既定値のままにします。

    | 設定 | 値 |
    | --- | --- |
    | 認証タイプ | **アカウント キー**  |
    | BLOB タイプ | **Block blob** |
    | ブロック サイズ | **4 MB** |
    | アクセス階層 | **Hot** |
    | フォルダへのアップロード | **licenses** |

    > **注意**: アクセス階層は、個々の BLOB に対して設定できます。

1. 「**アップロード**」 をクリックします。

    > **注意**: アップロードによって、**licenses** という名前のサブフォルダーが自動的に作成されます。

1. 「**az104-07-container**」 ブレードに戻って 「**licenses**」 をクリックし、「**LICENSE**」 をクリックします。

1. 「**licenses-LICENSE**」 ブレードで、使用可能なオプションを確認します。 

    > **注意**: BLOB をダウンロードする。アクセス層を変更する (現在の設定は **ホット**)。リースを取得しリース状態を **ロック** に変更して (現在の設定は **ロック解除**) BLOB を変更または削除できないように保護する。カスタム メタデータを割り当てる (任意のキーと値のペアを指定する) 、などのオプションがあります。また、ファイルを最初にダウンロードすることなく、Azure portal インターフェイス内で直接ファイルを**編集**することもできます。スナップショットを作成したり、SAS トークンを生成することもできます (このオプションは次のタスクで確認します)。 

#### タスク 4: Azure Storage の認証と認可を管理する

このタスクでは、Azure Storage の認証と認可を構成します。

1. 「**licenses-LICENSE**」 ブレードの 「**概要**」 タブで、「**URL**」 エントリの横にある 「**クリップボードにコピー**」 ボタンをクリックします。

1. InPrivate モードを使用して別のブラウザー ウィンドウを開いて、前の手順でコピーした URL に移動します。 

1. **ResourceNotFound**という XML 形式のメッセージが表示されます。

    > **注意**: これは正常の動作であり、作成したコンテナーのパブリック アクセス レベルが 「**プライベート (匿名アクセスなし)**」 に設定されているためです。

1. InPrivate モードのブラウザー ウィンドウを閉じて、Azure Storage コンテナーの 「**licenses-LICENSE**」 ブレードが表示されているブラウザー ウィンドウ に戻り、「**SAS の生成**」 タブに切り替 えます。

1. 「**licenses-LICENSE**」 ブレードの 「**SAS の生成**」 タブで、次の設定を指定します (他の設定は既定値のままにします)。

    | 設定 | 値 |
    | --- | --- |
    | アクセス許可 | **読み取り** |
    | 開始日 | 昨日の日付 |
    | 開始時刻 | 現在の時刻 |
    | 有効期限 | 明日の日付 |
    | 有効期限 | 現在の時刻 |
    | 使用できる IP アドレス | 空白のままにする |
    | 許可されるプロトコル | **HTTP** |
    | 署名キー | **キー 1** |

1. 「**SAS トークンと URL の生成**」 をクリックします。

1. 「**BLOB SAS URL**」 エントリの横にある 「**クリップボードにコピー**」 ボタンをクリックします。

1. InPrivate モードを使用して別のブラウザー ウィンドウを開き、前の手順でコピーした URL に移動します。 

    > **注意**: Microsoft Edge または Internet Explorer を使用している場合は、**MIT ライセンス (MIT)** ページが表示されます。Chrome または Firefox を使用している場合は、ファイルをダウンロードしてメモ帳で開くと、ファイルの内容を表示できます。

    > **注意**: 新しく生成された SAS トークンに基づいてアクセスが承認されるため、これは正常の動作です。 

    > **注意**: BLOB SAS URL を保存します。このラボの後半で使用します。

1. InPrivate モードのブラウザー ウィンドウを閉じ、Azure Storage コンテナーの 「**ライセンス/ライセンス**」 ブレードが表示されているブラウザー ウィンドウに戻り、そこから 「**az104-07-container**」 ブレードに戻ります。

1. 「**認証方法**」 ラベルの横にある 「**Azure AD ユーザー アカウントに切り替える**」 リンクをクリックします。

    > **注意**: この時点で、コンテナーにアクセスできなくなります。 

1. 「**az104-07- コンテナ―**」 ブレードで、「**アクセス制御 (IAM)**」 をクリックします。

1. 「**ロールの割り当てを追加**」 セクションで 「**追加**」 をクリックします。

1. 「**ロール割り当ての追加**」 ブレードで、次の設定を指定します。

    | 設定 | 値 |
    | --- | --- |
    | ロール | **ストレージ BLOB データ所有者** |
    | アクセスの割り当て | **Azure AD ユーザー、グループ、またはサービス プリンシパル** |
    | 選択 | ご自身のユーザー アカウントの名前 |

1. 変更を保存して、「**az104-07- コンテナー**」 コンテナーの 「**概要**」 ブレードに戻り、コンテナーに再度アクセスできることを確認します。

#### タスク 5: Azure Files 共有を作成して構成する

このタスクでは、Azure Files 共有を作成して構成します。

   > **注意**: このタスクを開始する前に、このラボの最初のタスクでプロビジョニングした仮想マシンが実行されていることを確認します。

1. Azure portal で、このラボの最初のタスクで作成したストレージ アカウントのブレードに戻り、「**File サービス**」 セクションの 「**ファイル共有**」 をクリックします。

1. 「**+ ファイル共有**」 をクリックし、次の設定でファイル共有を作成します。 

    | 設定 | 値 |
    | --- | --- |
    | 名前 | **az104-07-share** |
    | クォータ | **1024** |

1. 新しく作成したファイル共有をクリックし、「**接続**」 をクリックします。 

1. 「**接続**」 ブレードで、「**Windows**」 タブが選択されていることを確認し、「**クリップボードにコピー**」 をクリックします。

1. Azure portal で 「**仮想マシン**」 を検索して選択し、仮想マシンの一覧で 「**az104-07-vm0**」 をクリックします。

1. 「**az104-07-vm0**」 ブレードの 「**操作**」 セクションで、「**コマンドの実行**」 をクリックします。 

1. 「**az104-07-vm0- コマンドの実行**」 ブレードで、「**RunPowerShellScript**」 をクリックします。 

1. 「**コマンドスクリプトの実行**」 ブレードで、このタスクの前半でコピーしたスクリプトを 「**PowerShell スクリプト**」 ウインドウに貼り付け、「**実行**」 をクリックします。

1. コマンドが正常に完了したことを確認します。 

1. 「**PowerShell スクリプト**」 ペインの内容を次のスクリプトに置き換え、「**実行**」 をクリックします。

   ```pwsh
   New-Item -Type Directory -Path 'Z:\az104-07-folder'

   New-Item -Type File -Path 'Z:\az104-07-folder\az-104-07-file.txt'
   ```

1. コマンドが正常に完了したことを確認します。 

1. 「**az104-07-share** ファイル共有」 ブレードに戻り、「**更新**」 をクリックして、フォルダの一覧に 「**az104-07-folder**」 があることを確認します。 

1. 「**az104-07-folder**」 をクリックし、ファイルの一覧に 「**az104-07-file.txt**」 があることを確認します。

#### タスク 6: Azure Storage のネットワーク アクセスを管理する

このタスクでは、Azure Storage のネットワーク アクセスを構成します。

1. Azure portal で、このラボの最初のタスクで作成したストレージ アカウントのブレードに戻り、「**設定**」 セクションの 「**ファイアウォールと仮想ネットワーク**」 をクリックします。

1. 「**選択したネットワーク**」 オプションをクリックして、このオプションを有効にしたら使用できる構成設定を確認します。

    > **注意**: これらを設定すると、仮想ネットワークの指定サブネット上の Azure 仮想マシンと、ストレージ アカウントとの直接接続をサービス エンドポイントを使用して構成できます。 

1. 「**クライアント IP アドレスの追加**」 チェック ボックスをクリックして、変更を保存します。

1. InPrivate モードを使用して別のブラウザー ウィンドウを開き、前のタスクで生成した BLOB SAS URL に移動します。 

1. 「**MIT ライセンス (MIT)**」 ページの内容が表示されます。

    > **注意**: これは正常の動作であり、クライアント IP アドレスから接続しているためです。

1. InPrivate モードのブラウザー ウィンドウを閉じ、Azure Storage コンテナーの**ライセンス/LICENSE** ブレードが表示されているブラウザー ウィンドウに戻り、Azure Cloud Shell ウインドウを開きます。

1. Azure portal で右上にあるアイコンをクリックして **Azure Cloud Shell** を開きます。

1. **Bash** または **PowerShell** のいずれかを選択するように求められた場合、**PowerShell**を選択します。 

1. Cloud Shell ペインから以下を実行して、ストレージアカウントの **az104-07-container** コンテナーから LICENSE BLOB のダウンロードを試みます (`[blob SAS URL]` プレースホルダーを前のタスクで生成した BLOB SAS URL に置き換えます)。

   ```pwsh
   Invoke-WebRequest -URI '[blob SAS URL]'
   ```
1. ダウンロードの試行が失敗したことを確認します。 

    > **注意**: **AuthorizationFailure** を示すメッセージが表示されます。**この要求には、この操作を実行する権限がありません**。Cloud Shell インスタンスをホストする Azure VM に割り当てられた IP アドレスから接続しているため、これは正常な動作です。

1. Cloud Shell ウインドウを閉じます。

#### リソースのクリーンアップ

   >**注**: 新しく作成された Azure リソースのうち、使用しないリソースは必ず削除してください。使用していないリソースを削除することで、予期しないコストが発生しなくなります。

1. Azure portalの 「**Cloud Shell**」 ウインドウで、「**PowerShell**」 セッションを開始します。

1. 次のコマンドを実行して、このモジュールのラボ全体で作成されたすべてのリソース グループを一覧表示します。

   ```pwsh
   Get-AzResourceGroup -Name 'az104-07*'
   ```

1. 次のコマンドを実行して、このモジュールのラボ全体で作成したすべてのリソース グループを削除します。

   ```pwsh
   Get-AzResourceGroup -Name 'az104-07*' | Remove-AzResourceGroup -Force -AsJob
   ```

    >**メモ**: コマンドは非同期に実行されるため (-AsJob パラメーターによって決まります)、同じ PowerShell セッション内ですぐに別の PowerShell コマンドを実行できますが、リソース グループが実際に削除されるまでに数分かかります。

#### レビュー

このラボでは次の内容を学習しました。

- ラボ環境の構成
- Azure Storage アカウントの作成と構成 
- BLOB ストレージの管理
- Azure Storage の認証と認可の管理
- Azure Files 共有の作成および構成
- Azure Storage のネットワーク アクセス管理