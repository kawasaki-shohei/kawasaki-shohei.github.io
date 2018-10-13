---
title: "【Rails】jira-rubyでJira課題の追加と更新"
date: 2018-10-07T22:52:30+09:00
draft: false
---
最近、Railsアプリからjira-rubyというgemを使って、プロジェクト管理ツールであるJiraのREST APIでJiraに課題を追加したり更新したりできるようにしたので、その方法を書いていきます。
jira-rubyに関する日本語の記事は少ないみたいなので、誰かの手助けになれば幸いです。

今回は、前回の[JiraのOAuth認証の実装](./jira-ruby1.md)ができていることを前提として、  
課題の更新や追加に関して書いていきます。。  
 
# バージョン
ruby 2.2.0  
rails 4.2.1  
Jira Software Server 7.11.2

# Jiraのプロジェクトの取得方法
プロジェクトはプロジェクトIDやkeyを指定して取得できます。  

```rb
# 全てのプロジェクトを取得
@client.Project.all
# プロジェクトのidを指定して取得
@client.Project.find(2)
# プロジェクトのキーを指定して取得
@client.Project.find('SAMPLEPROJECT-1')
```

# 課題の取得方法
課題もプロジェクト同様に、課題IDとkeyを指定して取得できます。
存在しない課題の場合、JIRA::HTTPErrorになります。

```rb
# 全ての課題を取得
@client.Issue.all
# 課題IDを指定して取得
@client.Issue.find(1000)
```

# ステータスの取得方法
ステータスもよく似た方法で取得できます。

```rb
# 全てのステータスを取得
@client.Status.all
# ステータスIDを指定して取得
@client.Status.find(id)
```

# 優先度取得方法
優先度も同様です。

```rb
# 全ての優先度を取得
@client.Priority.all
# IDを指定して取得
@client.Priority.find(id)
```

# 課題タイプの取得方法
課題タイプはプロジェクトに紐づいているので、プロジェクトを特定してから取得します。

```rb
@client.Project.find(id).issuetypes
```

# 課題の更新方法
課題を更新するときは、まず更新したい課題を取得します。

```rb
issue = @client.Issue.find(id)
```

次に、更新したいパラメータのみfieldsをkeyとして、値に付与します。  
idが必要なものはidをkeyにしましょう。

```rb
issue.save({"fields" => {
  "project" => {"id" => 2},
  "summary" => "hogehogesummary"],
  "issuetype" => {"id" => 3}
}})
issue.fetch
```

# 課題の追加方法
課題の追加はより複雑です。  
単純に課題を追加するには、以下のようにできます。

```rb
issue = client.Issue.build
labels = ['label1', 'label2']
issue.save({
 "fields" => {
   "summary"   => "blarg from in example.rb",
   "project"   => {"key" => "SAMPLEPROJECT"},
   "issuetype" => {"id" => "3"},
   "labels"    => labels,
   "priority"  => {"id" => "1"}
 }
})
issue.fetch
```

ただ、Jiraでは非常に細かく設定をカスタマイズできます。  
課題のフィールドの入力の許可設定ページ(/secure/admin/ViewIssueFields.jspa)では、  
必須フィールド以外のフィールドを入力禁止にすることができます。  
禁止されているフィールドを追加しようとするとエラーになるので、  
私の場合は、以下のように、必須項目以外は一つずつ更新していきました。  

```rb
issue = @client.Issue.build
# dataは送信したい詳細データ
issue = create_or_update_issue(data, issue)
# 必須フィールド入力してとりあえず課題を生成。
def create_or_update_issue(data, issue)
  begin
    issue.save({"fields" => {
      "project" => {"id" => "#{data["projectId"]}"},
      "summary" => data["summary"],
      "issuetype" => {"id" => "#{data["issueTypeId"]}"}
    }})
  rescue JIRA::HTTPError
    issue
  end
  issue
end

# 上手く課題を生成できれば、残りのフィールドを追加するために、その課題を更新していく。
unless issue.try(:errors) || issue.try(:errorMessages)
  update_unrequire_fields(data, issue.id)
end

# 必須フィールド以外のフィールドの追加
def update_unrequire_fields(data, issue_id)
  unrequire_keys = %w(description priorityId duedate)
  unrequire_keys.each do |key|
    begin
      issue = @client.Issue.find(issue_id)
      issue.save({"fields" => {new_key => (key.include?('Id') ? {"id" => "#{data[key]}"} : "#{data[key]}")}})
    rescue JIRA::HTTPError
      nil
    end
  end
end
```


# 課題のステータスの更新方法
ステータスを更新するのは一苦労入ります。  
Jiraをお使いであれば、わかると思いますが、Jiraではステータスを変更するときはトランジションを指定します。  
REST APIでも一緒です。  
ステータスを変更するときはトランジションを指定しなければいけません。  

```rb
issue = @client.Issue.find(issue_id)
# 変更したいステータスを取得
new_status_name = self.Status.find(status_id).name
# 課題に紐づいているトランジションを全て取得します。
# トランジションとステータスは1対１の関係にあり、toメソッドで変更されるステータスを取得することができます。
# トランジションで変更されるステータスと変更したいステータスを照合して、合致するものを取得します。
begin
  jira_transitions_id = issue.transitions.all.find{|jt| jt.to.name == new_status_name }.id
rescue StandardError
  jira_transitions_id = false
end
if jira_transitions_id
  # 課題に紐づくトランジションをbuildしてからseveします。
  issue_transition = issue.transitions.build
  issue_transition.save!('transition' => { 'id' => jira_transitions_id})
end
```

# エラーハンドリング
エラーとしては、APIではほぼ課題に関するものはほぼ全て、JIRA::HTTPErrorになります。  
ステータスコードは大抵400です。  
詳しくは[こちら](https://docs.atlassian.com/software/jira/docs/api/REST/7.6.1/)をご覧ください。  
ただ、saveするときは失敗した場合は、issue.errorsもしくはissue.errorMessagesにエラーメッセージが入っています。  
アクセストークンが有効ではない場合と認証タイプが違う場合はOAuth::Problemになります。


# まとめ
私が試したJiraの課題に関するとこは以上となります。
Jiraは非常にカスタマイズできる項目が多いため、中々REST APIで対応するのは難しかったです。
何かご指摘があればコメントください。
