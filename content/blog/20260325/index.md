+++
date = '2026-03-25T20:06:14+09:00'
draft = false
title = 'Hugoの使い方を間違っていた'
+++

年始にサイトを立ち上げて以来、ずっとRSSが更新されていないのを不思議に思っていた。

原因はずっと `hugo new content 記事のディレクトリ/_index.md` で記事を作っていたこと。 `_index.md` はセクション（section）の目印として扱われるので、通常のページ（regular page）にはならない。ページでないのでRSSにも載らない。

参考

- [index page - Content management](https://gohugo.io/content-management/organization/#index-pages-_indexmd)
- [section - Glossary](https://gohugo.io/quick-reference/glossary/#section)
- [section - Glossary](https://gohugo.io/quick-reference/glossary/#regular-page)


[Directory structure](https://gohugo.io/getting-started/directory-structure/#article) でもとくに述べられていないし、見た目の上ではページは表示されているしで問題に気づいていなかった。

そんなわけでまとめてファイル名を直した。一度にRSSを流してしまってすみません。