# Concourseを使ってアプリケーションをアップデートする
ここでは`Conocurse`を利用してアプリケーションの継続的デリバリを体験します。`Concourse`はPivotalが中心に開発をするOSSのパイプラインベースのCI/CDツールです。`Configration As Code`と言われているようにマニフェストにコードとしてパイプラインを定義できることが特徴です。

## flyコマンドのセットアップ
`https://concourse.sys.pas.ik.am`にアクセスしてしてみましょう。

該当するOSのアイコンを押下し、flyコマンドをダウンロードします。
ダウンロードをしたら自身のOSに合わせて実行権限を付与し、パスを通します。
以下はMacの例です。
```console
chmod +x path/to/fly
mv path/to/fly /usr/local/bin
```
新しい端末を立ち上げ以下を実行しflyがインストールされていることを確認して下さい。

```console
$ fly --version
4.2.2
```

`fly`でConcourseへログインします。
```console
$ fly login -t ci -c https://concourse.sys.pas.ik.am/ -n handson-<student id>
```

`https://concourse.sys.pas.ik.am/sky/login?redirect_uri=http://127.0.0.1:54840/auth/callback`のようなURLへのアクセスが指示されるはずなのでブラウザでアクセスをし、`CLOUD FOUNDRY`の認証から自身のログインIDでログインしていください。

ログインが完了したら、ターミナルに戻り、`target saved`と表示されていればログイン成功です。

## Gir Repositoryの作成
[Github](https://github.com)にログインし、`Repositories` -> `New`と進みます。

![image](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/img/cc-1.png)

`Repository Name`に`api-<studentID>`を入力し作成します。`api-<studentID>`プロジェクトのディレクトリに移動し、以下のコマンドを実行します。
```shell
git init
git add .
git commit -m "first commit"
git remote add origin https://github.com/tkaburagi/api-<studentID>.git
git push -u origin master
```

作成したプロジェクトにブラウザで戻りソースコードがpushされていることを確認してください。

**ここまで完了したら進捗シートにチェックをしてください。**

## パイプラインのセットアップ
`api-<studentID>`プロジェクトのディレクトリに`ci`ディレクトリを作成し、下記のファイルを追加します。

ファイル名: `pipeline.yml`

```yaml
---
resources:
  - name: api-<studentID>
    type: git
    source:
      uri: <GIT_URI>
      branch: master
    check_every: 10s
jobs:
- name: unit-test
  plan:
  - get: api-<studentID>
    trigger: true
  - task: mvn-test
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: maven
      inputs:
      - name: api-<studentID>
      caches:
      - path: api-<studentID>/m2   
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          cd api-<studentID>
          rm -rf ~/.m2
          ln -fs $(pwd)/m2 ~/.m2
```

`fly`コマンドでConcourseのパイプラインを更新します。

```shell
fly -t ci set-pipeline -p simple-pipeline -c pipeline.yml
```

![](https://github.com/tkaburagi1214/pcf-workshop/blob/master/image/Screen%20Shot%200030-03-02%20at%201.19.12%20PM.png)

アップロードが成功したらWeb GUIで確認してみます。
ブラウザで`https://concourse.sys.pas.ik.am`にアクセスしてしてみましょう。ユーザ名とパスワードはPCFと同じです。
![](https://github.com/tkaburagi1214/pcf-workshop/blob/master/image/Screen%20Shot%200030-03-02%20at%201.20.54%20PM.png)

CLIからパイプラインを実行します。

```shell
fly -t ci unpause-pipeline -p simple-pipeline
fly -t ci trigger-job -j simple-pipeline/unit-test
```

ブラウザに戻ってパイプラインを確認すると、パイプラインが実行状態に遷移ています。
![](https://github.com/tkaburagi1214/pcf-workshop/blob/master/image/ci-running.png)

ジョブが終了しパイプラインが緑色に変化すれば処理が成功しています。
![](https://github.com/tkaburagi1214/pcf-workshop/blob/master/image/ci-green.png)


パイプラインの`unit-test`のボックスをクリックすると実行内容のログが確認できます。
![](https://github.com/tkaburagi1214/pcf-workshop/blob/master/image/ci-log.png)

**ここまで完了したら進捗シートにチェックをしてください。**

## アプリケーションの修正
次にアプリケーションを修正し、gitにコミット、Concourseからその更新情報を取得し、アプリケーションをテストし、アップグレードという流れを実施します。ここでは先ほどの`circuit-breaker`のコードを修正していきます。端末を開いて以下のコマンドを実行し、アプリケーションからのレスポンス内容を監視します。
```shell
while true;do curl -LI http://api-<studentID>.apps.pcf.pcflab.jp -o /dev/null -w '%{http_code}\n' -s;echo;sleep 1;done
```

まずはConcourseのパイプラインにアプリケーションをビルドとアップグレードするための処理を追加します。
以下のテキストをコピーして`pipeline.yml`をすべて上書きします。

```yaml
---
resources:
  - name: api-<studentID>
    type: git
    source:
      uri: <GIT_URI>
      branch: master
    check_every: 10s
  - name: deploy-to-cf
    type: cf
    source:
      api: api.sys.pcf.pcflab.jp
      username: <STUDENT_ID>
      password: <YOUR_PASSWD>
      organization: handson-<STUDENT_ID>
      space: development
      skip_cert_check: true
jobs:
- name: unit-test
  plan:
  - get: api-<studentID>
    trigger: true
  - task: mvn-test
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: maven
      inputs:
      - name: api-<studentID>
      caches:
      - path: api-<studentID>/m2   
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          cd api-<studentID>
          rm -rf ~/.m2
          ln -fs $(pwd)/m2 ~/.m2
- name: build-and-deploy
  plan:
  - get: api-<studentID>
    passed: [ unit-test ]
    trigger: true
  - task: build
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: maven
      inputs:
      - name: api-<studentID>
      caches:
      - path: api-<studentID>/m2   
      run:
        path: bash
        args:
        - -c
        - |
          set -e
          cd api-<studentID>
          rm -rf ~/.m2
          ln -fs $(pwd)/m2 ~/.m2
          mvn clean package -DskipTests=true
  - put: deploy-to-cf
    params: 
      manifest: api-<studentID>/manifest.yml
      current_app_name: api-<studentID>
```

Concourseのパイプラインを更新します。
```shell
fly -t ci set-pipeline -p simple-pipeline -c pipeline.yml
```

Webブラウザを更新して確認してください。パイプラインがアップデートされていることがわかります。
![](https://github.com/tkaburagi1214/pcf-workshop/blob/master/image/Screen%20Shot%200030-03-02%20at%201.40.26%20PM.png)


次にアプリケーションのソースコードを下記のように変更してください。`api-tkaburagi`アプリの`ApiController.java`に以下のメソッドを追加します。
```java
@RequestMapping(method = RequestMethod.GET, value = "/dummy")
public Object getText() {
    return "I'm available!";
}
```

アプリケーションを修正したらGithubにコミットします。GitにコミットするとConcourseから更新情報が取得され自動的にパイプラインが稼働します。`pcf-workshop-app`ディレクトリにいることを確認してください。
```console
$ pwd 
path/to/api-tkaburagi
$ git add .
$ git commit -m "modified"
$ git push origin master
```

Web GUIでConcourseが実行中のステータスに遷移していることを確認してください。10秒ほど経過すると実行されるはずです。
`unit-test`が完了すると`build-and-artifact`に遷移し、ビルドとデプロイが実行されます。

![](https://github.com/tkaburagi1214/pcf-workshop/blob/master/image/ci-running-2.png)

すべてのタスクが実行されると成功の状態に遷移します。

![](https://github.com/tkaburagi1214/pcf-workshop/blob/master/image/ci-green-2.png)

curlコマンドのコンソールを見るとエラーが発生せずアプリケーションのアップグレードが完了していることがわかります。Webブラウザでアプリケーションにアクセスしてみましょう。

![image](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/img/cc-2.png)

**ここまで完了したら進捗シートにチェックをしてください。**


