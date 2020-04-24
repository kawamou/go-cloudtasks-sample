# ローカル→Tasks→GAE
## 概要
- Cloud Tasks試してみる
## Tasksって？
- Task Queueの進化版。構造はほぼ同じ。Pushができるキューイングサービス
- 最初の100万回まで無料
- GAE等がfor文ぶん回してTasksにタスクを投入。TasksはworkerにPOSTリクエスト飛ばす(Push)
- POSTリクエストを契機にworkerが動き出し、処理が終わったら2xxなりを返す
- レスポンスなかったらTasksはもう一度タスクを実行する(Push)
- この図が分かりやすい: https://cloud.google.com/tasks/docs/dual-overview
## 実践
### ローカルからGCP叩けるようにアクセストークン取得
```
❯ gcloud auth application-default login
```
### GAEにデプロイ
- TasksにタスクをPushされるワーカーインスタンス作る
```
❯ go get github.com/kawamou/go-cloudtasks-sample
❯ cd $GOPATH/src/github.com/kawamou/go-cloudtasks-sample/handle_task
❯ gcloud app deploy
# 動作確認
❯ gcloud app browse
```
```
# cat app.yaml
runtime: go112
env: standard
instance_class: F1 #AutomaticはF以上
service: worker #サービス名を指定

automatic_scaling:
    min_idle_instances: 0
    max_idle_instances: 2
    max_pending_latency: 2000ms
    #GAEはRequestをインスタンスに割り振る際にPending Request Queueに保持する
    #Requestを何秒待たせるかの設定
    #長くするとAutoscale発生を抑えられる
```
### Cloud Tasks作成
- tasksコマンドだとqueue.yml使うの推奨されてないっぽい
- gcloud app deployする
```
❯ cd ..
❯ gcloud app deploy queue.yml
# 動作確認
❯ gcloud tasks queues describe first-queue
```
- Token Bucketアルゴリズムに従う
- Bucketに貯められたToken(Pushして良いですよ〜っていう許可証みたいな感じ)をひとつ消費してPushする
- Tokenが枯渇するとキューにタスクが貯まる。Tokenが補充されるとPushを再開する
- Tokenがある限りタスクは無限に実行され続けるので、max_concurrent_requestsを設定し、同時実行できるタスク数を制限する
```
# cat queue.yml
queue:
  - name: first-queue
    bucket_size: 50 #Token BucketアルゴリズムのToken数
    rate: 0.5/s #Tokenの補充レート
    target: worker #ターゲットのインスタンス(サービス名)
    max_concurrent_requests: 2 #同時実行の数
    retry_parameters:
      task_retry_limit: 0
      task_age_limit: 1s
      min_backoff_seconds: 2 #タスク処理の裁定間隔
      max_backoff_seconds: 3600
      max_doublings: 16
```
### 実行
```
❯ cd create_task
# LOCATION_ID確認
❯ gcloud tasks locations list
❯ export PROJECT_ID=PROJECT_ID
❯ export LOCATION_ID=LOCATION_ID
❯ export QUEUE_ID=first-queue
# タスクをキューに突っ込むとworkerにPushして処理される
❯ go run . $PROJECT_ID $LOCATION_ID $QUEUE_ID hello
# 疎通確認
❯ gcloud app logs read --service=worker
```
## 参考
- https://cloud.google.com/tasks/docs/quickstart-appengine
- https://note.com/umaaai/n/nc383034d96d9
- https://addsict.hatenablog.com/entry/2018/09/24/125820