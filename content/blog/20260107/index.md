+++
date = '2026-01-07T20:16:03+09:00'
draft = false
title = 'Linux/ROCmでのComfyUIにはバグがある（対処方法あり）'
+++

## 事象

Linux+AMD GPU上で [ComfyUI](https://github.com/Comfy-Org/ComfyUI) を使っていると以下のようなエラーでセグフォする。

__Page not present or supervisor privilege.__ という言葉が穏やかではない。

```
Memory access fault by GPU node-1 (Agent handle: 0x3f18fe70) on address 0x7f925bdff000. Reason: Page not present or supervisor privilege.
Failed to write segment data to pipe: Broken pipe
GPU coredump: handler exited with error (status: 1)
GPU core dump failed
```

### （付記）インストール内容

ComfyUIのコミット `b7d7cc1d496afe3c82279eec74c4d47399aab8ea` で確認している。

インストールには `uv` を使用した。

```sh
uv venv --python 3.13 --seed
uv pip install --pre torch torchvision torchaudio --index-url https://rocm.nightlies.amd.com/v2/gfx120X-all/
uv pip install -r requirements.txt
uv pip install -r manager_requirements.txt
```

## 回避方法

おそらくROCm7.2あたりで解消すると思われる。上流で修正されるまでの間は以下の2つを両方とも行うことで回避できる。

### 1. カーネルパラメータ

カーネルパラメータに `amdgpu.cwsr_enable=0` をつける。GRUBであれば `/etc/default/grub` に追記して `grub-mkconfig` でコンフィグを再生成すればよい。再起動して反映させる。

### 2. ComfyUIのオプション

ComfyUIの起動引数に `--disable-smart-memory` をつける。

```sh
uv run main.py --enable-manager --disable-smart-memory
```

## 参考

- [ROCm 7: finally a workaround for ComfyUI · Issue #157 · YanWenKun/ComfyUI-Docker](https://github.com/YanWenKun/ComfyUI-Docker/issues/157)
- [ComfyUI seems to ignore the --reserve-vram and/or --disable-smart-memory ? Is there anything going wrong ? · Issue #6314 · Comfy-Org/ComfyUI](https://github.com/Comfy-Org/ComfyUI/issues/6314)
- [ROCm + RDNA 4 GPU で発生していたメモリアクセスエラーが修正される | Coelacanth's Dream](https://www.coelacanth-dream.com/posts/2025/12/13/rocm-rdna4-memory-access-error/)
