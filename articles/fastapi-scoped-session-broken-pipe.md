---
title: 'FastAPIでscoped_sessionが静かにBroken Pipeを引き起こす理由'
emoji: '🔧'
type: 'tech'
topics: ['python', 'fastapi', 'sqlalchemy', 'backend']
published: false
---

> この記事は [Namsang Labs 技術ブログ](https://blog.namsang.co/ja/posts/fastapi-scoped-session-broken-pipe/) からの転載です。

```
(2006, "MySQL server has gone away (BrokenPipeError(32, 'Broken pipe'))")
```

このエラーは月に一、二度、主に明け方や朝一番のリクエストで落ちていた。リトライすれば消え、翌月の同じ時間帯にまた出る。しばらく優先度が低いまま放っておいたが、頻度がふだんの十倍に跳ね上がったある火曜日に、最後まで追いかけることにした。

## 結論と当てはまる条件

ざっくり言えば、リクエストが終わってもDB接続を返さなかったセッションがスレッドに残り、夜のうちにDBが切ってしまったその接続で朝一番のクエリを送ったのだと見ている。接続を検査するオプションはプールから接続を借りるときにしか動かないので、すでに握っている接続は検査できなかった。

FastAPIで`scoped_session`をデフォルト設定のまま使うと、同期エンドポイントがワーカースレッドで実行されるため、セッションはリクエストではなくそのスレッドに紐づく。リクエストが`commit()`も`close()`もせずに終わり、セッションがトランザクションを開いたままコネクションを握っていると、長く休んでいたスレッドのコネクションがMySQLの`wait_timeout`(サーバーがアイドル接続を切るまでの時間、デフォルト8時間)で切れ、次のリクエストで`Broken Pipe`(2006)や`Lost connection`(2013)が起きる。

解決策は、スレッドごとに引いていたセッションをリクエストごとに引くように変え、リクエストが終わるたびにそのセッションを片付けることだった。具体的には、セッションを引くキーを決める`scopefunc`を`ContextVar`のリクエストIDを返す関数に替え、ミドルウェアでリクエストごとに`remove()`を呼んだ。業務コードには手を入れていない。デプロイ後の一か月、このエラーは一度も出ていない。

次の条件がすべて当てはまるなら、同じ問題である可能性が高い。

- `scoped_session`を`scopefunc`なしで使っている。
- リクエストが`commit()`も`close()`もせずに終わり、セッションがトランザクションを開いたままコネクションを握る。
- `def`で宣言した同期エンドポイントがワーカースレッドプールで実行される。
- MySQLやMariaDBのように、`wait_timeout`でアイドルコネクションを切るDBを使っている。
- トラフィックにむらがあり、一部のスレッドが長く休む。DBを使わない`def`のヘルスチェックが絶えず来ていても当てはまる。

条件を一つずつ確かめにくければ、処理中のリクエストがない時間にプールの状態をログに出してみればよい。リクエストもバックグラウンド処理もないのに、プールから借りたまま返していないコネクション(チェックアウトされたコネクション)が0でなければ、どこかで返却が漏れている合図だ。ワーカープロセスが複数あるとプールもプロセスごとに別なので、それぞれを見る。SQLAlchemy 2.1.5で、コミットせずに終わったセッションを二つスレッドに残しておくと、次のように出た。

```python
print(engine.pool.status())
# Pool size: 5  Connections in pool: 0 Current Overflow: -3 Current Checked out connections: 2
```

この数字だけでは、誰がコネクションを握っているのかはわからない。怪しければ`create_engine(..., echo_pool="debug")`でチェックアウトと返却のログを有効にし、リクエストが終わったあとも返却ログが出ないコネクションを探す。

解決コードは、ミドルウェアがレスポンスを返すときにセッションを片付ける。ストリーミングレスポンスやバックグラウンド処理でも同じセッションを使うなら、先に[注意点](#3-basehttpmiddlewareはレスポンスヘッダーが出た時点で片付ける)の制約を見てほしい。一つのリクエストの中でDB処理を並列に走らせるなら、[セッションを別に開く必要がある](#6-一つのリクエストの中でセッションを同時に使わない)。

## 症状と最初に疑ったもの

パターンははっきりしていた。

- トラフィックがしばらく途切れたあと、再びリクエストが来たときに起きる。
- MySQLの`wait_timeout`のデフォルト値8時間と噛み合う。
- 特定のエンドポイントに偏らず、複数のAPIに散らばって出る。
- `pool_pre_ping=True`をすでに有効にしているのに消えない。

いちばん奇妙だったのは最後の点だ。アプリケーションはDBコネクションを毎回新しく開くのではなく、コネクションプールに溜めておいて借りて使う。プールからコネクションを取り出す瞬間をチェックアウトと呼ぶ。`pool_pre_ping`はチェックアウトのときにコネクションが生きているかを先に確かめ、死んでいれば新しいものに取り替えてくれる。SQLAlchemyのドキュメントが切断された接続への備えとして紹介しているオプションで、私たちは運用のデフォルト値として有効にしてからだいぶ経っていた。

まずMySQL側を疑った。`wait_timeout`は、クエリが一つも来ないコネクションをサーバーが切るまで待つ時間で、デフォルトは28800秒、つまり8時間だ。うちのトラフィックが8時間も空くことはめったにないと思っていた。当時はエンドポイント単位で空白を数えていたが、明け方4時から8時のあいだにその日最初のリクエストが来るエンドポイントは、その前の8時間リクエストを受けていなかった。`wait_timeout`を16時間に延ばした。翌週、同じことがまた起きた。

延ばしたあとも再発した理由は、そのときは確かめられなかった。いま考えると、説明は二つありうる。MySQLは、アプリのように非対話型でつなぐコネクションが確立されるとき、グローバルの`wait_timeout`の値をそのコネクションの設定値にコピーする。グローバル値だけを変えても、すでに開いていたコネクションは8時間のままで、後で見るように私たちのセッションはその古いコネクションを握り続けていた。週末を挟むと空白が16時間を超えるという点もある。新しい値を受け取ったコネクションでも、タイムアウトを延ばすやり方は切れる時点を先送りするだけだった。

`pool_recycle=3600`も入れた。チェックアウトのとき、作られてから1時間を過ぎたコネクションを新しいものに替えるオプションだ。改善はわずかだった。二つのオプションがどちらもチェックアウトのときにしか働かないことに、そこでようやく気づいた。チェックアウトを経ずに再利用されるコネクションがあるなら、どちらのオプションも手を出せない。原因へ向かう最初の手がかりはそこにあった。

あとで確かめると、`pool_pre_ping`そのものは壊れていなかった。チェックアウトが起きるコネクションではきちんと働いていて、エラーが出たのは、セッションがすでに握っていたコネクションを新たなチェックアウトなしに使い回した場合だった。あの火曜日に頻度が十倍に跳ねた理由は、最後まで確かめられなかった。

## 原因: セッションがリクエストではなくスレッドに紐づく

先に用語を三つ整理しておく。コネクションは、アプリとDBのあいだに実際に開いた接続だ。トランザクションは一緒に確定するか一緒に取り消すDB処理のまとまりで、この構成では参照するだけでも始まる。SQLAlchemyのセッションは、ORMオブジェクトとトランザクションを管理するオブジェクトだ。この記事のようにエンジン一つに紐づいたセッションは、DBに初めてアクセスするときにプールからコネクションをチェックアウトして握り、トランザクションをコミットかロールバックで終えるか、セッションを閉じるまで、そのコネクションをプールに返さない。だから参照だけのGETリクエストでも、レスポンスは出たのにコネクションは握ったまま終わることがある。

記事によく出てくる言葉も二つ押さえておく。スレッドはプログラムの中で仕事を一つずつ受け持って実行する単位で、ミドルウェアはすべてのリクエストの前後で共通に走るコードだ。

### `scoped_session`のデフォルトのスコープはスレッドだ

`scoped_session`は、セッションをいちいち渡さなくても、どこで`session_factory()`を呼んでも「いまこの場所」のセッションを返してくれる道具だ。デフォルト設定では、この場所はスレッドになる。スレッドごとにロッカーを一つずつ割り当て、最初に呼ばれたらセッションを作って入れておき、それ以降は同じロッカーから同じセッションを出してくれる、と考えればいい。内部では`threading.local()`ベースの`ThreadLocalRegistry`がこのロッカーの役を担う。

このデフォルトはWSGIの構成によく合う。WSGIは、Flask、Django、Pyramidのような伝統的なPythonのWebフレームワークがWebサーバーと噛み合うための規約だ。伝統的な同期構成ではリクエスト一つが最初から最後までワーカースレッド一つの中で処理されるので、リクエストが終わるときにそのスレッドでセッションを片付ければよい。Flask-SQLAlchemyもリクエストの終わりに`remove()`を呼ぶが、3.xからはスレッドの代わりにアプリコンテキストをスコープに使う。SQLAlchemyのドキュメントもこの前提を書いている。

> _"most Python web frameworks, with notable exceptions such as the asynchronous frameworks Twisted and Tornado, use threads in a simple way, such that a particular web request is received, processed, and completed within the scope of a single worker thread."_
>
> 「TwistedやTornadoのような非同期フレームワークを除けば、ほとんどのPython Webフレームワークはスレッドを単純に使う。特定のWebリクエストは単一のワーカースレッドの範囲内で受信、処理、完了される。」
>
> 出典: [SQLAlchemyドキュメント、Webアプリケーションでスレッドローカルスコープを使う](https://docs.sqlalchemy.org/en/20/orm/contextual.html#using-thread-local-scope-with-web-applications)

スレッドがそのままリクエストなら、セッションをスレッドに紐づけることはリクエストに紐づけることと同じだ。WSGIでは、リクエストの終わりに`remove()`さえ呼べばデフォルト設定で足りていた。

### ASGIではリクエスト一つが複数のスレッドを渡り歩く

FastAPIとStarletteはASGIの上で動く。ASGIは非同期のWebサーバーとアプリが噛み合うための規約で、イベントループという窓口一つが複数のリクエストに代わる代わる応対し、時間のかかる同期処理は裏手のワーカースレッドに回す。ミドルウェアと`async def`エンドポイントはイベントループのスレッドで走り、`def`で宣言した同期エンドポイントと同期の依存関係はanyioが管理するワーカースレッドプールに回される。リクエストの前後はイベントループのスレッドにあり、セッションを作る同期エンドポイントはワーカースレッドにある。同期の依存関係と同期エンドポイントが同じワーカースレッドに割り当てられる保証もない。

以下のワーカーの再利用と片付けは、FastAPIがデフォルトで使うanyioのasyncioバックエンドを4.15.1のソースで確かめた内容だ。ワーカースレッドは、anyioの`CapacityLimiter`が同時実行をデフォルト40に制限したプールから出てきて、同じスレッドが複数のリクエストを順に受け持つ。anyioは新しい呼び出しが来ると、いちばん最近休みに入ったワーカーから再利用し、10秒以上休んだ残りのワーカーはそのときに片付ける。`threading.local()`の格納領域はスレッドが終わらないとガベージコレクションされないので、ワーカーが生きているあいだはそのスレッドのセッションも残る。

夜通し呼び出しがなければ片付けるきっかけが来ないので、朝一番のリクエストは最後に働いていたワーカーに回る。プロセスが完全に空にならなくても同じことが起きる。DBを使わない`def`のヘルスチェックがそのワーカーを使い回し続ければ、ワーカーは生きたままで、セッションが握るコネクションだけが休む。

事故当時どのワーカーがどれだけ休んでいたかは、ログがなく確かめられなかった。この経路はanyioの実装を読んで再構成したものだ。

![同期エンドポイントはワーカースレッドで動く。セッションはそのスレッドにくっつき、コミットせずに終わったリクエストのセッションはコネクションを握ったまま残る](https://blog.namsang.co/diagrams/fastapi-scoped-session-broken-pipe/ja-1-thread-bound.svg)

片付けを忘れると、セッションを使ったワーカースレッドごとにセッションが一つずつ残りうる。エラーが毎日出なかったのもこの構造のせいだと見ている。空白のあと最初のリクエストを受けるワーカーで、直前のリクエストがトランザクションを開いたまま終わっていたときだけ、切れたコネクションを使い回すことになる。コミットで終わっていればコネクションはすでにプールに戻っていて、次のDBアクセスでチェックアウトを通り、そのとき`pool_recycle`や`pool_pre_ping`が古いコネクションを弾く。

つまり切れるのはワーカースレッドのセッションが握るコネクションで、8時間を数える起点も、エンドポイントの最後のリクエストではなく、そのコネクションが最後に使われた時刻だ。

同じ構造が、接続エラー以外に残しうる症状がもう一つある。ワーカーの次のリクエストが前のリクエストが開いたトランザクションの中で続けて走ると、InnoDBのデフォルト分離レベルであるREPEATABLE READでは、通常のSELECTが最初に読んだ時点のスナップショットを使い続け、そのあいだにコミットされた変更が見えないことがある。ときどき古いデータを読んでいたなら、同じ原因を疑ってみる価値がある。

### `Broken Pipe`までの流れ

私たちのコードは、リクエストが終わるときに`close()`を呼んでいなかった。セッションはトランザクションを開いたままコネクションを握り、ワーカースレッドに残った。

![コミットせずに終わったリクエストのセッションが夜通しコネクションを握り、サーバーが閉じたそのコネクションで朝一番のクエリを送る](https://blog.namsang.co/diagrams/fastapi-scoped-session-broken-pipe/ja-2-timeline.svg)

セッションがコネクションを直接握っているので、プールからのチェックアウトは起きない。チェックアウトのときにしか働かない`pool_pre_ping`には、割り込む隙がない。

図の最後にある二つの番号は、この流れでは同じ原因で、切れたソケットがどういう状態にあるかで分かれる。FastAPIとMySQL 8.4、PyMySQLで再現してみると、サーバーが接続を閉じてまもなく送ったクエリは、書き込みには成功し、応答を読む段階で2013 `Lost connection to MySQL server during query`になった。

私たちが見た2006 `BrokenPipeError`は、TCP keepaliveを経由した経路で説明できる。keepaliveは長く静かな接続にOSが送る生存確認の信号で、PyMySQLはこれを有効にしている。サーバーがすでに閉じた接続をクライアントが握ったままだと、この確認にサーバー側が「そんな接続はない」という接続リセットで応えることがある。そのあとでクエリを書き込むと書き込み自体が失敗し、2006になる。Linuxでリクエストのあいだを80秒空けて比べると、keepaliveの待ち時間がデフォルトの7200秒のときは2013が、5秒に縮めたときはぴったり2006が出た。

Linuxのデフォルトの待ち時間は、サーバーが閉じるときに送った最後のパケットからさらに2時間なので、上の図のように6時に閉じられて9時に最初のリクエストが来たのなら、この経路に当てはまる。運用での実際のアイドル時間とkeepaliveの設定は確かめていない。

リトライで消えた理由は確かめていない。接続エラーが起きるとSQLAlchemyはそのコネクションを無効化し、同じセッションで再びDBにアクセスするには、`rollback()`か`close()`で前のトランザクションを先に片付ける必要がある。リトライが同じセッションに行ったのなら例外処理のどこかでその片付けが起きたはずで、別のワーカーのセッションに行った可能性もある。片付けずに同じセッションで再アクセスすると`PendingRollbackError`が出るので、ログで一緒に探してみるとよい。

### ミドルウェアの`remove()`はワーカースレッドに届かない

リクエストが終わるときにミドルウェアでセッションを片付ければよさそうに見える。こんなコードだ。

```python
class CleanupMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        try:
            return await call_next(request)
        finally:
            database.session_factory.remove()  # イベントループのスレッドのセッションだけが対象になる
```

このコードはワーカースレッドのセッションを消せない。デフォルトのレジストリはスレッドごとなので、`remove()`は呼んだスレッドのセッションしか見ない。ミドルウェアはイベントループのスレッドで走り、セッションはワーカースレッドにあるので、`remove()`は何も消さずに戻ってくる。同じ理由で、ミドルウェアから呼ぶ`close()`もワーカースレッドのセッションには届かない。この記事を直すにあたって、SQLAlchemy 2.1.5とワーカースレッド一本のプールで改めて確かめた。再現ではメインスレッドがイベントループのスレッドの役で、スレッドごとのレジストリの挙動だけを見たものだ。

```
default registry: ThreadLocalRegistry
after remove() from main thread, worker still has session: True
after remove() from worker thread, worker gets new session: True
```

FastAPIとMySQL 8.4でリクエスト経路全体も別に再現したが、ミドルウェアで`remove()`を呼んでもチェックアウトされたコネクションは1のまま残り、次のリクエストは先の再現と同じ2013で失敗した。

`remove()`は、セッションを作ったワーカースレッドで呼ばないと効かない。どのワーカーがこのリクエストを受け持つかはanyioが決め、リクエスト側からは選べない。

### `close()`と`remove()`は別の仕事をする

> _"The `scoped_session.remove()` method first calls `Session.close()` on the current Session, which has the effect of releasing any connection/transactional resources owned by the Session first, then discarding the Session itself."_
>
> 「`scoped_session.remove()`メソッドは、まず現在のSessionに対して`Session.close()`を呼び出してコネクションとトランザクションのリソースを解放し、そのあとSession自体を破棄する。」
>
> 出典: [SQLAlchemyドキュメント、コンテキストセッション](https://docs.sqlalchemy.org/en/20/orm/contextual.html)

- `close()`はORMオブジェクトとトランザクションの状態を片付け、コネクションをプールに返す。セッションオブジェクトはレジストリに残り、次の呼び出しでは同じオブジェクトが返る。
- `remove()`は`close()`を実行したあと、レジストリからセッションを消す。次の呼び出しでは新しいセッションが作られる。

エンドポイントやRepositoryの中で、つまりセッションを作ったワーカースレッドで`close()`を毎回呼んでいれば、コネクションはプールに戻り、次のリクエストはチェックアウトを通っていたはずだ。そうなれば`pool_recycle`と`pool_pre_ping`が働けるので、リクエストをまたいで握ったコネクションを検査なしで使い回す経路はふさがれていた。`scoped_session`を使えばセッションを引数で渡さなくてもいまのスコープのセッションが手に入るが、リクエストが終わるときの片付けは自分でやらなければならない。私たちのコードはその片付けを漏らしていた。

## 解決: `scopefunc`を`ContextVar`に替える

### アイデア

`scoped_session`に自作の`scopefunc`を渡すと、`ThreadLocalRegistry`の代わりに`ScopedRegistry`を使う。このレジストリはキーとセッションを対にして持つ対応表、つまりただの辞書だ。

```python
# SQLAlchemy ScopedRegistry 内部 (簡略化)
class ScopedRegistry:
    def __init__(self, createfunc, scopefunc):
        self.createfunc = createfunc
        self.scopefunc = scopefunc
        self.registry = {}

    def __call__(self):
        key = self.scopefunc()  # このキーが何であるかがすべて
        if key not in self.registry:
            self.registry[key] = self.createfunc()
        return self.registry[key]
```

キーをリクエストごとに固有の値にすれば、同じスレッドで実行されても、リクエストごとに別のセッションを使う。リクエストごとに番号札を発行し、その番号でセッションを探すようなものだ。

![変わるのはセッションを引くキー一つだけだ。キーがリクエストIDになれば、ミドルウェアのremove()がリクエストごとにそのセッションを見つけて片付けられる](https://blog.namsang.co/diagrams/fastapi-scoped-session-broken-pipe/ja-3-scope-key.svg)

リクエストIDは`ContextVar`に入れる。`ContextVar`は実行の流れごとに別々の値を持つPython標準ライブラリのコンテキスト変数で、リクエストごとに付ける番号札の役を果たす。ミドルウェアで付けた番号札は、ワーカースレッドまでついていく。

### 実装はファイル3つ、要の変更は1行

新しいモジュールと新しいミドルウェアを一つずつ作り、既存の`Database`クラスに`scopefunc`を1行足した。

#### `ContextVar`モジュール (新規)

```python
# context_session.py
from contextvars import ContextVar, Token

_session_context: ContextVar[str] = ContextVar("session_context")

def set_session_context(context_id: str) -> Token:
    return _session_context.set(context_id)

def get_session_context() -> str:
    return _session_context.get()

def reset_session_context(token: Token) -> None:
    _session_context.reset(token)
```

#### `Database`クラス (1行変更)

```python
class Database:
    def __init__(self, engine: Engine) -> None:
        self.session_factory = orm.scoped_session(
            orm.sessionmaker(autocommit=False, autoflush=False, bind=engine),
            scopefunc=get_session_context  # ← この1行が要だ
        )
```

#### ミドルウェア (新規)

```python
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from uuid import uuid4

class ContextSessionMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        context_id = str(uuid4())
        token = set_session_context(context_id)
        try:
            response = await call_next(request)
            return response
        finally:
            # remove() → reset の順序が重要:
            # remove()は内部でscopefunc()を呼び、現在のコンテキストのセッションを探す。
            # resetが先に実行されるとscopefunc()がLookupErrorを出すか別の値を返し、
            # このリクエストのセッションが片付けられない。
            database.session_factory.remove()  # 現在のコンテキストのセッションを探して削除
            reset_session_context(token)        # そのあとコンテキストをリセット

app.add_middleware(ContextSessionMiddleware)
```

運用に入れたコードは上のとおりだ。いま新しく書くなら、`remove()`が例外を出してもコンテキストが元に戻るよう、`finally`をもう一段重ねる。

```python
        finally:
            try:
                database.session_factory.remove()
            finally:
                reset_session_context(token)
```

### なぜエラーが消えるのか

先の`CleanupMiddleware`と形は同じだが、今度は`remove()`が効く。セッションを引くキーがリクエストIDになったので、イベントループのスレッドから呼んでもこのリクエストのセッションが見つかる。

![修正後はリクエストごとにプールからコネクションを借りるのでpool_pre_pingが再び検査し、リクエストの終わりのremove()がコネクションを返す](https://blog.namsang.co/diagrams/fastapi-scoped-session-broken-pipe/ja-4-fixed-flow.svg)

`remove()`はまず`close()`を呼ぶので、リクエストが終わるたびにコネクションがプールに戻る。次のリクエストは新しいセッションで始まり、DBに初めてアクセスするときにチェックアウトを通る。その瞬間、`pool_recycle=3600`は作られてから1時間を過ぎたコネクションを新しいものに替え、`pool_pre_ping`は再利用するコネクションが切れていないかを検査する。今回の修正は、二つのオプションに働く機会を取り戻させたものだ。

同じ再現環境で`wait_timeout`を5秒にし、リクエストのあいだを8秒空けて修正前後を比べた。既存のコードと、ミドルウェアで`remove()`だけを呼ぶコードは、リクエストが終わったあともチェックアウトされたコネクションが1つ残り、次の参照が2013で失敗した。`scopefunc`を替えたコードはリクエストが終わるたびにチェックアウトが0に戻り、同じ空白のあとの参照は`pool_pre_ping`が切れたコネクションを新しいものに替えて成功した。同時実行の上限をデフォルトの40にしても結果は同じだった。

この方式が動くには前提が一つある。Starletteの`run_in_threadpool`はanyioの`to_thread.run_sync`を呼び、anyioは呼び出し時点のコンテキストをコピーしてワーカースレッドで実行する。リクエストに付けた番号札を、ワーカーに仕事を渡すときに一緒に手渡すようなもので、ミドルウェアで入れたリクエストIDがエンドポイントの中の`session_factory()`まで同じ値で届く。

デプロイの翌日から`MySQL server has gone away`は出なくなり、一か月が過ぎたいままで一度も出ていない。ただ、もともと月に一、二度だったエラーなので、一か月は結論を出すのに長い期間とは言えず、引き続き様子を見ている。負荷試験は別にしておらず、運用ダッシュボードのメモリ使用量と応答時間のP99にも、デプロイ前後の差は見えなかった。

## 依存性注入パターンに切り替えなかった理由

FastAPIの公式ドキュメントが勧めるのは、`yield`依存関係でリクエストごとにセッションを開いて閉じるパターンだ。下は以前のSQLチュートリアルに載っていた`Depends(get_db)`の例で、いまのドキュメントはSQLModelのセッションで同じことをしている。

```python
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/users")
def get_users(db: Session = Depends(get_db)):
    return db.query(User).all()
```

セッションのライフサイクルがコードに表れ、リクエストごとに別のセッションを使う。ただ、コードベースが依存性注入コンテナで`scoped_session`をグローバルに注入する構造だと、切り替えのコストが大きい。すべての`UseCase`と`Repository`のセッションの渡し方を変え、テストコードも直さなければならない。セッションを使う箇所ごとに`close()`を入れる手もあるが、セッションをあちこちで取り出して使う構造なので、一か所漏らすだけで同じことが起きる。

この方式にも報告された問題があった。[FastAPI Discussion #6628](https://github.com/fastapi/fastapi/discussions/6628)では、スレッドプールがコネクション待ちで埋まり、依存関係の`finally`ブロックが実行されるスレッドを得られずにアプリ全体が止まった。メンテナーは、2022年9月の0.82.0での修正でおそらく解決したと答えている。

`scopefunc`の差し替えは既存の構造をそのままに、問題になる箇所だけを直す。SQLAlchemyのドキュメントは、できれば`scoped_session`の代わりにWebフレームワークの統合ツールを使うよう勧め、そうでない場合でもセッションは現在のスレッドよりリクエストと直接結びつけるほうがよいと付け加えている。この修正は後者の勧めに従った折衷案だ。

## ほかの実装はセッションを何に紐づけるか

調べてみると、ほかの実装も同じやり方を使っている。違いはセッションを紐づける単位にある。

| 実装                                   | スコープの決め方                                    | 紐づける単位       |
| -------------------------------------- | --------------------------------------------------- | ------------------ |
| `scoped_session`のデフォルト           | `threading.local()`                                 | スレッド           |
| Flask-SQLAlchemy                       | Flaskのアプリコンテキスト                           | アプリコンテキスト |
| `async_scoped_session`のドキュメント例 | `current_task()`                                    | 非同期タスク       |
| Galaxy Project                         | `ContextVar`、フォールバックは`threading.get_ident` | リクエスト         |
| この記事の解決策                       | `ContextVar.get()`                                  | リクエスト         |

大規模なバイオインフォマティクス基盤であるGalaxy Projectは、FastAPIの上で[同じパターン](https://github.com/galaxyproject/galaxy/blob/7761eeb325600358ada3025cb32549b4fa06f427/lib/galaxy/model/base.py#L100-L107)を使っている。リクエストIDがなければスレッドIDにフォールバックする。

## 適用時の注意点

### 1. `remove()`を漏らすとメモリがリークする

スレッドごとのレジストリでは、セッションはワーカースレッドの数までしかできなかった。リクエストIDをキーにするとリクエストの数だけできうるので、`remove()`が漏れると`ScopedRegistry`の辞書が際限なく膨らむ。ミドルウェアの`finally`で必ず呼ばなければならない。

### 2. リクエストIDの設定と片付けはミドルウェア一か所で行う

この例の`BaseHTTPMiddleware`は下位のアプリを別タスクで実行し、同期の依存関係に回るときはanyioがコンテキストをコピーする。そのため、ミドルウェアで入れた値は依存関係まで下りていくが、依存関係で入れたり変えたりした値はミドルウェアまで上がってこない。

[dependency-injector Discussion #493](https://github.com/ets-labs/python-dependency-injector/discussions/493)にも、ミドルウェアと依存関係が同じセッションを共有できなかった例があり、投稿者はコンテキストが分かれたせいだと推測している。

[FastAPI Discussion #8017](https://github.com/fastapi/fastapi/discussions/8017)には、リクエストIDはミドルウェアで設定し、セッションの片付けは依存関係で行うコードが投稿された。これに対し、ミドルウェアがコンテキスト変数を先にリセットするので、依存関係で呼んだ`remove()`がこのリクエストのセッションを見つけられず、結局プールが枯渇するという指摘が続いた。当時の報告なので、いまの構成にそのまま当てはめることはできない。Pythonのタスクとanyioのワーカーはコンテキストをコピーしていくので、親があとでリセットしても、コピーされたリクエストIDはそのままだ。私たちが`ContextVar`の設定と`remove()`をミドルウェア一か所に置いたのは、入れて消す順序をひと目で管理するためだ。

### 3. `BaseHTTPMiddleware`はレスポンスヘッダーが出た時点で片付ける

`call_next()`はレスポンスヘッダーが出た時点で戻るので、例の`finally`はストリーミングの本文やバックグラウンド処理が終わる前に実行されうる。本文を作ったり後続の処理をしたりするあいだも同じセッションを使うなら、片付けのタイミングを別に設計する必要がある。`yield`依存関係の片付けコードもレスポンスのあとに走りうるので、その中でセッションを使っていないか確かめる。

`remove()`は同期呼び出しなので、セッションがコネクションを握っていた場合、返却時に送るロールバックが終わるまでイベントループを止める。運用でストリーミングレスポンスが必要なら、[純粋なASGIミドルウェア](https://starlette.dev/middleware/#pure-asgi-middleware)に移して、レスポンス本文が終わったあとに片付けることを検討するとよい。

### 4. `pool_pre_ping`と`pool_recycle`はそのまま残す

この二つは今回の問題を単独では防げなかったが、修正後はリクエストごとにチェックアウトが起きるので、プールの中で切れたコネクションを弾く役を再び担う。外すと、プールの切れたコネクションをそのまま使い回して、最初のDB処理で接続エラーが起きうる。どちらもチェックアウト時点の検査なので、トランザクションの途中で切れる接続までは防げない。

### 5. リクエストの外でセッションを使うコードは別に面倒を見る

例の`ContextVar`にはデフォルト値がない。ミドルウェアを通らないアプリ起動フック、バッチ、CLI、テストで`session_factory()`を呼ぶと`LookupError`が出る。そうした経路では`set_session_context()`でIDを直接入れて最後に`remove()`と`reset_session_context()`を呼ぶか、GalaxyのようにスレッドIDにフォールバックさせる。フォールバック経路はまたスレッド単位になるので、その経路の片付けも自分で行う必要がある。

### 6. 一つのリクエストの中でセッションを同時に使わない

スレッドスコープではワーカースレッドごとにセッションが違ったが、リクエストIDをキーにすると、一つのリクエストの中のコードはすべて同じセッションを受け取る。一つのリクエストで複数のDB処理をワーカースレッドに同時に送ると、二つのスレッドが一つのセッションを一緒に使うことになる。anyioで再現すると、ワーカースレッド二つが同じセッションオブジェクトを受け取った。[SQLAlchemyのセッションは同時使用に対して安全ではないので](https://docs.sqlalchemy.org/en/20/orm/session_basics.html#is-the-session-thread-safe-is-asyncsession-safe-to-share-in-concurrent-tasks)、そうした処理ではセッションを別に開いて使う。

## 適用前後の比較

| 項目                               | 既存: デフォルトスコープ、片付け漏れ         | `scopefunc`と`ContextVar`                                    |
| ---------------------------------- | -------------------------------------------- | ------------------------------------------------------------ |
| セッションの分離単位               | スレッド                                     | リクエスト                                                   |
| リクエスト終わりの片付け`remove()` | セッションを作ったスレッドでのみ有効         | イベントループのスレッドでも有効                             |
| 次のリクエストのチェックアウト     | セッションが握るコネクションを再利用して省略 | 最初のDBアクセス時にチェックアウト、プールのオプションが働く |
| `Broken Pipe`のリスク              | あり、アイドルスレッド経路                   | アイドルスレッド経路を除去                                   |
| 一つのリクエスト内の同時DB処理     | スレッドごとに別のセッション                 | 同じセッションを共有、別に開く必要あり                       |
| 追加したコード                     | なし                                         | モジュール1つ、ミドルウェア1つ、1行変更                      |
| `UseCase`と`Repository`の修正      | 不要                                         | 不要                                                         |

直接の原因は、リクエストが終わってもセッションを片付けなかったことにあった。ミドルウェアに`remove()`を入れるだけでは直せなかったのは、「スレッド一つがリクエスト一つ」というWSGI時代の前提を、ASGIの環境にそのまま持ち込んでいたからだ。要の変更は1行だったが、その1行を入れる場所を見つけるまでに数日かかった。

FastAPIで`scoped_session`を使っていて、明け方に`errno 32`や2013のエラーをときどき見かけるなら、`pool_pre_ping`を足す前に、セッションが何に紐づいているかをまず確かめてみてほしい。`scopefunc`を渡していなければ、答えはスレッドだ。

## 参考資料

- [SQLAlchemy 2.0ドキュメント、コンテキストセッション](https://docs.sqlalchemy.org/en/20/orm/contextual.html): `scoped_session`の設計前提、`close()`と`remove()`の違い、カスタム`scopefunc`を説明している。
- [MySQLドキュメント、`wait_timeout`](https://dev.mysql.com/doc/refman/8.4/en/server-system-variables.html#sysvar_wait_timeout): コネクションの確立時にグローバル値でセッション値を初期化すると説明している。
- [fastapi/fastapi Discussion #8017](https://github.com/fastapi/fastapi/discussions/8017): `scoped_session`、依存性注入、ミドルウェアの方式を比べ、`ContextVar`と`scopefunc`で解決した例を共有している。
- [fastapi/fastapi Discussion #6628](https://github.com/fastapi/fastapi/discussions/6628): 依存性注入方式のセッションで、スレッドプールが詰まって起きたデッドロックの報告だ。
- [sqlalchemy/sqlalchemy Discussion #8891](https://github.com/sqlalchemy/sqlalchemy/discussions/8891): FastAPIが絡む並行性の問題が増えたというzzzeekのコメントがある。
- [ets-labs/python-dependency-injector Discussion #493](https://github.com/ets-labs/python-dependency-injector/discussions/493): FastAPIのミドルウェアと依存関係が同じセッションを共有できなかったという例だ。

---

*広告やCRMからAIエージェントまで、大規模なプラットフォームを作り、率いてきた経験を [Namsang Labs 技術ブログ](https://blog.namsang.co/ja/?utm_source=zenn&utm_medium=crosspost) に書いています。*
