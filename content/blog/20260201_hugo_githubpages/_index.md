+++
date = '2026-02-01T14:46:08+09:00'
draft = false
title = 'GitHub PagesでHugoのページを公開したときの罠'
+++


どういうわけか、 `_index.md` より `README.md` が優先されるらしい。

## 優先度

私のHugoプロジェクトはおおよそ以下のディレクトリ構成となっていた。 `/content/_index.md` がトップページで、 `/blog/` と `/docs/` に各ページを格納している。 `/README.md` はGitHubプロジェクトのトップに表示するためのものだ。

```
├── content
│   ├── _index.md
│   ├── blog
│   └── docs
├── themes
├── README.md
└── hugo.yaml
```

ここで [Host on GitHub Pages](https://gohugo.io/host-and-deploy/host-on-github-pages/) を参考にページをデプロイしたところ、なぜか `/README.md` が `/` に表示された。 

アーティファクトをダウンロードして中身を見たところ以下のようになっており、 `index.html` は `/content/_index.md` の内容となっている。


```
./artifact
├── 404.html
├── index.html
├── robots.txt
├── sitemap.xml
...
```

そのくせ `/robots.txt` や `/sitemap.xml` は表示できる。どうやらアーティファクトを作ったあとに `README.md` の内容を優先させる処理があるらしい。

## 結果

いったん `/README.md` を消すことにした。GitHub Pagesでのホストはいってしまえば副次的なものにすぎないので重視することはないと判断したため。