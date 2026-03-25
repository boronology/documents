+++
date = '2026-03-25T08:08:28+09:00'
draft = false
title = 'さくらVPS収容機材の移行でCPUが変わった'
+++


## TL;DR

ホストサーバー移行でCPUがCascadelakeからSapphireRapidsになってうれしい。

## 何があったか

私はさくらVPSにいくつかのアプリケーションをホストしている。このたび [ホストサーバー収容機材変更に伴うVPS移行手順 — さくらの VPS マニュアル](https://manual.sakura.ad.jp/vps/support/technical/migration.html) にしたがって作業をしろとメールが届いていたので実施した。

ところでさくらVPSは新旧機材が混ざっており、契約時にどちらを引くかはギャンブルという話があった（参考： [【随時更新】国内VPSに使われているCPU一覧・比較表](https://doudonn.com/saba/2826/) ）。今回もCPUをチェックしておくことにした。

### CPU情報

`cat /proc/cpuinfo` の結果（flagより上のみ）を以下に示す。

{{< details title="旧機材のCPU" closed="true" >}}

```
processor	: 0
vendor_id	: GenuineIntel
cpu family	: 6
model		: 85
model name	: Intel(R) Xeon(R) Gold 6212U CPU @ 2.40GHz
stepping	: 7
microcode	: 0x1
cpu MHz		: 2399.998
cache size	: 4096 KB
physical id	: 0
siblings	: 1
core id		: 0
cpu cores	: 1
apicid		: 0
initial apicid	: 0
fpu		: yes
fpu_exception	: yes
cpuid level	: 13
wp		: yes
...
```

{{< /details >}}

{{< details title="新機材のCPU" closed="true" >}}

```
processor       : 3
vendor_id       : GenuineIntel
cpu family      : 6
model           : 143
model name      : Intel Xeon Processor (SapphireRapids)
stepping        : 4
microcode       : 0x1
cpu MHz         : 2000.000
cache size      : 16384 KB
physical id     : 3
siblings        : 1
core id         : 0
cpu cores       : 1
apicid          : 3
initial apicid  : 3
fpu             : yes
fpu_exception   : yes
cpuid level     : 32
wp              : yes
...
```

{{< /details >}}

## まとめ

やったね！

## 最後に

「TL;DR」を一度使ってみたかった。
