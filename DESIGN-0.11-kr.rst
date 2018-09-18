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


Profiles
~~~~~~~~

In 0.10
"""""""

In previous versions of Duct, modules and normal Integrant keys
existed within the same configuration:

.. code-block:: clojure

  {:duct.core/project-ns foo
   :duct.module/example  {}}

However, this meant that the configuration had to be split into module
keys and non-module keys before it could be initiated. It also lead to
problems if module keys and non-module keys shared refs.

While it's convenient to have module and non-module keys in the same
configuration map, it also produces a leaky abstraction.


In 0.11
"""""""

The solution to this is to explicitly separate module keys from
non-module keys. The way Duct handles this in version 0.11 is to make
every key in the configuration a module. Non-module keys are placed
into profiles, like so:

.. code-block:: clojure

  {:duct.profile/base   {:duct.core/project-ns foo}
   :duct.module/example {}}

Profiles are just modules that meta-merge their value into the
configuration.

This results in a tiered structure with a clear separation between
tiers: a Duct configuration produces an Integrant configuration, which
is then used to create a running system of dependent components.

To ensure that profiles are run before any other module, we need a way
of defining a set of dependencies. Integrant 0.7 introduces refsets to
solve this problem. Refsets act like refs, except they produce a set
of all matching keys.

We can use refsets to ensure that any key derived from
``:duct/module`` must be applied after a key deriving from
``:duct/profile``. In ``duct.core`` there is the following definition
that does exactly that:

.. code-block:: clojure

  (defmethod ig/prep-key :duct/module [_ profile]
    (assoc profile ::requires (ig/refset :duct/profile)))

Between refs, refsets and keyword inheritance, we can set up
sophisticated but predictable dependency graphs.


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
