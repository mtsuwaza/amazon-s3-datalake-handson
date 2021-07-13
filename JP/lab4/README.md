------------------------------------------------------------------------------------
Copyright <first-edit-year> Amazon.com, Inc. or its affiliates. All Rights Reserved.  
SPDX-License-Identifier: MIT-0

------------------------------------------------------------------------------------


# Lab4：アプリケーションログの永続化と長期間データの分析と可視化
ストリームデータを Amazon Kinesis Data Firehose（以降、Kinesis Data Firehose）に送信後、 Amazon S3（以降、S3）に保存することで長期保存します。その後、 Amazon Athena（以降、Athena）を用いて、アドホックな分析を行い、 Amazon QuickSight（以降、QuickSight）で可視化します。


## Section1：S3, Kinesis Data Firehose の設定
### Step1：S3 バケットの作成

 1. AWS マネジメントコンソールのサービス一覧から **S3** を選択し、画面の **[バケットを作成する]** をクリックします。
 
 2. バケット名を以下のルールに従って入力し、画面左下の **[作成]** をクリックします。  

    - バケット名：[YYYYMMDD]-handson-minilake-[Your Name][Your Birthday]
    - [YYYYMMDD]：ハンズオン実施日
    - [Your Name]：ご自身のお名前
    - [Your Birthday]：ご自身の誕生日の日にち


    **Note：** S3 バケット名はグローバルで一意である必要がありますが、バケット作成ができればバケット名は任意でも構いません。
  

### Step2：Kinesis Data Firehose の作成

 1. AWS マネジメントコンソールのサービス一覧から **Kinesis** を選択し、 Kinesis Data Firehose の **[配信ストリームを作成]** をクリックします。
 
 2. **[Delivery stream name]** に「 **minilake1-userID**（任意）」と入力し、 **[Next]** をクリックします。
 
    **Note：** 後続の手順において、「 **/etc/td-agent/td-agent.conf** 」のファイルにある「 **delivery_stream_name minilake1** 」の指定を合わせて変更いただく必要があります。
 
 3. **[Data transformation]** を **[Disabled]** 、 **[Convert record format]** を **[Disabled]** のまま、 **[Next]** をクリックします。
 
 4. **[Choose a destination]** で、「 **Amazon S3** 」を選択します。
 
 5. **[S3 bucket]** は **Step1** で作成したバケットを選択します。 **[S3 Prefix]** に「 **minilake-in1/** 」を入力します。
 
    **Note：** **[S3 Prefix]** の最後の「 **/** 」を忘れないように注意してください。 S3 への出力時のディレクトリとなり、デフォルトの場合、指定プレフィックス配下に「 **YYYY/MM/DD/HH** 」が作られます。
 
 6. 画面右下の **[Next]** をクリックします。
 
 7. **[Buffer interval]** を「 **60** seconds」に設定します。バッファリングは、 Buffer size か Buffer interval のいずれかの条件がみたされるとS3に配信されます。  

    **Note：** 今回は設定しませんが、データの圧縮、暗号化も可能です。大規模データやセキュリティ要件に対して、有効に働きます。 
 
 8. **[Permissions]** で **[Create or update IAM role (自動で割り当てられた IAM ロール名)]** を選択し、  **[Next]** をクリックします。

 9. 続いて、 Review 画面になるので設定値に問題なければ、 **[Create delivery stream]** をクリックします。
 
 10. **[Status]** が「 **Creating** 」となります。数分で「 **Active** 」になるので次の手順に進めてください。


## Section2：EC2 の設定変更
### Step1：IAM ロールのポリシー追加

作成済の「 **handson-minilake-userID**（任意）」の IAM ロールに以下のようにポリシーを追加します。 

 1. AWS マネジメントコンソールのサービス一覧から **IAM** を選択し、 **[Identity and Access Management (IAM)]** 画面の左ペインから **[ロール]** を選択し、「 **handson-minilake-userID**（任意）」のロール名をクリックします。
 
 2. **[アクセス権限]** タブを選択し、 **[ポリシーのアタッチ]** をクリックします。
 
 3. 検索窓で「 **amazonkinesisfirehose** 」と入れ検索し、  **[AmazonKinesisFirehoseFullAccess]** にチェックを入れ、 **[ポリシーのアタッチ]** をクリックします。  
    **Note：** **[AmazonKinesisAnalyticsFullAccess]** ではなく、 **[AmazonKinesisFirehoseFullAccess]** となります。
 
 4. 変更実施したロールの **[アクセス権限]** タブを選択し、 **[AmazonKinesisFirehoseFullAccess]** がアタッチされたことを確認します。

### Step2：Fluentd の設定
Fluentd から Kinesis Data Firehose にログデータを送信するための設定を行います。  

   1. Kinesis Data Firehose のプラグインをインストールします。
 
      **Asset** 資料：[4-cmd.txt](asset/us-east-1/4-cmd.txt)
 
 ```
 $ sudo su -
 # td-agent-gem install fluent-plugin-kinesis -v 2.1.0
 ```
 
   2. プラグインのインストールを確認します。

      **Asset** 資料：[4-cmd.txt](asset/us-east-1/4-cmd.txt)

 ```
 # td-agent-gem list | grep plugin-kinesis
 ```
   **[実行結果例]**  
   
   ```
  fluent-plugin-kinesis (2.1.0)
   ```
 
   3. **Asset** 資料：[4-td-agent2.conf](asset/us-east-1/4-td-agent2.conf)

 3-1. 「 **/etc/td-agent/td-agent.conf** 」の中身を削除（vi のコマンドの「:%d」などで削除）し、**Asset** 資料の「 **4-td-agent2.conf** 」ファイルをエディタで開き中身をコピーし、「**delivery_stream_name minilake1** 」の指定を「**delivery_stream_name minilake1-userID** 」に変更して貼り付けます。
	
 ```
 # vi /etc/td-agent/td-agent.conf
 ```     
 
 3-2. 「 **/etc/init.d/td-agent** 」ファイルを開き、 14 行目辺りに以下の行を追加します。
 
 ```
 # vi /etc/init.d/td-agent
 ```  
 
 **[追記する行の例]**
 
 **Asset** 資料：[4-cmd.txt](asset/us-east-1/4-cmd.txt)
 
 ```
 export AWS_REGION="us-east-1"
 ```
 
  **Note：** リージョンを変更した場合は、適宜変更します。
  

 #### 以下の手順からは、上記両方の場合において実施します。
 
   4. Fluentd を再起動します。
 
       **Asset** 資料：[4-cmd.txt](asset/us-east-1/4-cmd.txt)
 
 ```
 # /etc/init.d/td-agent restart
 ```
 
   5. S3 にデータが出力されていることを確認します。  
   
      **Note：** 数分かかります。（S3 のパスの例：20190927-handson-minilake-test01/minilake-in1/2019/09/27/13）

   6. Kinesis Data Firehose の画面において、作成した **Delivery stream** の「 **minilake1-userID**（任意）」を選択し、 **[Monitoring]** タブをクリック、表示に時間がかかる為、次の手順に進みます。


環境を削除される際は、[こちら](../clean-up/README.md)の手順をご覧ください。
