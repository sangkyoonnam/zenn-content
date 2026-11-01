---
title: 'Flink Agents 0.4.0で見るストリーミングエージェント'
emoji: '🔧'
type: 'tech'
topics: ['flink', 'flinkagents', 'aiagent', 'opensource']
published: true
---

> この記事は [Namsang Labs 技術ブログ](https://blog.namsang.co/ja/posts/flink-agents-0-4-0/) からの転載です。

![紫の背景にFlinkのリスのロゴと、Apache Flink Agents 0.4.0という大きな文字が入った画像](https://blog.namsang.co/og-custom/flink-agents-0-4-0.png)

10月30日にApache Flink Agents 0.4.0がリリースされた。本番向けの1.0を控え、機能とAPIを整理するプレビュー版だ。今回の0.4.0には、私もコントリビューターとして参加した。リリースを機に、Flink Agentsがどんなプロジェクトで、エージェントをどう実行し、障害からどう復旧させるのかをまとめた。

## Flinkの上でエージェントを実行する理由

決済が失敗したり、センサーが異常な値を送ってきたりしたら、システムはそのイベントを受け取り、原因を判断して必要な処置を進めなければならない。人がチャット画面で質問するまで待てる仕事ではない。[Flink Agentsは、こうした処理のために始まったプロジェクトだ](https://flink.apache.org/2025/10/15/apache-flink-agents-0.1.0-release-announcement/)。流れ続けるイベントをエージェントが受け取って判断し、ツールを呼び出して必要な作業を行うことを目指している。

イベントが増えれば複数のサーバーに処理を分散する必要があり、顧客や機器ごとに、それまでに処理した内容も覚えておかなければならない。動き続けるシステムなので、サーバーに障害が起きたり実行が中断したりしても、作業を続けられる必要がある。モデルを呼ぶコードだけでは、この実行環境は作れない。

Flinkには、データストリームを分散処理し、キーごとに状態を管理しながら、障害時にはチェックポイントから復旧する機能がある。Flink Agentsはこのランタイムの上に、モデルとツール、メモリを使うエージェントを実行する仕組みを加えたApache Flinkのサブプロジェクトだ。イベントの受信からエージェントの判断、結果の処理までを、一つのFlinkジョブの中でつなぐ。

## 0.1.0から0.4.0まで

最初のプレビュー版となる[0.1.0](https://flink.apache.org/2025/10/15/apache-flink-agents-0.1.0-release-announcement/)は、2025年10月にリリースされた。作業をアクションという単位に分けてイベントでつなぎ、Flinkの中でモデルやツールを呼ぶ基本の仕組みができた。実行を追うためのイベントログと、完了したアクションの記録を復旧時に再利用する機能も、この時点で含まれていた。Flink 1.20.3を基盤にPythonとJavaのAPIを提供していたが、初期の機能は主にPythonで実装されていた。

2026年2月の[0.2.0](https://flink.apache.org/2026/02/06/apache-flink-agents-0.2.0-release-announcement/)ではJava APIの機能が補われ、対応するFlinkのバージョンも1.20から2.2まで広がった。別の言語で実装されたリソースも使えるようになった。メモリは、1回の実行だけで使うものと次の実行にも残すものを分けられるようになった。復旧時には、完了したアクションの実行記録に加えて、まだ終わっていないアクションの中で保存済みの個々の呼び出し結果も再利用できるようになった。

[0.3.0](https://flink.apache.org/2026/06/19/apache-flink-agents-0.3.0-release-announcement/)は2026年6月にリリースされた。モデルやツールの設定をYAMLで宣言し、別の言語で書いたアクションも一緒に実行できるようになった。長期メモリはMem0を使って情報を抽出、整理する方式に変わり、作業の指示や参考資料を提供するAgent Skillsも追加された。実行記録の保存先としてApache Flussを選べるようになったほか、Web UIでイベントログを確認する機能も加わった。復旧時に結果の分からない外部呼び出しを確認する調整関数も、0.3.0で追加された。それまでは保存済みの呼び出し結果を再利用する方式だったが、呼び出しの途中で中断して結果が残っていない場合にも、外部システムの状態を確認する手段ができた。

今回リリースされた[0.4.0](https://flink.apache.org/2026/10/30/apache-flink-agents-0.4.0-release-announcement/)では、そこにサブエージェントやモデルルーティングなどの機能が加わった。モデルを呼んだ後のロジックを同じアクションに書ける`ctx.chat`も追加され、イベントだけでなく、モデルやツールの呼び出しまで追えるようになった。1.0からAPIの互換性を約束するのに先立ち、既存のAPIとデフォルトの動作も整理された。

## エージェントを実行する仕組み

エージェントはFlinkジョブの中のオペレーターとして実行される。前段からデータを受け取って作業し、結果を次のオペレーターに渡す。エージェントの内部でも、処理はイベントでつながっている。[基本の構成要素](https://nightlies.apache.org/flink/flink-agents-docs-release-0.4/docs/development/create-agent/overview/)はイベントとアクション、リソースだ。イベントが届くと条件に合うアクションが実行され、必要な処理をした後で新たなイベントを送る。一つのイベントに複数のアクションが反応することもあり、分岐や繰り返しを作れる。モデルやツール、プロンプトのように複数のステップで使うものは、リソースとして登録して名前で参照する。MCPサーバーのツールやプロンプトも接続でき、0.3.0からはリソースをYAMLで宣言できる。

商品レビューを要約するエージェントなら、入力レコードが`InputEvent`として届き、それを受け取ったアクションが`ChatRequestEvent`で要約を依頼する。組み込みのチャットアクションがモデルを呼んで`ChatResponseEvent`を送ると、応答を処理するアクションが要約文を`OutputEvent`に入れる。その値がFlinkの次のオペレーターに渡る。こうしてステップ同士をイベントでつなぐので、イベントの記録をたどれば、どの入力からどんな判断と出力が続いたのかを確認できる。

処理の順序をコードで決めることも、次の行動をモデルに任せることもできる。組み込みの[`ReActAgent`](https://nightlies.apache.org/flink/flink-agents-docs-release-0.4/docs/development/react_agent/)にはモデルを指定し、必要に応じてツールやプロンプトを加える。モデルは目標を達成するために必要なツールを選び、その結果を見て次の行動を決める。全体の手順はコードで決め、一部の調査だけをこうしたエージェントに任せる形で、二つの方式を組み合わせることもできる。

エージェントを定義したら、Flinkのジョブにつなぐ。0.4.0のドキュメントのPython例から、その部分を抜き出すと次のようになる。

```python
env = StreamExecutionEnvironment.get_execution_environment()
agents_env = AgentsExecutionEnvironment.get_execution_environment(env)

output_stream = (
    agents_env.from_datastream(
        input=input_stream, key_selector=lambda x: x.id
    )
    .apply(your_agent)
    .to_datastream()
)
```

入力はキーごとに分かれ、前段から届いたレコードごとに新しい実行が始まる。ドキュメントではこれをagent runと呼ぶ。上のコードの`x.id`が顧客IDなら、同じ顧客の入力同士でメモリを共有するように構成できる。キーを選ぶ関数を渡せばその関数を使い、省略すれば`DataStream`にすでに指定されているキーを使う。どちらもなければ接続できない。[`Table`](https://nightlies.apache.org/flink/flink-agents-docs-release-0.4/docs/development/integrate_with_flink/)も接続できるが、入力テーブルは行の追加だけを行うものに限られる。既存の行を更新したり削除したりする集計結果は、そのままではつなげられない。

メモリは、どれだけ長く残し、どう読み出すかによって分かれる。[感覚メモリと短期メモリ](https://nightlies.apache.org/flink/flink-agents-docs-release-0.4/docs/development/memory/overview/)は、どちらもFlinkの状態に保存される。感覚メモリは1回の実行が終わると消え、短期メモリは同じキーで届いた次の実行でも読み出せる。顧客ごとのエージェントなら、今回のリクエストを処理するあいだだけ必要な値は感覚メモリに、次のリクエストでも使う値は短期メモリに置く。

長期メモリは複数の実行で得た情報を整理して残し、セマンティック検索で探す。0.3.0からは[Mem0](https://nightlies.apache.org/flink/flink-agents-docs-release-0.4/docs/development/memory/long_term_memory/)が情報の抽出と整理を担当し、実際のデータは外部のベクトルストアに保存する。そのため、チャットモデルと埋め込みモデル、ベクトルストアをあわせて設定する必要がある。長期メモリの検索範囲も、デフォルトでは同じジョブと入力キーの中に限られる。

モデルに作業の指示を与える[Agent Skills](https://nightlies.apache.org/flink/flink-agents-docs-release-0.4/docs/development/calling-llms/skills/)もある。`SKILL.md`に書いた指示と関連するスクリプト、参考資料をまとめておくと、エージェントが必要に応じて内容を読み込んで使う。

## 障害から復旧する仕組み

エージェントは一つの作業を終えるまでに、モデルや外部ツールを何度も呼ぶことがある。その途中でタイムアウトや呼び出し回数の制限にかかることもあれば、エージェントを実行していたプロセスが止まることもある。呼び出しエラーへのリトライ方針とは別に、実行が中断した後、完了済みの作業を繰り返さずに残りを続ける方法も必要になる。最初から実行し直しても、前と同じ結果になるとは限らない。同じ入力でもモデルが別の答えを返すことがあり、外部システムから取得した値も、そのあいだに変わっているかもしれない。送信済みのメールや決済リクエストまで繰り返せば、外部の作業が重複する可能性もある。復旧するには、何が完了していて、何を再実行する必要があるのかを区別しなければならない。

Flinkは障害が起きると、最後のチェックポイントから状態と入力の位置を復元し、それ以降の入力を再処理する。外部ストアに結果を書き込むときは、トランザクションや冪等性に対応したシンクを使い、同じ結果が重複して反映されるのを防げる。だがエージェントが入力を処理し直すと、モデルやツールも再び呼ぶことがあり、すでに行った外部の作業をチェックポイントで元に戻すことはできない。

完了したアクションの実行記録を復旧に使うには、Flinkのチェックポイントに加えて、アクション状態ストア(Action State Store)を設定する。KafkaやApache Flussにアクションの実行記録を残す仕組みで、完了したアクションの記録を再利用する機能は0.1.0から提供されている。[0.4.0の一貫性に関するドキュメント](https://nightlies.apache.org/flink/flink-agents-docs-release-0.4/docs/operations/consistency/)によれば、完了記録のあるアクションは本体を実行し直さず、保存しておいた感覚メモリと短期メモリの変更、出力イベントをそのまま再生する。ただし、非同期で実行されていた複数のアクションが共有のメモリを読み書きした順序までは再現しない。その順序で結果が変わるなら、必要な順序をイベントの依存関係として明示しなければならない。

0.2.0では、復旧時に再実行しない範囲をアクションの中の呼び出し単位まで狭めた。アクションが終わっていなくても、その中で耐久実行(durable execution)で包んだ呼び出しが終わって結果が保存されていれば、復旧時にその呼び出しはもう行わない。[アクションのドキュメント](https://nightlies.apache.org/flink/flink-agents-docs-release-0.4/docs/development/create-agent/actions/#durable-execution)はプロフィールの取得を例に挙げている。

```python
class ProfileAgent(Agent):
    @action(EventType.InputEvent)
    @staticmethod
    def process_input(event: Event, ctx: RunnerContext) -> None:
        user_id = InputEvent.from_event(event).input
        profile = ctx.durable_execute(
            profile_service.get_profile,
            user_id,
            durable_id=f"profile:{user_id}",
        )
        ctx.send_event(OutputEvent(output=profile))
```

プロフィールの取得が終わり、結果まで保存された後でアクションが失敗すると、復旧したアクションはサービスを呼び直さず、保存されたプロフィールを取り出して使う。ドキュメントにもあるとおり、耐久実行の呼び出しの外にあるコードは再実行されることがある。完了記録と呼び出しの結果は外部ストアに同期的に書き込まれ、復元するチェックポイントと、その復旧に必要な実行記録も一緒に保持しておかなければならない。復旧後の実行が以前の記録を見つけて使うには、キーごとの入力順序とキーの型、シリアライザの設定、アクションの名前が変わっていないことが条件になる。耐久実行の呼び出しも同じ入力と順序、安定した識別子を保つ必要があり、保存された値は復元の時点でも読める状態でなければならない。

呼び出しを始めてから結果を保存するまでのあいだに障害が起きると、呼び出しが実際に成功したかどうかは記録だけでは分からない。外部サービスが同じリクエストを繰り返しても処理を重複させないなら、再試行できる。そうでなければ、そのリクエストが処理されたかを先に確かめる必要がある。[0.3のロードマップ解説](https://zhuanlan.zhihu.com/p/2029555366648653207)もこの問題を取り上げ、ユーザーが外部の状態を確認するための関数を提案していた。

0.3.0では、この確認処理を実装できる調整関数(reconciler)が追加された。復旧時に、実行中の呼び出しの記録はあるのに結果がなければ、この関数が実行される。調整関数には、外部システムの処理状況を確かめ、元の結果を返すか失敗を知らせるコードを書く。こうして得た結果も記録に残る。調整関数がなければ、元の呼び出しをもう一度実行する。

![復旧時に何を再実行するかは、障害時点の記録で決まる。完了記録があれば再生し、結果が保存された呼び出しはそれを再利用し、実行中だった呼び出しは調整関数で確かめるか再実行する](/images/flink-agents-0-4-0/ja-recovery.png)

アクション状態ストアは、外部の動作とその結果を保存する処理を一つのトランザクションにまとめられない。そのため外部の状態を変える呼び出しは、繰り返されうるものとして設計する必要がある。外部サービスの冪等性の仕組みを使うか、耐久実行の呼び出しに元の結果を突き止められる調整関数を付けるのが手だ。

## 新たに追加された機能

今回のバージョンでは、作業を別のエージェントに任せたり、リクエストに合うモデルを選んだりする機能が追加された。モデル呼び出しと応答処理を一つのアクションに書けるようになり、運用中にモデルやツールの呼び出し記録も確認できるようになった。

![Flink Agents 0.4.0の新機能。サブエージェント、意思決定モデル、モデルルーティング、マルチモーダル入力、構造化出力、ctx.chat、条件付きトリガー、ツール引数の注入、トレースログ、メトリクス](/images/flink-agents-0-4-0/ja-features.png)

表のテキスト版は[元の記事](https://blog.namsang.co/ja/posts/flink-agents-0-4-0/)で読めます。

0.3.0のリリース後にあたる2026年9月15日、TypeSafeは[Jev](https://typesafe.ai/blog/introducing-system-one-models-and-jev)を公開した。与えられた文脈をあらかじめ決めた基準に照らして判断し、確率や選択肢、スコアとして結果を返すモデルだ。Flink Agentsは今回のバージョンに[意思決定モデルAPI](https://nightlies.apache.org/flink/flink-agents-docs-release-0.4/docs/development/decision_models/)を追加し、JevとOpenAI、アリババクラウドのModel Studioを同じインターフェースで使えるようにした。たとえば、顧客の問い合わせを請求担当と技術サポートのどちらに回すか判断したり、あらかじめ決めた尺度でアラートのリスクを評価したりできる。モデルに渡す事実と判断基準、結果を受けて実行する後続の処理は、アプリケーション側で決める。

開発ツールとモデル連携も補強された。Javaの非同期実行は、JDK 21に加えてJDK 11と17にも対応した。チャットモデルは、JavaでGeminiを、JavaとPythonでvLLMとIBM watsonx.aiを新たに使える。JavaにはOpenAIの埋め込みモデルも追加された。コーディングエージェントがFlink Agentsのアプリケーションを作って検証するときに参照する`flink-agents-dev`スキルも追加された。

これらの機能を使って、深刻なアラートだけを選んで処理するエージェントを作るとしよう。条件トリガーで、深刻度の高いアラートに対してだけアクションを実行する。そのアクションの中で`ctx.chat`を使ってモデルを呼び、応答をその場で処理する。モデルが登録済みのツールを使いたいと返せば、フレームワークが代わりに実行する。

[イベント駆動のチャットAPI](https://nightlies.apache.org/flink/flink-agents-docs-release-0.4/docs/development/calling-llms/chat/)では、リクエストと応答をそれぞれ別のアクションで処理する。この方式は0.4.0でも引き続き使える。`ctx.chat`は、モデルを呼んだ後のロジックを同じアクションに続けて書きたいときに使うAPIだ。耐久実行を有効にしておけば、`ctx.chat`の応答も復旧時に再利用できる。

追加の調査はサブエージェントに任せられ、その呼び出しも耐久実行に対応している。決まった基準でリスクを評価したいなら意思決定モデルを使い、リクエストごとに別のモデルを使いたいならモデルルーティングを使う。単純なリクエストは軽いモデルに送り、複雑な作業にだけ大きなモデルを使う、といった使い分けだ。

運用では、トレースログで一つの入力がどのステップや呼び出しを経て処理されたのかを確認する。組み込みメトリクスで失敗や遅延を見つけたら、このログでどのステップに問題があったのかを絞り込める。[デフォルトの記録範囲](https://nightlies.apache.org/flink/flink-agents-docs-release-0.4/docs/operations/monitoring/)はイベントだけなので、ステップや呼び出しまで見るには`trace-log.targets`を設定する必要がある。イベント同士の因果関係をツリーの形に描き直すツールも追加された。このツールはイベントの記録だけを読み、実行ステップや呼び出しの記録は読み飛ばす。トレースログの書き込みに失敗しても、イベント処理とジョブの実行は続く。だから、トレースログだけを見てすべての実行が漏れなく記録されたと言い切ることはできない。

## 0.4.0へのアップグレード方法

0.xはまだプレビュー版なので、APIや設定が互換性のない形で変わることがある。今回は、初期の本番ユーザーからのフィードバックやAPIレビューで分かった問題を整理したため、互換性のない変更がこれまでのどのリリースよりも多くなった。0.4.0を安定させてから小さな調整を経て1.0をリリースし、そこからAPIの互換性を約束する計画だ。具体的な互換性ポリシーは別途発表される予定になっている。0.3.xから0.4.0にアップグレードするときは、まず次の変更点を確認する必要がある。

- アクションの宣言: Javaの`@Action(listenEventTypes = ...)`は`@Action(...)`に、Pythonの`add_action(events=...)`とYAMLの`listen_to`は`trigger_conditions`に変わる。よく使うPythonの`@action("event_type")`という宣言はそのまま使える。
- Pythonの実行経路: `from_list()`と`to_list()`でFlinkなしに実行していた経路がなくなった。Flinkの`StreamExecutionEnvironment`にDataStreamかTableでつなぐ必要があり、ローカルで開発するときはローカルのFlinkランタイムを使う。
- リトライのデフォルト: フレームワークの`max-retries`のデフォルトが3から0になり、`error-handling-strategy`はなくなった。失敗したチャットの応答はアプリケーション側で処理しなければならない。モデル提供者のSDKのリトライは別に設定する。
- 並行実行のデフォルト: デフォルトの非同期設定では、互いに独立したツール呼び出しが並行して実行されることがある。順序が大事なツールなら、ステップを分けるか並列度を絞る必要がある。JDK 11から20でJavaのアクションだけを使う場合は、実験的な機能であるアクションの並列実行がデフォルトで有効になり、`parallel-execution.enabled`で無効にできる。
- ツール引数の検証: 未知のフィールドや誤った型、不正な値が来ると実行前に失敗する。数値の文字列を数値に変換することもない。
- ログの設定: イベントログがトレースログに置き換わり、`event-log.*`のような以前の設定キーは受け付けられない。`trace-log.*`のキーに移す必要がある。
- 実行記録: 0.3.xが残したアクション状態ストアのデータは0.4.0と互換性がないので、以前の実行記録を再利用しないアップグレード計画が要る。Flinkのチェックポイントやセーブポイントを復元できるかどうかは、このストアのデータ互換性とは別に確認する必要がある。
- 処理中のアクション: 処理中のアクションを含むオペレーターのスナップショットは、0.4.0にアップグレードするときに復元に失敗することがある。以前のバージョンにもあった制約なので、実際に近い状態でまず復元を試しておくのがよい。

メッセージの形式も変わった。`ChatMessage`は内容を`blocks`に、追加の属性を`metadata`に持つ。モデル呼び出しは、応答メッセージとモデルの情報、使用量を分けて持つ`ChatResult`を返し、ツールの結果は`ToolResponse`に統一された。モデル連携を自作していたり、メッセージをJSONで保存していたりするなら、まずこの部分から手を入れる必要がある。

## コントリビューターから見た0.4.0

告知のコントリビューター一覧には38人が載っており、私も今回のリリースから貢献できた。今回取り込まれた私のPR 23件のうち5件は、先に説明した耐久実行の再生経路を直したものだった。記録された結果が`null`だと再生時に例外が出ていた [#1171](https://github.com/apache/flink-agents/pull/1171)。pickleで保存はできるが復元できないPythonの例外は、見当違いの`TypeError`として再生されていた [#1182](https://github.com/apache/flink-agents/pull/1182)。調整関数なしで完了結果だけを記録するPythonの耐久実行の呼び出しは、Java側で結果の保存に失敗しても呼び出し元に知らせていなかった [#1211](https://github.com/apache/flink-agents/pull/1211)。

通常の呼び出しと耐久実行の呼び出しを非同期でまとめて待っているときに、通常の呼び出しがキャンセルされると、入力順でそれより前にあった耐久実行の呼び出しは完了していても結果が保存されないケースがあった [#1239](https://github.com/apache/flink-agents/pull/1239)。組み込みのPythonチャット呼び出しの識別子が、プロセスごとに変わりうる問題も直した [#1240](https://github.com/apache/flink-agents/pull/1240)。そのほか、モデルルーティングの検証やメモリの参照、ランタイムのキュー、メトリクス、ドキュメントを修正した。

リリースの直前には最初のリリース候補を検証し、投票にも参加した。その経緯は「[オープンソースにも投票がある](https://blog.namsang.co/ja/posts/open-source-has-votes/)」に別に書いた。検証中に、バージョンを上げた後で既存のスナップショットを復元するときの問題を確認し、その内容が正式な告知のアップグレード時の注意事項に反映された。その後、[バージョンをまたぐ状態の復元を検証するテスト基盤](https://github.com/apache/flink-agents/issues/1332)を作る作業も担当することになった。1.0ではAPIが安定し、アップグレードの検証も充実して、運用中のエージェントを今より保守しやすくなることを期待している。

## 始めるには

[実行環境の設定](https://nightlies.apache.org/flink/flink-agents-docs-release-0.4/docs/get-started/setup/)で使用するFlinkとJava、Pythonのバージョンを確認したうえで、[最初のエージェントを作る](https://nightlies.apache.org/flink/flink-agents-docs-release-0.4/docs/get-started/quickstart/create-agent/single-agent/)の例を実行すればよい。処理の手順を書くより先にモデルにツールを任せてみたいなら、[ReActAgentのクイックスタート](https://nightlies.apache.org/flink/flink-agents-docs-release-0.4/docs/get-started/quickstart/react_agent/)が向いている。コーディングエージェントで開発するなら、`flink-agents-dev`スキルを追加できる。これはコーディングエージェントがAPIやサンプルを参照するための開発資料であり、FlinkとFlink Agentsの実行環境は上の案内に従って別にインストールする。

```bash
npx skills add 'apache/flink-agents#release-0.4@flink-agents-dev'
```

バグや機能の要望は[GitHubのイシュー](https://github.com/apache/flink-agents/issues)で受け付けており、利用事例やアイデアは[GitHubのディスカッション](https://github.com/apache/flink-agents/discussions)と、Apache FlinkのSlackにある`#flink-agents-user`チャンネルで話せる。

## 参考資料

- [Apache Flink Agents 0.4.0 Release Announcement](https://flink.apache.org/2026/10/30/apache-flink-agents-0.4.0-release-announcement/)、Wenjin Xie、2026年。0.4.0で追加された機能と互換性のない変更、そして0.4.0が1.0前の最後のリリースだという計画を書いている。
- [Apache Flink Agents 0.3.0 Release Announcement](https://flink.apache.org/2026/06/19/apache-flink-agents-0.3.0-release-announcement/)、2026年。Flinkの上でエージェントを実行する理由と、0.4.0と1.0へと続くAPI互換性の計画を説明している。
- [Apache Flink Agents 0.2.0 Release Announcement](https://flink.apache.org/2026/02/06/apache-flink-agents-0.2.0-release-announcement/)、2026年。3種類のメモリと呼び出し単位の耐久実行、複数のFlinkバージョンへの対応を紹介している。
- [Apache Flink Agents 0.1.0 Release Announcement](https://flink.apache.org/2025/10/15/apache-flink-agents-0.1.0-release-announcement/)、2025年。プロジェクトがなぜ必要なのかと、最初のプレビュー版の主な機能を紹介している。
- [Flink Agents 0.3 Roadmap解读](https://zhuanlan.zhihu.com/p/2029555366648653207)、Flink中国語コミュニティ、2026年。0.3.0のロードマップをコントリビューターの視点で解説し、障害復旧で残る問題と解決策の候補を挙げている。同コミュニティのこの記事には、まだ0.4.0の内容が反映されていない。
- [Flink Agents 0.4.0のドキュメント](https://nightlies.apache.org/flink/flink-agents-docs-release-0.4/)。はじめにと開発ガイド、一貫性と復旧の流れを説明する運用の章がある。

---

*広告やCRMからAIエージェントまで、大規模なプラットフォームを作り、率いてきた経験を [Namsang Labs 技術ブログ](https://blog.namsang.co/ja/?utm_source=zenn&utm_medium=crosspost) に書いています。*
