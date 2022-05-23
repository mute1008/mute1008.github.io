+++
title = "スマートコントラクトの静的解析ツール調査"
date = "2022-05-18"
tags = [
    "security",
]
draft = true
+++

スマートコントラクトに対する静的解析ツールについて調査しました。

TODO 概要は後で書く

#### [Mythril](https://github.com/ConsenSys/mythril)

MythrilはEVMバイトコードに対する検査ツールです。
Symbolic execution / SMT resolver / taint analysisを使用して、問題の検出を行います。

検出可能なモジュールは以下のページで確認できます。

[Analysis Modules](https://mythril-classic.readthedocs.io/en/master/analysis-modules.html)

#### [MythX](https://mythx.io/)

複数の静的解析ツールを組み合わせたSaaSサービスです。

詳細は以下のブログで確認できます。

[MythX Tech: Behind the Scenes of Smart Contract Security Analysis](https://blog.mythx.io/features/mythx-tech-behind-the-scenes-of-smart-contract-analysis/)

このブログによると、内部的には以下のツールが用いられているようです。

- Maru (Static code analysis)
- Harvey (Grebox Fuzzing)
- Mythril (Symbolic Execution and SMT Solving)

Maruについては情報が見つかりませんでしたが、Harveyというツールは、論文として発表されているツールのようです。

[Harvey: A Greybox Fuzzer for Smart Contracts](https://arxiv.org/abs/1905.06944)

#### [slither](https://github.com/crytic/slither)

#### [Echidna](https://github.com/crytic/echidna)

#### [Manticore](https://github.com/trailofbits/manticore)
