---
title: "【Rails】jira-rubyでOAuth認証"
date: 2018-10-06T22:52:24+09:00
draft: false
---
最近、Railsアプリからjira-rubyというgemを使って、プロジェクト管理ツールであるJiraのREST APIでJiraに課題を追加したり更新したりできるようにしたので、その方法を書いていきます。
jira-rubyに関する日本語の記事は少ないみたいなので、誰かの手助けになれば幸いです。

まずは、JiraのOAuth認証についてです。  
課題の更新や追加に関しては[こちら](./jira-ruby2.md)をご覧ください。  
 
# バージョン
ruby 2.2.0  
rails 4.2.1  
Jira Software Server 7.11.2

# 概要
JiraのREST APIは認証タイプが3タイプあります。  
- OAuth  
- HTTP Basic Auth  
- Cookie-Based Auth  
今回はJiraからも推奨されている、OAuth認証を使います。OAuth認証といえば、UUIDで違うアプリログインするときのものを想像することが多いですが、ログインするのではなく、OAuth認証で発行されたアクセストークンをキーにして、APIで課題を作成したりする方法を紹介します。

# gemのインストール
今回使うのはjira-rubyというgemです。  

```ruby:Gemfile
gem 'jira-ruby', :require => 'jira-ruby'
```

bundle installします。

# 必要なキーの生成
コンソールで以下を打ってコンシューマーキー、パブリックキーとプライベートキーを生成します。  
コンシューマーキーの生成  

```bash
rake jira:generate_consumer_key
```

パブリックキープライベートキーの生成  

```bash
rake jira:generate_public_cert
```
生成されたコンシューマーキーとパブリックキーをどこかにコピペしておきます。  
パブリックキーを生成した際に、表示される  
-----BEGIN CERTIFICATE-----  
や   
-----END CERTIFICATE-----  
あと、その前後にある改行も必要なのでまるっとコピペしておきます。


# 生成したキーをJiraのアプリケーションリンク設定で設定する
1. jiraのアプリケーションリンクを作成  
  アプリケーションリンクを作成するページ(ホスト名/plugins/servlet/applinks/listApplicationLinks)でリンクを作成します。  
ローカルの場合、http://localhost:3000と作成しておきます。  
ここで、無効なアプリケーションかもしれませんとアラートが表示される。  
これはAtlassian製品かどうかを確認するものなので、次のページへ行くまで「続行」を押します。

2. 受信設定
  作成したアプリケーションリンクを編集します。  
  サイドバーから受信設定を選択し、コンシューマーキーやパブリックキーなど必要項目欄を入力し、保存します。

# clientの生成
### JIRA::Clientインスタンスを生成
以下のoptionsの指定されたキーに値を設定することで、そのJiraのclientを作成できます。  
この時点で当然Jiraへリクエストは送りません。

```rb
options = {
   :site => "https://jira.hogedev.jp/", # 自分のjiraサーバーURL
   :context_path => '',
   :auth_type => :oauth,
   :private_key_file => "rsakey.pem",
   :consumer_key => hogefugaconsumerkey, # 先ほど生成したコンシューマーキー
}
JIRA::Client.new(options)
```
context_pathはドメイン以下のパスを指定するものなので、何も指定しなければ'/jira'になってしまうので、空文字''を設定してあげる必要があります。  
2-Legged-OAuthで認証したい場合はauth_typeを`:oauth_2legged`にします。  
生成した公開キーはroot直下に置き(デフォルト)、private_key_fileで指定します。

### リクエストトークンの生成
コールバックURLを引数に入れて、リクエストトークンを生成します。  
コールバックURLとはこの後、Jiraの認証画面に飛ばすのですが、その認証後にリダイレクトされる、アプリ側のURLです。  
コールバックURLはアプリケーションリンクの受信設定でも指定できます。  
ここで、ホスト名が違っていればエラーになるので、キャッチしてあげるといいでしょう。  
リクエストトークンが生成できれば、アクセストークンを取得する際にもう一度使うのでセッションに入れておいて、認証画面にリダイレクトさせます。

```rb
callback_url = passed_jira_oauth_url
begin
  request_token = @client.request_token(oauth_callback: callback_url)
rescue StandardError
  request_token = false
end
if request_token
  session[:request_token] = request_token.token
  session[:request_secret] = request_token.secret
  redirect_to request_token.authorize_url
end
```

### アクセストークンを取得
Jiraの認証画面にリダイレクトされ、Jiraにログインしていなければログインし、ログインしていれば、「許可」と「拒否」を選択する画面になります。  
ここで、許可をするとアクセストークンにを取得でき、拒否をすると違うパラメータが返ってきます。  
拒否を押すと、  

```rb
params[:oauth_verifier] == "denied"
```

というパラメータでリクエストを受けられるので、許可した場合と拒否した場合のエラーハンドリングをしておきましょう。


### アクセストークンをセット
セッションに入れておいたリクエストトークンと受け取ったoauth_verifiewを元にアクセストークンを生成します。  
生成したアクセストークンはセッションに入れておき、毎回Jiraにリクエストを送るときに取り出します。  
アクセストークンを生成できれば、clientにセットしてあげます。  
これで、ようやくJiraのプロジェクトや課題などをAPIで取得できるようになりました。  

```rb
request_token = client.set_request_token(session[:request_token], session[:request_secret])
access_token = client.init_access_token(oauth_verifier: params[:oauth_verifier])
session[:jira_auth] = {access_token: access_token.token, access_key: access_token.secret}
session.delete(:request_token)
session.delete(:request_secret)
client.set_access_token(session[:jira_auth]['access_token'], session[:jira_auth]['access_key'])
```

アプリケーションリンクの受信設定で、2-Legged-OAuthを許可している場合は、set_access_tokenで空文字を挿入します。  
2-Legged-OAuthではアクセストークンが正しくセットできていなくてもエラーにならず、空配列が返ってくるので、注意してください。  

### アクセストークンをチェックする
もちろん、一度アクセストークンを取得してしまえば、APIを使う度に毎回認証画面に遷移して、アクセストークンを取得し直すみたいなことはしなくてもいいです。

```rb
if session[:jira_auth]
  client.set_access_token(session[:jira_auth]['access_token'], session[:jira_auth]['access_key'])
else
  # アクセストークンを取得する処理
end
```

しかし、アクセストークンは有効期限がありますし、無効化もできます。さらに、セッションに入れているだけなので、セッションがなくなれば使えません。  
なので、毎回clientを生成したらアクセストークンが有効かどうか判断する必要があります。  
ただ、それ用のメソッドを見つけられなかったので、何か他にいい方法があれば教えて欲しいのですが、私は、  

```rb
client.User.myself
```

で一度アクセスしてみて、それで上手く取得できれば有効、できなければ無効ということにしました。

# ルーティング
これまでのものをまとめたルーティングの一例です。  

```rb
# jiraへリクエストを送る前に、毎回アクセストークンを確認するメソッド。
def check_jira_authorization
  @client = jira_new_instance
  if session[:jira_auth]
    @client.set_access_token(session[:jira_auth][:access_token], session[:jira_auth][:access_key])
    if check_access_token(@client)
      return @client
    else
      redirect_jira_authorization
    end
  else
    redirect_jira_authorization
  end
end

# jiraのclientを作るメソッド
def jira_new_instance
  options = {
     :site => "https://hogehoge.atlassian.net/", # 自分のjiraサーバーURL
     :context_path => '',
     :auth_type => :oauth,
     :private_key_file => "rsakey.pem",
     :consumer_key => hogefugaconsumerkey,
  }
  JIRA::Client.new(options)
end

# アクセストークンを判定するためにユーザー情報を取得してみる
def check_access_token(client)
  begin
    client.User.myself
  rescue JIRA::HTTPError
    false
rescue OAuth::Problem  # アクアストークンが無効だった場合はこのエラーになる。
    false
  end
end

# 初めてアクセストークンを取得したい時
def send_jira_oauth
  @client = jira_new_instance
  redirect_jira_authorization
end

# 認証画面にリダイレクトさせるメソッド
def redirect_jira_authorization
  session.delete('jira_auth') if session[:jira_auth]
  callback_url = passed_jira_oauth_url   # action passed_jira_oauthのURL
  begin
    request_token = @client.request_token(oauth_callback: callback_url)
  rescue StandardError
    request_token = false
  end
  if request_token
    session[:request_token] = request_token.token
    session[:request_secret] = request_token.secret
    redirect_to request_token.authorize_url and return
  else
    # URLや認証タイプが間違っっていたときはrequest_tokenが生成できないのでその時の処理
  end
end

# Jiraの認証画面の後に返ってくるアクション
def passed_jira_oauth
  if params[:oauth_verifier] == "denied"
    # 認証画面で拒否をクリックしたときの処理
  else
    @client = jira_new_instance
    request_token = @client.set_request_token(session[:request_token], session[:request_secret])
    access_token = @client.init_access_token(:oauth_verifier => params[:oauth_verifier])
    session[:jira_auth] = {:access_token => access_token.token, :access_key => access_token.secret}
    session.delete(:request_token)
    session.delete(:request_secret)
    @client.set_access_token(session[:jira_auth][:access_token], session[:jira_auth][:access_key])
    if check_access_token(@client)
      # アクセストークンが有効だった場合の処理
    else
      session.delete('jira_auth')
      flash[:alert] = "Jiraの接続に失敗しました。"
      # アクセストークンが無効だった場合の処理
    end
  end
end
```

# まとめ
こちらのOAuth認証は実装は難しくないですが、Jiraのアプリケーションリンクの設定も少しややこしいですね。  
内容のほとんどはjira-rubyのREADMEに書いてあることですが、私が詰まったところなどを付け足して書いてみました。  
長くなってしまったので、課題の更新や追加に関しては[こちら](./jira-ruby2.md)に書くことにします。  
何かご指摘があればコメントください。
