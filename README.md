# shrv32
[![SecHack365](https://img.shields.io/badge/SecHack365-2020-ffd700.svg)](https://sechack365.nict.go.jp/)

暗号アクセラレータなどを搭載したRISC-V，RV32Iと暗号拡張命令を実装．

# 使いかた．
コンパイルして書き込み．CYC1000で動作確認済み．
verilatorなどのエミュレータの場合は乱数生成器をコメントアウトするなどして，romの初期化データを./binに置くと実行できる．

# リソースの消費状況
LE: 24,047 / 24,624 ( 98 % )

# RV32Iの実装について
RV32Iに沿って実装してあるが，現在signedとunsignedの区別はしていない．後で実装する．
パイプライン化を考慮して5ステージにした．しかしまだパイプライン化は行なっていない．1命令が終了するのに5クロック必要である．

# 拡張領域について
暗号計算など256ビット演算のため拡張された領域がある．通常の領域とのデータの交換は後述する拡張命令によって行なう．
- 256ビットのレジスタであるreg256を6本搭載．X[0,5]という名前．なおx0と違いX0はゼロレジスタではない．アセンブリ言語中ではx[0,5]と表記しているためわかりづらい，注意．
- 256ビット用のALUであるexaluを搭載．通常の領域とは異なり各命令が1クロックで終わると限らない，拡張命令が利用するアクセラレータはこのALU内部に搭載される．
  - 例えばAES128の暗号化は10クロック程度，復号化は20クロック程度を要する．
  - 拡張領域が動作している間，通常領域は動作を止める．(複数クロック掛かるため)

# 拡張命令について
開発中のため変更の可能性があります．というか汚いのでそのうち整理します．

opcode=0001011 (RV32 custom-0, from Table24.1, risv-spec-20191213)
主にレジスターレジスタ演算で利用
| 命令          | 定数タイプ | funct3 | オペランド      | 解説                                            | 備考           |
|---------------|------------|--------|-----------------|-------------------------------------------------|----------------|
| aesencrypt128 | R-type     | 1      | xrd, xrs1, xrs2 | xrd = aesencrypt128(plaintext=xrs1,secret=xrs2) | 10クロック必要 |
| aesdecrypt128 | R-type     | 2      | xrd, xrs1, xrs2 | xrd = aesdecrypt128(cipher=xrs1,   secret=xrs2) | 20クロック必要 |
| xd2r          | R-type     | 3      | rd,  xrs1, rs2  | rd  = (xrs1>>rs2)&0xffffffff                    |                |
| r2x           | I-type     | 4      | xrd             | xrd = x[24,31]                                  | 削除予定       |
| exlb          | R-type     | 5      | xrd, xrs1, rs2  | xrd = xr1<<8 + [rs2]                            |                |
| x2r           | R-type     | 6      | rd,  xrs1, rs2  | rd  = (xrs1>>rs2)&0xffffffff                    |                |
| exxor         | R-type     | 7      | xrd, xrs1, xrs2 | rd  = xrs1 ^ xrs2                               |                |

opcode=0101011 (RV32 custom-1, from Table24.1, risv-spec-20191213)
主にレジスター定数演算で利用
| 命令   | 定数タイプ | funct3 | オペランド     | 解説                         | 備考 |
|--------|------------|--------|----------------|------------------------------|------|
| exaddi | I-type     | 0      | xrd, xrs1, imm | xrd = xrs1 + imm             |      |
| xd2ri  | I-type     | 3      | rd,  xrs1, imm | rd  = (xrs1>>imm)&0xffffffff |      |
| exlbi  | I-type     | 5      | xrd, xrs1, imm | xrd = xrs1<<8 + [imm]        |      |

# AES暗号化/復号化
暗号化はAES128の10ラウンドを10クロックを掛けて計算する．復号化は10クロックをかけて10ラウンド分のラウンド鍵を生成し，その後10クロックをかけて復号を行なう．AES-NIみたいに1ラウンドずつ命令わければよかったと後悔している．(鍵長256ビットに対応させやすいし1命令を1クロックで終わらせた方が都合がいいので．) なおレジスタと行列の対応関係はAES-NIにならっており，[7:0]が(1,1)成分，[15:8]が(2,1)成分...のようになる．

# 乱数生成器
複数リングオシレータ型の乱数生成器を搭載している．3段のNOTを使ったリングオシレータ(以下RO)を121個搭載，1個は100で分周しクロック生成に用い，残り120個はXORで出力を1本にまとめて，生成したクロックのタイミングでメモリにマップされたシフトレジスタに転送している．ROのクロックで32クロック必要なため，乱数生成器の使用後は適当な時間RNGBusyを立てている．時間がないため詳細な検証はしていないが分布を見た感じ完全に均一なわけではないようだ．鍵には使わないように．

# UART MMIO
開発中のため変更の可能性があります．
0x200 status register
0x201 Rx buffer
0x202 Tx buffer
0x203 random number generator's 32bits shift register

0x200 status register mapping
| 31         5 | 4       | 3      2 | 1      | 0       |
|--------------|---------|----------|--------|---------|
| --not used-- | RNGBusy | reserved | TxBusy | RxReady |