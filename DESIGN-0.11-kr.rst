Duct 0.11 설계 의사 결정
==========================

머리말
~~~~~~~

Duct 0.11 프로젝트의 주요 변경 사항을 소개합니다. 이 문서는 변경 사항을 자세히 설명하고 그렇게 구현 된
이유를 설명합니다.

이 변경 사항은 알파 버전에 구현되어 있고 아래와 같이 테스트 해볼 수 있습니다::

  lein new duct-alpha <project> <hints...>


모듈
~~~~~~~

0.10 버전에서
"""""""

0.10 이하 버전에서는 모듈을 만들 때 아래 처럼 코드를 작성했습다:

.. code-block:: clojure

  (defmulti ig/init-key ::module [_ options]
    {:req [::precondition]
     :fn  (fn [config] config)})

모듈을 초기화 하는 키는 맵을 리턴합니다. ``:fn`` 키는 설정을 변경하는데 사용하는 순수 함수입니다.
``:req`` 키는 모듈이 초기화 되기 전에 설정에 필요한 필수 키 컬렉션입니다. 필수 키는 Duct가 각 모듈이
의존성에 맞춰 순서대로 적용하기 위해 필요합니다.

문제는 이미 Integrant에 의존성을 관리하기 위한 잘 만들어진 기능이 있다는 것입니다. 의존성 순서를 위한
두 개의 개별 시스템이 있는 것 보다 모듈 키와 일반 키를 하나의 시스템으로 쓰는 것이 더 합리적입니다.

0.11 버전에서
"""""""

이전 버전의 Integrant은 설정을 변경하는 방법 외에 참조 키를 추가할 방법이 없었습니다. 모듈이 다른 키에
의존 한다면 자동으로 참조를 추가할 수 있는 방법이 필요합니다.

이를 위해 Integrant 0.7에 ``prep-key`` 멀티메서드가 추가되었습니다. ``prep-key`` 멀티메서드는
설정이 초기화되기 전에 ``prep`` 함수에 의해 불립니다. 이렇게 하면 설정을 작성한 후에도 참조를 추가해
의존성 순서를 만들 수 있습니다.

Duct 버전 0.11 이후 모듈은 다음과 같이 작성합니다:

.. code-block:: clojure

  (defmulti ig/prep-key ::module [_ options]
    (assoc options ::requires (ig/ref ::precondition)))

  (defmulti ig/init-key ::module [_ options]
    (fn [config] config))

prep 단계에 모듈이 필요한 다른 키에 대한 참조는 ``::requires`` 프라이빗 키에 추가 됩니다.
모듈이 초기화 될때 ``::precondition`` 키 다음에 ``::module`` 키가 초기화 되도록 보장해 줍니다.


프로필
~~~~~~~~

0.10 버전에서
"""""""

이전 버전의 duct에는 모듈 키와 일반 Integrant 키가 같은 설정에 있었습니다:

.. code-block:: clojure

  {:duct.core/project-ns foo
   :duct.module/example  {}}

하지만 이런 방식을 쓰면 초기화를 하기 전에 모듈 키와 일반 키가 분리되어야 하는 문제가 있습니다. 
또 모듈 키와 일반 키가 레퍼런스를 공유하면 문제가 될 수도 있습니다.

같은 설정 맵에 모듈 키와 일반 키가 함께 있는 것은 편리하지만 추상화를 깰 수 있습니다.


0.11 버전에서
"""""""

해결 방법은 모듈 키를 일반 키로 부터 명확하게 분리하는 것입니다. Duct 0.11 버전에는 모든 모듈 키에
이 방법을 적용되었습니다. 일반 키는 아래와 같이 프로필 안에 씁니다:

.. code-block:: clojure

  {:duct.profile/base   {:duct.core/project-ns foo}
   :duct.module/example {}}

프로필은 단순히 프로필 설정 값을 전체 설정 값에 meta-merge 하는 모듈 입니다.

그 결과 계층 사이에 명확히 분리된 계층 구조를 만듭니다: Duct 설정은 동작하는 시스템에 필요한 의존성과
함께 Integrant 설정을 만듭니다.

프로필이 다른 모듈이 시작되기 전에 실행 되는 것을 보장하려면 의존성을 정의할 방법이 필요합니다.
이 문제를 해결하기 위해 Integrant 0.7 버전에 refsets 기능이 추가되었습니다. Refset은 ref 처럼
동작하는데 매칭되는 모든 키의 set을 생성해 준다는 점이 ref와 다릅니다.

``:duct/profile`` 을 상속한 키가 실행 된 후에 ``:duct/module`` 을 상속한 키를 적용하기 위해
refset을 쓸 수 있습니다. ``duct.core`` 에 아래와 같이 정의 되어 있습니다:

.. code-block:: clojure

  (defmethod ig/prep-key :duct/module [_ profile]
    (assoc profile ::requires (ig/refset :duct/profile)))

ref와 refset, 키워드 상속 간에 복잡하지만 예측 가능한 의존성 그래프를 설정 할 수 있습니다.


Includes
~~~~~~~~

In 0.10
"""""""

In version 0.10 and below, includes were handled by a special key,
``:duct.core/include``:

.. code-block:: clojure

  {:duct.core/include ["example"]}

This will look for a resource named ``example.edn`` and meta-merge it
into the configuration.

There are two problems with this approach.

The first and most obvious issue is that it requires one key to have a
special function, one that isn't defined by a standard multimethod. It
cannot be a module because it's side-effectful.

The second issue is that it introduces new side-effects after the
configuration has been read. Ideally we want reading the configuration
to happen at the same step.

In 0.11
"""""""

In version 0.11 the ``:duct.core/include`` key is replaced with the
``#duct/include`` reader tag. The tag is replaced by the contents of
the referenced resource. If we want to merge it into the
configuration, we place it in a profile:

.. code-block:: clojure

  {:duct.profile/example #duct/include "example"]}

This ensures all the included configurations are read together by
``read-config``, and moves the complexities of merging into the
profiles.

This approach also allows smaller chunks of data to be included from
external files, rather than full configurations.


Summary
~~~~~~~

The changes represent an overall simplification of the module and
include system:

- Modules and normal components are separated.
- Modules no longer use their own dependency management.
- Merging is separated out into profiles.
- Including other configurations happens at read time.
- No keys with 'special' functionality.

In addition to the simplification, extra functionality has been added:

- ``prep-key`` removes the need for modules in simple cases
- ``refset`` allows for more sophisticated dependencies
