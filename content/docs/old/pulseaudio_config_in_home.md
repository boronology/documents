# PulseAudio 15.0時代の設定置き場
PulseAudio 15.0が2021年7月にリリースされた。このバージョンで便利な機能が入ったので紹介する。

## 過去の記事と現在の状況
[PulseAudioで音量制限をかける](https://boronology.github.io/documents/pulseaudio_volume_limit)ではデフォルトの音量が非常に大きいデバイスに対して音量の上限を下げるために */usr/share/alsa-card-profile/mixer/paths/* 以下を書き換える方法を扱った。

その後、Arch LinuxではPipeWireが導入さたことに伴ってディレクトリ構成が変更された。具体的には */usr/share/alsa-card-profile/mixer/paths/* 以下はPipeWire用となり、 PulseAudioは代わりに */usr/share/pulseaudio/alsa-mixer/paths/* 以下を参照するようになっている。

## 過去の記事の方法の問題点
前述の記事の方法を使った場合、以下のような問題が存在する。

* */usr* 以下の書き換えのため基本的に管理者権限が必要である
* これらのファイルはパッケージマネージャにより上書きされるため、PulseAudioが更新されるたびに手動で変更する必要がある

## PulseAudio 15.0における変更と解決方法
[PulseAudio 15.0のリリースノート](https://www.freedesktop.org/wiki/Software/PulseAudio/Notes/15.0/)に以下の新機能が紹介されている。

> ALSA path configuration files can now be placed in user home directory
> 
> The code that loads the ALSA path configuration files now checks if the files exist in the directories specified with the XDG_DATA_HOME or XDG_DATA_DIRS environment variables (under pulseaudio/alsa-mixer/paths subdirectory). Those environment variables are defined by the XDG Base Directory Spec, and even if those environment variables aren't set, the XDG Base Directory Spec defines default locations that PulseAudio uses. In particular, in the usual case where XDG_DATA_HOME isn't set, the default value is $HOME/.local/share, so PulseAudio will look for path configuration files from that directory under the user's home. This is useful when it's necessary to customize the path configuration files. Previously the files in /usr/share/pulseaudio/alsa-mixer/paths had to be modified, and the modifications were lost whenever upgrading PulseAudio.

ユーザーのHomeに設定ファイルを置けばそちらを優先して読むようになった（[該当のコミットはこちら](https://gitlab.freedesktop.org/pulseaudio/pulseaudio/-/commit/9b0ae8327d990584bb9a966d8d7bee6badbdb8c0?merge_request_iid=293)）。PulseAudio14のリリース後に取り込まれた機能のため15.0リリースをずっと待ちわびていたが、これにより上述の問題はどちらも解決するようになった。


## 使ってみた
前回同様にスピーカーの設定ファイルは *analog-output.conf.common* である。

```ShellSession
$ mkdir -p ~/.local/share/pulseaudio/alsa-mixer/paths
$ cp /usr/share/pulseaudio/alsa-mixer/paths/analog-output.conf.common ~/.local/share/pulseaudio/alsa-mixer/paths

# ~/.local/share/pulseaudio/alsa-mixer/paths/analog-output.conf.commonを編集
$ $EDITOR ~/.local/share/pulseaudio/alsa-mixer/paths/analog-output.conf.common

# pulseaudioを再起動
$ pulseaudio -k
```

もちろん編集する内容は以前と同じで

```
[Element PCM]
switch = mute
volume = merge
override-map.1 = all
override-map.2 = all-left,all-right
```

から

```
[Element PCM]
switch = mute
; volume = merge
volume-limit = 50
override-map.1 = all
override-map.2 = all-left,all-right
```

のように`volume`パラメータを`volume-limit`に置き換える。

## その他
今回はpathの設定を扱ったが、 */usr/share/pulseaudio/* 以下にはprofile-setも含まれている。これらも当然ホームディレクトリで管理できるようになる。