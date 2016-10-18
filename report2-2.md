##複数スイッチ対応版 ラーニングスイッチ
複数スイッチに対応したラーニングスイッチ (multi_learning_switch.rb) の動作を説明しよう。



##解答
* start
コントローラが起動したタイミングでstartハンドラが呼ばれる。複数スイッチを管理するために、FDBを連想配列として扱っている。、

```
  def start(_argv)
    @fdbs = {}
    logger.info 'MultiLearningSwitch started.'
  end
```

* switch_ready
コントローラがスイッチに接続したときにswitch_readyハンドラが呼び出される。先ほど生成した連想配列@fdbsにスイッチのデータパスIDを用いて新たなFDBを格納している。
```
  def switch_ready(datapath_id)
    @fdbs[datapath_id] = FDB.new
  end

```
                                                  
* packet_in
PacketInが発生するとpacket_inハンドラが呼び出される。このとき更新するFDBを特定するためにfetchを用いてデータパスIDのFDBから目的となるFDBを特定し、学習するようにしている。その後、flow_mod_and_packet_out messageによりFlowModやPacketOutを行っている。
```
  def packet_in(datapath_id, message)
    return if message.destination_mac.reserved?
    @fdbs.fetch(datapath_id).learn(message.source_mac, message.in_port)
    flow_mod_and_packet_out message
  end
```
* flow_mod_and_packet_out

@fdbs.fetch(message.dpid).lookup(message.destination_mac)によりFDBに登録されているMACアドレスを参照し、送信先のMACアドレスが登録されているかされていないかでその後分岐する。登録されているときはフローテーブルにフローエントリーを追加して、送信先のMACアドレスが接続しているポート番号に向けてPacketOutする。登録されていない場合にはfloodオプションをつけフラッディングするようにする。
```
  def flow_mod_and_packet_out(message)
    port_no = @fdbs.fetch(message.dpid).lookup(message.destination_mac)
    flow_mod(message, port_no) if port_no
    packet_out(message, port_no || :flood)
  end

  def flow_mod(message, port_no)
    send_flow_mod_add(
      message.datapath_id,
      match: ExactMatch.new(message),
      actions: SendOutPort.new(port_no)
    )
  end

  def packet_out(message, port_no)
    send_packet_out(
      message.datapath_id,
      packet_in: message,
      actions: SendOutPort.new(port_no)
    )
  end



```

