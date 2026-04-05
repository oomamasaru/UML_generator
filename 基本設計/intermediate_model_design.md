# 中間モデル設計書

## 1. 文書概要

### 1.1 目的
本書は、UML自動生成ツールにおける中間モデルの設計内容を定義することを目的とする。  
中間モデルは、言語依存の構文解析結果を図生成可能な共通表現へ正規化し、クラス図、モジュール依存図、シーケンス図の生成、および型推論・参照解決の基盤として利用する。

### 1.2 適用範囲
本設計書は初版実装を対象とし、以下を前提とする。

- 実装言語は Python とする。
- 初版解析対象言語は Python のみとする。
- 初版出力記法は PlantUML のみとする。
- 初版対象図はクラス図、モジュール依存図、シーケンス図とする。
- 将来の多言語対応を見据え、初版時点で中間モデルの骨格を確定する。

### 1.3 設計方針
- 中間モデルは言語非依存の単一モデルとする。
- AST をそのまま保持せず、意味単位へ正規化する。
- `Symbol` は共通ヘッダ、`Decl / Payload` は詳細本体として責務分離する。
- `Relation` は一次情報ではなく派生ビューとして扱う。
- `Behavior IR` は分岐・ループ・呼び出し追跡の一次情報として扱う。
- 型推論結果は宣言型を上書きせず、`TypeFact` として蓄積する。
- 図表示用の名称は中間モデルに保持せず、図生成直前に派生生成する。
- 言語固有情報は `Language Extension` に隔離し、共通モデルを汚染しない。

---

## 2. 中間モデル全体構成

中間モデルは以下の6層で構成する。

1. CoreModel
2. Semantic Payload Layer
3. Behavior IR Layer
4. Language Extension Layer
5. Inference Layer
6. Analysis Support Layer

### 2.1 各レイヤの責務

| レイヤ | 役割 |
|---|---|
| CoreModel | 解析対象全体で共通利用する基礎情報を保持する |
| Semantic Payload Layer | 型、関数、フィールド、変数などの宣言本体を保持する |
| Behavior IR Layer | 関数・メソッド内の振る舞いを構造と遷移の両面から保持する |
| Language Extension Layer | Python 固有情報など、共通モデルへ昇格しない言語固有情報を保持する |
| Inference Layer | 型推論・束縛推論の結果を事実として保持する |
| Analysis Support Layer | 起点関数候補や未対応構文など、解析補助情報を保持する |

---

## 3. 基本設計原則

### 3.1 一次情報と派生情報

中間モデルでは、一次情報と派生情報を明確に分離する。

#### 一次情報
- `SourceUnit`
- `Scope`
- `Symbol`
- `SymbolReference`
- `TypeDecl`
- `CallableDecl`
- `FieldDecl`
- `PropertyDecl`
- `VariableDecl`
- `ParameterDecl`
- `BehaviorGraph`
- `BehaviorNode`
- `BehaviorEdge`
- `TypeFact`
- `TypeConstraint`
- `SymbolBindingFact`

#### 派生情報
- `Relation`
- `BehaviorSummary`
- `CallTraceStep`
- 図表示用ラベル

### 3.2 名前の責務分離

名称は以下のように分離する。

| 項目 | 用途 |
|---|---|
| `decl_name` | ソースコード上の宣言名そのもの |
| `canonical_name` | シンボル解決・比較・識別のための完全修飾名 |
| 図表示名 | 図生成直前に派生生成する短縮表示名 |

### 3.3 図表示名の扱い

図表示名は中間モデルの正本には保持しない。  
図生成直前に、図出力対象のシンボル集合をもとに派生生成する。

#### 基本ルール
- 原則としてピリオド区切りの末尾要素のみを表示する。
- 同名衝突が発生した場合は、右から1階層ずつ表示範囲を広げる。
- 解決用名称と表示用名称は必ず分離する。

---

## 4. CoreModel 設計

### 4.1 ProjectModel

プロジェクト全体の中間モデルを表現するルートオブジェクトとする。

#### 保持情報
- `project_id`
- `project_name`
- `source_units`
- `scopes`
- `symbols`
- `symbol_references`
- `resolution_candidates`
- `type_refs`
- `relations`
- `type_facts`
- `diagnostics`

#### 補足
- `relations` は派生ビューであるが、再利用性の高い基本関係を materialize して保持する。
- 図固有の補助関係は保持しない。

### 4.2 SourceUnit

1ファイル単位の解析結果を表す。

#### 保持情報
- `source_unit_id`
- `path`
- `language`
- `module_name`
- `imports`
- `top_level_symbol_ids`
- `root_scope_id`

### 4.3 SourceSpan

ソースコード上の位置情報を保持する。

#### 保持情報
- `file_path`
- `start_line`
- `start_column`
- `end_line`
- `end_column`

### 4.4 Scope

解析用スコープを表現する。

#### 保持情報
- `scope_id`
- `kind`
- `owner_symbol_id`
- `parent_scope_id`
- `source_unit_id`
- `span_id`

#### `kind`
- `module`
- `class`
- `callable`
- `block`

#### 補足
`block` は Python 言語仕様上の正式スコープではなく、分岐・ループ・型制約管理のための**解析用擬似スコープ**として扱う。

### 4.5 Symbol

宣言の共通ヘッダを表現する。

#### 保持情報
- `symbol_id`
- `kind`
- `decl_name`
- `canonical_name`
- `owner_symbol_id`
- `scope_id`
- `source_unit_id`
- `span_id`
- `visibility`
- `modifiers`
- `payload_kind`
- `payload_id`
- `extension_id`

#### 補足
- `display_name` は保持しない。
- `decl_name` はソースコード上の宣言名を保持する。
- 図表示名はここには持たせない。

### 4.6 SymbolReference

コード中の参照を表現する。

#### 保持情報
- `ref_id`
- `owner_symbol_id`
- `scope_id`
- `usage_kind`
- `text`
- `span_id`
- `receiver_ref_id`
- `arg_index`
- `is_attribute_access`

### 4.7 ResolutionCandidate

参照解決結果候補を表現する。

#### 保持情報
- `ref_id`
- `candidate_symbol_id`
- `status`
- `confidence`
- `reason`
- `rank`

### 4.8 TypeRef

型参照を表現する不変オブジェクトとする。

#### 保持情報
- `type_id`
- `kind`
- `display_name`
- `canonical_name`
- `namespace`
- `arg_type_ids`
- `element_type_id`
- `pointee_type_id`
- `union_member_type_ids`
- `nullable`
- `resolved_symbol_id`
- `raw_text`

#### `kind`
- `BUILTIN`
- `NAMED`
- `GENERIC`
- `ARRAY`
- `POINTER`
- `UNION`
- `UNKNOWN`
- `VOID`
- `ANY`

#### 型正規化方針
- 同一意味の型は Type Pool により再利用する。
- `UNKNOWN`、`VOID`、`ANY` は共有 TypeRef として扱う。
- `canonical_name` は主に `NAMED` 型に対して保持する。
- `UNION`、`GENERIC`、`ARRAY` などの合成型は構造情報で表現する。

### 4.9 Relation

シンボル間・型間の関係を表す派生ビューとする。

#### 主な `kind`
- `DECLARES`
- `EXTENDS`
- `IMPLEMENTS`
- `HAS_A`
- `USES`
- `CALLS`
- `CREATES`
- `IMPORTS`
- `RETURNS`
- `PARAM_TYPE`

#### 保持情報
- `kind`
- `source_symbol_id`
- `target_symbol_id` または `target_type_id`
- `label`
- `confidence`
- `span_id`

#### 保持方針
- 基本的な Relation は `ProjectModel` に materialize して保持する。
- 図固有の一時的 Relation は保持せず、図生成時に on-demand 生成する。

### 4.10 Diagnostic

解析時の診断情報を表す。

#### 保持情報
- `diagnostic_id`
- `level`
- `code`
- `message`
- `span_id`
- `related_symbol_id`
- `related_ref_id`

#### `level`
- `ERROR`
- `WARNING`
- `INFO`

#### `code` プレフィックス
- `PARSE`
- `RESOLVE`
- `MODEL`
- `INFER`
- `GENERATE`
- `CONFIG`
- `GUI`

#### 補足
詳細なコード体系は異常系設計で定義する。中間モデル設計ではプレフィックスまでを固定する。

---

## 5. Semantic Payload Layer 設計

### 5.1 TypeDecl

クラス、インターフェース、構造体、列挙体などの型宣言本体を表現する。

#### 保持情報
- `symbol_id`
- `type_kind`
- `type_param_ids`
- `base_type_ids`
- `member_symbol_ids`
- `nested_type_symbol_ids`

### 5.2 CallableDecl

関数・メソッド・コンストラクタを表現する。

#### 保持情報
- `symbol_id`
- `parameter_symbol_ids`
- `return_type_id`
- `type_param_ids`
- `throws_text`
- `behavior_graph_id`

### 5.3 FieldDecl

クラスのフィールドを表現する。

#### 保持情報
- `symbol_id`
- `declared_type_id`
- `initializer_text`
- `assigned_from_ref_ids`

#### Python 初版での扱い
- `self.xxx = ...` を検知した場合、基本的に `FieldDecl` 候補として扱う。
- `__init__` 内か否かによる主/補助の区別は行わない。
- 必要に応じて初出位置などの補助情報を将来拡張可能とする。

### 5.4 PropertyDecl

property 相当の宣言を表現する。

#### 保持情報
- `symbol_id`
- `declared_type_id`
- `has_getter`
- `has_setter`

### 5.5 VariableDecl

ローカル変数、グローバル変数、トップレベル変数を表現する。

#### 保持情報
- `symbol_id`
- `declared_type_id`
- `initializer_text`

### 5.6 ParameterDecl

Callable の引数を表現する。

#### 保持情報
- `symbol_id`
- `declared_type_id`
- `default_text`
- `optional`
- `variadic`
- `keyword_name`

---

## 6. Behavior IR Layer 設計

### 6.1 目的
Behavior IR は、分岐・ループ・呼び出し・代入・生成・戻りを含む振る舞いを、シーケンス図生成および型推論に再利用可能な形で保持するために用いる。

### 6.2 BehaviorGraph

Callable 単位の振る舞いグラフを表現する。

#### 保持情報
- `behavior_graph_id`
- `owner_callable_symbol_id`
- `root_node_id`

### 6.3 BehaviorNode

振る舞いの構造ノードを表現する。

#### 保持情報
- `node_id`
- `owner_callable_symbol_id`
- `parent_node_id`
- `kind`
- `order_index`
- `span_id`
- `guard_text`
- `ref_ids`
- `child_node_ids`

#### `kind`
- `BLOCK`
- `CALL`
- `ASSIGN`
- `RETURN`
- `IF`
- `LOOP`
- `CREATE`

#### 役割
`BehaviorNode` の親子構造は**構文の内包関係**を表す。  
例として、`IF` ノードの配下に then ブロック、then ブロックの配下に `RETURN` ノードを保持する。

### 6.4 BehaviorEdge

ノード間の実行遷移を表現する。

#### 保持情報
- `edge_id`
- `from_node_id`
- `to_node_id`
- `edge_kind`
- `guard_text`

#### 役割
`BehaviorEdge` は**実行遷移**を表す。  
構文木の親子関係とは別に、条件分岐やループ分岐を含む実行順序を表現する。

### 6.5 構造と遷移の分担

| 要素 | 役割 |
|---|---|
| `parent_node_id` / `child_node_ids` | 構文の内包関係 |
| `BehaviorEdge` | 実行遷移 |

この分担により、以下を両立する。

- 分岐・ループの構造的把握
- シーケンス図生成時の経路追跡
- 分岐内限定の型制約管理

### 6.6 BehaviorSummary

Behavior IR から派生生成される要約情報とする。  
一次情報ではなく、必要に応じて生成する。

### 6.7 CallTraceStep

シーケンス図生成時に派生生成する呼び出しステップとする。

#### 保持情報
- `caller_symbol_id`
- `callee_symbol_id`
- `receiver_type_ids`
- `call_text`
- `line`
- `branch_condition`
- `loop_context`

---

## 7. Language Extension Layer 設計

### 7.1 基本方針
Language Extension は、共通中間モデルへ直接昇格させない言語固有情報を保持するための層とする。

### 7.2 初版方針
初版では Python のみを対象とし、辞書型ではなく**型付き dataclass**で保持する。

### 7.3 初版で定義する Extension

#### 7.3.1 PythonTypeExtension

型宣言に付随する Python 固有情報を保持する。

##### 想定項目
- class decorator 一覧
- metaclass 補助情報
- raw AST 補助情報

#### 7.3.2 PythonCallableExtension

関数・メソッドに付随する Python 固有情報を保持する。

##### 想定項目
- decorator 名一覧
- `is_classmethod`
- `is_staticmethod`
- `is_property_getter`
- `is_property_setter`
- raw AST 補助情報

#### 7.3.3 PythonReferenceExtension

参照表現に付随する Python 固有情報を保持する。

##### 想定項目
- alias import 補助情報
- attribute access 補助情報
- raw AST 由来の参照種別

### 7.4 紐づけ方針
- `Symbol` または対象 Payload から `extension_id` により参照可能とする。
- 言語固有項目を共通 dataclass へ追加しない。
- 生の `dict[str, Any]` は採用しない。

---

## 8. Inference Layer 設計

### 8.1 基本方針
推論結果は宣言型を上書きせず、事実として蓄積する。

### 8.2 TypeFact

型推論結果を保持する。

#### 保持情報
- `fact_id`
- `target_kind`
- `target_id`
- `inferred_type_id`
- `reason`
- `source_ref_id`
- `guard_scope_id`
- `confidence`

#### 補足
- 分岐内のみ有効な型推論は `guard_scope_id` で表現する。
- 1対象に対して複数の `TypeFact` を保持可能とする。

### 8.3 TypeConstraint

推論に用いる型制約を表現する。

### 8.4 SymbolBindingFact

引数束縛、代入元、属性更新元などの対応関係を表現する。

---

## 9. Analysis Support Layer 設計

### 9.1 CandidateEntryPoint

起点関数候補を表現する。

#### 保持情報
- `symbol_id`
- `module_path`
- `class_name`
- `function_name`
- `arg_count`
- `line`
- `score`
- `priority`

### 9.2 AnalysisOptionSnapshot

解析時点のオプション状態を保持する。

### 9.3 UnsupportedConstructRecord

初版未対応構文を記録する。

---

## 10. Python 初版の具体的な扱い

### 10.1 `self.xxx` の扱い

```python
class UserService:
    def __init__(self, repo):
        self.repo = repo
        self.cache = {}

    def refresh(self, repo):
        self.repo = repo
```

上記では、`self.repo`、`self.cache` を基本的に `FieldDecl` 候補として扱う。  
`__init__` とその他メソッドで主/補助の区別は行わない。

### 10.2 block scope の扱い

```python
if isinstance(result, User):
    return result.name
```

この場合、`if` 内だけ `result` を `User` とみなすため、解析用の `block` scope を生成し、`TypeFact.guard_scope_id` により適用範囲を管理する。

### 10.3 図表示名の扱い

```python
pkg.domain.user.User
pkg.repositories.user.User
```

図生成時はまず `User` を候補とし、衝突時に以下のように右から表示範囲を広げる。

- `User`
- `domain.user.User`
- `repositories.user.User`

### 10.4 TypeRef の扱い

```python
def set_repo(self, repo):
    self.repo = repo
```

型ヒントがない `repo` は `TypeRef(kind=UNKNOWN)` を参照する。  
`UNKNOWN` は共有 TypeRef として再利用する。

---

## 11. 今後の拡張余地

本設計では、将来の以下の拡張を妨げない構造とする。

- 対応言語追加
- 出力図種追加
- 出力記法追加
- `self.xxx` に対する補助情報強化
- Diagnostic コード体系の詳細化
- Language Extension の項目追加
- 図表示名生成ルールの高度化

---

## 12. まとめ

本設計では、中間モデルを単一の共通モデルとして維持しつつ、以下を明確に分離した。

- 解決用名称と図表示用名称
- 構文の内包関係と実行遷移
- 宣言情報と推論情報
- 共通モデルと言語固有拡張
- 一次情報と派生情報

これにより、初版実装の現実性を保ちながら、将来の多言語対応・図種追加・出力拡張に耐えうる中間モデル基盤を構築する。
