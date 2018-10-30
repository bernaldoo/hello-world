# hello-world
iroha test
defaults: &defaults
  working_directory: /opt/iroha
  environment:
    IROHA_HOME: /opt/iroha
    IROHA_BUILD: /opt/iroha/build

defaults: &version
  environment:
    IROHA_VERSION: ${CIRCLE_TAG:-0x$CIRCLE_SHA1}

version: 2

jobs:
  build:
    <<: *defaults
    <<: *version
    docker:
      - image: hyperledger/iroha-docker-develop:v1
      - image: postgres:9.5
    steps:
      - checkout
      - run: echo $(eval echo "$IROHA_VERSION")
      - restore_cache:
          keys:
            - build-linux-debug-{{ arch }}-{{ .Branch }}-
            - build-linux-debug-{{ arch }}-
            - build-linux-debug-
          paths:
            - ~/.ccache

      - run:
          name: ccache setup
          command: |
            ccache --version
            ccache --show-stats
            ccache --zero-stats
            # debug/coverage builds produce 2G cache
            ccache --max-size=2G
      - run:
          name: cmake
          command: >
            cmake \
              -DCOVERAGE=ON \
              -H$IROHA_HOME \
              -B$IROHA_BUILD \
              -DCMAKE_BUILD_TYPE=Debug \
              -DIROHA_VERSION=$(eval echo "$IROHA_VERSION")
      - run:
          name: make
          command: |
            cmake --build $IROHA_BUILD -- -j2
      - run: ccache --show-stats
      - run:
          name: unit tests, generate xunit-*.xml
          command: cmake --build $IROHA_BUILD --target test
      - run:
          name: coverage info
          command: cmake --build $IROHA_BUILD --target gcovr
      - run:
          name: cppcheck
          command: cmake --build $IROHA_BUILD --target cppcheck
      - run:
          name: codecov.io
          command: bash <(curl -s https://codecov.io/bash) -f $IROHA_BUILD/reports/gcovr.xml || echo "Codecov did not collect coverage reports"
      - save_cache:
          key: build-linux-debug-{{ arch }}-{{ .Branch }}-{{ epoch }}
          paths:
            - ~/.ccache
          when: always
      - persist_to_workspace:
          root: /opt/iroha/build
          paths:
            - compile_commands.json
            - reports



  build-linux-release:
    <<: *defaults
    <<: *version
    docker:
      - image: hyperledger/iroha-docker-develop:v1
    steps:
      - checkout
      - restore_cache:
          keys:
            - build-linux-release-{{ arch }}-{{ .Branch }}-
            - build-linux-release-{{ arch }}-
            - build-linux-release-
          paths:
            - ~/.ccache
      - run:
          name: ccache setup
          command: |
            ccache --version
            ccache --show-stats
            ccache --zero-stats
            # release builds produce cache <100M
            ccache --max-size=500M
      - run:
          name: cmake
          command: >
            cmake \
              -DCOVERAGE=OFF \
              -DTESTING=OFF \
              -H$IROHA_HOME \
              -B$IROHA_BUILD \
              -DCMAKE_BUILD_TYPE=Release \
              -DPACKAGE_DEB=ON \
              -DPACKAGE_TGZ=ON \
              -DIROHA_VERSION=$(eval echo "$IROHA_VERSION")
      - run:
          name: make and package
          command: |
            cmake --build $IROHA_BUILD --target package -- -j$(nproc --all)
      - run: ccache --show-stats
      - save_cache:
          key: build-linux-release-{{ arch }}-{{ .Branch }}-{{ epoch }}
          paths:
            - ~/.ccache
          when: always
      - run:
          name: copy deb file
          command: |
            IROHA_VERSION=$(eval echo "$IROHA_VERSION")
            cd $IROHA_BUILD
            # iroha-$IROHA_VERSION-Linux.{tar.gz, deb}, always matches 1 file
            cp iroha-*.deb ./iroha.deb
            cp iroha-*.tar.gz ./iroha.tar.gz
            # if IROHA_VERSION is not set, then CIRCLE_SHA1
            echo ${IROHA_VERSION:-$CIRCLE_SHA1} > version.txt
      - store_artifacts:
          path: build/iroha.deb
          destination: iroha-linux.deb
      - store_artifacts:
          path: build/iroha.tar.gz
          destination: iroha-linux.tar.gz
      - store_artifacts:
          path: build/version.txt
          destination: version.txt
      - persist_to_workspace:
          root: build
          paths:
            - iroha.deb



  dockerize:
    <<: *defaults
    <<: *version
    docker:
      - image: hyperledger/iroha-docker-develop:v1
      - image: docker:17.05.0-ce-git
    steps:
      - checkout
      - setup_remote_docker
      - attach_workspace:
          at: /opt/iroha/build
      - run:
          name: Copy deb
          command: |
            cp $IROHA_BUILD/iroha.deb $IROHA_HOME/docker/release/iroha.deb
      - run:
          name: Docker login
          command: |
            docker login -u "$DOCKER_USER" -p "$DOCKER_PASS"
      - run:
          name: Build and push iroha image
          command: |
            tag=
            if [ -n "$CIRCLE_TAG" ]; then
              # if CIRCLE_TAG is not empty
              tag="$CIRCLE_TAG"
            else
              if [ "$CIRCLE_BRANCH" == "develop" ]; then
                tag=develop
              elif [ "$CIRCLE_BRANCH" == "master" ]; then
                tag=latest
              fi
            fi
            docker build -t hyperledger/iroha-docker:$tag $IROHA_HOME/docker/release
            docker push hyperledger/iroha-docker:$tag
  # used to push info about issues to github
  sonar-pr:
    <<: *defaults
    <<: *version
    docker:
      - image: hyperledger/iroha-docker-develop:v1
    steps:
      - checkout
      - restore_cache:
          keys:
            - v3-sonar-
      - attach_workspace:
          at: /opt/iroha/build
      - run:
          name: execute sonar-scanner to analyze PR
          command: >
            if [ ! -z "$SONAR_TOKEN" ] && \
              [ ! -z "$SORABOT_TOKEN" ] && \
              [ ! -z "$CIRCLE_BUILD_NUM" ] && \
              [ ! -z "$CI_PULL_REQUEST" ]; then
              sonar-scanner \
                -Dsonar.github.disableInlineComments \
                -Dsonar.github.repository="hyperledger/iroha" \
                -Dsonar.analysis.mode=preview \
                -Dsonar.login=${SONAR_TOKEN} \
                -Dsonar.projectVersion="${CIRCLE_BUILD_NUM}" \
                -Dsonar.github.oauth="${SORABOT_TOKEN}" \
                -Dsonar.github.pullRequest="$(echo $CI_PULL_REQUEST | egrep -o "[0-9]+")"
            else
              echo "required env vars not found"
            fi
      - save_cache:
          key: v3-sonar-{{ epoch }}
          paths:
            - ~/.sonar
          when: always


  # executed only for develop, master and release branches
  sonar-release:
    <<: *defaults
    <<: *version
    docker:
      - image: hyperledger/iroha-docker-develop:v1
    steps:
      - checkout
      - restore_cache:
          keys:
            - v3-sonar-
      - attach_workspace:
          at: /opt/iroha/build
      - run:
          name: execute sonar-scanner
          command: >
            if [ ! -z "$SONAR_TOKEN" ] && \
              [ ! -z "$CIRCLE_BUILD_NUM" ] && \
              [ ! -z "$CIRCLE_BRANCH" ]; then
              sonar-scanner \
                -Dsonar.login="${SONAR_TOKEN}" \
                -Dsonar.projectVersion="${CIRCLE_BUILD_NUM}" \
                -Dsonar.branch="${CIRCLE_BRANCH}"
            else
              echo "required env vars not found"
            fi
      - save_cache:
          key: v3-sonar-{{ epoch }}
          paths:
            - ~/.sonar
          when: always


  build-macos-release:
    <<: *version
    macos:
      xcode: 9.0.0
    steps:
      - checkout
      - restore_cache:
          keys:
            - brew-
          paths:
            - ~/Library/Caches/Homebrew
      - run:
          name: install dev dependencies
          command: brew install cmake boost postgres grpc autoconf automake libtool ccache || true
      - save_cache:
          key: brew-{{ epoch }}
          paths:
            - ~/Library/Caches/Homebrew
          when: always
      - restore_cache:
          keys:
            - build-mac-release-{{ arch }}-{{ .Branch }}-
            - build-mac-release-{{ arch }}-
            - build-mac-release-
          paths:
            - ~/.ccache
      - restore_cache:
          keys:
            - mac-external-{{ arch }}-{{ .Branch }}-
            - mac-external-{{ arch }}-
            - mac-external-
          paths:
            - external
      - run:
          name: ccache setup
          command: |
            ccache --version
            ccache --show-stats
            ccache --zero-stats
            # release build produces <20M cache per build
            ccache --max-size=100M
      - run:
          name: cmake
          command: >
            cmake -H. -Bbuild \
              -DCOVERAGE=OFF \
              -DTESTING=OFF \
              -DPACKAGE_TGZ=ON \
              -DCMAKE_BUILD_TYPE=Release \
              -DIROHA_VERSION=$(eval echo "$IROHA_VERSION")
      - run:
          name: make
          command: |
            cmake --build build --target package -- -j$(sysctl -n hw.ncpu)
      - run: ccache --show-stats
      - save_cache:
          key: build-mac-release-{{ arch }}-{{ .Branch }}-{{ epoch }}
          paths:
            - ~/.ccache
          when: always
      - save_cache:
          key: mac-external-{{ arch }}-{{ .Branch }}-{{ epoch }}
          paths:
            - external
          when: always
      - run:
          name: rename artifacts
          command: |
            IROHA_VERSION=$(eval echo "$IROHA_VERSION")
            # iroha-$IROHA_VERSION-Darwin.tar.gz, always matches 1 file
            cp build/iroha-*.tar.gz build/iroha-darwin.tar.gz
            # if IROHA_VERSION is not set, then CIRCLE_SHA1
            echo ${IROHA_VERSION:-$CIRCLE_SHA1} > build/version.txt
      - store_artifacts:
          path: build/iroha-darwin.tar.gz
          destination: iroha-darwin.tar.gz
      - store_artifacts:
          path: build/version.txt
          destination: version.txt

     

workflows:
  version: 2
  full_pipeline:
    jobs:
      # build-macos-release + build -> sonar(pr/release)
      - build-macos-release
      - build
      - sonar-pr:
          requires:
            - build
          filters:
            branches:
              ignore:
                - develop
                - master
            tags:
              # do not invoke sonar-pr for tags
              ignore: /.*/
      # invoke sonar-release whenever anything to master, develop is pushed
      # or any commit tagged with v[\.0-9]+.*
      - sonar-release:
          requires:
            - build
          filters:
            branches:
              only:
                - develop
                - master
                - /trunk\/.*/
            tags:
              only: /v[\.0-9]+.*/

      # build-linux-release -> dockerize
      - build-linux-release:
          filters:
            branches:
              only:
                - develop
                - master
            tags:
              only: /v[\.0-9]+.*/
      - dockerize:
          requires:
            - build-linux-release
          filters:
            branches:
              only:
                - develop
                - master
            tags:
only: /v[\.0-9]+.*/
