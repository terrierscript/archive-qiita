
2018-11-19追記
https://github.com/CircleCI-Public/circleci-dockerfiles/blob/master/ruby/images/
いつの間にか公開されてました

# 動機
* circleciが提供しているdockerファイルで、rubyのバージョンとして、`node`, `browswer` , `node-browsers`とかがあった
	* [setup-project](https://circleci.com/setup-project)したときのデフォルトは`node-browsers`だった
	* 地味にsetup projectでボイラープレート出てくるのも知らなかった。。。
* なんとなくフロント好きおじさんからするとnodeってバージョン何入ってるんだ！が気になった。

# dockerの中身が見たい。
残念ながらDockerfileは公開されていないっぽい

https://discuss.circleci.com/t/official-ruby-dockerfile/12853/2

ここで`docker history`コマンドがあると知った。

```
$ docker pull circleci/ruby:2.4.1-node-browsers
$ docker history --no-trunc --format "{{.CreatedBy}}" circleci/ruby:2.4.1-node-browsers
```

今回はコマンド何やってるか知れれば良いだけなのでformatかける

## `circleci/ruby:2.4.1-node-browsers`は何が入っているっぽいか荒目に見る
historyなので、実行が逆順。先頭の方が後に行っているコマンド

### entrypoint

`docker-entrypoint.sh`はssh接続したら見れそうだけど、今のところ興味ないのでスルー

```
#(nop)  CMD ["/bin/sh"]
#(nop)  ENTRYPOINT ["/docker-entrypoint.sh"]
#(nop)  LABEL com.circleci.preserve-entrypoint=true
```

### browser関連インストール

怒号感ある。

```
printf '#!/bin/sh\nXvfb :99 -screen 0 1280x1024x24 &\nexec "$@"\n' > /tmp/entrypoint  && chmod +x /tmp/entrypoint         && sudo mv /tmp/entrypoint /docker-entrypoint.sh
#(nop)  ENV DISPLAY=:99
export CHROMEDRIVER_RELEASE=$(curl --location --fail --retry 3 http://chromedriver.storage.googleapis.com/LATEST_RELEASE)       && curl --silent --show-error --location --fail --retry 3 --output /tmp/chromedriver_linux64.zip "http://chromedriver.storage.googleapis.com/$CHROMEDRIVER_RELEASE/chromedriver_linux64.zip"       && cd /tmp       && unzip chromedriver_linux64.zip       && rm -rf chromedriver_linux64.zip       && sudo mv chromedriver /usr/local/bin/chromedriver       && sudo chmod +x /usr/local/bin/chromedriver
curl --silent --show-error --location --fail --retry 3 --output /tmp/google-chrome-stable_current_amd64.deb https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb       && (sudo dpkg -i /tmp/google-chrome-stable_current_amd64.deb || sudo apt-get -fy install)        && rm -rf /tmp/google-chrome-stable_current_amd64.deb       && sudo sed -i 's|HERE/chrome"|HERE/chrome" --disable-setuid-sandbox --no-sandbox|g'            "/opt/google/chrome/google-chrome"
curl --silent --show-error --location --fail --retry 3 --output /tmp/firefox.deb https://s3.amazonaws.com/circle-downloads/firefox-mozilla-build_47.0.1-0ubuntu1_amd64.deb   && echo 'ef016febe5ec4eaf7d455a34579834bcde7703cb0818c80044f4d148df8473bb  /tmp/firefox.deb' | sha256sum -c   && sudo dpkg -i /tmp/firefox.deb || sudo apt-get -f install    && sudo apt-get install -y libgtk3.0-cil-dev   && rm -rf /tmp/firefox.deb
export PHANTOMJS_VERSION=$(curl --location --fail --retry 3  https://api.github.com/repos/ariya/phantomjs/tags | jq -r '.[0].name')   && sudo apt-get update; sudo apt-get install libfontconfig   && curl --silent --show-error --location --fail --retry 3 --output /tmp/phantomjs.tar.bz2 https://bitbucket.org/ariya/phantomjs/downloads/phantomjs-${PHANTOMJS_VERSION}-linux-x86_64.tar.bz2   && tar -x -C /tmp -f /tmp/phantomjs.tar.bz2   && sudo mv /tmp/phantomjs-${PHANTOMJS_VERSION}-linux-x86_64/bin/phantomjs /usr/local/bin   && rm -rf /tmp/phantomjs.tar.bz2 /tmp/phantomjs-*
#(nop)  USER [circleci]
```

こんな感じが読み取れる

* xvfbが入ってるっぽい
* chromedriver入れてる
* chrome入れてる
* firefox入れてる
* phantomjs入れてる

### node & yarnインストール

```
set -ex   && for key in     6A010C5166006599AA17F08146C2130DFD2497F5   ; do     gpg --keyserver pgp.mit.edu --recv-keys "$key" ||     gpg --keyserver keyserver.pgp.com --recv-keys "$key" ||     gpg --keyserver ha.pool.sks-keyservers.net --recv-keys "$key" ;   done   && curl -fSLO --compressed "https://yarnpkg.com/downloads/$YARN_VERSION/yarn-v$YARN_VERSION.tar.gz"   && curl -fSLO --compressed "https://yarnpkg.com/downloads/$YARN_VERSION/yarn-v$YARN_VERSION.tar.gz.asc"   && gpg --batch --verify yarn-v$YARN_VERSION.tar.gz.asc yarn-v$YARN_VERSION.tar.gz   && mkdir -p /opt/yarn   && tar -xzf yarn-v$YARN_VERSION.tar.gz -C /opt/yarn --strip-components=1   && ln -s /opt/yarn/bin/yarn /usr/local/bin/yarn   && ln -s /opt/yarn/bin/yarn /usr/local/bin/yarnpkg   && rm yarn-v$YARN_VERSION.tar.gz.asc yarn-v$YARN_VERSION.tar.gz
#(nop)  ENV YARN_VERSION=0.24.6
curl -SLO "https://nodejs.org/dist/v$NODE_VERSION/node-v$NODE_VERSION-linux-x64.tar.xz"   && curl -SLO --compressed "https://nodejs.org/dist/v$NODE_VERSION/SHASUMS256.txt.asc"   && gpg --batch --decrypt --output SHASUMS256.txt SHASUMS256.txt.asc   && grep " node-v$NODE_VERSION-linux-x64.tar.xz\$" SHASUMS256.txt | sha256sum -c -   && tar -xJf "node-v$NODE_VERSION-linux-x64.tar.xz" -C /usr/local --strip-components=1   && rm "node-v$NODE_VERSION-linux-x64.tar.xz" SHASUMS256.txt.asc SHASUMS256.txt   && ln -s /usr/local/bin/node /usr/local/bin/nodejs
#(nop)  ENV NODE_VERSION=8.1.4
#(nop)  ENV NPM_CONFIG_LOGLEVEL=info
```

* yarn入れてる
* node入れてる
    * nodeが8.1ということも判明

なるほど。このへんまでで`-node-browswer`とわかった。わりと想像通り。

### dockerなどツール系インストール

```
set -ex   && for key in     9554F04D7259F04124DE6B476D5A82AC7E37093B     94AE36675C464D64BAFA68DD7434390BDBE9B9C5     FD3A5288F042B6850C66B31F09FE44734EB7990E     71DCFD284A79C3B38668286BC97EC7A07EDE3FC1     DD8F2338BAE7501E3DD5AC78C273792F7D83545D     B9AE9905FFD7803F25714661B63B535A4C206CA9     C4F0DFFF4E8C1A8236409D08E73BC641CC11F4C8     56730D5401028683275BD23C23EFEFE93C4CFFFE   ; do     gpg --keyserver pgp.mit.edu --recv-keys "$key" ||     gpg --keyserver keyserver.pgp.com --recv-keys "$key" ||     gpg --keyserver ha.pool.sks-keyservers.net --recv-keys "$key" ;   done
groupadd --gid 1000 node   && useradd --uid 1000 --gid node --shell /bin/bash --create-home node
#(nop)  USER [root]
#(nop)  CMD ["/bin/sh"]
#(nop)  USER [circleci]
groupadd --gid 3434 circleci   && useradd --uid 3434 --gid circleci --shell /bin/bash --create-home circleci   && echo 'circleci ALL=NOPASSWD: ALL' >> /etc/sudoers.d/50-circleci   && echo 'Defaults    env_keep += "DEBIAN_FRONTEND"' >> /etc/sudoers.d/env_keep
DOCKERIZE_URL=$(curl --location --fail --retry 3 https://api.github.com/repos/jwilder/dockerize/releases/latest | jq -r '.assets[] | select(.name | startswith("dockerize-linux-amd64")) | .browser_download_url')   && curl --silent --show-error --location --fail --retry 3 --output /tmp/dockerize-linux-amd64.tar.gz $DOCKERIZE_URL   && tar -C /usr/local/bin -xzvf /tmp/dockerize-linux-amd64.tar.gz   && rm -rf /tmp/dockerize-linux-amd64.tar.gz
COMPOSE_URL=$(curl --location --fail --retry 3 https://api.github.com/repos/docker/compose/releases/latest | jq -r '.assets[] | select(.name == "docker-compose-Linux-x86_64") | .browser_download_url')   && curl --silent --show-error --location --fail --retry 3 --output /usr/bin/docker-compose $COMPOSE_URL   && chmod +x /usr/bin/docker-compose
set -ex   && export DOCKER_VERSION=$(curl --silent --fail --retry 3 https://get.docker.com/builds/  | grep -P -o 'docker-\d+\.\d+\.\d+-ce\.tgz' | head -n 1)   && DOCKER_URL="https://get.docker.com/builds/Linux/x86_64/${DOCKER_VERSION}"   && echo Docker URL: $DOCKER_URL   && curl --silent --show-error --location --fail --retry 3 --output /tmp/docker.tgz "${DOCKER_URL}"   && ls -lha /tmp/docker.tgz   && tar -xz -C /tmp -f /tmp/docker.tgz   && mv /tmp/docker/* /usr/bin   && rm -rf /tmp/docker /tmp/docker.tgz
JQ_URL=$(curl --location --fail --retry 3 https://api.github.com/repos/stedolan/jq/releases/latest  | grep browser_download_url | grep '/jq-linux64"' | grep -o -e 'https.*jq-linux64')   && curl --silent --show-error --location --fail --retry 3 --output /usr/bin/jq $JQ_URL   && chmod +x /usr/bin/jq
```

docker関連とか色々瑣末なやつ入れている
* jq
    * jsonパーサー
* dockerize
    * https://github.com/jwilder/dockerize 
    * 専門知識が薄い部分なのでよくわかってないが、コンテナの起動関連？
* compose
    * https://github.com/docker/compose/
    * docker-compose。複数のdockerを扱うやつ

### apt-get

gitやxvfbなどapt-getしてる

```
#(nop)  ENV LANG=C.UTF-8
locale-gen C.UTF-8 || true
ln -sf /usr/share/zoneinfo/Etc/UTC /etc/localtime
apt-get update   && apt-get install -y     git mercurial xvfb     locales sudo openssh-client ca-certificates tar gzip parallel     net-tools netcat unzip zip bzip2
#(nop)  ENV DEBIAN_FRONTEND=noninteractive
echo 'APT::Get::Assume-Yes "true";' > /etc/apt/apt.conf.d/90circleci   && echo 'APT::Get::force-Yes "true";' >> /etc/apt/apt.conf.d/90circleci   && echo 'DPkg::Options "--force-confnew";' >> /etc/apt/apt.conf.d/90circleci
```

* apt-getで色々入れてる
    * git / mercurial 
    * xvfb
    * locales / sudo
    * openssh-client / ca-certificates
    * tar / gzip parallel 
    * net-tools / netcat 
    * unzip / zip / bzip2
* このへんはdockerhubに書いてる感じ
    * https://hub.docker.com/r/circleci/ruby/

### rubyインストール

```
#(nop)  CMD ["irb"]
mkdir -p "$GEM_HOME" "$BUNDLE_BIN"  && chmod 777 "$GEM_HOME" "$BUNDLE_BIN"
#(nop)  ENV PATH=/usr/local/bundle/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
#(nop)  ENV BUNDLE_PATH=/usr/local/bundle BUNDLE_BIN=/usr/local/bundle/bin BUNDLE_SILENCE_ROOT_WARNING=1 BUNDLE_APP_CONFIG=/usr/local/bundle
#(nop)  ENV GEM_HOME=/usr/local/bundle
gem install bundler --version "$BUNDLER_VERSION"
#(nop)  ENV BUNDLER_VERSION=1.15.1
set -ex   && buildDeps='   bison   dpkg-dev   libgdbm-dev   ruby  '  && apt-get update  && apt-get install -y --no-install-recommends $buildDeps  && rm -rf /var/lib/apt/lists/*   && wget -O ruby.tar.xz "https://cache.ruby-lang.org/pub/ruby/${RUBY_MAJOR%-rc}/ruby-$RUBY_VERSION.tar.xz"  && echo "$RUBY_DOWNLOAD_SHA256 *ruby.tar.xz" | sha256sum -c -   && mkdir -p /usr/src/ruby  && tar -xJf ruby.tar.xz -C /usr/src/ruby --strip-components=1  && rm ruby.tar.xz   && cd /usr/src/ruby   && {   echo '#define ENABLE_PATH_CHECK 0';   echo;   cat file.c;  } > file.c.new  && mv file.c.new file.c   && autoconf  && gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"  && ./configure   --build="$gnuArch"   --disable-install-doc   --enable-shared  && make -j "$(nproc)"  && make install   && apt-get purge -y --auto-remove $buildDeps  && cd /  && rm -r /usr/src/ruby   && gem update --system "$RUBYGEMS_VERSION"
#(nop)  ENV RUBYGEMS_VERSION=2.6.12
#(nop)  ENV RUBY_DOWNLOAD_SHA256=4fc8a9992de3e90191de369270ea4b6c1b171b7941743614cc50822ddc1fe654
#(nop)  ENV RUBY_VERSION=2.4.1
#(nop)  ENV RUBY_MAJOR=2.4
mkdir -p /usr/local/etc  && {   echo 'install: --no-document';   echo 'update: --no-document';  } >> /usr/local/etc/gemrc
set -ex;  apt-get update;  apt-get install -y --no-install-recommends   autoconf   automake   bzip2   file   g++   gcc   imagemagick   libbz2-dev   libc6-dev   libcurl4-openssl-dev   libdb-dev   libevent-dev   libffi-dev   libgdbm-dev   libgeoip-dev   libglib2.0-dev   libjpeg-dev   libkrb5-dev   liblzma-dev   libmagickcore-dev   libmagickwand-dev   libncurses-dev   libpng-dev   libpq-dev   libreadline-dev   libsqlite3-dev   libssl-dev   libtool   libwebp-dev   libxml2-dev   libxslt-dev   libyaml-dev   make   patch   xz-utils   zlib1g-dev     $(    if apt-cache show 'default-libmysqlclient-dev' 2>/dev/null | grep -q '^Version:'; then     echo 'default-libmysqlclient-dev';    else     echo 'libmysqlclient-dev';    fi   )  ;  rm -rf /var/lib/apt/lists/*
```
```
apt-get update && apt-get install -y --no-install-recommends   bzr   git   mercurial   openssh-client   subversion     procps  && rm -rf /var/lib/apt/lists/*
apt-get update && apt-get install -y --no-install-recommends   ca-certificates   curl   wget  && rm -rf /var/lib/apt/lists/*
#(nop)  CMD ["bash"]
#(nop) ADD file:9c48682ff75c756544d4491472081a078edf5dd0bb5038d1cb850a1f9c480e3e in / 
```

* ruby系のインストール
* bundlerのインストール
* apt-getでビルドに必要なもの入れているっぽい

# 結論
* `circleci/ruby:2.4.1-node-browsers`はわりと名前の通り
    * xvfbとかphantomjsとかchromeとかそのへん使う場合に有益っぽい
    * nodeは8系。stableを使うポリシーなのかedgeを利用するポリシーなのかとかは不明
	    * 地味に一番知りたかったのはこれだった。目的達成
* 他のtagは？
    * 普通にそれぞれが必要な分入ってる
    * `circleci/ruby:2.4.1-node`だとchromeとかxvfb入ってない
    * `circleci/ruby:2.4.1-browsers`だとnodeとかyarn入らない

