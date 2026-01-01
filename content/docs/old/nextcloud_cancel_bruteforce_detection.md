# ブルートフォース防止に引っかかって締め出された場合

nextcloudを使っているとどういうわけか時々Bruteforce detectionに引っかかり、ログインできなくなることがある。原因はまだ調べていない。iPhoneからログインしようとするとしばしば失敗してやりなおすことになっているので、それのせいかもしれない。

以下ではこうなったときの解除方法をまとめる。

## 対処方法
Bruteforce detectionのDBテーブルを直接編集する。テーブル名は`oc_bruteforce_attempts`。

## 具体的な手順
私はDockerでnextcloudを動かしている。なのでまず`docker-compose exec db bash`でDBのコンテナに入り、mysqlに接続する。そうしたら
```sql
select * from oc_bruteforce_attempts;
```
で本当に怪しいアクセスがないか確認したうえで
```sql
delete from oc_bruteforce_attempts where ip = 'IPアドレス';
```
で自分のIPアドレスを含む行を削除する。これで再びログインできるようになるはず。