+++
title = "スマートコントラクトの検査ツール調査"
date = "2022-05-18"
tags = [
    "security",
]
+++

スマートコントラクトに対する検査ツールについて調査しました。

##### [Mythril](https://github.com/ConsenSys/mythril)

MythrilはEVMバイトコードに対する検査ツールです。
Symbolic execution / SMT resolver / taint analysisを使用して、問題の検出を行います。

検出可能なモジュールは以下のページで確認できます。

[Analysis Modules](https://mythril-classic.readthedocs.io/en/master/analysis-modules.html)

##### [MythX](https://mythx.io/)

複数の静的解析ツールを組み合わせたSaaSサービスです。

詳細は以下のブログで確認できます。

[MythX Tech: Behind the Scenes of Smart Contract Security Analysis](https://blog.mythx.io/features/mythx-tech-behind-the-scenes-of-smart-contract-analysis/)

このブログによると、内部的には以下のツールが用いられているようです。

- Maru (Static code analysis)
- Harvey (Greybox Fuzzing)
- Mythril (Symbolic Execution and SMT Solving)

Maruについては情報が見つかりませんでしたが、Harveyというツールは、論文として発表されているツールのようです。

[Harvey: A Greybox Fuzzer for Smart Contracts](https://arxiv.org/abs/1905.06944)

##### [slither](https://github.com/crytic/slither)

[SlithIR](https://github.com/crytic/slither/wiki/SlithIR)という中間表現を利用し、テイント解析を行うツールです。

2022年5月現在、76の[Detector](https://github.com/crytic/slither/wiki/Detector-Documentation)に対応しているようです。

以下のページで、今まで検出した脆弱性の一覧がリストされています。

https://github.com/crytic/slither/blob/master/trophies.md

##### [Echidna](https://github.com/crytic/echidna)

スマートコントラクトのファジングツールです。古典的なファザーのようにクラッシュを探すのではなく、不変状態を破ろうとするプロパティベースのファジングツールです。

以下のサイトにチュートリアルがあります。

https://github.com/crytic/building-secure-contracts/tree/master/program-analysis/echidna

##### [Manticore](https://github.com/trailofbits/manticore)

シンボリック実行を利用した検査ツールです。

詳細は以下の論文に記述してあります。

https://arxiv.org/abs/1907.03890

論文によると、Manticoreは平均65.64%のカバレッジを達成しているそうです。

こちらに[チュートリアル](https://github.com/trailofbits/manticore/wiki/Tutorial:-Exercise)があり、使う際にはこちらも利用できそうです。
