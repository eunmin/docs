Duct Framework 가이드
===========================

머리말
~~~~~~~

Duct는 Clojure_ 프로그래밍 언어로 서버 애플리케이션을 만들기 위한 data-driven 프레임워크입니다.
이 가이드는 웹 애플리케이션 중점을 두고 Duct 사용법을 자세히 설명하기 위해 섰습니다.

또 이 가이드는 Leiningen_\이 설치되어 있고 Clojure 실무 지식이 있다는 가정을 하고 썼습니다.
꼭 필요한 것은 아니지만 Ring_\의 기초적인 부분을 알고 있으면 좋습니다.

.. _Clojure:   https://clojure.org/
.. _Leiningen: https://leiningen.org/
.. _Ring:      https://github.com/ring-clojure/ring


시작하기
~~~~~~~~~~~~~~~

Duct Leiningen 템플릿으로 바로 시작해 볼 수 있습니다. Duct로 다양한 서버 애플리케이션을 만들 수 있지만
이 가이드의 목적을 위해 SQLite_ 데이터베이스를 기반으로 하는 웹 서비스를 만들어 보겠습니다.

프로젝트 만들기
""""""""""""""""""""

쉘 에서 다음과 같이 실행합니다::

  $ lein new duct todo +api +ataraxy +sqlite

이런 결과가 나옵니다::

  Generating a new Duct project named todo...
  Run 'lein duct setup' in the project directory to create local config files.

``+``\로 시작하는 파라미터는 프로필 힌트로 웹 서비스 (``+api``)와 Ataraxy_ 라우팅 라이브러리
(``+ataraxy``), SQLite 데이터베이스 (``+sqlite``)를 사용하는 프로젝트를 만들어 줍니다.

사용할 수 있는 프로필 힌트를 모두 보려면 아래 명령어를 실행합니다::

  $ lein new :show duct

이제 조금 전에 만든 ``todo`` 프로젝트 디렉터리로 들어가 봅시다::

  $ cd todo

그리고 로컬 셑업을 실행합니다::

  $ lein duct setup

그러면 소스 컨트롤에는 제외되어 있는 파일 4개가 생깁니다::

  Created profiles.clj
  Created .dir-locals.el
  Created dev/resources/local.edn
  Created dev/src/local.clj

Git_\을 사용하면 이 파일들은 ``.gitignore``\에 추가되어 있습니다. 하지만 다른 소스 컨트롤을 사용한다면
소스 컨트롤에 관리되지 않도록 수동으로 처리해줘야 합니다.

.. _SQLite:  https://sqlite.org/
.. _Ataraxy: https://github.com/weavejester/ataraxy
.. _Git:     https://git-scm.com/


REPL 시작하기
"""""""""""""""""

Duct는 REPL을 중심으로 개발 하도록 되어 있습니다. 그래서 Cursive_\나 Emacs_\의 CIDER_, Vim_\의
`fireplace.vim`_, Atom_\의 `Proto REPL`_\같은 에디터 REPL 통합 환경을 사용하는 것을 추천합니다.
하지만 이 가이드에서는 에디터 통합 없이 커맨드 라인에서 직접 실행해볼 수 있도록 되어 있습니다.

REPL을 시작합니다::

  $ lein repl

먼저 개발 환경을 로드하기 위해 프롬프트에 다음과 같이 입력합니다:

.. code-block:: clojure

  user=> (dev)
  :loaded
  dev=>

개발 환경에 에러가 있는 경우 REPL이 실행되지 않을 수 있기 때문에 개발 환경은 자동으로 로드하지 않습니다.

``dev`` 네임스페이스에 들어오면 애플리케이션을 실행해볼 수 있습니다:

.. code-block:: clojure

  dev=> (go)
  :duct.server.http.jetty/starting-server {:port 3000}
  :initiated

웹 서버는 3000번 포트로 실행됩니다. HTTP 리퀘스트를 보내 잘 실행되었는지 확인해봅시다.
보통 커맨드 라인에서 curl_\이나 wget_\으로 웹 서비스를 테스트 하지만 저는 HTTPie_\를 더 좋아합니다::

  $ http :3000
  HTTP/1.1 404 Not Found
  Content-Length: 21
  Content-Type: application/json; charset=utf-8
  Date: Wed, 06 Dec 2017 11:27:22 GMT
  Server: Jetty(9.2.21.v20170120)

  {
      "error": "not-found"
  }

"not found" 응답을 받았습니다. 하지만 아직 라우터를 추가하지 않았기 때문에 예상된 결과입니다.

.. _Cursive:       https://cursive-ide.com/
.. _Emacs:         https://www.gnu.org/software/emacs/
.. _CIDER:         https://github.com/clojure-emacs/cider
.. _Vim:           http://www.vim.org/
.. _fireplace.vim: https://github.com/tpope/vim-fireplace
.. _Atom:          https://atom.io/
.. _Proto Repl:    https://atom.io/packages/proto-repl
.. _curl:          https://curl.haxx.se/
.. _wget:          https://www.gnu.org/software/wget/
.. _HTTPie:        https://httpie.org/


Configuration
~~~~~~~~~~~~~

Duct applications are built around an edn_ configuration file. This
defines the structure and dependencies of the application. In the
project we're writing in this guide, the configuration is located at:
``resources/todo/config.edn``.


Adding a Static Route
"""""""""""""""""""""

Let's take a look at the configuration file:

.. code-block:: edn

  {:duct.core/project-ns  todo
   :duct.core/environment :production

   :duct.module/logging {}
   :duct.module.web/api {}
   :duct.module/sql {}

   :duct.module/ataraxy
   {}}

We're going to start by adding in a static index route, and to do that
we're going to add to the ``:duct.module/ataraxy`` key, since Ataraxy
is our router:

.. code-block:: edn

  :duct.module/ataraxy
  {[:get "/"] [:index]}

This connects a route ``[:get "/"]`` with a result ``[:index]``. The
Ataraxy module automatically looks for a Ring handler in the
configuration with a matching name to pair with the result. Since the
result key is ``:index``, the handler key is ``:todo.handler/index``.
Let's add in a configuration entry with that name:

.. code-block:: edn

  [:duct.handler.static/ok :todo.handler/index]
  {:body {:entries "/entries"}}

This time we're using a vector as the key; in Duct parlance, this is
known as a *composite key*. Composite keys inherit the properties of
all the keywords contained in them; because the vector contains the
key ``:duct.handler.static/ok``, the configuration entry produces a
static handler.

Let's apply this change to the application. Go to back to the REPL and
run:

.. code-block:: clojure

  dev=> (reset)
  :reloading (todo.main dev user)
  :resumed

This reloads the configuration and any changed files. When we send a
HTTP request to the web server, we now get the expected response::

  $ http :3000
  HTTP/1.1 200 OK
  Content-Length: 22
  Content-Type: application/json; charset=utf-8
  Date: Wed, 06 Dec 2017 13:28:52 GMT
  Server: Jetty(9.2.21.v20170120)

  {
      "entries": "/entries"
  }

.. _edn: https://github.com/edn-format/edn

Adding a Database Migration
"""""""""""""""""""""""""""

We want to begin adding more dynamic routes, but before we can we need
to create our database schema. Duct uses Ragtime_ for migrations, and
each migration is defined in the configuration.

Add two more keys to the configuration:

.. code-block:: edn

  :duct.migrator/ragtime
  {:migrations [#ig/ref :todo.migration/create-entries]}

  [:duct.migrator.ragtime/sql :todo.migration/create-entries]
  {:up ["CREATE TABLE entries (id INTEGER PRIMARY KEY, content TEXT)"]
   :down ["DROP TABLE entries"]}

The ``:duct.migrator/ragtime`` key contains an ordered list of
migrations. Individual migrations can be defined by including
``:duct.migrator.ragtime/sql`` in a composite key. The ``:up`` and
``:down`` options contains vectors of SQL to execute; the former to
apply the migration, the latter to roll it back.

To apply the migration we run ``reset`` again at the REPL:

.. code-block:: clojure

  dev=> (reset)
  :reloading ()
  :duct.migrator.ragtime/applying :todo.migration/create-entries#b34248fc
  :resumed

Suppose after applying the migration we change our mind about the
schema. We could write another migration, but if we haven't committed
the code or deployed it to production it's often more convenient to
edit the migration we have.

Let's change the migration and rename the ``content`` column to
``description``:

.. code-block:: edn

  [:duct.migrator.ragtime/sql :todo.migration/create-entries]
  {:up ["CREATE TABLE entries (id INTEGER PRIMARY KEY, description TEXT)"]
   :down ["DROP TABLE entries"]}

Then ``reset``:

.. code-block:: clojure

  dev=> (reset)
  :reloading ()
  :duct.migrator.ragtime/rolling-back :todo.migration/create-entries#b34248fc
  :duct.migrator.ragtime/applying :todo.migration/create-entries#5c2bb12a
  :resumed

The old version of the migration is automatically rolled back, and the
new version of the migration applied in its place.

.. _Ragtime: https://github.com/weavejester/ragtime

Running Database Migrations in Production
"""""""""""""""""""""""""""""""""""""""""

We can easily run migrations in production::

  $ lein run :duct/migrator

If you are using Heroku for deployment, this can easily be added to the release phase via your Procfile::

  web: java -jar target/sstandalone.jar
  release: lein run :duct/migrator

Adding a Query Route
""""""""""""""""""""

Now that we have a database table, it's time to write some routes to
query it. To do this, we're going to use a library called
``duct/handler.sql``, which should be added to the ``:dependencies``
key in your ``project.clj`` file:

.. code-block:: clojure

  [duct/handler.sql "0.3.1"]

Your dependencies should now look something like:

.. code-block:: clojure

  :dependencies [[org.clojure/clojure "1.9.0-RC1"]
                 [duct/core "0.6.1"]
                 [duct/handler.sql "0.3.1"]
                 [duct/module.logging "0.3.1"]
                 [duct/module.web "0.6.3"]
                 [duct/module.ataraxy "0.2.0"]
                 [duct/module.sql "0.4.2"]
                 [org.xerial/sqlite-jdbc "3.20.1"]]

Adding dependencies is one of the few times we have to restart the
REPL. So first we exit:

.. code-block:: clojure

  dev=> (exit)
  Bye for now!

Then we restart::

  $ lein repl

And start the application running again:

.. code-block:: clojure
  user=> (dev)
  :loaded
  dev=> (go)
  :duct.server.http.jetty/starting-server {:port 3000}
  :initiated

We can now turn back to the project configuration. Let's start by
adding a new Ataraxy route:

.. code-block:: edn

  :duct.module/ataraxy
  {[:get "/"]        [:index]
   [:get "/entries"] [:entries/list]}

As before, the result ``[:entries/list]`` needs to be paired with an
appropriately named Ring handler. The Ataraxy module expects this
handler to be named ``:todo.handler.entries/list``, so we'll use that
name, along with the ``:duct.handler.sql/query`` key:

.. code-block:: edn

  [:duct.handler.sql/query :todo.handler.entries/list]
  {:sql ["SELECT * FROM entries"]}

Once the handler is defined in the configuration, we can ``reset``:

.. code-block:: clojure

  dev=> (reset)
  :reloading (todo.main dev user)
  :resumed

Then we check the route by sending a HTTP request to it::

  $ http :3000/entries
  HTTP/1.1 200 OK
  Content-Length: 2
  Content-Type: application/json; charset=utf-8
  Date: Thu, 07 Dec 2017 10:13:34 GMT
  Server: Jetty(9.2.21.v20170120)

  []

We get a valid, though empty response. This makes sense, as we've yet
to populate the ``entries`` table with any data.


Adding an Update Route
""""""""""""""""""""""

Next we'd like to add a route that updates the database. Again we're
going to be making use of the ``duct/handler.sql`` library, but both
the route and handler are going to be more complex.

First, the new route:

.. code-block:: edn

  :duct.module/ataraxy
  {[:get "/"]        [:index]
   [:get "/entries"] [:entries/list]

   [:post "/entries" {{:keys [description]} :body-params}]
   [:entries/create description]}

The new Ataraxy route not only matches the method and URI of the
request, it also destructures the request body and places the
description of the todo entry into the result.

When we come to write the associated handler, we need some way of
getting the information from the result. Ataraxy places the result
into the ``:ataraxy/result`` key on the request map, so we can
destructure the request to find the description of the new entry:

.. code-block:: edn

  [:duct.handler.sql/insert :todo.handler.entries/create]
  {:request {[_ description] :ataraxy/result}
   :sql     ["INSERT INTO entries (description) VALUES (?)" description]}

Next we ``reset``:

.. code-block:: clojure

  dev=> (reset)
  :reloading (todo.main dev user)
  :resumed

And test::

  $ http post :3000/entries description="Write Duct guide"
  HTTP/1.1 201 Created
  Content-Length: 0
  Content-Type: application/octet-stream
  Date: Thu, 07 Dec 2017 11:29:46 GMT
  Server: Jetty(9.2.21.v20170120)


  $ http get :3000/entries
  HTTP/1.1 200 OK
  Content-Length: 43
  Content-Type: application/json; charset=utf-8
  Date: Thu, 07 Dec 2017 11:29:51 GMT
  Server: Jetty(9.2.21.v20170120)

  [
      {
          "description": "Write Duct guide",
          "id": 1
      }
  ]

We can now have the bare bones of a useful application.


Becoming More RESTful
"""""""""""""""""""""

We can now GET and POST to lists of entries for our Todo application,
but ideally we'd also like to DELETE particular entries as well. In
order to do that, each entry needs to have a distinct URI.

Let's start by adding some hypertext references to our list handler:

.. code-block:: edn

  [:duct.handler.sql/query :todo.handler.entries/list]
  {:sql   ["SELECT * FROM entries"]
   :hrefs {:href "/entries/{id}"}}

The ``:hrefs`` option allows hypertext references to be added to the
response using `URI templates`_. If we ``reset``:

.. code-block:: clojure

  dev=> (reset)
  :reloading (todo.main dev user)
  :resumed

And test::

  $ http :3000/entries
  HTTP/1.1 200 OK
  Content-Length: 63
  Content-Type: application/json; charset=utf-8
  Date: Thu, 07 Dec 2017 21:13:20 GMT
  Server: Jetty(9.2.21.v20170120)

  [
      {
          "description": "Write Duct guide",
          "href": "/entries/1",
          "id": 1
      }
  ]

We can see that each list entry now has a new key. Let's write two new
Ataraxy routes:

.. code-block:: edn

  :duct.module/ataraxy
  {[:get "/"]        [:index]
   [:get "/entries"] [:entries/list]

   [:post "/entries" {{:keys [description]} :body-params}]
   [:entries/create description]

   [:get    "/entries/" id] [:entries/find    ^int id]
   [:delete "/entries/" id] [:entries/destroy ^int id]}

These routes show how we can pull data out of the URI, and coerce it
into a new type.

The routes require associated handlers. As before, we'll make use of
the `duct/handler.sql` library, using the `query-one` and `execute`
handler types:

.. code-block:: edn

  [:duct.handler.sql/query-one :todo.handler.entries/find]
  {:request {[_ id] :ataraxy/result}
   :sql     ["SELECT * FROM entries WHERE id = ?" id]
   :hrefs   {:href "/entries/{id}"}}

  [:duct.handler.sql/execute :todo.handler.entries/destroy]
  {:request {[_ id] :ataraxy/result}
   :sql     ["DELETE FROM entries WHERE id = ?" id]}


We also want to improve the entry creation route and give it a
`Location` header to the resource it creates:

.. code-block:: edn

  [:duct.handler.sql/insert :todo.handler.entries/create]
  {:request  {[_ description] :ataraxy/result}
   :sql      ["INSERT INTO entries (description) VALUES (?)" description]
   :location "/entries/{last_insert_rowid}"}

The `last_insert_rowid` is a resultset column specific to
SQLite. Other databases will return the generated row ID in different
ways.

With all that done we `reset`:

.. code-block:: clojure

  dev=> (reset)
  :reloading ()
  :resumed

And test::

  $ http :3000/entries/1
  HTTP/1.1 200 OK
  Content-Length: 61
  Content-Type: application/json; charset=utf-8
  Date: Sat, 09 Dec 2017 12:59:05 GMT
  Server: Jetty(9.2.21.v20170120)

  {
      "description": "Write Duct guide",
      "href": "/entries/1",
      "id": 1
  }

  $ http delete :3000/entries/1
  HTTP/1.1 204 No Content
  Content-Type: application/octet-stream
  Date: Sat, 09 Dec 2017 12:59:12 GMT
  Server: Jetty(9.2.21.v20170120)


  $ http :3000/entries/1
  HTTP/1.1 404 Not Found
  Content-Length: 21
  Content-Type: application/json; charset=utf-8
  Date: Sat, 09 Dec 2017 12:59:18 GMT
  Server: Jetty(9.2.21.v20170120)

  {
      "error": "not-found"
  }

  $ http post :3000/entries description="Continue Duct guide"
  HTTP/1.1 201 Created
  Content-Length: 0
  Content-Type: application/octet-stream
  Date: Sat, 09 Dec 2017 13:18:46 GMT
  Location: http://localhost:3000/entries/1
  Server: Jetty(9.2.21.v20170120)

.. _URI templates: https://tools.ietf.org/html/rfc6570


코드
~~~~

지금까지 Duct 애플리케이션을 만들기 만들기 위해 설정을 사용하는 방법에 대해 알아봤습니다.
단순한 기능에는 이 방법으로 잘 동작하지만 대부분의 애플리케이션은 코드를 작성해야 합니다.

데이터를 기반으로 핸들러를 정의하는 것은 장점이 있지만 너무 과하지 않도록 하는 것이 중요합니다.
애플리케이션에서 설정은 골격으로 코드는 근육과 기관으로 생각하세요.

사용자 추가하기
""""""""""""

지금까지 사용자가 한명인 애플리케이션을 만들었습니다. 이제 ``users`` 테이블을 추가해서 바꿔 봅시다.
먼저 설정에 새 마이그레이션 참조를 추가합니다:

.. code-block:: edn

  :duct.migrator/ragtime
  {:migrations [#ig/ref :todo.migration/create-entries
                #ig/ref :todo.migration/create-users]}

다음에 마이그레이션을 만듭니다:

.. code-block:: edn

  [:duct.migrator.ragtime/sql :todo.migration/create-users]
  {:up ["CREATE TABLE users (id INTEGER PRIMARY KEY, email TEXT UNIQUE, password TEXT)"]
   :down ["DROP TABLE users"]}

다음은 새 마이그레이션을 적용하기 위해 ``reset``\을 실행합니다:

.. code-block:: clojure

  dev=> (reset)
  :reloading ()
  :duct.migrator.ragtime/applying :todo.migration/create-users#66d6b1f8
  :resumed

이제 사용자를 저장할 테이블이 생겼으니 다음으로 사용자들이 웹 서비스에서 가입할 수 방법이 필요합니다.
``duct/handler.sql`` 라이브러리로 핸들러를 만들 수 있지만 데이터베이스에 직접 비밀번호를 저장하는
것은 보안에 좋지 않습니다.

대신 비밀번호 보안 방식 중 하나인 `key derivation function`_\(또는 KDF)를 이용해서 핸들러 함수를
직접 만들어 봅시다. 먼저 아래와 같이 새로운 라이브러리를 프로젝트 디펜던시에 추가합니다:

.. code-block:: clojure

  [buddy/buddy-hashers "1.3.0"]

이 라이브러리를 추가하면 KDF를 사용할 수 있습니다. 디펜던시를 추가한 후에 REPL을 종료합니다:

.. code-block:: clojure

  dev=> (exit)
  Bye for now!

그리고 다시 시작합니다::

  $ lein repl

다음은 애플리케이션을 시작합니다:

.. code-block:: clojure
  user=> (dev)
  :loaded
  dev=> (go)
  :duct.server.http.jetty/starting-server {:port 3000}
  :initiated

다음으로 사용자를 생성하기 위한 Ataraxy 라우터를 추가합니다:

.. code-block:: edn

  :duct.module/ataraxy
  {[:get "/"]        [:index]
   [:get "/entries"] [:entries/list]

   [:post "/entries" {{:keys [description]} :body-params}]
   [:entries/create description]

   [:get    "/entries/" id] [:entries/find    ^int id]
   [:delete "/entries/" id] [:entries/destroy ^int id]

   [:post "/users" {{:keys [email password]} :body-params}]
   [:users/create email password]}

그리고 핸들러 설정을 추가합니다:

.. code-block:: edn

  :todo.handler.users/create
  {:db #ig/ref :duct.database/sql}

방금 추가한 설정에는 컴포지트 키를 사용하지 않았습니다. 왜냐하면 기존에 있는 기능이 아니고 새로운 기능을
만들기 때문입니다.

그리고 데이터베이스 참조를 추가했습니다. Duct에 있는 모든 SQL 데이터베이스 키는 ``:duct.database/sql``\를
상속 받습니다. Duct는 이 키를 이용해서 첫번째로 사용 가능한 SQL 데이터베이스를 찾습니다.

You may wonder why the ``duct.handler.sql`` keys didn't include a
database key. This is because they all inherit from the
``:duct.module.sql/requires-db`` keyword, which is a indicator to the
``:duct.module/sql`` module to automatically insert the reference. We
could also do this, but for now we'll keep the reference explicit.

이제 핸들러 코드를 만들어 봅시다. 키워드에 사용한 네임스페이스는 ``todo.handler.users`` 입니다.
그래서 코드에 네임스페이스도 같은 것을 사용하려고 합니다. ``src/todo/handler/users.clj`` 파일을
만들고 네임스페이스를 선언합니다:

.. code-block:: clojure

  (ns todo.handler.users
    (:require [ataraxy.response :as response]
              [buddy.hashers :as hashers]
              [clojure.java.jdbc :as jdbc]
              duct.database.sql
              [integrant.core :as ig]))

KDF를 쓰기 위해 ``buddy.hashers``\가 필요하고 데이터베이스에 접근하기 위해 ``clojure.java.jdbc``\가
필요합니다. ``integrant.core`` 네임스페이스는 Integrant 멀티메서드를 만들기 위해 필요하지만
``ataraxy.response``\와 ``duct.database.sql``\는 추가하는 목적이 약간 명확하지 않습니다.

이제 새 사용자를 데이터베이스에 추가하는 함수를 만들고 추가된 row 아이디를 리턴하도록 함수를 만들어봅시다:

.. code-block:: clojure

  (defprotocol Users
    (create-user [db email password]))

  (extend-protocol Users
    duct.database.sql.Boundary
    (create-user [{db :spec} email password]
      (let [pw-hash (hashers/derive password)
            results (jdbc/insert! db :users {:email email, :password pw-hash})]
        (-> results ffirst val))))

Duct를 처음 사용한다면 여기에 프로토콜을 쓴다는 점이 생소할 것입니다. 왜 함수를 바로 쓰지 않죠?
왜 이상한 ``duct.database.sql.Boundary`` 타입에 프로토콜을 구현을 하는거죠?

답은 분명히 함수를 *사용할 수* 있고 그러면 코드를 몇 줄 더 줄일 수 있습니다. 하지만 프로토콜을 사용하면
개발이나 테스트 환경에 데이터베이스를 Mock으로 대체할 수 있다는 장점이 있습니다. 이런 이유로 Duct는
``duct.database.sql.Boundary`` 라고 부르는 빈 '바운더리' 레코드를 제공합니다. 이것이 앞에서
``duct.database.sql`` 네임스페이스를 포함시킨 이유입니다. 그렇지 않으면 레코드가 로드되지 않습니다.

마지막으로 create 키워드를 위한 ``init-key`` 메서드를 만듭니다:

.. code-block:: clojure

  (defmethod ig/init-key ::create [_ {:keys [db]}]
    (fn [{[_ email password] :ataraxy/result}]
      (let [id (create-user db email password)]
        [::response/created (str "/users/" id)])))

Ataraxy는 Ring 응답 맵 대신 백터를 리런 할 수 있습니다. 이 기능은 추상화와 편리함을 줍니다.
Ataraxy는 ``201 Created`` 응답을 내려주게 됩니다.

이제 ``reset``\을 해봅시다:

.. code-block:: clojure

  dev=> (reset)
  :reloading (todo.main todo.handler.users dev user)
  :resumed

그리고 확인해봅니다::

  $ http post :3000/users email=bob@example.com password=hunter2
  HTTP/1.1 201 Created
  Content-Length: 0
  Content-Type: application/octet-stream
  Date: Mon, 11 Dec 2017 14:10:31 GMT
  Location: http://localhost:3000/users/1
  Server: Jetty(9.2.21.v20170120)

아직 어떤 시작적 정보도 없습니다. 이제 데이터베이스를 살펴볼 필요가 있습니다.

.. _key derivation function: https://en.wikipedia.org/wiki/Key_derivation_function


데이터베이스에 쿼리하기
"""""""""""""""""""""


개발을 하는 동안 우리가 작성한 코드가 데이터베이스에 데이터를 잘 넣고 있는지 확인할 필요가 있습니다.
이 일을 쉽게 하기 위해 ``dev/src/dev.clj`` 파일에 ``dev`` 네임스페이스를 추가합시다.

먼저 ``clojure.java.jdbc`` 네임스페이스가 필요합니다:

.. code-block:: clojure

  [clojure.java.jdbc :as jdbc]

다음으로 데이터베이스 연결을 얻을 수 있어야 합니다. Duct는 개발하는 동안 ``system`` var에 동작하고
있는 시스템 정보를 저장합니다. 그래서 JDBC 데이터베이스 스펙을 가져오는 간단한 함수를 아래와 같이 만들 수
있습니다:

.. code-block:: clojure

  (defn db []
    (-> system (ig/find-derived-1 :duct.database/sql) val :spec))

데이터베이스을 얻었으니 이제 쿼리를 도와주는 간단한 함수를 만들어 봅시다:

.. code-block:: clojure

  (defn q [sql]
    (jdbc/query (db) sql))

다 했으면 ``reset``\을 실행해 줍니다:

.. code-block:: clojure

  dev=> (reset)
  :reloading (dev)
  :resumed

다음에 ``users`` 테이블에 쿼리를 실행해 봅시다:

.. code-block:: clojure

  dev=> (q "SELECT * FROM users")
  ({:id 1,
    :email "bob@example.com",
    :password
    "bcrypt+sha512$f4c1bc592ecd1869d0bf802f7c8f6e36$12$19a9ae3ed9118cb6cbfcd8c4a31aadb6b00162288b1fce50"})

잘 된 것 같습니다. ID, 이메일, 해쉬된 비밀번호가 있네요.
