# 中間モデルと Python AST 対応整理資料

## 1. この資料の目的

本資料は、UML自動生成ツールにおける**中間モデル**について、以下をできるだけわかりやすく整理したものです。

- 中間モデルとは何か
- Python を解析対象とした場合、どの情報がどのデータクラスに入るのか
- Python の AST 各ノードが中間モデルのどこに対応するのか
- 実装時にどう分解して考えればよいのか

本資料は、要件定義書で定めた中間モデル方針を、**実装イメージが湧く粒度**まで噛み砕いて説明することを目的とします。

---

## 2. まず全体像

中間モデルは、Pythonコードをそのまま保持するものではありません。
AST を直接後続処理に使い回すのではなく、**図生成や型推論に必要な意味情報へ正規化した表現**として保持します。

ざっくり言うと、Pythonコードの情報は次の6種類に分けて持ちます。

| 観点 | 何を表すか | 主な保持先 |
|---|---|---|
| 宣言 | 何が定義されたか | `Symbol` + `Decl/Payload` |
| 参照 | どこで何を使ったか | `SymbolReference` |
| 解決 | その参照が最終的に何を指すか | `ResolutionCandidate` |
| 型 | それが何型か | `TypeRef`, `TypeFact`, `TypeConstraint` |
| 振る舞い | どう実行が流れるか | `BehaviorGraph`, `BehaviorNode`, `BehaviorEdge` |
| 図用関係 | 継承・呼び出し・依存など | `Relation` |

つまり、以下の役割分担になります。

- **定義そのもの**は `Symbol + Payload`
- **使われた痕跡**は `SymbolReference`
- **それが何を指すか**は `ResolutionCandidate`
- **処理の流れ**は `Behavior IR`
- **図に引く線**は `Relation`
- **型の確定・推論**は `TypeRef / TypeFact / TypeConstraint`

---

## 3. なぜ AST をそのまま使わないのか

AST はあくまで**Python構文木**です。

AST の役割は「コードがどのような構文で書かれているか」を表すことであり、図生成や言語非依存な解析にそのまま使うには、次の問題があります。

| AST をそのまま使う場合の問題 | 内容 |
|---|---|
| 言語依存が強い | Python AST の都合がそのまま後続に漏れる |
| 図生成に不要な構文差分が多い | 同じ意味でも書き方の違いに引っ張られやすい |
| 型推論に必要な意味情報が散らばる | 宣言、参照、制御構造が別々に存在する |
| 将来の多言語対応に弱い | Java / C# / C などに広げにくい |

そのため、AST から一度 **意味単位の共通表現** に落としてから、
クラス図・依存図・シーケンス図の生成に使う構造にします。

---

## 4. 中間モデルのレイヤ構造

要件定義では、中間モデルを次の6層で構成します。

| 層 | 役割 |
|---|---|
| CoreModel | すべての言語で共通に使う最小要素 |
| Semantic Payload Layer | 宣言の詳細本体 |
| Behavior IR Layer | 実行の流れや分岐・ループ |
| Language Extension | Python 固有情報などの退避先 |
| Inference Layer | 型推論結果や型制約 |
| Analysis Support Layer | 起点候補や未対応構文などの補助情報 |

以降では、Python を例にこの6層がどう使われるかを具体的に説明します。

---

## 5. Python コード例

以降の説明では、次のコードを例にします。

```python
from app.service import Service as Svc


class User:
    def __init__(self, service: Svc, name):
        self.service = service
        self.name = name

    def run(self, retry: int = 3):
        result = self.service.execute(self.name)

        if isinstance(result, str):
            return result

        for _ in range(retry):
            result = self.service.execute(self.name)

        return result
```

---

## 6. このコードが中間モデルにどう分かれるか

## 6.1 ファイル単位

`user.py` のようなファイルは `SourceUnit` で表現します。

| Python要素 | 対応先 |
|---|---|
| `.py` ファイル | `SourceUnit` |
| ファイルパス | `SourceUnit.path` |
| モジュール情報 | `SourceUnit.package / module` |
| import 一覧 | `SourceUnit.import 情報` |
| トップレベル宣言 | `SourceUnit.top_level_declarations` |

---

## 6.2 スコープ

Python では同名の変数や条件分岐内だけ有効な型制約があるため、スコープ管理が必要です。

| Python要素 | `Scope.kind` |
|---|---|
| モジュール全体 | `module` |
| `class User:` の中 | `class` |
| `def run(...):` の中 | `callable` |
| `if` / `for` の中 | `block` |

例としてこのコードからは、少なくとも以下のスコープができます。

- ファイル全体のモジュールスコープ
- `User` クラススコープ
- `__init__` callable スコープ
- `run` callable スコープ
- `if isinstance(...)` 真側の block スコープ
- `for _ in range(retry)` の loop block スコープ

---

## 6.3 クラス定義

`class User:` は次の2層に分かれます。

| 役割 | 対応先 |
|---|---|
| 共通ヘッダ | `Symbol(kind=TYPE)` |
| クラス詳細 | `TypeDecl(type_kind=CLASS)` |

### `Symbol` に持たせるもの

- `symbol_id`
- `kind`
- `display_name = "User"`
- `canonical_name = "app.user.User"` のような完全修飾名
- `scope_id`
- `source_unit_id`
- `span_id`

### `TypeDecl` に持たせるもの

- `type_kind = CLASS`
- `base_type_ids`
- `member_symbol_ids`
- `nested_type_symbol_ids`

---

## 6.4 メソッド定義

`def __init__` や `def run` は次の2層です。

| 役割 | 対応先 |
|---|---|
| 共通ヘッダ | `Symbol(kind=CALLABLE)` |
| メソッド詳細 | `CallableDecl` |

### `CallableDecl` に持たせるもの

- `parameter_symbol_ids`
- `return_type_id`
- `type_param_ids`
- `behavior_graph_id`

`run` の場合、後で `BehaviorGraph` を持つことになります。

---

## 6.5 引数

`service: Svc`, `name`, `retry: int = 3` は `ParameterDecl` です。

| Python要素 | 対応先 |
|---|---|
| `service: Svc` | `Symbol(kind=PARAMETER)` + `ParameterDecl` |
| `name` | `Symbol(kind=PARAMETER)` + `ParameterDecl` |
| `retry: int = 3` | `Symbol(kind=PARAMETER)` + `ParameterDecl` |

### `ParameterDecl` に持たせるもの

- `declared_type_id`
- `default_text`
- `optional`
- `variadic`
- `keyword_name`

例:

- `service: Svc` → `declared_type_id = TypeRef(NAMED, "Svc")`
- `name` → `declared_type_id = TypeRef(UNKNOWN)`
- `retry: int = 3` → `declared_type_id = TypeRef(BUILTIN, "int")`

---

## 6.6 フィールド

`self.service = service` や `self.name = name` は、クラスの属性なので `FieldDecl` 対象です。

| Python要素 | 対応先 |
|---|---|
| `self.service = service` | `FieldDecl` + `BehaviorNode(ASSIGN)` |
| `self.name = name` | `FieldDecl` + `BehaviorNode(ASSIGN)` |

ここで重要なのは、1行の代入でも情報が1個ではないことです。

### `self.service = service` から得たい情報

| 観点 | 対応先 |
|---|---|
| `service` というフィールドの存在 | `FieldDecl` |
| この位置で代入が起きた | `BehaviorNode(ASSIGN)` |
| 右辺 `service` を参照した | `SymbolReference` |
| フィールド型は右辺由来かもしれない | `TypeFact` / `SymbolBindingFact` |

---

## 6.7 ローカル変数

`result = ...` の `result` は `VariableDecl` です。

| Python要素 | 対応先 |
|---|---|
| `result` | `Symbol(kind=VARIABLE)` + `VariableDecl` |
| `result = ...` | `BehaviorNode(ASSIGN)` |

`VariableDecl` は変数そのものの宣言情報、`BehaviorNode(ASSIGN)` はその変数への代入イベントです。

---

## 6.8 import

`from app.service import Service as Svc` は複数の側面を持ちます。

| 観点 | 対応先 |
|---|---|
| import 文の記録 | `SourceUnit.import 情報` |
| `Svc` という名前参照 | `SymbolReference` |
| `Svc` が何を指すか | `ResolutionCandidate` |
| モジュール依存としての線 | `Relation(IMPORTS)` |

つまり、import 文は単純に1個の構造ではありません。
「生の import 情報」「参照」「解決結果」「図用依存線」が別管理されます。

---

## 6.9 呼び出し

`self.service.execute(self.name)` はかなり多くの情報を持ちます。

| 観点 | 対応先 |
|---|---|
| 呼び出し行為そのもの | `BehaviorNode(CALL)` |
| `self.service` の参照 | `SymbolReference` |
| `execute` の参照 | `SymbolReference` |
| `self.name` の参照 | `SymbolReference` |
| `execute` がどのメソッドか | `ResolutionCandidate` |
| 呼び出し関係 | `Relation(CALLS)` |

### 実装上の見方

この1行は、後続処理では以下のように分解して扱うと考えやすいです。

1. `self.service` という receiver を取る
2. `execute` という属性参照を取る
3. 引数 `self.name` を取る
4. それらをまとめて `BehaviorNode(CALL)` とする
5. 解決結果から `User.run -> Service.execute` を派生させて `Relation(CALLS)` を作る

---

## 6.10 条件分岐

`if isinstance(result, str):` は `BehaviorNode(IF)` が中心です。

| Python要素 | 対応先 |
|---|---|
| `if ...:` | `BehaviorNode(IF)` |
| 条件式 | `guard_text` + `SymbolReference` |
| 真偽の遷移 | `BehaviorEdge` |
| 分岐内だけ有効な型制約 | `TypeConstraint`, `TypeFact` |

この分岐が重要なのは、**分岐内だけ `result` の型が `str` に絞り込まれる**ことです。

つまり、

- 通常時: `result = Unknown` あるいは `Union[...]`
- 真側ブロック内: `result = str`

という条件付き型事実を持てます。

---

## 6.11 ループ

`for _ in range(retry):` は `BehaviorNode(LOOP)` です。

| Python要素 | 対応先 |
|---|---|
| `for _ in range(retry):` | `BehaviorNode(LOOP)` |
| `range(retry)` | `SymbolReference` / 必要なら `BehaviorNode(CALL)` |
| ループ本体 | `child_node_ids` またはブロック構造 |
| ノード間の流れ | `BehaviorEdge` |

初版では、ループを実行回数まで完全展開するのではなく、**ループ構造として保持**する方針です。

---

## 6.12 return

`return result` は `BehaviorNode(RETURN)` です。

| Python要素 | 対応先 |
|---|---|
| `return result` | `BehaviorNode(RETURN)` |
| `result` 参照 | `SymbolReference` |
| 戻り値型候補 | `TypeFact` |
| 図用の戻り値関係 | `Relation(RETURNS)` |

---

## 7. 型はどう扱うか

## 7.1 宣言型

型ヒントなど、コード上に明示された型は `TypeRef` に正規化します。

| Python表現 | `TypeRef.kind` |
|---|---|
| `int` | `BUILTIN` |
| `str` | `BUILTIN` |
| `Svc` | `NAMED` |
| `list[str]` | `GENERIC` または `ARRAY` |
| `A | B` | `UNION` |
| 型ヒントなし | `UNKNOWN` |
| 戻り値なし | `VOID` |

---

## 7.2 推論型

宣言時点で型がなくても、後から `TypeFact` を積みます。

例:

```python
svc = Svc()
```

この場合、

- 宣言上は `svc: Unknown`
- 推論上は `svc = Svc`

という扱いにできます。

| 観点 | 対応先 |
|---|---|
| 宣言時の型不明 | `VariableDecl.declared_type_id = Unknown` |
| 生成式からの推論 | `TypeFact(target=svc, inferred_type=Svc)` |

---

## 7.3 型制約

分岐で型が絞られる場合は `TypeConstraint` を使います。

例:

```python
if isinstance(result, str):
```

この場合、

- `result` はこの分岐内では `str`

という制約を持たせ、その制約に基づく `TypeFact` を付与します。

---

## 8. Relation は正本ではなく派生ビュー

`Relation` は図に線を引くための便利な情報ですが、**正本ではありません**。

つまり、

- クラスそのものは `TypeDecl`
- メソッドそのものは `CallableDecl`
- 呼び出しの痕跡は `BehaviorNode(CALL)`
- 参照の痕跡は `SymbolReference`

として保持し、それらを集約して

- `CALLS`
- `HAS_A`
- `IMPORTS`
- `RETURNS`
- `PARAM_TYPE`

などの `Relation` を派生生成します。

### Python から派生される `Relation` 例

| Pythonコード | 派生 `Relation` |
|---|---|
| `class A(B):` | `EXTENDS` |
| `from x import Y` | `IMPORTS` |
| `self.svc = svc` | `HAS_A` |
| `self.svc.execute()` | `CALLS` |
| `svc = Svc()` | `CREATES` |
| `def f(x: A)` | `PARAM_TYPE` |
| `def f() -> B` | `RETURNS` |

---

## 9. Behavior IR の見え方

`Behavior IR` は、シーケンス図生成と型推論の心臓部です。

先ほどの `run` メソッドを、ざっくり意味単位で表すと次のようになります。

```text
BehaviorGraph(run)
  root -> BLOCK
            ├─ ASSIGN(result = self.service.execute(self.name))
            ├─ IF(isinstance(result, str))
            │    ├─ true -> RETURN(result)
            │    └─ false -> LOOP(for _ in range(retry))
            │                 └─ ASSIGN(result = self.service.execute(self.name))
            └─ RETURN(result)
```

このように、AST の木構造をそのまま使うのではなく、**実行の意味に寄せた形**で保持します。

初版で必須対応の `BehaviorNode.kind` は次の通りです。

| kind | 意味 |
|---|---|
| `BLOCK` | ブロック |
| `CALL` | 関数・メソッド呼び出し |
| `ASSIGN` | 代入 |
| `RETURN` | return |
| `IF` | 分岐 |
| `LOOP` | ループ |
| `CREATE` | クラス生成 |

---

## 10. Python AST ノードと中間モデルの対応表

ここからは、AST の各ノードが中間モデルのどこに落ちるかを整理します。

## 10.1 モジュール・import 系

| AST ノード | 主な意味 | 中間モデル対応 |
|---|---|---|
| `ast.Module` | ファイル全体 | `SourceUnit`, モジュール `Scope` |
| `ast.Import` | `import x` | `SourceUnit.import 情報`, `SymbolReference`, `Relation(IMPORTS)` |
| `ast.ImportFrom` | `from x import y` | `SourceUnit.import 情報`, `SymbolReference`, `ResolutionCandidate`, `Relation(IMPORTS)` |
| `ast.alias` | import 要素1件 | import 明細、alias 解決情報 |

### 補足

`import` 系は宣言ではなく、基本的には

- 外部名を持ち込む
- モジュール依存を作る
- ローカル名と実体名を結びつける

という役割です。

---

## 10.2 宣言系

| AST ノード | 主な意味 | 中間モデル対応 |
|---|---|---|
| `ast.ClassDef` | クラス定義 | `Symbol(TYPE)` + `TypeDecl`, クラス `Scope` |
| `ast.FunctionDef` | 関数 / メソッド定義 | `Symbol(CALLABLE)` + `CallableDecl`, callable `Scope` |
| `ast.AsyncFunctionDef` | 非同期関数定義 | 初版では未対応寄り。必要に応じ `UnsupportedConstructRecord` |
| `ast.arguments` | 引数群 | `ParameterDecl` 群 |
| `ast.arg` | 引数1件 | `Symbol(PARAMETER)` + `ParameterDecl` |
| `ast.Return` | return 文 | `BehaviorNode(RETURN)` |
| `ast.Global` | global 宣言 | `Scope` / 解決補助 / extension |
| `ast.Nonlocal` | nonlocal 宣言 | `Scope` / 解決補助 / extension |

### `ClassDef` の扱い

`ClassDef` からは最低限以下を取りたいです。

- クラス `Symbol`
- `TypeDecl`
- 継承候補の `TypeRef`
- メンバー一覧
- クラススコープ

### `FunctionDef` の扱い

`FunctionDef` からは最低限以下を取りたいです。

- メソッド `Symbol`
- `CallableDecl`
- 引数 `ParameterDecl` 群
- 戻り値型 `TypeRef`
- `BehaviorGraph`
- メソッドスコープ

---

## 10.3 文 (`stmt`) 系

| AST ノード | 主な意味 | 中間モデル対応 |
|---|---|---|
| `ast.Assign` | 代入 | `BehaviorNode(ASSIGN)` + 必要に応じ `FieldDecl` / `VariableDecl` |
| `ast.AnnAssign` | 型注釈付き代入 | `BehaviorNode(ASSIGN)` + `declared_type_id` 取得 |
| `ast.AugAssign` | `+=` など | `BehaviorNode(ASSIGN)` 相当として扱う |
| `ast.Expr` | 単独式 | 中身が `Call` なら `BehaviorNode(CALL)`、DocString なら extension or metadata |
| `ast.If` | if 文 | `BehaviorNode(IF)` + `BehaviorEdge` |
| `ast.For` | for 文 | `BehaviorNode(LOOP)` |
| `ast.While` | while 文 | `BehaviorNode(LOOP)` |
| `ast.Break` | break | 初版では loop 制御用の edge / 未対応記録 |
| `ast.Continue` | continue | 初版では loop 制御用の edge / 未対応記録 |
| `ast.Pass` | pass | 省略可能または no-op ノード |
| `ast.Try` | try 文 | 初版対象外寄り。必要なら `UnsupportedConstructRecord` |
| `ast.With` | with 文 | 初版では詳細展開対象外 |
| `ast.Raise` | 例外送出 | `BehaviorNode` 拡張候補 or unsupported |
| `ast.Assert` | assert | 条件制約として拡張候補 |
| `ast.Delete` | del | 変数束縛解除として拡張候補 |

### `Assign` の分解が重要

`Assign` は見た目以上に情報が多く、たとえば

```python
self.service = service
```

は次のように扱います。

- 左辺が `self.xxx` なので `FieldDecl` 候補
- 代入イベントとして `BehaviorNode(ASSIGN)`
- 右辺の `service` は `SymbolReference`
- 型伝播候補として `SymbolBindingFact`

### `Expr` の注意点

`Expr` は単独では意味が広いです。

例:

```python
func()
"""docstring"""
```

どちらも AST 上は `Expr` ですが、中身で分けて扱う必要があります。

- `Expr(Call(...))` → `BehaviorNode(CALL)`
- 先頭の `Expr(Constant(str))` → DocString

---

## 10.4 式 (`expr`) 系

| AST ノード | 主な意味 | 中間モデル対応 |
|---|---|---|
| `ast.Call` | 関数 / メソッド呼び出し | `BehaviorNode(CALL)` または `BehaviorNode(CREATE)` |
| `ast.Name` | 名前参照 | `SymbolReference` |
| `ast.Attribute` | 属性参照 | `SymbolReference` |
| `ast.Constant` | 定数 | リテラル情報、型推論根拠 |
| `ast.keyword` | キーワード引数 | `ParameterDecl.keyword_name` 対応、呼び出し引数情報 |
| `ast.List` | リスト生成 | リテラル / `TypeRef(ARRAY)` 推論根拠 |
| `ast.Tuple` | タプル生成 | リテラル / 複合値 |
| `ast.Dict` | 辞書生成 | リテラル / 推論根拠 |
| `ast.Set` | set 生成 | リテラル / 推論根拠 |
| `ast.BinOp` | 二項演算 | 式評価、型推論根拠 |
| `ast.UnaryOp` | 単項演算 | 式評価、型推論根拠 |
| `ast.BoolOp` | `and/or` | 条件式、分岐条件 |
| `ast.Compare` | 比較 | 条件式、分岐条件 |
| `ast.IfExp` | 三項演算子 | 初版では専用ノード対象外 |
| `ast.Lambda` | lambda | 初版では詳細追跡対象外寄り |
| `ast.Subscript` | 添字アクセス | `SymbolReference` / `TypeRef(GENERIC/ARRAY)` 補助 |
| `ast.Slice` | スライス | 添字詳細 |
| `ast.ListComp` | リスト内包表記 | 初版詳細追跡対象外 |
| `ast.DictComp` | 辞書内包表記 | 初版詳細追跡対象外 |
| `ast.SetComp` | set内包表記 | 初版詳細追跡対象外 |
| `ast.GeneratorExp` | generator 式 | 初版詳細追跡対象外 |
| `ast.Await` | await | 初版対象外 |
| `ast.Yield` | yield | 初版対象外 |
| `ast.YieldFrom` | yield from | 初版対象外 |
| `ast.NamedExpr` | walrus 演算子 | `ASSIGN` 相当の拡張候補 |

### `Call` は CREATE になる場合がある

`ast.Call` は常に `CALL` ではありません。

例:

```python
svc = Svc()
```

この場合、`Svc` がクラス解決されれば、これは

- `BehaviorNode(CREATE)`
- `Relation(CREATES)`

として扱うのが自然です。

一方で

```python
svc.execute()
```

は通常の `BehaviorNode(CALL)` です。

---

## 10.5 型注釈・デコレータ系

| AST ノード / 項目 | 主な意味 | 中間モデル対応 |
|---|---|---|
| `FunctionDef.returns` | 戻り値型注釈 | `CallableDecl.return_type_id` |
| `arg.annotation` | 引数型注釈 | `ParameterDecl.declared_type_id` |
| `AnnAssign.annotation` | 変数型注釈 | `FieldDecl/VariableDecl.declared_type_id` |
| `ClassDef.bases` | 継承元 | `TypeRef` + `Relation(EXTENDS)` |
| `decorator_list` | デコレータ | `Language Extension` + 一部は `PropertyDecl` / modifier |

### デコレータの扱い

Python では decorator が意味に影響します。

代表例:

| デコレータ | 基本方針 |
|---|---|
| `@property` | `PropertyDecl` 化 |
| `@classmethod` | `CallableDecl` + classmethod modifier |
| `@staticmethod` | `CallableDecl` + staticmethod modifier |
| その他 | `Language Extension` へ保持 |

---

## 10.6 DocString の扱い

Python の DocString は AST 上では、通常以下のように現れます。

- クラスや関数の先頭要素にある `Expr(Constant(str))`

したがって、DocString は専用ノードではありません。

| 対象 | AST 上の見え方 | 中間モデル上の扱い |
|---|---|---|
| モジュール DocString | `Module.body[0] = Expr(Constant(str))` | `SourceUnit` 補助情報 or extension |
| クラス DocString | `ClassDef.body[0] = Expr(Constant(str))` | `TypeDecl` 補助情報 or extension |
| 関数 DocString | `FunctionDef.body[0] = Expr(Constant(str))` | `CallableDecl` 補助情報 or extension |

初版で図生成の正本にする必要は薄いため、必要なら extension または補助メタデータで十分です。

---

## 11. AST ノード別の実装イメージ

ここでは、実際の変換処理でどう考えるかを、擬似フローで整理します。

## 11.1 `ClassDef`

```text
入力: ast.ClassDef

1. Type 用の Symbol を作る
2. TypeDecl を作る
3. bases から base_type_ids を作る
4. クラス Scope を作る
5. body 内を走査してメンバーを収集する
6. member_symbol_ids に紐付ける
7. 継承 Relation(EXTENDS) を派生生成できるよう情報を保持する
```

---

## 11.2 `FunctionDef`

```text
入力: ast.FunctionDef

1. Callable 用の Symbol を作る
2. CallableDecl を作る
3. callable Scope を作る
4. arguments から ParameterDecl 群を作る
5. returns から return_type_id を作る
6. body を走査して BehaviorGraph を作る
7. decorator_list を見て property / classmethod / staticmethod を反映する
```

---

## 11.3 `Assign`

```text
入力: ast.Assign

1. 左辺を解析する
   - Name なら VariableDecl 候補
   - Attribute(self.xxx) なら FieldDecl 候補
2. 右辺を解析して参照・呼び出し・生成を拾う
3. BehaviorNode(ASSIGN) を作る
4. 左辺と右辺の束縛関係を SymbolBindingFact に残す
5. 型が推測できるなら TypeFact 候補を作る
```

---

## 11.4 `Call`

```text
入力: ast.Call

1. 呼び出し対象を解析する
   - Name(...) か
   - Attribute(obj.method) か
2. receiver / callee / args の SymbolReference を作る
3. 解決結果を見て CREATE か CALL か判定する
4. BehaviorNode(CALL or CREATE) を作る
5. Relation(CALLS or CREATES) を後段で派生可能にする
6. 戻り値型推論に使う情報を残す
```

---

## 11.5 `If`

```text
入力: ast.If

1. guard 式の参照を集める
2. BehaviorNode(IF) を作る
3. then / else 用の block ノードを作る
4. BehaviorEdge を張る
5. isinstance などの型制約を TypeConstraint に落とす
6. 分岐内だけ有効な TypeFact を蓄積できるようにする
```

---

## 11.6 `For`

```text
入力: ast.For

1. iterator と target を解析する
2. BehaviorNode(LOOP) を作る
3. 本体の block を作る
4. body 内で起きる ASSIGN / CALL / RETURN を child に持たせる
5. 必要なら反復変数の型推論根拠を残す
```

---

## 12. AST ノードごとの優先度整理

初版で優先して対応すべきノードを整理すると次のようになります。

## 12.1 最優先

| ノード | 理由 |
|---|---|
| `Module` | 解析開始点 |
| `Import`, `ImportFrom`, `alias` | 依存関係・解決に必須 |
| `ClassDef` | クラス図の基礎 |
| `FunctionDef` | メソッド解析の基礎 |
| `arguments`, `arg` | 引数型・起点候補に必須 |
| `Assign`, `AnnAssign` | フィールド・変数・型伝播に必須 |
| `Return` | 戻り値推論に必須 |
| `If` | 分岐表現と型絞り込みに必須 |
| `For`, `While` | ループ表現に必須 |
| `Call` | シーケンス図の中心 |
| `Name`, `Attribute` | 参照解析の中心 |
| `Constant` | リテラル型推論に必須 |

## 12.2 初版で簡易対応でもよいもの

| ノード | 方針 |
|---|---|
| `AugAssign` | `ASSIGN` 相当で簡易吸収 |
| `Expr` | 中身を見て call / docstring 判定 |
| `BoolOp`, `Compare` | guard_text 保存中心 |
| `List`, `Tuple`, `Dict`, `Set` | リテラルとして保持 |
| `Subscript` | 型注釈 / 添字アクセス補助 |
| `keyword` | キーワード引数対応 |

## 12.3 初版では未対応寄り

| ノード | 方針 |
|---|---|
| `AsyncFunctionDef` | 未対応または制限付き |
| `Await` | 未対応 |
| `Yield`, `YieldFrom` | 未対応 |
| `ListComp`, `DictComp`, `SetComp` | 詳細分解しない |
| `GeneratorExp` | 詳細追跡しない |
| `IfExp` | 専用ノード化しない |
| `Try`, `With` | 詳細展開しない |
| `Lambda` | 詳細追跡しない |

---

## 13. 実装で特に大事な考え方

## 13.1 `self.x = y` を1個の情報だと思わない

これは最低でも次の4要素に分かれます。

- `x` というフィールドの存在
- 代入イベント
- `y` 参照
- 型や束縛の伝播

ここを1レコードにまとめてしまうと、後で保守が難しくなります。

---

## 13.2 `Relation` を正本にしない

`CALLS` や `HAS_A` を直接持ち始めると、宣言や振る舞いと二重管理になりやすいです。

正本はあくまで次の系統です。

- `Symbol`
- `Decl / Payload`
- `SymbolReference`
- `Behavior IR`
- `TypeFact`

`Relation` はそこから派生すべきです。

---

## 13.3 型ヒントなしでも構造を確定させる

Python 初版では、型不明でも中間モデルを作れることが重要です。

例:

```python
def f(x):
    self.a = x
    return x
```

この時点でも最低限以下は成立します。

- `CallableDecl(f)`
- `ParameterDecl(x, Unknown)`
- `FieldDecl(a, Unknown)`
- `BehaviorNode(ASSIGN)`
- `BehaviorNode(RETURN)`

型は後で `TypeFact` により補います。

---

## 13.4 AST から直接 UML を作らない

実装時にやりたくなりがちですが、

- AST からその場で PlantUML を組み立てる
- AST の node 種別ごとに Generator 側で分岐する

という構成は避けるべきです。

必ず

**AST → 中間モデル → 図生成**

の形にすることで、将来の多言語対応や出力記法追加に耐えやすくなります。

---

## 14. Python 要素 → 中間モデル 早見表

## 14.1 宣言系

| Python要素 | 中間モデル |
|---|---|
| `class` | `Symbol(TYPE)` + `TypeDecl` |
| `def` | `Symbol(CALLABLE)` + `CallableDecl` |
| 引数 | `Symbol(PARAMETER)` + `ParameterDecl` |
| `self.x = ...` | `Symbol(FIELD)` + `FieldDecl` |
| ローカル変数 | `Symbol(VARIABLE)` + `VariableDecl` |
| `@property` | `Symbol(PROPERTY)` + `PropertyDecl` |

## 14.2 参照系

| Python要素 | 中間モデル |
|---|---|
| 変数参照 | `SymbolReference` |
| 属性参照 | `SymbolReference` |
| 関数参照 | `SymbolReference` |
| import 名 | `SymbolReference` |
| 解決結果 | `ResolutionCandidate` |

## 14.3 振る舞い系

| Python要素 | 中間モデル |
|---|---|
| `a = b` | `BehaviorNode(ASSIGN)` |
| `f()` | `BehaviorNode(CALL)` |
| `Cls()` | `BehaviorNode(CREATE)` |
| `if` | `BehaviorNode(IF)` |
| `for`, `while` | `BehaviorNode(LOOP)` |
| `return` | `BehaviorNode(RETURN)` |

## 14.4 型系

| Python要素 | 中間モデル |
|---|---|
| 型ヒント `: int` | `TypeRef(BUILTIN)` |
| 型ヒント `: User` | `TypeRef(NAMED)` |
| 型ヒントなし | `TypeRef(UNKNOWN)` |
| 実行追跡でわかった型 | `TypeFact` |
| 分岐内だけ成り立つ型 | `TypeConstraint` + `TypeFact` |

## 14.5 図系

| Python要素 | 派生 `Relation` |
|---|---|
| 継承 | `EXTENDS` |
| import | `IMPORTS` |
| フィールド保持 | `HAS_A` |
| メソッド呼び出し | `CALLS` |
| インスタンス生成 | `CREATES` |
| 引数型 | `PARAM_TYPE` |
| 戻り値型 | `RETURNS` |

---

## 15. まとめ

Python 解析時の中間モデルは、次の考え方で整理すると理解しやすいです。

### 宣言
「何が存在するか」

- クラス
- メソッド
- 引数
- フィールド
- 変数

→ `Symbol + Decl`

### 参照
「どこで何を使ったか」

- 名前参照
- 属性参照
- import 名
- メソッド参照

→ `SymbolReference`

### 解決
「その参照が何者か」

- `Svc` はどのクラスか
- `execute` はどのメソッドか

→ `ResolutionCandidate`

### 振る舞い
「どう実行されるか」

- 代入
- 呼び出し
- 生成
- 分岐
- ループ
- return

→ `BehaviorGraph / BehaviorNode / BehaviorEdge`

### 型
「その値は何型か」

- 宣言型
- 推論型
- 条件付き型

→ `TypeRef / TypeFact / TypeConstraint`

### 図
「最終的に UML にどう線を引くか」

- 継承
- 呼び出し
- 依存
- 生成
- 保持

→ `Relation`

要するに、

**AST は入力**、
**中間モデルは意味の整理結果**、
**UML はその出力**

です。

したがって、設計上は必ず

**Python AST → 中間モデル → PlantUML**

という流れを崩さないことが重要です。

---

## 16. 次にやるとよいこと

次の設計工程としては、以下を進めると実装に入りやすくなります。

1. `Symbol` / `Decl` / `BehaviorNode` の dataclass 定義を確定する
2. AST 走査処理を「宣言収集」「参照収集」「Behavior IR 生成」に分ける
3. `Call` を `CALL` と `CREATE` に分類する判定ルールを決める
4. `self.xxx` / `cls.xxx` 解決のルールを具体化する
5. `TypeFact` と `TypeConstraint` の生成条件を一覧化する
6. PlantUML Generator が参照する中間モデル項目を明文化する

