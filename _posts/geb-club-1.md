title: "《GEB》读书会（一）：形式系统的形式与意义"
date: 2015-06-06 20:52:50
category: Reading
tags: [GEB, Book Club]
---

![Mathematical Symbols](thumbnail.jpg)

# 形式系统的形式与语义

## 第一章


                                              <-- inside the system [4,5] |  outside the system [4,5] -->
                                                                          |
                                             J-Mode (Mechanical-Mode) [5] |     W-Mode (Intelligent-Mode) [5]
                                           * can be totally unobservant   |   * can never be unobservant
                                             (but also can be!)           |
                                           * work within the system       |   * thinking about the system
                                                                          |
    Post Production System (r.e.) [1]           Terminology [2]           |
    |                                                                     |
    +--- string for free: WJ                 ----> axiom                  |
	+--- rules:                              \   * must have decision     |
    |    1. xJ -> xJU                         |    procedure [6]         ---    * lengthening rules [6]
    |    2. Wx -> Wxx                         +--> rules of production    |
    |    3. xJJJy -> xUy                      |  * characterizes the     ---    * shortening rules [6]
    |    4. xUUy -> xy                       /     system implicitly [6]  |
    +--- Requirement of Formality:                                        |
    |    MUST NOT do anything outside the rules                           |
    +--- example:                                                        ---    * any pattern in theorems? [4]
    |    (1) WJ        free                  \                            |     * ...
    |    (2) WJJ       from (1) by Rule 2     |                           |
    |    (3) WJJJJ     from (2) by Rule 2     |                           |        Decision Procedures [6]
    |    (4) WJJJJU    from (3) by Rule 1     +--> derivation / proof     |     * discriminate perfectly between
    |    (5) WUJU      from (4) by Rule 3     |                           |       theorems and non-theorem
    |    (6) WUJUUJU   from (5) by Rule 2     |                           |     * always terminate
    |    (7) WUJJU     from (6) by Rule 4    / --> theorem                |     * characterizes the system
    `--- problem: Can you get WU?                  (not Theorem)          |       explicitly
                                                                          |
    ------------------------------------------------------------------------------------------------------------

                                                              U-Mode (Un-Mode) [5]

[1] 形式系统
[2] 定理、公理、规则
[3] 系统内外
[4] 跳出系统
[5] W方式、J方式、U方式
[6] 判定过程

## 第二章

### pq系统

                                               |
    pq-system [1]                              |
    |                                          |
    +--- axiom: x-qxp-                         |
    |           where x is a string of '-'s    |
    `--- rule of production:                  ---  lengthening rule only [2]
         1. xqypz -> x-qypz-                   |
                                               |       Decision procedures
                                               |     * addition [2]
                                               |     * top-down [2]
                                               |     * bottom-up [3]
                                               |

[1] pq系统
[2] 判定过程
[3] 自底向上之别于自顶向下

### 形式系统的意义


                                                                         Isomorphism
                                          +-------------------------+-------------------+--------------------------+
                                          | [4]                     | [5]               | [7]                      |

                          theorem        <->    true statement     <->       ? ?       <->    true statement      <->  ...
                             :                         :                      :                      :
                             :                         :                      :                      :
                     well-formed string  <->  grammatical sentence <->       ? ?       <->  grammatical sentence  <->  ...
                             :                         :                      :                      :
                             :                         :                      :                      :
                          string         <->       sentence        <->    sentence     <->       sentence         <->  ...
                             :                         :                      :                      :
                             :                         :                      :                      :
                        /    p           <=>         plus          <=>      horse      <=>         equals         <=>  ...
                       |     q           <=>        equals         <=>      happy      <=>       taken from       <=>  ...
     Interpretation  --+     -           <=>          1            <=>      apple      <=>           1            <=>  ...
          [4]          |    --           <=>          2                                <=>           2            <=>  ...
                        \  ---           <=>          3                                <=>           3            <=>  ...
                           ...
                                                 (meaningful)           (meaningless)           (meaningful) [5]

           Passive meaning: the meaning adds no new theorems (Requirement of Formality)
                            e.g. --p--p--p--q-------- is not a theorem in pq-system
    [6] ------------------------------------------------------------------------------------------------------------------------
                                                                   Active meaning: bring new rules for creating sentences

                ?          ?          ?          ?          ?          ?          ?          ?          ?          ?

[4] 同构产生意义
[5] 有意义的解释与无意义的解释
[6] 主动意义之别于被动意义
[7] 双重意义！

## 数论系统初涉

                                    the Universe [8]
                                          .
                                          .
                                our thinking abilities
                                          .
                             numbers in natural languages [10]
                     ?                    .
                     .          theory of ideal numbers [11]             Euclid's Theorem and proof [12]
       consistent    .                    .                           ------------------------------------
          and        .       thought process of reasoning [13]       * a proof is a sequence of small steps
        complete     |                    .                            (like derivation in formal systems)
     formal systems  |                    .						     * what is 'number'? [10, 11]
          [9]        |                    .                          * what is 'all'? [13]
                     +---         addition (pq-system)

[8] 形式系统与现实
[9] 数学与符号处理
[10] 算术的基本法则
[11] 理想的数
[12] 欧几里得的证明
[13] 绕过无穷

## 第四章

### 《对位藏头诗》的多层意义


                     (difference between M and I ?) [6]

           vibration         underivability                                <=>    Bach's death     <=>   goblet broken      <-+
      +--->   :                     :                                                  :                       :              |
      |     sounds    <=>    true statement      <=>  the devilish method  <=>  'BACH' as subject  <=>       sound            | backfire
      +--->   :                     :                          :                       :                       :              |
      |   phonograph  <=>  formal number theory  <=>       Tortoise        <=>        Bach         <=>  autograph on goblet --+
      |      [2]       ^           [4]            ^           [3]           ^          [5]          ^         [3]
      |                +--------------------------+------------+------------+-----------------------+
      |                                                        |
      +-------------------------------------------------- Isomorphism
                gives meaning [1]

[1] 隐含意义与显明意义
[2] 《对位藏头诗》的显明意义
[3] 《对位藏头诗》的隐含意义
[4] 《对位藏头诗》与哥德尔定理之间的映射
[5] 赋格的艺术
[6] 哥德尔的结果造成的问题
