# ForestPlus
  app && op サーバは 25' 03/24 本番データを使用しています。

#### ディレクトリ構成
<pre>
  /* ルートディレクトリ */
 (d) forestplus/
 ┃
 ┃   /* dockerコンテンツ用イメージ app && op 以外は dockerhub から取得されます */
 ┣━ (d) IMG/
 ┃   ┃  
 ┃   ┣━ (f) forestp-ap.v{number}.{Ymd}.tar.xz	... (5.6G) app鯖 25' 03/31 v.1.2 です。
 ┃   ┃
 ┃   ┗━ (f) forestp-op.v{number}.{Ymd}.tar.xz	... (3.6G) op鯖 25' 04/02 v.1.2 です。
 ┃
 ┃   /* コンテンツ用ディレクトリ */
 ┣━ (d) forest/
 ┃  ┃
 ┃  ┃   /* ap コンテナ用 */
 ┃  ┣━ (d) app/
 ┃  ┃  ┃
 ┃  ┃　┣━ (d) forest/				... (app&&op) /var/www/forest/
 ┃  ┃  ┃
 ┃  ┃　┗━ (d) initdb.d/				... (app) /docker-entrypoint-initdb.d/
 ┃  ┃     ┃  
 ┃  ┃     ┣━ (f) init_db.sh			... /etc/rc.local にて実行定義。docker-compose run 時のみ実行される想定作成。
 ┃  ┃     ┃
 ┃  ┃     ┣━ (f) 1.reforest.tbl.sql		... (500KB) テーブル定義。
 ┃  ┃     ┃
 ┃  ┃     ┗━ (f) 2.reforest.insert.sql		... (19MB)  DBデータ。流し込み完了まで3分程度かかります。
 ┃  ┃
 ┃  ┃   /* minio コンテナ用 S3互換環境構築 */
 ┃  ┣━ (d) minio/
 ┃  ┃  ┃
 ┃  ┃  ┗━ (d) data/				... (minio) バケットが作成される共用ディレク
 ┃  ┃	
 ┃  ┃   /* op コンテナ用 */
 ┃  ┗━ (d) op/
 ┃     ┃
 ┃     ┗━ (d) cron.d/				... (op) /docker-entrypoint-cron.d/
 ┃        ┃  
 ┃        ┣━ (f) cron.sh			... /etc/rc.local にて実行定義。ユーザを指定する場合は要修正
 ┃        ┃
 ┃        ┗━ (f) {user}				... テキストは改行コードに注意。^M (CR) だと動作しません。
 ┃
 ┃   /* ホスト <-> コンテナ間データ共有用に用意 */	
 ┗━ (d) share/ 
     ┃
     ┣━ (d) ./app/				... (app) /mnt
     ┃
     ┗━ (d) ./op/				... (op)  /mnt

</pre>

#### docker イメージ登録 ( app 10min / op 5min )
```
[ubuntu]# sh install-img.sh
```
```
[ubuntu]# docker-compose up -d
```

各サービスの起動完了まで初回 4～5 分程度かかります。<br>
10分経過して各サービスに接続できないようであれば正常動作していない可能性大。


#### from Ubuntu mysql 接続
```
[ubuntu]# mysql -h 127.0.0.1 -P 23306 -uroot -pVworks%3314 --ssl-mode=DISABLED
```
* root(PASSWD) : Vworks%3314
* mysql-client 8.0 使用の場合sslモードを無効にして接続する。
* （コンテナ備考） root アクセス許可のため、mysql.user へ root ユーザ修正済み

```
mysql> USE mysql;
mysql> CREATE user 'root'@'%' IDENTIFIED BY 'Vworks%3314';
mysql> GRANT ALL PRIVILEGES ON * . * TO 'root'@'%' WITH GRANT OPTION;
mysql> FLUSH PRIVILEGES;
```

#### app鯖、DBデータ

#### 確認したこと
 * mysqlデータ領域を volumes: で共用使用とする場合、アクセス権限の問題で mysqld が起動できなくなる。
 * systemctl で sql テキストを流し込んだ場合、テーブル定義は読み込み完了するも、INSERTデータが大きいと流し込み完了する前に終了してしまう。

#### 対応
非推奨とのことですが、/etc/rc.local を使用したところ流し込みに成功しました。

* mysqld が起動完了してから rc.local を実行へ修正
```
/lib/systemd/system/rc-local.service
-> After=network.target mysqld.service
```

* init_db の実行権限がない場合に備えて sh シェルに喰わせる。
```
/etc/rc.local ( perm 0755 )
-> /bin/sh /docker-entrypoint-initdb.d/init_db.sh
```
#### DB 使用データ
* 2.reforest.insert.sql ( 19MB )

2025/03/31 時点のデータとなります。<br>
コンテナ起動から流し込み完了まで、3分程かかります。<br>
mysqldump 時に --skip-extended-insert を使用しているので、<br>
1行出力やめればもっと早くなるかも。<br>

## minio コンテナ注意点 ( S3互換 )
docker-compose.yml 指定の環境変数が以下非推奨となり指定しても使用できません。

##### 非推奨
```
MINIO_ACCESS_KEY={xxxxxxxxxxxxxxxx}
MINIO_SECRET_KEY={xxxxxxxxxxxxxxxx}
```

##### 管理画面用アカウント
```
MINIO_ROOT_USER=minio
MINIO_ROOT_PASSWORD=minio123
```

##### 接続先設定参考
```
endpoint = http://minio:9000
key	 = minio
secret	 = minio123
```
## OP コンテナ注意点 ( cron )
定期実行指定の cron テキスト、改行コードが ^M (CR) の場合、crontab に反映されません。<br>
～ bad minute 判定となり、cron 定期実行が正常動作しなくなります。<br>
cron テキスト修正した場合は、以下でエラーが出ないか確認した方がよいかも。<br>
```
[op]# crontab -u {user} {txt}
```
```
/lib/systemd/system/rc-local.service
-> After=network.target crond.service
```
* ./cron.sh { add /del }
```
/etc/rc.local ( perm 0755 )
-> /bin/bash /docker-entrypoint-cron.d/cron.sh add
```

#### URL
## Top
https://localhost:20443/

## OP (admin)
http://localhost:38080/

## phpMyAdmin
http://localhost:23380/

## for MailHog
http://localhost:28025/

## for minio
http://localhost:39001/
```
id : minio
pw : minio123
```

### system memo ( app && op )

#### MailHog 用 mhsendmail インストール
```
[forestp]# curl -sSL https://github.com/mailhog/mhsendmail/releases/download/v0.2.0/mhsendmail_linux_amd64 -o /usr/local/bin/mhsendmail
[forestp]# chmod +x /usr/local/bin/mhsendmail
```

#### php-fpm php で使用するメールを mhsendmail へ変更
```
[forestp]# vi /etc/php.ini
982c982
< sendmail_path = /usr/sbin/sendmail -t -i
---
> sendmail_path = "/usr/local/bin/mhsendmail --smtp-addr=mailhog:1025"
```

#### fuelphp 修正箇所
mhsendmail では使用できないオプションがあるためオプションを削除する。
```
[forestp]# vi /var/www/forest/bundle/lib/fuel/packages/email/classes/email/driver/mail.php
30c30
<               if ( ! @mail(static::format_addresses($this->to), $this->subject, $message['body'], $message['header'], '-oi -f '.$return_path))
---
>               if ( ! @mail(static::format_addresses($this->to), $this->subject, $message['body'], $message['header']))
```
