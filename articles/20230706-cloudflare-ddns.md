---
title: "CloudFlareでIPv6をddnsしようとしたらクライアントが対応してなくてカッとなってAPIを直接叩いた話"
emoji: "🖋"
type: "tech"
topics: ["cloudflare", "ddns"]
published: true
---

# はじめに

久々に外部からアクセスできる自宅サーバーを立てたくなったのだが、残念ながら昔と違ってIPoEを使っている関係でIPv4アドレスで公開ができなくなってしまっていた。仕方ないのでIPv6アドレスでddnsができないか調べてみたところ、CloudFlareでddnsをやっているような記事がいくつも見つかるので試してみた。
が、広く使われているddclientというソフトはAAAAレコードに対応していなくて使うことができなかった。AAAAレコードに対応しているフォークプロジェクトもあるようだが、ddclientを動かしてログをみていたところAPIを直接叩いてもそれほど難しくなさそうで、APIドキュメントを眺めたらすぐにできそうだったので自前で作ることにした。

# スクリプト

早速だが作成したスクリプト。
やっていることはシンプルで、NICからIPアドレスを取り出しておいて、叩きたいAPIを目指して順番にAPIを叩いていくだけ。

APIキーはあらかじめCloudFlareのマイページで取得しておく必要がある。
その他変数は適宜修正する。

```
#!/bin/bash

INTERFACE="enp3s0f0"
API_KEY=<api key>
EMAIL=<email>
ZONE="example.com"
DOMAIN="sub.example.com"
NAME="sub"

IP=`ip -o a |\
    grep ${INTERFACE} |\
    grep "scope global dynamic mngtmpaddr noprefixroute" |\
    awk '{print $4}' |\
    sed -e 's/\(.*\)\/64/\1/'`

echo "current ip: ${IP}"

ZONE_ID=`curl -H "x-Auth-Key: ${API_KEY}" \
              -H "x-Auth-Email: ${EMAIL}" \
              "https://api.cloudflare.com/client/v4/zones?name=${ZONE}" |\
	     jq -r .result[0].id`

echo "success to fetch zone id: ${ZONE_ID}"

DOMAIN_ID=`curl -H "x-Auth-Key: ${API_KEY}" \
	            -H "x-Auth-Email: ${EMAIL}" \
		        "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/dns_records?type=AAAA&name=${DOMAIN}" |\
           jq -r .result[0].id`

echo "success to fetch domain id: ${DOMAIN_ID}"

generate_body() {
    cat << EOS
{
    "type": "AAAA",
    "name": "$NAME",
    "content": "$IP"
}
EOS
}

curl -X PATCH \
     -H "x-Auth-Key: ${API_KEY}" \
     -H "x-Auth-Email: ${EMAIL}" \
     -H "Content-Type: application/json" \
     -d "$(generate_body)" \
     "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/dns_records/${DOMAIN_ID}"

echo "success to update address"
```

突貫工事で作ったのでエラー処理とか諸々は対応できてないが、とりあえずは動く。

# cronに登録する

ddclientはデフォルトの設定だと5分ごとに更新をかけるようなので、それに合わせてcronに登録した。

```
*/5 * * * * /path/to/ddns-update.sh
```

# まとめ

結構CloudFlareのAPIは整備されててわかりやすい気がする。1〜2時間ぐらいでここまで到達できたし結構なんとでもなるなぁ。
