## iOSプロジェクトの作成
Xcodeで適当なbundle identifierを持つ適当なプロジェクトを作成してください。  
(既存のプロジェクトと被らないようなbundle identifierで)
Xcodeのプロジェクト作成完了後、`git init`で初期化してください。

## ruby環境の構築(必要に応じて)
fastlaneを動かすruby環境を構築するために、rbenvを利用します。

```
$ brew install rbenv
$ brew install ruby-build
```

rbenvにパスを通すために、
`~/.bash_profile` に以下を追記します。

```
export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init -)"
```
`~/.bash_profile`を再読込し、  
rubyの2.3.1をインストール、  
それから、現在のディレクトリのrubyバージョンを2.3.1で指定します。
```
$ source ~/.bash_profile
$ rbenv install 2.3.1
$ rbenv local 2.3.1
```

## gemのインストール
今回必要なgemをGemfile.lockからインストールするために、  
Bundlerをインストールします。
```
$ gem install bundler
```

bunlderがインストールされたので、Gemfile.lockから
今回必要なgemをインストールします。

```
$ bundle install --path vendor/bundle
```

## match用のリポジトリを作成
matchが生成・更新・管理する証明書やProvisioning Profileを格納するためのリポジトリを作成します。  
証明書、Provisioning Profileを格納するため、絶対に**Private**であることを確認してください。


## iOSプロジェクトのディレクトリでmatchのセットアップ
一番最初に作成したiOSプロジェクトのディレクトリでmatchを使用する準備を行います。
```
$ bundle exec fastlane match init
```

これにより、証明書、Provisioning Profileの更新があった際に  
match用のリポジトリに証明書、Provisioning Profileがpushされるようになります。

## fastlaneのセットアップ
`bundle exec fastlane...`を実行する度にAppleIDやbundle identifierを何度も入力するのが面倒なので、  
このタイミングで`fastlane init`を実行し、Appfileを生成するようにします。
```
$ bundle exec fastlane init
```

## matchで証明書、Provisioning Profileの作成
準備が整ったので、以下のコマンドでAdHoc用の証明書、Provisioning Profileを作成します。
```
$ bundle exec fastlane match adhoc
```

これを実行することで、Apple Developer Center上に証明書とProvisioning Profileが作成され、  
Keychain Accessにも秘密鍵、証明書が取り込まれた状態になります。

## matchを利用したFastfileの作成
`bundle exec fastlane init`で生成されたFastfileの内容のうち、以下の内容のみを残すように編集します。

```
default_platform :ios

platform :ios do
  desc "Runs all the tests"
  lane :test do
  end

  desc "distribute latest app ipa via DeployGate"
  lane :beta do
    // AdHoc用の証明書、Provisioning Profileをダウンロード、インストール
    match(type: 'adhoc')
    // AdHoc用のipaファイルをビルド
    gym(export_method: 'ad-hoc')
  end
end
```

ローカルで以下を実行し、処理が成功することを確認します。
```
$ bundle exec fastlane beta
```

## CircleCIへのデプロイを準備
fastlaneをCircleCI上で実行するための準備を行います。

### circle.ymlをセットアップ
まずはcircle.ymlを作成し、以下の内容を記述します。
```
machine:
  xcode:
    version: '8.3.2'
  environment:
    override:
    LANG: en_US.UTF-8
dependencies:
  cache_directories:
    - 'vendor/bundle'
  override:
    - bundle install --path vendor/bundle
test:
  override:
    - bundle exec fastlane test
deployment:
  deploygate:
    branch: master
    commands:
      - bundle exec fastlane beta
```

### CircleCIの環境変数をセットアップ
fastlaneをCI環境で利用する際に必要になる以下の環境変数を設定します。
- MATCH_PASSWORD  
(暗号化した証明書、Provisioning Profileの復号のために必要)
- FASTLANE_PASSWORD  
(matchが必要に応じてiOS Developer Centerにアクセスするために必要)
- MATCH_KEYCHAIN_NAME  
(CircleCI特有。CircleCIがデフォルトのkeychainを`circle.keychain`としているため。)
- MATCH_KEYCHAIN_PASSWORD  
(keychainのパスワード。CircleCIの場合は`circle`)
