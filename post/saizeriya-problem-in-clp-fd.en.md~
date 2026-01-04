---
type: post
title: 「サイゼリヤで1000円あれば最大何kcal摂れるのか」を制約論理プログラミングで解く
date: 2020-01-29
hero: "/saizeriya-problem-in-clp-fd/saizeriya.jpg"
tags:
  - Prolog
  - CLP(FD)
  - サイゼリヤ
authors:
  - Yutaka Ichibangase
---
# はじめに

「サイゼリヤで1000円あれば最大何kcal摂れるのか」あるいはサイゼリヤ問題とは [「サイゼリヤのメニューを重複無しで合計1000円以下になるように選んだときに、最大の総カロリーになるようなメニューの組み合わせを求めよ」](https://qiita.com/tanakh/items/a1fb13f78e0576415de3#%E5%95%8F%E9%A1%8C) という問題で、様々な解法が知られている:

- [「サイゼリヤで1000円あれば最大何kcal摂れるのか」を量子アニーリング計算(Wildqat)で解いてみた。](https://qiita.com/hodaka0714/items/cf44b4ece992a39b5be4)
- [「サイゼリヤで1000円あれば最大何kcal摂れるのか」をSMTソルバー(Z3)で解いてみた。](https://qiita.com/tanakh/items/a1fb13f78e0576415de3)
- [「サイゼリヤで1000円あれば最大何kcal摂れるのか」を整数計画法ソルバー(PuLP)で解いてみた。](https://qiita.com/YSRKEN/items/dfc8604eb8598e5e9076)
- [「サイゼリヤで1000円あれば最大何kcal摂れるのか」をマルコフ連鎖モンテカルロで解いてみた。](https://qiita.com/kaityo256/items/b9a146d806dcb4f6edf9)
- [「サイゼリヤで1000円あれば最大何kcal摂れるのか」をExcel のソルバーでで解いてみた。](https://qiita.com/miki_iwa/items/5e0434da8b1150c21741)
- [【Excel】サイゼリヤ1000円で摂れるカロリーの最大値をVLOOKUP関数だけで求める方法](https://www.waenavi.com/entry/20190519/1558269886)
- [「サイゼリヤで1000円あれば最大何kcal摂れるのか」をなでしこでDPで解いてみた。](https://qiita.com/n_chiba_/items/694035ad880126230309)
- [「サイゼリヤで1000円あれば最大何kcal摂れるのか」を自作CPU上で解いてみた](https://anqou.net/poc/2019/05/27/post-2920/)
- [「サイゼリヤで1000円あれば最大何kcal摂れるのか」を全探索で解いてみた](https://primenumber.hatenadiary.jp/entry/2019/05/27/235459)
- [「サイゼリヤで1000円あれば最大何kcal摂れるのか」をSQLで解いてみた。](https://qiita.com/ahinore@github/items/c9a71ac56055a94d9e54)
- [「サイゼリヤで1000円あれば最大何kcal摂れるのか」をTeX言語で計算する ～TeX言語で動的計画法(DP)～](https://doratex.hatenablog.jp/entry/20190520/1558323856)
- [「サイゼリヤで1000円あれば最大何kcal摂れるのか」をPandocで解いてみた](https://zrbabbler.hatenablog.com/entry/2019/05/25/162221)

ここでは [SWI-Prolog](https://www.swi-prolog.org/) の [CLP(FD)](https://www.swi-prolog.org/pldoc/man?section=clpfd) を用いた制約論理プログラミングでの解法を紹介する

# コード

メニューのデータは [サイゼリヤ1000円ガチャをつくってみた(Heroku + Flask + LINEbot)](https://qiita.com/marusho_summers/items/a2d3681fac863734ec8a) で紹介されている [Saizeriya_100yenの#1ブランチ](https://github.com/marushosummers/Saizeriya_1000yen/tree/%231) からとった

大きな流れは [「サイゼリヤで1000円あれば最大何kcal摂れるのか」をSMTソルバー(Z3)で解いてみた。](https://qiita.com/tanakh/items/a1fb13f78e0576415de3) と同じであろう。総額と総カロリーに関する制約を記述し、全メニューに対して0か1かの値の割り当てを行う

```prolog
#!/usr/bin/env swipl

:- use_module(library(clpfd)).
:- initialization(main, main).

main :- time(saizeriya).

saizeriya :-
    % 必要なカラムだけに絞り込む
    findall(menu(Name, Price, Calorie), menu(_, Name, _, _, Price, Calorie, _), Menus),

    % すべてのメニューに対して、注文しない・一つ注文するをそれぞれ0/1で表す
    same_length(Menus, Vars),
    Vars ins 0..1,

    % 合計金額が1,000円を超えない
    maplist(arg(2), Menus, AllPrices),
    scalar_product(AllPrices, Vars, #=<, 1000),

    % 総カロリー
    maplist(arg(3), Menus, AllCalories),
    scalar_product(AllCalories, Vars, #=, Cal),

    % カロリーを最大化するんだという強い意志
    labeling([max(Cal)], Vars),

    print_result(Menus, Vars).

print_result([], []).
print_result([_|Menus], [0|Vars]) :-
    print_result(Menus, Vars).
print_result([menu(Name, Price, Calorie)|Menus], [1|Vars]) :-
    format('~w~n~t~w円~20|~t~wkcal~30|~n', [Name, Price, Calorie]),
    print_result(Menus, Vars).

% https://github.com/marushosummers/Saizeriya_1000yen/tree/%231
% menu(id integer, name text, category text, type text, price integer, calorie integer, salt real)
menu(1, '彩りガーデンサラダ', sidedish, salad, 299, 130, 1.1).
menu(2, '小エビのサラダ', sidedish, salad, 349, 115, 1.3).
menu(3, 'やわらかチキンのサラダ', sidedish, salad, 299, 134, 1.2).
menu(4, 'わかめサラダ', sidedish, salad, 299, 92, 2.1).
menu(5, 'イタリアンサラダ', sidedish, salad, 299, 196, 0.7).
menu(6, 'シーフードサラダ', sidedish, salad, 599, 229, 2.4).
menu(7, '半熟卵とポークのサラダ', sidedish, salad, 599, 433, 2.3).
menu(8, 'コーンクリームスープ', sidedish, soup, 149, 142, 1.1).
menu(9, '冷たいパンプキンスープ(季節限定)', sidedish, soup, 149, 105, 0.9).
menu(10, 'たっぷり野菜のミネストローネ(季節限定)', sidedish, soup, 299, 222, 2.1).
menu(11, '削りたてペコリーノチーズ', sidedish, appetizer, 100, 59, 0.6).
menu(12, 'ミニフィセル', sidedish, appetizer, 169, 188, 1.0).
menu(13, 'ガーリックトースト', sidedish, appetizer, 189, 252, 1.1).
menu(14, 'セットドリンクバー', drink, drinkbar, 190, 0, 0.0).
menu(15, '辛味チキン', sidedish, appetizer, 299, 374, 2.2).
menu(16, 'アスパラガスのオーブン焼き(季節限定)', sidedish, appetizer, 299, 221, 1.1).
menu(17, 'ポップコーンシュリンプ', sidedish, appetizer, 299, 215, 1.4).
menu(18, 'エスカルゴのオーブン焼き', sidedish, appetizer, 399, 256, 1.6).
menu(19, 'ムール貝のガーリック焼き', sidedish, appetizer, 399, 164, 1.3).
menu(20, '野菜ソースのグリルソーセージ', sidedish, appetizer, 399, 570, 3.1).
menu(21, 'チョリソー', sidedish, appetizer, 399, 393, 2.0).
menu(22, '柔らか青豆の温サラダ', sidedish, appetizer, 199, 213, 1.1).
menu(23, 'ほうれん草のソテー', sidedish, appetizer, 199, 138, 1.2).
menu(24, 'キャベツとアンチョビのソテー', sidedish, appetizer, 199, 80, 1.5).
menu(25, 'ポテトのグリル', sidedish, appetizer, 199, 366, 2.0).
menu(26, 'セロリのピクルス(季節限定)', sidedish, appetizer, 199, 52, 1.3).
menu(27, '真イカのパプリカソース', sidedish, appetizer, 199, 138, 1.1).
menu(28, 'フォッカチオ', sidedish, appetizer, 119, 214, 0.8).
menu(29, 'プチフォッカ', sidedish, appetizer, 139, 214, 0.8).
menu(30, 'セットプチフォッカ', sidedish, appetizer, 79, 107, 0.4).
menu(31, 'グラスワイン', drink, alcohol, 100, 0, 0.0).
menu(32, 'デカンタ・250ml', drink, alcohol, 200, 0, 0.0).
menu(33, 'デカンタ・500ml', drink, alcohol, 399, 0, 0.0).
menu(34, 'キリン一番搾り（ジョッキ）', drink, alcohol, 399, 0, 0.0).
menu(35, 'キリン一番搾り（グラス）', drink, alcohol, 299, 0, 0.0).
menu(36, 'ストロングゼロダブルレモン', drink, alcohol, 379, 0, 0.0).
menu(37, 'マグナム・1500ml', drink, alcohol, 1080, 0, 0.0).
menu(38, 'グラッパ', drink, alcohol, 379, 0, 0.0).
menu(39, 'ランブルスコセッコ（赤）辛口', drink, alcohol, 1080, 0, 0.0).
menu(40, 'ランブルスコ（ロゼ）甘口', drink, alcohol, 1080, 0, 0.0).
menu(41, 'ベルデッキオ（辛口・中）', drink, alcohol, 1080, 0, 0.0).
menu(42, 'キャンティ（辛口・やや重い）', drink, alcohol, 1080, 0, 0.0).
menu(43, 'キャンティルフィナリゼルパ（辛口・重い）', drink, alcohol, 2160, 0, 0.0).
menu(44, 'サイゼリヤプレミアム（辛口・重い）', drink, alcohol, 2160, 0, 0.0).
menu(45, 'サントリーオールフリー', drink, alcohol, 259, 0, 0.0).
menu(46, 'フレッシュチーズとトマトのサラダ', sidedish, appetizer, 299, 203, 0.4).
menu(47, 'フレッシュチーズとトマトのサラダ(Wサイズ)', sidedish, appetizer, 598, 406, 0.8).
menu(48, 'プロシュート', sidedish, appetizer, 399, 162, 1.8).
menu(49, 'プロシュート(Wサイズ)', sidedish, appetizer, 798, 324, 3.6).
menu(50, '熟成ミラノサラミ', sidedish, appetizer, 299, 95, 1.1).
menu(51, '熟成ミラノサラミ(Wサイズ)', sidedish, appetizer, 598, 190, 2.2).
menu(52, 'マルゲリータピザ', meal, pizza, 399, 568, 2.5).
menu(53, 'パンチェッタのピザ', meal, pizza, 399, 646, 2.9).
menu(54, '野菜ときのこのピザ', meal, pizza, 399, 610, 2.7).
menu(55, 'やわらかイカのアンチョビのピザ', meal, pizza, 499, 593, 4.6).
menu(56, 'バッファローモッツァレラのピザ', meal, pizza, 499, 575, 2.3).
menu(57, 'ミラノサラミのピザ', meal, pizza, 499, 606, 3.5).
menu(58, 'ほうれん草のグラタン(季節限定)', meal, gratin, 399, 521, 1.9).
menu(59, 'シーフードグラタン', meal, gratin, 499, 537, 2.2).
menu(60, 'アラビアータ', meal, pasta, 399, 591, 4.2).
menu(61, 'ミートソースボロニア風', meal, pasta, 399, 582, 4.3).
menu(62, '半熟卵のミートソースボロニア風', meal, pasta, 468, 672, 4.5).
menu(63, 'アーリオ・オーリオ', meal, pasta, 299, 560, 3.2).
menu(64, 'キャベツのペペロンチーノ', meal, pasta, 399, 686, 3.5).
menu(65, 'タラコソースシシリー風', meal, pasta, 399, 605, 3.7).
menu(66, 'スープ入りトマト味ボンゴレ(季節限定)', meal, pasta, 499, 686, 4.8).
menu(67, 'パルマ風スパゲッティ', meal, pasta, 399, 700, 4.2).
menu(68, 'イカの墨入りスパゲッティ', meal, pasta, 499, 610, 3.8).
menu(69, 'カルボナーラ', meal, pasta, 499, 865, 4.1).
menu(70, 'アスパラガスとエビのクリームスパゲッティ(季節限定)', meal, pasta, 499, 711, 3.5).
menu(71, 'アラビアータ(Wサイズ)', meal, pasta, 770, 1182, 8.4).
menu(72, 'ミートソースボロニア風(Wサイズ)', meal, pasta, 770, 1164, 8.6).
menu(73, 'アーリオ・オーリオ(Wサイズ)', meal, pasta, 574, 1120, 6.4).
menu(74, 'キャベツのペペロンチーノ(Wサイズ)', meal, pasta, 770, 1372, 7.0).
menu(75, 'タラコソースシシリー風(Wサイズ)', meal, pasta, 770, 1210, 7.4).
menu(76, 'スープ入りトマト味ボンゴレ(季節限定)(Wサイズ)', meal, pasta, 976, 1372, 9.6).
menu(77, 'パルマ風スパゲッティ(Wサイズ)', meal, pasta, 770, 1400, 8.4).
menu(78, 'イカの墨入りスパゲッティ(Wサイズ)', meal, pasta, 976, 1220, 7.6).
menu(79, 'カルボナーラ(Wサイズ)', meal, pasta, 976, 1730, 8.2).
menu(80, 'アスパラガスとエビのクリームスパゲッティ(季節限定)(Wサイズ)', meal, pasta, 976, 1422, 7.0).
menu(81, 'トッピング半熟卵', sidedish, appetizer, 69, 90, 0.2).
menu(82, 'ミラノ風ドリア', meal, doria, 299, 542, 2.7).
menu(83, '半熟卵のミラノ風ドリア', meal, doria, 368, 632, 2.9).
menu(84, 'セットプチフォッカ付きミラノ風ドリア', meal, doria, 378, 649, 3.1).
menu(85, 'いろどり野菜のミラノ風ドリア', meal, doria, 399, 590, 3.1).
menu(86, 'エビとイカのドリア', meal, doria, 499, 624, 2.9).
menu(87, 'シーフードパエリア', meal, rice, 599, 602, 3.6).
menu(88, 'エビと野菜のトマトクリームリゾット', meal, rice, 399, 302, 2.2).
menu(89, 'ハヤシ&ターメリックライス', meal, rice, 499, 638, 3.3).
menu(90, '半熟卵のハヤシ＆ターメリックライス', meal, rice, 568, 728, 3.5).
menu(91, 'ミックスグリル', meal, hamburg, 599, 823, 3.8).
menu(92, 'ハンバーグステーキ', meal, hamburg, 399, 514, 2.3).
menu(93, 'デミグラスソースのハンバーグ', meal, hamburg, 499, 628, 3.6).
menu(94, '野菜ソースのハンバーグ(ディアボラ風)', meal, hamburg, 499, 585, 2.6).
menu(95, 'イタリアンハンバーグ', meal, hamburg, 499, 633, 2.5).
menu(96, '焼肉とハンバーグの盛合せ', meal, hamburg, 599, 709, 3.4).
menu(97, '若鶏のグリル(ディアボラ風)', meal, chicken, 499, 541, 2.1).
menu(98, '柔らかチキンのチーズ焼き', meal, chicken, 499, 588, 2.0).
menu(99, 'パンチェッタと若鶏のグリル', meal, chicken, 599, 663, 2.5).
menu(100, 'リブステーキ', meal, steak, 999, 621, 1.5).
menu(101, 'ライス', sidedish, rice, 169, 303, 0.0).
menu(102, 'ラージライス', sidedish, rice, 219, 454, 0.0).
menu(103, 'スモールライス', sidedish, rice, 119, 151, 0.0).
menu(104, 'カプチーノ(アイスケーキ)(季節限定)', sidedish, dessert, 199, 114, 0.1).
menu(105, 'ティラミス(アイスケーキ)', sidedish, dessert, 199, 131, 0.1).
menu(106, 'シナモンフォッカチオ', sidedish, dessert, 169, 246, 0.8).
menu(107, 'プリンとカプチーノの盛合せ', sidedish, dessert, 399, 330, 0.2).
menu(108, 'プリンとティラミスの盛合せ', sidedish, dessert, 399, 347, 0.2).
menu(109, 'ミルクアイスのせシナモンフォッカチオ', sidedish, dessert, 319, 346, 0.9).
menu(110, 'ミルクジェラート', sidedish, dessert, 199, 100, 0.1).
menu(111, 'シチリア産レモンのソルベ', sidedish, dessert, 199, 127, 0.0).
menu(112, 'イタリアンプリン', sidedish, dessert, 249, 216, 0.1).
menu(113, 'チョコレートケーキ', sidedish, dessert, 299, 166, 0.1).
menu(114, 'コーヒーゼリー', sidedish, dessert, 299, 162, 0.1).
menu(115, 'トリフアイスクリーム', sidedish, dessert, 369, 164, 0.1).
```

# 実行結果

上記のコードを `saizeriya_1000yen.pl` として保存し、 `$ chmod +x saizeriya_1000yen.pl` などで実行権限を付与しておく。 `$ ./saizeriya_1000yen.pl` で実行され、しばらくした後、最もカロリーが高い1,000円以下の組み合わせが出力される

```console
$ ./saizeriya_1000yen.pl 
ポテトのグリル
                199円   366kcal
アーリオ・オーリオ(Wサイズ)
                574円  1120kcal
ラージライス
                219円   454kcal
% 2,117,842,889 inferences, 417.754 CPU in 419.840 seconds (100% CPU, 5069588 Lips)
```

# おわりに

SWI-PrologのCLP(FD)を用いたサイゼリヤ問題の解法を紹介した

最近業務でCLP(FD)を使う機会があり、なかなか便利で楽しかったので他のことにも使えないかと思っていたところ、サイゼリヤ問題を思い出したのでやってみた

