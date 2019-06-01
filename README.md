# functiona-spring-handson
Functional Spring Handson

本ハンズオンで、次の図のような簡易家計簿のAPIサーバーをSpring WebFlux.fnを使って実装します。
あえてSpring BootもDependency Injectionも使わないシンプルなWebアプリとして実装します。

![image](https://user-images.githubusercontent.com/106908/58406552-d2bb6880-80a4-11e9-8edf-e22d6015ebef.png)


## Prerequisite / 前提条件

Reactorの知識が必要になります。以下のハンズオンを事前に実施しておくことを強くオススメします。

* https://github.com/reactor/lite-rx-api-hands-on
* https://docs.google.com/presentation/d/1-0NopTfA-CGiCNvKPDOH9ZDMHhazKuoT-_1R69Wp8qs

## Contents

1. [ウォームアップ](00-warm-up.md) (当日は実施しませんので、事前に実施しておいてください。)
1. [簡易家計簿Moneygerプロジェクトの作成](01-getting-started.md)
1. [YAVIによるValidationの実装](02-validation.md)
1. [R2DBCによるデータベースアクセス](03-r2dbc.md)
1. [Web UIの追加](04-web-ui.md)
1. [例外ハンドリングの改善](05-exception-handling.md)

## Further reading / 参考資料

* https://docs.spring.io/spring/docs/5.2.0.M2/spring-framework-reference/web-reactive.html#webflux-fn
* https://docs.spring.io/spring-data/r2dbc/docs/1.0.0.M2/reference/html/#reference
* https://github.com/making/yavi
