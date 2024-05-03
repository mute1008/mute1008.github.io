+++
title = "Container Escape: Introduction"
date = "2024-04-21"
tags = [
    "security",
]
draft = true
+++

#### コンテナランタイムの基礎

まずは Container Escape を知るための前提知識として、コンテナランタイムについての説明を行います。

私たちがコンテナを利用するとき、コンテナランタイムと呼ばれるものを通じて利用しています。
これらのランタイムは内部的には2つのコンポーネントからなり、それぞれ High-level Runtime と Low-level Runtime と呼称されています。

これら2つのランタイムは以下のような機能を果たします。

##### High-level Runtime

High-level Runtime はコンテナのライフサイクルを管理するもので、CRIという仕様が策定されています。

- [kubernetes/cri-api](https://github.com/kubernetes/cri-api)

CRI の実装としては、[containerd](https://github.com/containerd/containerd) などが有名です。

Low-level Runtimeほどアタックサーフェスは大きくないものの、コンテナの設定に関わるコンポーネントであるため `CVE-2022-23648` のような脆弱性が発生することがあります。

- [Issue 2244: containerd: Insecure handling of image volumes](https://bugs.chromium.org/p/project-zero/issues/detail?id=2244)

##### Low-level Runtime

Low-level Runtimeはコンテナの生成や実行を担当するランタイムで、以下のように仕様が策定されています。
このランタイムの実装としては、[runc](https://github.com/opencontainers/runc) や [runsc (gVisor)](https://gvisor.dev/) が知られています。

- [Open Container Initiative Runtime Specification](https://github.com/opencontainers/runtime-spec/blob/main/spec.md)
- [Open Container Initiative Image Format Specification](https://github.com/opencontainers/image-spec/blob/main/spec.md)
- [Open Container Initiative Distribution Specification](https://github.com/opencontainers/distribution-spec/blob/main/spec.md)

このランタイムは実際のコンテナ作成処理を行うランタイムであるので、実装ミスや設定のミスによりContainer Escapeを引き起こす脆弱性を埋め込んでしまいやすいです。

2024年4月現在、runcでContainer Escape可能な脆弱性は以下の通りです。

- `CVE-2024-21626`
- `CVE-2021-30465`
- `CVE-2019-19921`
- `CVE-2019-5736`
- `CVE-2016-9962`

ただランタイムがセキュアな場合でも、ユーザーが脆弱な設定を行ったコンテナを作成した場合はそれによってContainer Escapeが発生することがあります。

#### Container Escape 手法

ここからは具体的な Container Escape の手法についてまとめます。
断りのない限り、runc を対象とした説明です。

##### コンテナランタイムの脆弱性

上記でも記述しましたが、コンテナランタイムに脆弱性がある場合はコンテナ環境からホストOSへ影響を与えられます。

// TODO Leaky Vessels

##### ホストのファイルシステム


#### まとめ

#### 参考文献
- https://www.container-security.site/attackers/container_breakout_vulnerabilities.html
- https://south37.hatenablog.com/entry/2020/12/11/%E6%89%8B%E3%82%92%E5%8B%95%E3%81%8B%E3%81%97%E3%81%A6%E5%AD%A6%E3%81%B6%E3%82%B3%E3%83%B3%E3%83%86%E3%83%8A%E6%A8%99%E6%BA%96_-_Container_Runtime_%E7%B7%A8
- https://www.cybereason.com/blog/container-escape-all-you-need-is-cap-capabilities
- https://container-security.dev/security/breakout-to-host.html
- https://www.panoptica.app/research/7-ways-to-escape-a-container
