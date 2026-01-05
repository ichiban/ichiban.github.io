---
type: post
title: Solving “How many kcal can you consume at Saizeriya with 1,000 yen?” with Constraint Logic Programming
date: 2025-12-30
tags:
  - Prolog
  - CLP(FD)
authors:
  - Yutaka Ichibangase
---

> **Translation note:** This post was translated from Japanese to English by **ChatGPT**.  
> **About Saizeriya:** *Saizeriya* is a popular Japanese low-cost “family restaurant” chain (casual dining) known for very affordable Italian-style dishes.

# Introduction

“How many kcal can you consume at Saizeriya with 1,000 yen?”—also known as the *Saizeriya Problem*—is the following challenge: [“Select Saizeriya menu items without duplicates such that the total price is at most 1,000 yen, and find the combination that maximizes total calories.”](https://qiita.com/tanakh/items/a1fb13f78e0576415de3#%E5%95%8F%E9%A1%8C) Various solutions are known:

* [Tried solving “How many kcal can you consume at Saizeriya with 1,000 yen?” with Quantum Annealing (Wildqat).](https://qiita.com/hodaka0714/items/cf44b4ece992a39b5be4)
* [Tried solving “How many kcal can you consume at Saizeriya with 1,000 yen?” with an SMT solver (Z3).](https://qiita.com/tanakh/items/a1fb13f78e0576415de3)
* [Tried solving “How many kcal can you consume at Saizeriya with 1,000 yen?” with an Integer Programming solver (PuLP).](https://qiita.com/YSRKEN/items/dfc8604eb8598e5e9076)
* [Tried solving “How many kcal can you consume at Saizeriya with 1,000 yen?” with Markov Chain Monte Carlo.](https://qiita.com/kaityo256/items/b9a146d806dcb4f6edf9)
* [Tried solving “How many kcal can you consume at Saizeriya with 1,000 yen?” with Excel Solver.](https://qiita.com/miki_iwa/items/5e0434da8b1150c21741)
* [[Excel] How to compute the maximum calories you can consume at Saizeriya with 1,000 yen using only VLOOKUP.](https://www.waenavi.com/entry/20190519/1558269886)
* [Tried solving “How many kcal can you consume at Saizeriya with 1,000 yen?” with DP in Nadeshiko.](https://qiita.com/n_chiba_/items/694035ad880126230309)
* [Tried solving “How many kcal can you consume at Saizeriya with 1,000 yen?” on a homemade CPU.](https://anqou.net/poc/2019/05/27/post-2920/)
* [Tried solving “How many kcal can you consume at Saizeriya with 1,000 yen?” with exhaustive search.](https://primenumber.hatenablog.jp/entry/2019/05/27/235459)
* [Tried solving “How many kcal can you consume at Saizeriya with 1,000 yen?” with SQL.](https://qiita.com/ahinore@github/items/c9a71ac56055a94d9e54)
* [Compute “How many kcal can you consume at Saizeriya with 1,000 yen?” in TeX — Dynamic Programming (DP) in TeX.](https://doratex.hatenablog.jp/entry/20190520/1558323856)
* [Tried solving “How many kcal can you consume at Saizeriya with 1,000 yen?” with Pandoc.](https://zrbabbler.hatenablog.com/entry/2019/05/25/162221)

Here, I introduce a solution using constraint logic programming with [CLP(FD)](https://www.swi-prolog.org/pldoc/man?section=clpfd) in [SWI-Prolog](https://www.swi-prolog.org/).

# Code

The menu dataset is taken from the [#1 branch of Saizeriya_100yen](https://github.com/marushosummers/Saizeriya_1000yen/tree/%231), introduced in [“I tried making a Saizeriya 1,000 yen gacha (Heroku + Flask + LINEbot)”](https://qiita.com/marusho_summers/items/a2d3681fac863734ec8a).

The overall approach is likely the same as [“I tried solving ‘How many kcal can you consume at Saizeriya with 1,000 yen?’ with an SMT solver (Z3).”](https://qiita.com/tanakh/items/a1fb13f78e0576415de3): describe constraints on total price and total calories, and assign a 0/1 value for each menu item.

```prolog
#!/usr/bin/env swipl

:- use_module(library(clpfd)).
:- initialization(main, main).

main :- time(saizeriya).

saizeriya :-
    % Restrict to only the necessary columns
    findall(menu(Name, Price, Calorie), menu(_, Name, _, _, Price, Calorie, _), Menus),

    % For each menu item, represent “do not order / order one” as 0/1
    same_length(Menus, Vars),
    Vars ins 0..1,

    % Total price must not exceed 1,000 yen
    maplist(arg(2), Menus, AllPrices),
    scalar_product(AllPrices, Vars, #=<, 1000),

    % Total calories
    maplist(arg(3), Menus, AllCalories),
    scalar_product(AllCalories, Vars, #=, Cal),

    % Firm intent to maximize calories
    labeling([max(Cal)], Vars),

    print_result(Menus, Vars).

print_result([], []).
print_result([_|Menus], [0|Vars]) :-
    print_result(Menus, Vars).
print_result([menu(Name, Price, Calorie)|Menus], [1|Vars]) :-
    format('~w~n~t~w yen~20|~t~wkcal~30|~n', [Name, Price, Calorie]),
    print_result(Menus, Vars).

% https://github.com/marushosummers/Saizeriya_1000yen/tree/%231
% menu(id integer, name text, category text, type text, price integer, calorie integer, salt real)
menu(1, 'Colorful Garden Salad', sidedish, salad, 299, 130, 1.1).
menu(2, 'Small Shrimp Salad', sidedish, salad, 349, 115, 1.3).
menu(3, 'Tender Chicken Salad', sidedish, salad, 299, 134, 1.2).
menu(4, 'Wakame Seaweed Salad', sidedish, salad, 299, 92, 2.1).
menu(5, 'Italian Salad', sidedish, salad, 299, 196, 0.7).
menu(6, 'Seafood Salad', sidedish, salad, 599, 229, 2.4).
menu(7, 'Soft-Boiled Egg & Pork Salad', sidedish, salad, 599, 433, 2.3).
menu(8, 'Corn Cream Soup', sidedish, soup, 149, 142, 1.1).
menu(9, 'Chilled Pumpkin Soup (Seasonal)', sidedish, soup, 149, 105, 0.9).
menu(10, 'Hearty Vegetable Minestrone (Seasonal)', sidedish, soup, 299, 222, 2.1).
menu(11, 'Freshly Shaved Pecorino Cheese', sidedish, appetizer, 100, 59, 0.6).
menu(12, 'Mini Ficelle', sidedish, appetizer, 169, 188, 1.0).
menu(13, 'Garlic Toast', sidedish, appetizer, 189, 252, 1.1).
menu(14, 'Drink Bar (Set)', drink, drinkbar, 190, 0, 0.0).
menu(15, 'Spicy Chicken', sidedish, appetizer, 299, 374, 2.2).
menu(16, 'Oven-Baked Asparagus (Seasonal)', sidedish, appetizer, 299, 221, 1.1).
menu(17, 'Popcorn Shrimp', sidedish, appetizer, 299, 215, 1.4).
menu(18, 'Oven-Baked Escargot', sidedish, appetizer, 399, 256, 1.6).
menu(19, 'Garlic-Grilled Mussels', sidedish, appetizer, 399, 164, 1.3).
menu(20, 'Grilled Sausage with Vegetable Sauce', sidedish, appetizer, 399, 570, 3.1).
menu(21, 'Chorizo', sidedish, appetizer, 399, 393, 2.0).
menu(22, 'Warm Tender Green Pea Salad', sidedish, appetizer, 199, 213, 1.1).
menu(23, 'Sautéed Spinach', sidedish, appetizer, 199, 138, 1.2).
menu(24, 'Sautéed Cabbage & Anchovy', sidedish, appetizer, 199, 80, 1.5).
menu(25, 'Grilled Potatoes', sidedish, appetizer, 199, 366, 2.0).
menu(26, 'Pickled Celery (Seasonal)', sidedish, appetizer, 199, 52, 1.3).
menu(27, 'Squid with Paprika Sauce', sidedish, appetizer, 199, 138, 1.1).
menu(28, 'Focaccia', sidedish, appetizer, 119, 214, 0.8).
menu(29, 'Petit Focaccia', sidedish, appetizer, 139, 214, 0.8).
menu(30, 'Set Petit Focaccia', sidedish, appetizer, 79, 107, 0.4).
menu(31, 'Glass of Wine', drink, alcohol, 100, 0, 0.0).
menu(32, 'Decanter 250ml', drink, alcohol, 200, 0, 0.0).
menu(33, 'Decanter 500ml', drink, alcohol, 399, 0, 0.0).
menu(34, 'Kirin Ichiban (Mug)', drink, alcohol, 399, 0, 0.0).
menu(35, 'Kirin Ichiban (Glass)', drink, alcohol, 299, 0, 0.0).
menu(36, 'Strong Zero Double Lemon', drink, alcohol, 379, 0, 0.0).
menu(37, 'Magnum 1500ml', drink, alcohol, 1080, 0, 0.0).
menu(38, 'Grappa', drink, alcohol, 379, 0, 0.0).
menu(39, 'Lambrusco Secco (Red), Dry', drink, alcohol, 1080, 0, 0.0).
menu(40, 'Lambrusco (Rosé), Sweet', drink, alcohol, 1080, 0, 0.0).
menu(41, 'Verdicchio (Dry, Medium-bodied)', drink, alcohol, 1080, 0, 0.0).
menu(42, 'Chianti (Dry, Slightly Full-bodied)', drink, alcohol, 1080, 0, 0.0).
menu(43, 'Chianti Rufina Riserva (Dry, Full-bodied)', drink, alcohol, 2160, 0, 0.0).
menu(44, 'Saizeriya Premium (Dry, Full-bodied)', drink, alcohol, 2160, 0, 0.0).
menu(45, 'Suntory All-Free', drink, alcohol, 259, 0, 0.0).
menu(46, 'Fresh Cheese & Tomato Salad', sidedish, appetizer, 299, 203, 0.4).
menu(47, 'Fresh Cheese & Tomato Salad (W size)', sidedish, appetizer, 598, 406, 0.8).
menu(48, 'Prosciutto', sidedish, appetizer, 399, 162, 1.8).
menu(49, 'Prosciutto (W size)', sidedish, appetizer, 798, 324, 3.6).
menu(50, 'Aged Milano Salami', sidedish, appetizer, 299, 95, 1.1).
menu(51, 'Aged Milano Salami (W size)', sidedish, appetizer, 598, 190, 2.2).
menu(52, 'Margherita Pizza', meal, pizza, 399, 568, 2.5).
menu(53, 'Pancetta Pizza', meal, pizza, 399, 646, 2.9).
menu(54, 'Vegetable & Mushroom Pizza', meal, pizza, 399, 610, 2.7).
menu(55, 'Tender Squid & Anchovy Pizza', meal, pizza, 499, 593, 4.6).
menu(56, 'Buffalo Mozzarella Pizza', meal, pizza, 499, 575, 2.3).
menu(57, 'Milano Salami Pizza', meal, pizza, 499, 606, 3.5).
menu(58, 'Spinach Gratin (Seasonal)', meal, gratin, 399, 521, 1.9).
menu(59, 'Seafood Gratin', meal, gratin, 499, 537, 2.2).
menu(60, 'Arrabbiata', meal, pasta, 399, 591, 4.2).
menu(61, 'Meat Sauce, Bologna Style', meal, pasta, 399, 582, 4.3).
menu(62, 'Meat Sauce, Bologna Style with Soft-Boiled Egg', meal, pasta, 468, 672, 4.5).
menu(63, 'Aglio e Olio', meal, pasta, 299, 560, 3.2).
menu(64, 'Cabbage Peperoncino', meal, pasta, 399, 686, 3.5).
menu(65, 'Cod Roe Sauce, Sicilian Style', meal, pasta, 399, 605, 3.7).
menu(66, 'Soup-Style Tomato Vongole (Seasonal)', meal, pasta, 499, 686, 4.8).
menu(67, 'Parma-Style Spaghetti', meal, pasta, 399, 700, 4.2).
menu(68, 'Squid Ink Spaghetti', meal, pasta, 499, 610, 3.8).
menu(69, 'Carbonara', meal, pasta, 499, 865, 4.1).
menu(70, 'Asparagus & Shrimp Cream Spaghetti (Seasonal)', meal, pasta, 499, 711, 3.5).
menu(71, 'Arrabbiata (W size)', meal, pasta, 770, 1182, 8.4).
menu(72, 'Meat Sauce, Bologna Style (W size)', meal, pasta, 770, 1164, 8.6).
menu(73, 'Aglio e Olio (W size)', meal, pasta, 574, 1120, 6.4).
menu(74, 'Cabbage Peperoncino (W size)', meal, pasta, 770, 1372, 7.0).
menu(75, 'Cod Roe Sauce, Sicilian Style (W size)', meal, pasta, 770, 1210, 7.4).
menu(76, 'Soup-Style Tomato Vongole (Seasonal) (W size)', meal, pasta, 976, 1372, 9.6).
menu(77, 'Parma-Style Spaghetti (W size)', meal, pasta, 770, 1400, 8.4).
menu(78, 'Squid Ink Spaghetti (W size)', meal, pasta, 976, 1220, 7.6).
menu(79, 'Carbonara (W size)', meal, pasta, 976, 1730, 8.2).
menu(80, 'Asparagus & Shrimp Cream Spaghetti (Seasonal) (W size)', meal, pasta, 976, 1422, 7.0).
menu(81, 'Topping: Soft-Boiled Egg', sidedish, appetizer, 69, 90, 0.2).
menu(82, 'Milano-Style Doria', meal, doria, 299, 542, 2.7).
menu(83, 'Milano-Style Doria with Soft-Boiled Egg', meal, doria, 368, 632, 2.9).
menu(84, 'Milano-Style Doria with Set Petit Focaccia', meal, doria, 378, 649, 3.1).
menu(85, 'Milano-Style Doria with Colorful Vegetables', meal, doria, 399, 590, 3.1).
menu(86, 'Shrimp & Squid Doria', meal, doria, 499, 624, 2.9).
menu(87, 'Seafood Paella', meal, rice, 599, 602, 3.6).
menu(88, 'Shrimp & Vegetable Tomato Cream Risotto', meal, rice, 399, 302, 2.2).
menu(89, 'Hayashi & Turmeric Rice', meal, rice, 499, 638, 3.3).
menu(90, 'Hayashi & Turmeric Rice with Soft-Boiled Egg', meal, rice, 568, 728, 3.5).
menu(91, 'Mixed Grill', meal, hamburg, 599, 823, 3.8).
menu(92, 'Hamburger Steak', meal, hamburg, 399, 514, 2.3).
menu(93, 'Hamburger with Demi-Glace Sauce', meal, hamburg, 499, 628, 3.6).
menu(94, 'Hamburger with Vegetable Sauce (Diavola-Style)', meal, hamburg, 499, 585, 2.6).
menu(95, 'Italian Hamburger Steak', meal, hamburg, 499, 633, 2.5).
menu(96, 'Grilled Meat & Hamburger Assortment', meal, hamburg, 599, 709, 3.4).
menu(97, 'Grilled Young Chicken (Diavola-Style)', meal, chicken, 499, 541, 2.1).
menu(98, 'Tender Chicken Cheese Bake', meal, chicken, 499, 588, 2.0).
menu(99, 'Pancetta & Grilled Young Chicken', meal, chicken, 599, 663, 2.5).
menu(100, 'Rib Steak', meal, steak, 999, 621, 1.5).
menu(101, 'Rice', sidedish, rice, 169, 303, 0.0).
menu(102, 'Large Rice', sidedish, rice, 219, 454, 0.0).
menu(103, 'Small Rice', sidedish, rice, 119, 151, 0.0).
menu(104, 'Cappuccino (Ice Cake) (Seasonal)', sidedish, dessert, 199, 114, 0.1).
menu(105, 'Tiramisu (Ice Cake)', sidedish, dessert, 199, 131, 0.1).
menu(106, 'Cinnamon Focaccia', sidedish, dessert, 169, 246, 0.8).
menu(107, 'Pudding & Cappuccino Assortment', sidedish, dessert, 399, 330, 0.2).
menu(108, 'Pudding & Tiramisu Assortment', sidedish, dessert, 399, 347, 0.2).
menu(109, 'Cinnamon Focaccia with Milk Ice Cream', sidedish, dessert, 319, 346, 0.9).
menu(110, 'Milk Gelato', sidedish, dessert, 199, 100, 0.1).
menu(111, 'Sicilian Lemon Sorbet', sidedish, dessert, 199, 127, 0.0).
menu(112, 'Italian Pudding', sidedish, dessert, 249, 216, 0.1).
menu(113, 'Chocolate Cake', sidedish, dessert, 299, 166, 0.1).
menu(114, 'Coffee Jelly', sidedish, dessert, 299, 162, 0.1).
menu(115, 'Tartufo Ice Cream', sidedish, dessert, 369, 164, 0.1).
```

# Execution Result

Save the code above as `saizeriya_1000yen.pl`, and give it execute permission with something like `$ chmod +x saizeriya_1000yen.pl`. Run it with `$ ./saizeriya_1000yen.pl`. After a while, it outputs the highest-calorie combination whose total price is at most 1,000 yen.

```console
$ ./saizeriya_1000yen.pl 
Grilled Potatoes
             199 yen   366kcal
Aglio e Olio (W size)
             574 yen  1120kcal
Large Rice
             219 yen   454kcal
% 2,112,422,450 inferences, 101.088 CPU in 101.671 seconds (99% CPU, 20896821 Lips)
```

# Conclusion

I introduced a solution to the Saizeriya Problem using SWI-Prolog’s CLP(FD).

Recently, I had a chance to use CLP(FD) at work. It was quite convenient and fun, and I wondered whether I could use it for other things. Then I remembered the Saizeriya Problem, so I gave it a try.
