# Pipewire環境でalsa-card-profileをホームディレクトリに置く
BluetoothヘッドホンでLDACを使うために[pulseaudio-modules-bt](https://aur.archlinux.org/packages/pulseaudio-modules-bt/)を使っていたが、[開発終了がアナウンスされた](https://github.com/EHfive/pulseaudio-modules-bt/issues/154)ためPipewireに乗り換えることにした。移ってみるとネイティブでLDACをサポートしているし使い勝手もよい。

だが、スピーカーは大音量で鳴る。

そういうわけで音量に制限をかける方法をまとめる。

## alsa-card-profileを編集する
PipewireもPulseaudioと同様にalsa-card-profileを読み込む。なのでこのファイルで`volume-limit`を設定すればPulseaudio同様に音量にリミットがかけられる。

ファイルのありかはPulseaudioとは変わって */usr/share/alsa-card-profile/mixer/paths* となる（もともとはPulseaudioが使っていたパスだが）。前回同様に *analog-output.conf.common* を編集する。

```
[Element PCM]
switch = mute
; volume = merge
volume-limit = 1
override-map.1 = all
override-map.2 = all-left,all-right
```

終わったらPipewireを再起動する。


## ホームディレクトリにalsa-card-profileを置く
上記の方法では */usr* 以下のファイルを編集するため、当然ながら以下のような問題がある。

* 管理者でないと変更できない
* 全ユーザーに反映される
* パッケージが更新されるたびに手動で直す必要がある

結論から述べると、Pipewireも[Pulseaudioと同様](https://boronology.github.io/documents/pulseaudio_config_in_home)にホームディレクトリ（正しくは任意のディレクトリ）からalsa-card-profileを読み込める。必要な手続きは以下のとおり。

1. ホームディレクトリにalsa-card-profileをコピーする
2. alsa-card-profileを編集する
3. `ACP_PATHS_DIR`環境変数で読み込み先（ */usr/share/alsa-card-profile/mixer/paths* に相当するパス）を指定する

これだけ。

```sh
$ mkdir -p ~/.config/pipewire/alsa-card-profile/mixer/paths
$ cp /usr/share/alsa-card-profile/alsa-card-profile/mixer/paths/* ~/.config/pipewire/alsa-card-profile/mixer/paths/

#編集
$ vim ~/.config/pipewire/alsa-card-profile/mixer/paths/analog-output.conf.common

$ echo "export ACP_PATHS_DIR=~/.config/pipewire/alsa-card-profile/mixer/paths" >> ~/.xprofile
```

ログインしなおせば環境変数のセットとpipewireの再起動が行われる。


## 調査
環境変数`ACP_PATHS_DIR`はドキュメント化されていない。見つけた方法をまとめておく。

最初はそもそもpipewireが`volume-limit`を解釈できるのかという疑問からはじまった。pipewireのソースで **volume-limit** をgrepしてみる。

```c
static int element_parse_volume_limit(pa_config_parser_state *state) {
    pa_alsa_path *p;
    pa_alsa_element *e;
    long volume_limit;

    pa_assert(state);

    p = state->userdata;

    if (!(e = pa_alsa_element_get(p, state->section, true))) {
        pa_log("[%s:%u] volume-limit makes no sense in '%s'", state->filename, state->lineno, state->section);
        return -1;
    }

    if (pa_atol(state->rvalue, &volume_limit) < 0 || volume_limit < 0) {
        pa_log("[%s:%u] Invalid value for volume-limit", state->filename, state->lineno);
        return -1;
    }

    e->volume_limit = volume_limit;
    return 0;
}
```

こんな関数がみつかる。`volume-limit`を読む機能はちゃんとある。呼び出し元を探すとこんなのが見つかる。

```c
pa_alsa_path* pa_alsa_path_new(const char *paths_dir, const char *fname, pa_alsa_direction_t direction) {
    pa_alsa_path *p;
    char *fn;
    int r;
    const char *n;
    bool mute_during_activation = false;

    pa_config_item items[] = {
        /* [General] */
        { "priority",            pa_config_parse_unsigned,          NULL, "General" },
        { "description-key",     pa_config_parse_string,            NULL, "General" },
        { "description",         pa_config_parse_string,            NULL, "General" },
        { "mute-during-activation", pa_config_parse_bool,           NULL, "General" },
        { "type",                parse_type,                        NULL, "General" },
        { "eld-device",          parse_eld_device,                  NULL, "General" },

        /* [Option ...] */
        { "priority",            option_parse_priority,             NULL, NULL },
        { "name",                option_parse_name,                 NULL, NULL },

        /* [Jack ...] */
        { "state.plugged",       jack_parse_state,                  NULL, NULL },
        { "state.unplugged",     jack_parse_state,                  NULL, NULL },
        { "append-pcm-to-name",  jack_parse_append_pcm_to_name,     NULL, NULL },

        /* [Element ...] */
        { "switch",              element_parse_switch,              NULL, NULL },
        { "volume",              element_parse_volume,              NULL, NULL },
        { "enumeration",         element_parse_enumeration,         NULL, NULL },
        { "override-map.1",      element_parse_override_map,        NULL, NULL },
        { "override-map.2",      element_parse_override_map,        NULL, NULL },
        { "override-map.3",      element_parse_override_map,        NULL, NULL },
        { "override-map.4",      element_parse_override_map,        NULL, NULL },
        { "override-map.5",      element_parse_override_map,        NULL, NULL },
        { "override-map.6",      element_parse_override_map,        NULL, NULL },
        { "override-map.7",      element_parse_override_map,        NULL, NULL },
        { "override-map.8",      element_parse_override_map,        NULL, NULL },
#if POSITION_MASK_CHANNELS > 8
#error "Add override-map.9+ definitions"
#endif
        /* ... later on we might add override-map.3 and so on here ... */
        { "required",            element_parse_required,            NULL, NULL },
        { "required-any",        element_parse_required,            NULL, NULL },
        { "required-absent",     element_parse_required,            NULL, NULL },
        { "direction",           element_parse_direction,           NULL, NULL },
        { "direction-try-other", element_parse_direction_try_other, NULL, NULL },
        { "volume-limit",        element_parse_volume_limit,        NULL, NULL },
        { NULL, NULL, NULL, NULL }
    };
```

こんなのがみつかる。はあはあ、設定項目名に対して関数のポインタをテーブルにしているらしい。引数に`paths_dir`なんてのがあるからここから読むのだろう。続きを見るとこうなっている。

```c
    if (!paths_dir)
        paths_dir = get_default_paths_dir();
```

`get_default_paths_dir()`はこう。

```c
static const char *get_default_paths_dir(void) {
    const char *str;
#ifdef HAVE_RUNNING_FROM_BUILD_TREE
    if (pa_run_from_build_tree())
        return PA_SRCDIR "mixer/paths";
    else
#endif
    if (getenv("ACP_BUILDDIR") != NULL)
        return "mixer/paths";
    if ((str = getenv("ACP_PATHS_DIR")) != NULL)
        return str;
    return PA_ALSA_PATHS_DIR;
}
```

なるほど。 **ACP_BUILD_DIR** 環境変数がなくて **ACP_PATHS_DIR** 環境変数があれば **ACP_PATHS_DIR** に指定したパスから読んでくれるということか。なければ `PA_ALSA_PATHS_DIR` マクロの指す値だ。

念のため。本当に`pa_alsa_path_new()`の引数`paths_dir`には`NULL`が渡されているのか？呼び出しをたどってみる。

`mapping_paths_probe()` -> `pa_alsa_path_set_new()` -> `pa_alsa_path_new()`となっていて、`mapping_paths_probe()`の処理はこう。

```c
static void mapping_paths_probe(pa_alsa_mapping *m, pa_alsa_profile *profile,
                                pa_alsa_direction_t direction, pa_hashmap *used_paths,
                                pa_hashmap *mixers) {

    pa_alsa_path *p;
    void *state;
    snd_pcm_t *pcm_handle;
    pa_alsa_path_set *ps;
    snd_mixer_t *mixer_handle;

    if (direction == PA_ALSA_DIRECTION_OUTPUT) {
        if (m->output_path_set)
            return; /* Already probed */
        m->output_path_set = ps = pa_alsa_path_set_new(m, direction, NULL); /* FIXME: Handle paths_dir */
        pcm_handle = m->output_pcm;
    } else {
        if (m->input_path_set)
            return; /* Already probed */
        m->input_path_set = ps = pa_alsa_path_set_new(m, direction, NULL); /* FIXME: Handle paths_dir */
        pcm_handle = m->input_pcm;
    }
```

`pa_alsa_path_set_new()`の第3引数が`const char *paths_dir`なのでどのみち`NULL`が渡されている。（FIXMEが少し気がかりだが）これで問題解決。