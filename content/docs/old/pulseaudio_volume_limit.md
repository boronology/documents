# PulseAudioで音量制限をかける

## 更新
PulseAudio15.0により以下の記事は古くなっている。かわりに[PulseAudio 15.0時代の設定置き場](https://boronology.github.io/documents/pulseaudio_config_in_home)を参照のこと。


## 目的
非常に大きな音が出るスピーカーを実用的な音量の範囲で使いたい。
そのままの設定では以下のような困ったことがおきる。

* 1%あたりの音量の幅が大きいため細かい音量調整ができない
* 何かの拍子に音量がリセットされ100%になると爆音が鳴り響く

具体的には[Olasonic TW-S7](https://www.olasonic.jp/product/?id=1526376032-555870)。
USBバスパワー駆動のくせにかなりの音量が出る。

## 方法
**/usr/share/alsa-card-profile/mixer/paths** 以下で対象のデバイスに対応する`Element`を探す。
このスピーカーの場合は **analog-output.conf.common** に含まれている。

もともとはこうなっているので
```
[Element PCM]
switch = mute
volume = merge
override-map.1 = all
override-map.2 = all-left,all-right
```

以下のように変更する。

```
[Element PCM]
switch = mute
; volume = merge
volume-limit = 30
override-map.1 = all
override-map.2 = all-left,all-right
```

`volume-limit`の値に上限にしたい音量をパーセントで指定する。今回は30にしたので音量を100%にしても設定前の音量30%の音しか出なくなる。

設定したら`pulseaudio -k`でPulseAudioを再起動する。