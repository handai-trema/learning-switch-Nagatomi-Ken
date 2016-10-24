##マルチプルテーブルを読む
OpenFlow1.3 版スイッチの動作を説明しよう。

スイッチ動作の各ステップについて、trema dump_flows の出力 (マルチプルテーブルの内容) を混じえながら動作を説明すること。

##解答

まずtremaを起動した直後のフローテーブルを
```
./bin/trema dump_flows lsw
```

を用いて確認した結果を以下に示す。

```
enyuu2@ensyuu2-VirtualBox:~/learning-switch-Nagatomi-Ken$ ./bin/trema dump_flows lsw
cookie=0x0, duration=86.788s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=86.751s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=86.751s, table=0, n_packets=0, n_bytes=0, priority=1 actions=goto_table:1
cookie=0x0, duration=86.751s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=86.751s, table=1, n_packets=0, n_bytes=0, priority=1 actions=CONTROLLER:65535
```

この結果は上の行から、
* 宛先MACアドレスがマルチキャストMACアドレスならばdropする（優先度２）
* 宛先MACアドレスがIPv6マルチキャストMACアドレスならばdropする（優先度２）
* table1のフォワーディングテーブルに移行（優先度１）
* 宛先MACアド図がブロードキャストアドレスならばフラッディングをする（優先度３）
* コントローラに65535にPacketInする（優先度１）

となっている。

次にパケットをホスト1からホスト2に送信して、その後フローテーブルを確認する

```
ensyuu2@ensyuu2-VirtualBox:~/learning-switch-Nagatomi-Ken$ ./bin/trema send_packets --source host1 --dest host2
ensyuu2@ensyuu2-VirtualBox:~/learning-switch-Nagatomi-Ken$ ./bin/trema dump_flow lsw
cookie=0x0, duration=168.068s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=168.031s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=168.031s, table=0, n_packets=1, n_bytes=42, priority=1 actions=goto_table:1
cookie=0x0, duration=168.031s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=168.031s, table=1, n_packets=1, n_bytes=42, priority=1 actions=CONTROLLER:65535
```

特に先ほどと変化は見られない。これは、ホスト１宛のパケットに対しコントローラにPacketInが発生する。このときコントローラはhost1の情報を記録して、フラッティングを行うのみで、フローテーブルに変化が起きないからである。

次にホスト２からホスト１にパケット送信を行い、同様にフローテブルを確認した。

```
ensyuu2@ensyuu2-VirtualBox:~/learning-switch-Nagatomi-Ken$ ./bin/trema send_packets --source host2 --dest host1
ensyuu2@ensyuu2-VirtualBox:~/learning-switch-Nagatomi-Ken$ ./bin/trema dump_flow lsw
cookie=0x0, duration=209.958s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=209.921s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=209.921s, table=0, n_packets=2, n_bytes=84, priority=1 actions=goto_table:1
cookie=0x0, duration=209.921s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=5.167s, 　　table=1, n_packets=0, n_bytes=0, idle_timeout=180, priority=2,in_port=2,dl_src=d8:6a:fe:c8:f1:3f,dl_dst=09:51:10:d1:92:bb actions=output:1
cookie=0x0, duration=209.921s, table=1, n_packets=2, n_bytes=84, priority=1 actions=CONTROLLER:65535
```

下から２行目が追加されていることがわかる。
この行はポート２から、送信元アドレスd8:6a:fe:c8:f1:3f、宛先アドレス09:51:10:d1:92:bbのパケットが来た時に、ポート１にそのパケットを出力するという処理である（優先度２）


もう一度ホスト１からホスト２にパケットを送信した
```
ensyuu2@ensyuu2-VirtualBox:~/learning-switch-Nagatomi-Ken$ ./bin/trema send_packets --source host1 --dest host2
ensyuu2@ensyuu2-VirtualBox:~/learning-switch-Nagatomi-Ken$ ./bin/trema dump_flow lsw
cookie=0x0, duration=227.261s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=227.224s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=227.224s, table=0, n_packets=3, n_bytes=126, priority=1 actions=goto_table:1
cookie=0x0, duration=227.224s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=3.302s,　　 table=1, n_packets=0, n_bytes=0, idle_timeout=180, priority=2,in_port=1,dl_src=09:51:10:d1:92:bb,dl_dst=d8:6a:fe:c8:f1:3f actions=output:2
cookie=0x0, duration=22.47s, 　　table=1, n_packets=0, n_bytes=0, idle_timeout=180, priority=2,in_port=2,dl_src=d8:6a:fe:c8:f1:3f,dl_dst=09:51:10:d1:92:bb actions=output:1
cookie=0x0, duration=227.224s, table=1, n_packets=3, n_bytes=126, priority=1 actions=CONTROLLER:65535
```

下から３行目が追加された行である。これは先ほどと反対に、
ポート1から、送信元アドレス09:51:10:d1:92:bb、宛先アドレスd8:6a:fe:c8:f1:3fのパケットが来た時に、ポート2にそのパケットを出力するという処理である（優先度２）
