SNMPTRAP日本語トラップ表示ツール
======================
# やりたいこと

Zabbixなど、snmptrapdで監視していると、日本語版vCenterやWindowsからの日本語トラップが文字化けする。
その文字化けの対処。

# 確認した環境

他のバージョンでも問題ないと思うが念の為

* SNMPTT v1.4beta2
* NET-SNMP Version:  5.7.2

# 実現方法

snmpttは、通常snmptrapdのtraphandle経由で呼び出すが、
間に、マルチバイトを解釈するプログラムを挟み込むことで文字化けを回避する。

## プログラムの挙動

snmptrapからの標準入力の中で、ダブルクォート(")で囲まれた文字列が、
パターン(２桁１６進数+空白)であれば、
マルチバイトとみなしバイナリデータ(マルチバイト文字)に変換する。
その他の文字列は触らない。  
結果を標準出力する。

## プログラム通過後の後処理(snmptrapd.conf記述)

以降のSNMPTTで、UTF8の場合はそのまま可読文字となる。
が、CP932の場合は、iconvコマンドでUTF8に変換する。

# 組み込み方

## 当プログラムを、snmptrapdが稼働しているサーバ内に設置。

* ここでは/usr/local/bin/.に設置
* snmptrapd.confのPATHと合わせればどのPATHでも可能

```
# cp -p conv_snmptrap_mb /usr/local/bin/.
# chmod a+x /usr/local/bin/conv_snmptrap_mb
```

## snmptrapd.conf の変更

* snmptrapdコンフィグファイルのtraphandleで呼び出す
* 通常、/usr/sbin/snmptthandlerを呼び出すだけがだが、前処理として以下のように配置

```
disableAuthorization yes

traphandle 1.3.6.1.4.1.311.* /usr/local/bin/conv_snmptrap_mb | iconv -f cp932 | /usr/sbin/snmptthandler
traphandle 1.3.6.1.4.1.6876.* /usr/local/bin/conv_snmptrap_mb | /usr/sbin/snmptthandler
traphandle default /usr/sbin/snmptthandler
```
上記の例では、
* 1.3.6.1.4.1.311.* = windows eventlogのトラップ(CP932の可能性)
  * CP932(UTF8以外)の場合、後続処理でiconvし、UTF8にする
* 1.3.6.1.4.1.6876.* = vCenter eventlogのトラップ(UTF8の可能性)
* 上記以外のOIDは、今までどおり/usr/sbin/snmptthandlerを直接呼び出す
