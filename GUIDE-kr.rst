Duct Framework 가이드
===========================

머리말
~~~~~~~

Duct는 Clojure_ 프로그래밍 언어로 서버 애플리케이션을 만들기 위한 data-driven 프레임워크입니다.
이 가이드는 웹 애플리케이션을 예제로 Duct 사용법을 자세히 설명하기 위해 섰습니다.

이 가이드를 제대로 읽으려면 Leiningen_\이 설치되어 있고 Clojure 실무 지식이 있어야합니다.
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

``+``\로 시작하는 파라미터는 프로필 힌트입니다. 위 예제는 웹 서비스 (``+api``)와 Ataraxy_ 라우팅 라이브러리
(``+ataraxy``), SQLite 데이터베이스 (``+sqlite``)를 사용하는 프로젝트를 템플릿을 만드는 예제입니다.

사용할 수 있는 프로필 힌트를 모두 보려면 아래 명령어를 실행합니다::

  $ lein new :show duct

이제 조금 전에 만든 ``todo`` 프로젝트 디렉터리로 들어가 봅시다::

  $ cd todo

그리고 로컬 셑업을 실행합니다::

  $ lein duct setup

셑업을 실행하고 나면 소스 컨트롤에는 제외되어 있는 파일 4개가 생깁니다::

  Created profiles.clj
  Created .dir-locals.el
  Created dev/resources/local.edn
  Created dev/src/local.clj

이 파일들은 ``.gitignore``\에 추가되어 있기 때문에 Git_\을 사용하면 따로 해줄 일이 없습니다. 하지만
다른 소스 컨트롤을 사용한다면 이 파일들이 소스 컨트롤에 관리되지 않도록 수동으로 처리해줘야 합니다.

.. _SQLite:  https://sqlite.org/
.. _Ataraxy: https://github.com/weavejester/ataraxy
.. _Git:     https://git-scm.com/


REPL 시작하기
"""""""""""""""""

Duct는 REPL을 중심으로 개발 하도록 되어 있습니다. 그래서 Cursive_\나 Emacs_\의 CIDER_, Vim_\의
`fireplace.vim`_, Atom_\의 `Proto REPL`_\같은 에디터 REPL 통합 환경을 사용하는 것을 추천합니다.
하지만 이 가이드는 에디터 통합 없이 커맨드 라인에서 직접 실행해볼 수 있도록 되어 있습니다.

REPL을 시작합니다::

  $ lein repl

먼저 개발 환경을 로드하기 위해 프롬프트에 다음과 같이 입력합니다:

.. code-block:: clojure

  user=> (dev)
  :loaded
  dev=>

개발 환경에 에러가 있는 경우 REPL이 실행되지 않을 수 있기 때문에 개발 환경은 자동으로 로드되지 않습니다.

``dev`` 네임스페이스로 바뀌면 애플리케이션을 실행해볼 수 있습니다:

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

"not found" 응답을 받았습니다. 아직 아무 라우터도 추가하지 않았기 때문에 예상된 결과입니다.

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

Duct 어플리케이션은 edn_ 컨피그 파일 주위에서 빌드됩니다.
Configuration 파일은 어플리케이션의 구조와 디펜던시를 정의합니다.
이 가이드안에서 만든 프로젝트에서, 컨피그레이션 파일은 다음 위치에 있습니다:
``resources/todo/config.edn``.

정적 라우트 추가하기
"""""""""""""""""""""

Config 파일을 살펴보겠습니다:

.. code-block:: edn

  {:duct.core/project-ns  todo
   :duct.core/environment :production

   :duct.module/logging {}
   :duct.module.web/api {}
   :duct.module/sql {}

   :duct.module/ataraxy
   {}}

정적 index 라우트를 추가하는 것으로 시작할 수 있을텐데,
Ataraxy가 사용할 라우터이기 때문에 ``:duct.module/ataraxy`` 라고 한 줄을 추가합니다:

.. code-block:: edn

  :duct.module/ataraxy
  {[:get "/"] [:index]}

  이것은 라우트 ``[:get "/"]`` 를 ``[:index]`` 로 연결합니다.
  Ataraxy 모듈은 자동으로 컨피그에서 이름과 일치하는 Ring 핸들러를 찾아 쌍을 이룹니다.
  결과 키가 ``:index`` 이기 때문에, 핸들러 키는 ``:todo.handler/index`` 가 됩니다.
  컨피그에 그 이름을 가진 엔트리를 추가해봅시다:

.. code-block:: edn

  [:duct.handler.static/ok :todo.handler/index]
  {:body {:entries "/entries"}}

이번에는 벡터를 키로 사용합니다; Duct에서는 이것을 *복합 (composite key)* 라고 합니다.
복합 키는 복합 키에 속한 모든 키워드의 속성을 상속 받습니다;
벡터에 ``:duct.handler.static/ok`` 가 포함되어 있기 때문에,
컨피그레이션 엔트리가 정적 핸들러를 생성합니다.

이 변경사항을 어플리케이션에 적용해 보겠습니다.
레플로 돌아가서 실행해보세요:

.. code-block:: clojure

  dev=> (reset)
  :reloading (todo.main dev user)
  :resumed

이것은 컨피그와 변경된 파일을 다시 로드합니다.
이제는 웹 서버에 요청을 보내, 예상된 응답을 받을 수 있습니다::

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

데이터 마이그레이션 추가하기
"""""""""""""""""""""""""""

더 많은 동적 라우트를 추가하고 싶지만, 그전에 데이터베이스 스키마를 생성해야합니다.
Duct는 Ragtime_ 을 사용해 마이그레이션을 하고,
각 마이그레이션은 컨피그에서 정의됩니다.

컨피그에 두 개의 키를 더 추가합니다.

.. code-block:: edn

  :duct.migrator/ragtime
  {:migrations [#ig/ref :todo.migration/create-entries]}

  [:duct.migrator.ragtime/sql :todo.migration/create-entries]
  {:up ["CREATE TABLE entries (id INTEGER PRIMARY KEY, content TEXT)"]
   :down ["DROP TABLE entries"]}

``:duct.migrator/ragtime`` 키는 마이그레이션을 순서대로 가집니다.
각 마이그레이션은 복합키에서 ``:duct.migrator.ragtime/sql`` 을 포함시켜 정의할 수 있습니다.
``:up`` 과 ``:down`` 옵션은 실행할 SQL의 벡터를 가집니다;
up은 마이그레이션을, down은 롤백을 하게 됩니다.

마이그레이션을 위해서 REPL에서 ``reset`` 을 다시 실행합니다:

.. code-block:: clojure

  dev=> (reset)
  :reloading ()
  :duct.migrator.ragtime/applying :todo.migration/create-entries#b34248fc
  :resumed

마이그레이션을 적용한 이후에 스키마를 바꾸기로 했다고 가정해보겠습니다.
다른 마이그레이션을 새로 작성해볼수도 있지만, 코드가 커밋이 안되었거나 프로덕션에 배포하지 않은경우
가지고 있던 마이그레이션을 편집하는 것이 좀더 편리합니다.

마이그레이션을 변경하고,``content`` 컬럼의 이름을``description`` 으로 바꿔봅시다:

.. code-block:: edn

  [:duct.migrator.ragtime/sql :todo.migration/create-entries]
  {:up ["CREATE TABLE entries (id INTEGER PRIMARY KEY, description TEXT)"]
   :down ["DROP TABLE entries"]}

그리고 ``reset``:

.. code-block:: clojure

  dev=> (reset)
  :reloading ()
  :duct.migrator.ragtime/rolling-back :todo.migration/create-entries#b34248fc
  :duct.migrator.ragtime/applying :todo.migration/create-entries#5c2bb12a
  :resumed

이전 버전의 마이그레이션은 자동으로 롤백되고 새 버전의 마이그레이션이 대신 적용됩니다.

.. _Ragtime: https://github.com/weavejester/ragtime

프로덕션 환경에서 데이터베이스 마이그레이션 하기
"""""""""""""""""""""""""""""""""""""""""

프로덕션 환경에서도 쉽게 마이그레이션을 할 수 있습니다::

  $ lein run :duct/migrator

개발에서 Heroku를 쓰고 있다면, Procfile을 통해 릴리즈 단계에 쉽게 추가해볼수 있습니다.

  web: java -jar target/sstandalone.jar
  release: lein run :duct/migrator

쿼리 라우트 추가하기
""""""""""""""""""""

이제 데이터베이스 테이블이 생겼으므로 쿼리 라우트를 작성해야합니다.
``duct/handler.sql`` 라고 불리는 라이브러리를 사용할 것입니다.
이것은 ``project.clj`` 파일의 ``:dependencies`` 키에 추가돼야 합니다::

.. code-block:: clojure

  [duct/handler.sql "0.3.1"]

디펜던시는 이제 다음과 같이 보일 것입니다 :

.. code-block:: clojure

  :dependencies [[org.clojure/clojure "1.9.0-RC1"]
                 [duct/core "0.6.1"]
                 [duct/handler.sql "0.3.1"]
                 [duct/module.logging "0.3.1"]
                 [duct/module.web "0.6.3"]
                 [duct/module.ataraxy "0.2.0"]
                 [duct/module.sql "0.4.2"]
                 [org.xerial/sqlite-jdbc "3.20.1"]]

디펜던시를 추가했을 때에는 REPL을 다시 시작해야하므로,
일단 REPL에서 빠져나옵니다.

.. code-block:: clojure

  dev=> (exit)
  Bye for now!

그리고 다시 시작합니다::

  $ lein repl

그리고 어플리케이션을 다시 실행합니다::

.. code-block:: clojure
  user=> (dev)
  :loaded
  dev=> (go)
  :duct.server.http.jetty/starting-server {:port 3000}
  :initiated

이제 프로젝트 컨피그로 돌아가서,
새로운 Ataraxy 라우트를 추가하는 것으로 시작해봅시다:

.. code-block:: edn

  :duct.module/ataraxy
  {[:get "/"]        [:index]
   [:get "/entries"] [:entries/list]}

앞서 본 것과 같이, ``[:entries/list]`` 는 적절하게 이름 붙여진 Ring 핸들러와 쌍을 이뤄야합니다.
Ataraxy 모듈은 이 핸들러 이름이  ``:todo.handler.entries/list`` 이기를 기대하기 때문에,
``:duct.handler.sql/query`` 키와 함께 그 이름을 사용하게 됩니다:

.. code-block:: edn

  [:duct.handler.sql/query :todo.handler.entries/list]
  {:sql ["SELECT * FROM entries"]}

일단 핸들러가 컨피그에서 정의되면, ``reset`` 을 할 수 있습니다 :

.. code-block:: clojure

  dev=> (reset)
  :reloading (todo.main dev user)
  :resumed

그리고 HTTP 요청을 보내서 라우트를 확인합니다::

  $ http :3000/entries
  HTTP/1.1 200 OK
  Content-Length: 2
  Content-Type: application/json; charset=utf-8
  Date: Thu, 07 Dec 2017 10:13:34 GMT
  Server: Jetty(9.2.21.v20170120)

  []

유효한 응답이지만, 비어있는 응답입니다.
``entries`` 테이블에 아무 데이터도 넣지 않았기 때문인 것을 알수 있습니다.


업데이트 라우트 추가하기
""""""""""""""""""""""

다음으로는 데이터베이스를 업데이트 하는 라우트를 추가하려고합니다.
다시 ``duct/handler.sql`` 라이브러리를 사용할 것이지만,
라우트와 핸들러는 더 복잡해 질 것입니다.

일단, 새로운 라우트입니다:

.. code-block:: edn

  :duct.module/ataraxy
  {[:get "/"]        [:index]
   [:get "/entries"] [:entries/list]

   [:post "/entries" {{:keys [description]} :body-params}]
   [:entries/create description]}

새로운 Ataraxy 라우트는 요청의 메소드와 URI를 일치시킬뿐만 아니라,
요청의 body를 디스트럭처링 하고 todo 엔트리에 설명도 넣을 수 있습니다.

관련된 핸들러를 작성할 때, 결과에서 정보를 가져올 수 있는 방법이 필요합니다.
Ataraxy는 결과를 요청 맵의 ``:ataraxy/result`` 키에 넣습니다.
그래서 새 앤트리의 설명을 찾기 위해 요청을 디스트럭처링 할 수 있습니다:

.. code-block:: edn

  [:duct.handler.sql/insert :todo.handler.entries/create]
  {:request {[_ description] :ataraxy/result}
   :sql     ["INSERT INTO entries (description) VALUES (?)" description]}

그리고 ``reset``:

.. code-block:: clojure

  dev=> (reset)
  :reloading (todo.main dev user)
  :resumed

그리고 테스트::

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

이제 쓸만한 어플리케이션의 뼈대가 생겼습니다.

좀 더 RESTful하게 만들기
"""""""""""""""""""""

이제 엔트리의 목록에 GET과 POST를 Todo 어플리케이션에 날려볼 수 있지만,
DELETE도 만들어봅시다.
이를 위해서는 각 엔트리가 고유한 URI를 가져야합니다.

리스트 핸들러에 하이퍼텍스트 참조를 추가해봅시다.

.. code-block:: edn

  [:duct.handler.sql/query :todo.handler.entries/list]
  {:sql   ["SELECT * FROM entries"]
   :hrefs {:href "/entries/{id}"}}

``:hrefs`` 옵션은 `URI templates`_을 사용해
응답에 하이퍼텍스트 참조를 추가할 수 있게합니다.
 ``reset`` 을 하면:

.. code-block:: clojure

  dev=> (reset)
  :reloading (todo.main dev user)
  :resumed

그리고 테스트::

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

이제 각 리스트 엔트리에 새 키가 생긴 것을 볼 수 있습니다.
투가지 새로운 Ataraxy 라우트를 작성해보겠습니다:

.. code-block:: edn

  :duct.module/ataraxy
  {[:get "/"]        [:index]
   [:get "/entries"] [:entries/list]

   [:post "/entries" {{:keys [description]} :body-params}]
   [:entries/create description]

   [:get    "/entries/" id] [:entries/find    ^int id]
   [:delete "/entries/" id] [:entries/destroy ^int id]}

이 라우트는 URI에서 데이터를 가져와서, 새로운 타입으로 강제하는 방법을 보여줍니다.

라우트에는 관련된 핸들러가 필요합니다. 앞서 나온 `duct/handler.sql` 라이브러리의
`query-one` 와 `execute` 핸들러 타입을 사용해봅니다:

.. code-block:: edn

  [:duct.handler.sql/query-one :todo.handler.entries/find]
  {:request {[_ id] :ataraxy/result}
   :sql     ["SELECT * FROM entries WHERE id = ?" id]
   :hrefs   {:href "/entries/{id}"}}

  [:duct.handler.sql/execute :todo.handler.entries/destroy]
  {:request {[_ id] :ataraxy/result}
   :sql     ["DELETE FROM entries WHERE id = ?" id]}


또한 엔트리 생성 라우트를 개선하고, `Location`를 제공해 리소스를 생성할 수 있습니다:

.. code-block:: edn

  [:duct.handler.sql/insert :todo.handler.entries/create]
  {:request  {[_ description] :ataraxy/result}
   :sql      ["INSERT INTO entries (description) VALUES (?)" description]
   :location "/entries/{last_insert_rowid}"}

`last_insert_rowid`는 SQLite에서만 사용하는 결과 집합 컬럼입니다.
다른 데이터베이스는 생성된 row별 ID를 다른 방식으로 반환합니다.

완료했으면 `reset`을 합니다 :

.. code-block:: clojure

  dev=> (reset)
  :reloading ()
  :resumed

그리고 테스트::

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

지금까지 설정을 사용해서 Duct 애플리케이션을 만들어 봤습니다. 단순한 기능을 만들 때는 설정만으로 만들 수
있지만 대부분의 애플리케이션은 코드를 작성해야 합니다.

설정을 사용한 데이터 기반으 핸들러는 장점이 있지만 너무 과하지 않도록 하는 것이 중요합니다.
애플리케이션을 만들 때 설정은 골격으로 코드는 근육과 기관으로 생각하면 좋습니다.

사용자 추가하기
""""""""""""

지금까지 사용자가 한명인 애플리케이션을 만들었습니다. 이제 ``users`` 테이블을 추가해서 사용자가 여러명인
애플리케이션으로 바꿔 봅시다. 먼저 설정에 새 마이그레이션 참조를 추가합니다:

.. code-block:: edn

  :duct.migrator/ragtime
  {:migrations [#ig/ref :todo.migration/create-entries
                #ig/ref :todo.migration/create-users]}

그리고 마이그레이션을 만듭니다:

.. code-block:: edn

  [:duct.migrator.ragtime/sql :todo.migration/create-users]
  {:up ["CREATE TABLE users (id INTEGER PRIMARY KEY, email TEXT UNIQUE, password TEXT)"]
   :down ["DROP TABLE users"]}

새 마이그레이션을 적용하기 위해 ``reset``\을 실행합니다:

.. code-block:: clojure

  dev=> (reset)
  :reloading ()
  :duct.migrator.ragtime/applying :todo.migration/create-users#66d6b1f8
  :resumed

사용자를 저장할 테이블이 생겼으니 이제 사용자들이 웹 서비스에서 가입할 수 방법이 필요합니다.
``duct/handler.sql`` 라이브러리로 핸들러를 만들 수 있지만 그렇게 하면 비밀번호를 데이터베이스에
그대로 저장하게 되어 보안에 좋지 않습니다.

대신 비밀번호 보안 방식 중 하나인 `key derivation function`_\(또는 KDF)를 이용해서 암호화된
비밀번호를 저장하도록 핸들러 함수를 직접 만들어 봅시다. 먼저 아래 라이브러리를 프로젝트 디펜던시에 추가합니다:

.. code-block:: clojure

  [buddy/buddy-hashers "1.3.0"]

이 라이브러리를 추가하면 키 유도 함수(KDF)를 사용할 수 있습니다. 디펜던시를 추가한 후에 REPL을 종료합니다:

.. code-block:: clojure

  dev=> (exit)
  Bye for now!

그리고 다시 시작합니다::

  $ lein repl

다음은 애플리케이션을 시작해줍니다:

.. code-block:: clojure
  user=> (dev)
  :loaded
  dev=> (go)
  :duct.server.http.jetty/starting-server {:port 3000}
  :initiated

이제 사용자를 생성하기 위한 Ataraxy 라우터를 추가합니다:

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

방금 추가한 핸들러 설정은 복합 키(Composite Key)를 사용하지 않았습니다. 왜냐하면 기존에 있는 기능을
상속하지 않고 새로운 기능을 만들려고 하기 때문입니다.

그리고 데이터베이스 참조를 추가했습니다. Duct에 있는 모든 SQL 데이터베이스 키는 ``:duct.database/sql``\를
상속 받습니다. Duct는 이 키를 이용해서 사용 가능한 SQL 데이터베이스를 찾습니다.

``duct.handler.sql`` 키를 사용하면 ``:duct.module/sql`` 모듈을 추가해주는
``:duct.module.sql/requires-db`` 키워드를 상속하고 있기 때문 자동으로 데이터베이스 참조가 추가됩니다.
하지만 여기서는 ``duct.handler.sql`` 키를 사용하지 않고 명시적으로 데이터베이스 참조를 추가했습니다.

이제 핸들러 코드를 만들어 봅시다. 키워드에 사용한 네임스페이스는 ``todo.handler.users`` 입니다.
그래서 코드에 있는 네임스페이스도 같은 것을 사용하려고 합니다. ``src/todo/handler/users.clj``
파일을 만들고 네임스페이스를 선언합니다:

.. code-block:: clojure

  (ns todo.handler.users
    (:require [ataraxy.response :as response]
              [buddy.hashers :as hashers]
              [clojure.java.jdbc :as jdbc]
              duct.database.sql
              [integrant.core :as ig]))

키 유도 함수(KDF)를 쓰기 위해 ``buddy.hashers``\가 필요하고 데이터베이스에 접근하기 위해
``clojure.java.jdbc``\가 필요합니다. ``integrant.core`` 네임스페이스는 Integrant 멀티메서드를
만들기 위해 필요하지만 ``ataraxy.response``\와 ``duct.database.sql``\는 추가하는 목적이
아직 명확하지 않습니다. (뒤에서 알아 봅니다.)

이제 새 사용자를 데이터베이스에 추가하는 함수를 만들고 추가된 row 아이디를 리턴하는 함수를 만들어봅시다:

.. code-block:: clojure

  (defprotocol Users
    (create-user [db email password]))

  (extend-protocol Users
    duct.database.sql.Boundary
    (create-user [{db :spec} email password]
      (let [pw-hash (hashers/derive password)
            results (jdbc/insert! db :users {:email email, :password pw-hash})]
        (-> results ffirst val))))

Duct를 처음 사용한다면 여기서 프로토콜을 쓴다는 점이 생소할 것입니다. 왜 함수를 바로 쓰지 않을까요?
왜 이상한 ``duct.database.sql.Boundary`` 타입에 프로토콜을 구현을 하는걸까요?

분명한 점은 함수로 *만들어도* 되고 그렇게하면 코드를 몇 줄 더 줄일 수 있습니다. 하지만 프로토콜을 사용하면
개발 환경이나 테스트 환경에서 데이터베이스를 Mock으로 대체할 수 있다는 장점이 있습니다. 이런 이유로 Duct는
``duct.database.sql.Boundary``\라고 부르는 비어 있는 '바운더리' 레코드를 제공합니다. 앞에서
``duct.database.sql`` 네임스페이스를 포함시킨 이유입니다. 그렇지 않으면 레코드가 로드되지 않습니다.

마지막으로 create 키워드를 위한 ``init-key`` 메서드를 만듭니다:

.. code-block:: clojure

  (defmethod ig/init-key ::create [_ {:keys [db]}]
    (fn [{[_ email password] :ataraxy/result}]
      (let [id (create-user db email password)]
        [::response/created (str "/users/" id)])))

Ataraxy는 Ring 응답 맵 대신 백터를 리런 할 수 있습니다. 이 기능은 추상화와 편리함을 줍니다.
위 예제에서 Ataraxy는 ``201 Created`` 응답을 내려주게 됩니다.

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

아직 잘 되었는지 눈으로 확인해 볼 방법은 없습니다. 그래서 이제 데이터베이스를 살펴볼 필요가 있습니다.

.. _key derivation function: https://en.wikipedia.org/wiki/Key_derivation_function


데이터베이스에 쿼리하기
"""""""""""""""""""""


개발을 하면서 데이터베이스에 데이터가 잘 들어가고 있는지 확인할 필요가 있습니다.
개발의 편의를 위해 ``dev/src/dev.clj`` 파일에 ``dev`` 네임스페이스를 추가합시다.

먼저 ``clojure.java.jdbc`` 네임스페이스가 필요합니다:

.. code-block:: clojure

  [clojure.java.jdbc :as jdbc]

다음으로 데이터베이스 연결이 필요합니다. 개발 환경에서 Duct는 ``system`` var에 현재 동작하는
시스템 정보를 저장하고 있습니다. 그래서 JDBC 데이터베이스 스펙을 가져오는 간단한 함수를 아래와 같이
만들 수 있습니다:

.. code-block:: clojure

  (defn db []
    (-> system (ig/find-derived-1 :duct.database/sql) val :spec))

데이터베이스 연결을 가져왔으니 이제 쿼리를 도와주는 간단한 함수를 만들어 봅시다:

.. code-block:: clojure

  (defn q [sql]
    (jdbc/query (db) sql))

다 했으면 ``reset``\을 실행해 줍니다:

.. code-block:: clojure

  dev=> (reset)
  :reloading (dev)
  :resumed

다음에 ``users`` 테이블에 쿼리를 실행해 봅니다:

.. code-block:: clojure

  dev=> (q "SELECT * FROM users")
  ({:id 1,
    :email "bob@example.com",
    :password
    "bcrypt+sha512$f4c1bc592ecd1869d0bf802f7c8f6e36$12$19a9ae3ed9118cb6cbfcd8c4a31aadb6b00162288b1fce50"})

잘 된 것 같습니다. ID, 이메일, 해쉬된 비밀번호가 있네요.
