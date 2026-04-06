# Pythonマッピング表

## 1. 文書目的

本書は、初版 Python 解析において、Python 構文をどの中間表現へ落とし込むかを一覧化するものである。

本書は以下の資料に整合することを前提とする。

- `要件定義書.md`
- `基本設計/設計優先度_対応内容_出力資料一覧.md`
- `基本設計/レイヤー間インタフェース設計書.md`

特に、以下の設計方針を前提とする。

- 初版解析対象言語は Python のみ。
- AST は Parser Layer 内に閉じ込める。
- Collector 以降は Parser が出力した構文事実のみを扱う。
- 中間モデルは `ProjectModel` を正本とする。
- Generator は中間モデルのみを参照する。
- 型不明は `Unknown` として保持する。
- シンボル解決と型推論は別工程として扱う。

---

## 2. マッピングの見方

本書では、各 Python 構文について以下を定義する。

| 列 | 意味 |
|---|---|
| Python 構文 | 入力ソース上の構文要素 |
| 主な AST ノード | Python `ast` 上の代表ノード |
| Parser 出力 | `ParsedSyntaxSet` 内の主な record |
| Collector / Resolver 観点 | `Symbol` / `Scope` / `SymbolReference` / `ResolutionCandidate` 化の要点 |
| ProjectModel 正規化先 | 正本として保持する Core / Payload / Behavior / Extension |
| 派生 Relation / Fact | 必要に応じて派生生成・推論蓄積される要素 |
| 備考 | 初版制約や補足 |

---

## 3. 全体方針

## 3.1 宣言系の基本方針

- 宣言は `Symbol` を共通ヘッダとして起こし、詳細本体は `Decl / Payload` に分離する。
- `decl_name` はソースコード上の宣言名、`canonical_name` は解決用の完全修飾名とする。
- 型ヒントがない場合も宣言自体は作成し、型は `TypeRef(UNKNOWN)` とする。

## 3.2 参照系の基本方針

- 参照は `SymbolReference` として起こす。
- 解決結果は `ResolutionCandidate` に 0 件以上保持する。
- 参照未解決でも中間モデル生成は継続する。

## 3.3 振る舞い系の基本方針

- 呼び出し、生成、代入、return、分岐、ループは `BehaviorNode` / `BehaviorEdge` に正規化する。
- `BehaviorNode` の親子関係は構文の内包関係を表す。
- `BehaviorEdge` は実行遷移を表す。

## 3.4 派生情報の基本方針

- `Relation` は一次情報ではなく派生ビューとして扱う。
- 型推論結果は宣言型を書き換えず、`TypeFact` として蓄積する。
- シーケンス図用の `CallTraceStep` は推論・追跡後の派生結果として扱う。

---

## 4. Python 宣言系マッピング表

| Python 構文 | 主な AST ノード | Parser 出力 | Collector / Resolver 観点 | ProjectModel 正規化先 | 派生 Relation / Fact | 備考 |
|---|---|---|---|---|---|---|
| モジュール | `ast.Module` | `ParsedSourceUnit` | `SourceUnit`、module scope 作成 | `SourceUnit`、`Scope(kind=module)` | `Relation(DECLARES)` の派生元 | ファイル単位で 1 件 |
| `class C:` | `ast.ClassDef` | `DeclarationRecord(kind=class)` | `Symbol(kind=type)`、class scope 作成 | `TypeDecl(type_kind=CLASS)` | `DECLARES`、`EXTENDS` | `canonical_name` は module + class |
| `def f(...):` | `ast.FunctionDef` | `DeclarationRecord(kind=callable)` | `Symbol(kind=callable)`、callable scope 作成 | `CallableDecl` | `DECLARES` | トップレベル関数対象 |
| `def m(self, ...):` | `ast.FunctionDef` | `DeclarationRecord(kind=method)` | owner は class symbol | `CallableDecl` | `DECLARES` | クラス配下メソッド |
| `__init__` | `ast.FunctionDef` | `DeclarationRecord(kind=method)` | 通常の callable として収集 | `CallableDecl` | `DECLARES` | constructor 専用 payload は初版不要 |
| 引数 `x` | `ast.arg` | `DeclarationRecord(kind=parameter)` | `Symbol(kind=parameter)` | `ParameterDecl` | `PARAM_TYPE` | 型ヒントなしは `UNKNOWN` |
| 戻り値型ヒント `-> T` | `ast.FunctionDef.returns` | `TypeRecord` | `TypeRef` 解決候補生成 | `CallableDecl.return_type_id` | `RETURNS` | 未記載は `UNKNOWN` |
| クラス変数 `x: T = ...` | `ast.AnnAssign` | `DeclarationRecord(kind=field_or_variable)` | class scope 配下の symbol | `FieldDecl` | `HAS_A` | owner が class のため field |
| トップレベル変数 `x = ...` | `ast.Assign` | `DeclarationRecord(kind=variable)` | module scope 配下の symbol | `VariableDecl` | `USES` | 型不明可 |
| ローカル変数 `x = ...` | `ast.Assign` | `DeclarationRecord(kind=variable)` | callable / block scope 配下 | `VariableDecl` | `TypeFact` の対象 | ローカルも宣言化 |
| 属性代入 `self.x = ...` | `ast.Assign` + `ast.Attribute` | `DeclarationRecord(kind=field_candidate)` | `self` receiver を bind hint 化 | `FieldDecl` | `HAS_A`、`TypeFact` | `__init__` 外でも基本的に field 候補 |
| 属性代入 `cls.x = ...` | `ast.Assign` + `ast.Attribute` | `DeclarationRecord(kind=field_candidate)` | classmethod 文脈を考慮 | `FieldDecl` | `HAS_A` | class variable 相当として扱える |
| `@property` | `ast.FunctionDef.decorator_list` | `DeclarationRecord(kind=property)` | property symbol 作成 | `PropertyDecl` | `DECLARES` | getter 起点 |
| `@x.setter` | `ast.FunctionDef.decorator_list` | `DeclarationRecord(kind=property_setter)` | 既存 property と紐付け | `PropertyDecl.has_setter=True` | なし | property 統合が必要 |
| `@classmethod` | `ast.FunctionDef.decorator_list` | `DeclarationRecord(kind=method)` + modifier | resolver で `cls` を class 参照として扱う | `CallableDecl` + modifier | `DECLARES` | `PythonCallableExtension` に保持 |
| `@staticmethod` | `ast.FunctionDef.decorator_list` | `DeclarationRecord(kind=method)` + modifier | implicit receiver 無し | `CallableDecl` + modifier | `DECLARES` | `PythonCallableExtension` に保持 |
| 継承 `class C(B):` | `ast.ClassDef.bases` | `ReferenceRecord(kind=base_type)` | base symbol 候補解決 | `TypeDecl.base_type_ids` | `EXTENDS` | 複数継承も配列で保持 |
| ネストクラス | `ast.ClassDef` | `DeclarationRecord(kind=class)` | owner_symbol_id を外側 class に設定 | `TypeDecl` | `DECLARES` | 初版でも構造上は保持可能 |

---

## 5. Python import / 参照系マッピング表

| Python 構文 | 主な AST ノード | Parser 出力 | Collector / Resolver 観点 | ProjectModel 正規化先 | 派生 Relation / Fact | 備考 |
|---|---|---|---|---|---|---|
| `import pkg.mod` | `ast.Import` | `ImportRecord(kind=import)` | alias 無し import として解決 | `SourceUnit.imports` 相当補助情報、`SymbolReference` | `IMPORTS` | 外部は unresolved 可 |
| `import pkg.mod as m` | `ast.Import` | `ImportRecord(kind=import_alias)` | alias → canonical_name 対応付け | `PythonReferenceExtension` | `IMPORTS` | alias 解決対象 |
| `from pkg import x` | `ast.ImportFrom` | `ImportRecord(kind=from_import)` | モジュール + 名称分離 | `SymbolReference` | `IMPORTS` | 相対 import も対象 |
| `from .sub import x` | `ast.ImportFrom` | `ImportRecord(kind=relative_import)` | `level` と module path を解決 | `SymbolReference` | `IMPORTS` | project 内解決優先 |
| 名前参照 `x` | `ast.Name` | `ReferenceRecord(kind=name)` | scope を辿って候補列挙 | `SymbolReference` | `USES` | load/store context は parser で識別 |
| 属性参照 `obj.x` | `ast.Attribute` | `ReferenceRecord(kind=attribute)` | receiver_ref_id を保持 | `SymbolReference` | `USES` | `is_attribute_access=True` |
| `self.x` | `ast.Attribute` | `ReferenceRecord(kind=self_attribute)` | owner class へ束縛候補生成 | `SymbolReference` | `USES` / `HAS_A` 生成元 | `self.xxx` 解決対象 |
| `cls.x` | `ast.Attribute` | `ReferenceRecord(kind=cls_attribute)` | classmethod 前提で解決 | `SymbolReference` | `USES` / `HAS_A` 生成元 | `cls.xxx` 解決対象 |
| 型ヒント `list[T]` | `ast.Subscript` | `TypeRecord(kind=generic)` | named type + arg types 解決 | `TypeRef(kind=GENERIC)` | `PARAM_TYPE` 等 | Python の raw 表記も保持 |
| 型ヒント `A | B` | `ast.BinOp` or `ast.Subscript` 系 | `TypeRecord(kind=union)` | union member 解決 | `TypeRef(kind=UNION)` | `PARAM_TYPE` 等 | Python バージョン差は parser 吸収 |
| 文字列型ヒント `'A'` | `ast.Constant(str)` | `TypeRecord(kind=forward_ref)` | 遅延解決候補 | `TypeRef(raw_text='A')` | なし | unresolved 可 |

---

## 6. Python 振る舞い系マッピング表

| Python 構文 | 主な AST ノード | Parser 出力 | Collector / Resolver 観点 | ProjectModel 正規化先 | 派生 Relation / Fact | 備考 |
|---|---|---|---|---|---|---|
| ブロック本体 | `ast.Module.body` / `FunctionDef.body` / `If.body` 等 | `BehaviorRecord(kind=block)` | owner callable と親子構造を保持 | `BehaviorNode(kind=BLOCK)` | なし | ルートや子ブロックに使用 |
| 代入 `x = y` | `ast.Assign` | `BehaviorRecord(kind=assign)` | lhs / rhs の参照抽出 | `BehaviorNode(kind=ASSIGN)` | `TypeConstraint`、`TypeFact` | 複数代入は初版では 1 文として保持 |
| 注釈付き代入 `x: T = y` | `ast.AnnAssign` | `BehaviorRecord(kind=assign)` + type info | 宣言と代入を分けて扱う | `BehaviorNode(kind=ASSIGN)` | `TypeConstraint` | 宣言系とも連動 |
| return | `ast.Return` | `BehaviorRecord(kind=return)` | return value ref を保持 | `BehaviorNode(kind=RETURN)` | `RETURNS`、`TypeFact` | `return None` は `VOID` / `UNKNOWN` 方針要整理 |
| if | `ast.If` | `BehaviorRecord(kind=if)` | guard 参照抽出 | `BehaviorNode(kind=IF)` | `TypeConstraint` | then/else は child block 化 |
| for | `ast.For` | `BehaviorRecord(kind=loop)` | target / iter の参照抽出 | `BehaviorNode(kind=LOOP)` | `TypeFact` | `guard_text` に走査条件補助可 |
| while | `ast.While` | `BehaviorRecord(kind=loop)` | condition 参照抽出 | `BehaviorNode(kind=LOOP)` | `TypeConstraint` | loop として統一 |
| 関数呼び出し `f(x)` | `ast.Call` | `BehaviorRecord(kind=call)` | callee ref、arg refs を抽出 | `BehaviorNode(kind=CALL)` | `CALLS`、`PARAM_TYPE`、`SymbolBindingFact` | callee が関数/メソッド解決対象 |
| メソッド呼び出し `obj.m(x)` | `ast.Call` + `ast.Attribute` | `BehaviorRecord(kind=call)` | receiver_ref_id を保持 | `BehaviorNode(kind=CALL)` | `CALLS`、`SymbolBindingFact` | receiver 型は推論対象 |
| クラス生成 `C(x)` | `ast.Call` | `BehaviorRecord(kind=create_candidate)` | callee が type symbol か解決 | `BehaviorNode(kind=CREATE)` | `CREATES`、`TypeFact` | CALL と CREATE の判定は resolver / adapter で確定 |
| `isinstance(x, A)` | `ast.Call` | `ReferenceRecord(kind=type_guard)` | guard として特別扱い | `BehaviorNode(kind=CALL)` または IF 補助 | `TypeConstraint` | 分岐内型絞り込みの根拠 |
| 連鎖呼び出し `a().b()` | `ast.Call` + `ast.Attribute` | `BehaviorRecord(kind=call)` 複数 | 中間戻り値の receiver 候補を保持 | `BehaviorNode(kind=CALL)` | `TypeFact`、`CALLS` | 初版は構造保持優先 |

---

## 7. Python デコレータ / 修飾情報マッピング表

| Python 構文 | 主な AST ノード | Parser 出力 | ProjectModel 正規化先 | 備考 |
|---|---|---|---|---|
| `@property` | `ast.Name('property')` | decorator record | `PropertyDecl.has_getter=True` | property 本体化 |
| `@x.setter` | `ast.Attribute(value=Name('x'), attr='setter')` | decorator record | `PropertyDecl.has_setter=True` | 既存 property と統合 |
| `@classmethod` | `ast.Name('classmethod')` | decorator record | `CallableDecl` modifier + `PythonCallableExtension` | `cls` 解決前提 |
| `@staticmethod` | `ast.Name('staticmethod')` | decorator record | `CallableDecl` modifier + `PythonCallableExtension` | receiver 無し |
| その他デコレータ | 任意 | decorator record | `PythonCallableExtension` | 初版では意味解釈しないが raw 保持可 |

---

## 8. Python 型マッピング表

| Python 記述 | 主な AST ノード | TypeRef 正規化先 | 備考 |
|---|---|---|---|
| `int`, `str`, `bool` | `ast.Name` | `TypeRef(BUILTIN)` | builtins として共有化 |
| `None` 戻り値 | `ast.Constant(None)` / 無 return | `TypeRef(VOID)` または `UNKNOWN` | 運用詳細は別紙で固定 |
| 型ヒントなし | なし | `TypeRef(UNKNOWN)` | 初版標準 |
| `Sample` | `ast.Name` | `TypeRef(NAMED)` | `canonical_name` 解決対象 |
| `pkg.Sample` | `ast.Attribute` | `TypeRef(NAMED)` | namespace 付き |
| `list[Sample]` | `ast.Subscript` | `TypeRef(GENERIC)` | `arg_type_ids` 使用 |
| `list[int]` 相当配列表現 | `ast.Subscript` | `TypeRef(ARRAY)` または `GENERIC` | 正式ルールは TypeRef 設計で固定 |
| `A | B` | `ast.BinOp(BitOr)` | `TypeRef(UNION)` | 複数候補保持 |
| `typing.Any` | `ast.Attribute` / `ast.Name` | `TypeRef(ANY)` | shared type |
| forward ref `'A'` | `ast.Constant(str)` | `TypeRef(NAMED or UNKNOWN)` + `raw_text` | 遅延解決可 |

---

## 9. Relation 派生ルール表

| 元情報 | 条件 | 派生 Relation | source | target | 備考 |
|---|---|---|---|---|---|
| class 宣言 | class が module 配下 | `DECLARES` | module symbol | class symbol | 派生ビュー |
| method 宣言 | method が class 配下 | `DECLARES` | class symbol | method symbol | 派生ビュー |
| base class | `ClassDef.bases` 解決成功 | `EXTENDS` | derived class | base class | 複数継承可 |
| field 保持 | `FieldDecl` が class 配下 | `HAS_A` | owner class | target type | declared / inferred 統合可 |
| import | import 解決成功 | `IMPORTS` | source module | target module/symbol | unresolved は relation 未生成可 |
| callable 内 call | callee 解決成功 | `CALLS` | caller callable | callee callable | シーケンス/依存両用 |
| class create | callee が type 解決 | `CREATES` | caller callable | created type | `BehaviorNode(CREATE)` と整合 |
| parameter type | declared_type_id あり | `PARAM_TYPE` | callable symbol | target type | 引数ごとに派生可 |
| return type | return_type_id あり | `RETURNS` | callable symbol | target type | declared / inferred 統合可 |
| ref use | 参照解決成功 | `USES` | owner symbol | target symbol | 必要に応じて materialize |

---

## 10. 型推論連動ルール表

| 構文パターン | 根拠 | 生成対象 | 例 |
|---|---|---|---|
| `x = Sample()` | CREATE の戻り値型 | `TypeFact(target=x, inferred=Sample)` | ローカル変数型 |
| `self.a = x` | 右辺参照の型継承 | `TypeFact(target=self.a, inferred=type(x))` | フィールド型補完 |
| `return x` | 戻り値へ伝播 | `TypeFact(target=callable_return, inferred=type(x))` | return 型補完 |
| `func(x)` | 引数束縛 | `SymbolBindingFact`、`TypeFact(parameter)` | 呼び出し先引数型候補 |
| `if isinstance(x, A):` | 型ガード | `TypeConstraint`、分岐内 `TypeFact` | 分岐内型絞り込み |
| `x = cond()  # A or B` | 複数候補 | 複数 `TypeFact` または `TypeRef(UNION)` | Union 許容 |
| loop 内代入 | 繰り返し経路 | `TypeFact` 蓄積 | 経路爆発は制御対象 |

---

## 11. Parser 出力 record の推奨対応表

## 11.1 `DeclarationRecord`

| kind | 主対象 | ProjectModel 変換先 |
|---|---|---|
| `class` | `ClassDef` | `Symbol(type)` + `TypeDecl` |
| `callable` | `FunctionDef` | `Symbol(callable)` + `CallableDecl` |
| `parameter` | `arg` | `Symbol(parameter)` + `ParameterDecl` |
| `field_candidate` | `self.x` / `cls.x` 代入 | `Symbol(field)` + `FieldDecl` |
| `variable` | module/local 変数 | `Symbol(variable)` + `VariableDecl` |
| `property` | `@property` | `Symbol(property)` + `PropertyDecl` |

## 11.2 `ReferenceRecord`

| kind | 主対象 | `SymbolReference.usage_kind` 例 |
|---|---|---|
| `name` | `x` | `READ` / `WRITE` / `TYPE_HINT` |
| `attribute` | `obj.x` | `ATTRIBUTE_READ` |
| `self_attribute` | `self.x` | `SELF_MEMBER` |
| `cls_attribute` | `cls.x` | `CLASS_MEMBER` |
| `call_callee` | `f(...)` の f | `CALL_TARGET` |
| `call_arg` | `f(x)` の x | `ARGUMENT` |
| `base_type` | 継承先 | `BASE_TYPE` |
| `type_guard` | `isinstance(x, A)` | `TYPE_GUARD` |

## 11.3 `BehaviorRecord`

| kind | 主対象 | `BehaviorNode.kind` |
|---|---|---|
| `block` | 文ブロック | `BLOCK` |
| `assign` | 代入 | `ASSIGN` |
| `return` | return | `RETURN` |
| `if` | if | `IF` |
| `loop` | for / while | `LOOP` |
| `call` | 関数 / メソッド呼び出し | `CALL` |
| `create_candidate` | クラス生成候補 | `CREATE` |

---

## 12. 初版対象外構文の扱い

| 構文 | 主な AST ノード | 初版扱い | 記録方法 |
|---|---|---|---|
| list 内包表記 | `ast.ListComp` | 詳細追跡しない | `UnsupportedConstructRecord` + `Diagnostic(INFO)` |
| dict 内包表記 | `ast.DictComp` | 詳細追跡しない | 同上 |
| set 内包表記 | `ast.SetComp` | 詳細追跡しない | 同上 |
| 三項演算子 | `ast.IfExp` | 専用ノード化しない | 同上 |
| `async def` | `ast.AsyncFunctionDef` | 詳細追跡しない | 同上 |
| `await` | `ast.Await` | 詳細追跡しない | 同上 |
| `yield` | `ast.Yield` / `ast.YieldFrom` | 詳細追跡しない | 同上 |
| context manager 詳細展開 | `ast.With` | 詳細展開しない | 同上 |

補足:
- 初版対象外でも、可能な範囲で外側の構造は保持する。
- たとえば `async def` 自体を callable として収集するかどうかは、実装時に要判断だが、少なくとも黙って破棄はしない。

---

## 13. 実装時の注意点

## 13.1 `Call` と `Create` の判定

- Parser ではどちらも `ast.Call` で入る。
- そのため Parser 段階では `call` または `create_candidate` として保持し、最終的な `CALL` / `CREATE` 判定は resolver / adapter で callee 解決結果を見て確定する。

## 13.2 `self.x = ...` の扱い

- 要件定義書に従い、`self.xxx = ...` は基本的に `FieldDecl` 候補として扱う。
- `__init__` 内か否かで主/補助を分けない。
- 右辺型は declared type がなくても `TypeFact` で補完可能とする。

## 13.3 `PropertyDecl` の統合

- `@property` getter と `@x.setter` は別 `FunctionDef` として現れる。
- ただし ProjectModel 上は 1 つの `PropertyDecl` として統合する前提で扱う。
- 元の getter / setter callable を個別に残すかは実装詳細で決めるが、少なくとも property と callable の関係が追える必要がある。

## 13.4 scope 解決

- Python はブロックスコープが言語仕様上限定的だが、本ツール要件では block scope を `Scope(kind=block)` として保持する前提がある。
- したがって if / loop 内の変数参照も block scope を持てる設計で実装する。

## 13.5 Generator 非依存化

- 本表はあくまで Python → 共通中間モデルの写像であり、PlantUML 個別事情をここへ持ち込まない。
- 図表示名、省略名、participant 名衝突解消などは Generator 側の責務とする。

---

## 14. 決定事項まとめ

- Python AST は Parser Layer 内で record 群へ落とし込む。
- 宣言は `Symbol + Decl/Payload`、参照は `SymbolReference`、解決結果は `ResolutionCandidate` に分離する。
- 振る舞いは `BehaviorNode / BehaviorEdge` へ正規化する。
- `class`、`def`、`import`、継承、属性代入、`if`、`for`、`while`、`return`、関数呼び出し、クラス生成、`@property`、`@classmethod`、`@staticmethod` は初版対象とする。
- list 内包表記、dict / set 内包表記、三項演算子、`async` / `await`、`yield` は初版対象外とし、記録だけ残す。
- `self.xxx` / `cls.xxx` は resolver で解決し、必要に応じて `FieldDecl`、`USES`、`TypeFact` へつなぐ。
- `Call` と `Create` は parser では未確定候補として扱い、callee 解決後に確定する。
- 型不明は `TypeRef(UNKNOWN)` とし、推論は `TypeFact` 蓄積で補完する。

---

## 15. 次に詰めるべき点

- `DeclarationRecord` / `ReferenceRecord` / `BehaviorRecord` の正式 dataclass 項目
- `ReferenceBindHint` の詳細構造
- `PropertyDecl` と getter / setter callable の正規な紐付け方式
- `VOID` と `UNKNOWN` の使い分け
- `ARRAY` と `GENERIC` の切り分けルール
- `async def` を callable として最低限収集するかどうか
- 複数代入、tuple unpack、`with`、`match` など未記載構文の扱い
