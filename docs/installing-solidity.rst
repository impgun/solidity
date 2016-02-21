##################
Установка Solidity
##################

Solidity в браузере
===================

Если вы просто хотите попробовать Solidity с небольшими контрактами, вы можете попробовать `browser-solidity <https://chriseth.github.io/browser-solidity>`_, который вообще не нужно устанавливать. Если вы хотите использовать его без подключения к Интернету, вы также можете просто сохранить страницу локально или клонировать репозиторий http://github.com/chriseth/browser-solidity.

NPM / node.js
=============

Пожалуй, это наиболее портативный и удобный способ установки Solidity на локальном устройстве.

Платформенно-независимая библиотека JavaScript предоставляется путем компиляции исходного кода C++ в JavaScript с помощью Emscripten для browser-solidity, а также доступен пакет NPM.

Чтобы установить его, просто введите команду::

    npm install solc

Подробности использования пакета nodejs можно найти в `repository <https://github.com/chriseth/browser-solidity#nodejs-usage>`_.

Двоичные пакеты
===============

Двоичные пакеты Solidity вместе с IDE Mix доступны в `C++ bundle <https://github.com/ethereum/webthree-umbrella/releases>`_ Эфириума.

Сборка из исходных файлов
=========================

Сборка Solidity довольно похожа в MacOS X, Ubuntu и, вероятно, других Unix. Это руководство начинается с объяснения того, как установить зависимости для каждой платформы, а затем показывает, как собрать сам Solidity.

MacOS X
-------


Требования:

- OS X Yosemite (10.10.5)
- Homebrew
- Xcode

Настройте Homebrew:

.. code-block:: bash

    brew update
    brew install boost --c++11             # this takes a while
    brew install cmake cryptopp miniupnpc leveldb gmp libmicrohttpd libjson-rpc-cpp 
    # Только для IDE Mix и Alethzero
    brew install xz d-bus
    brew install llvm --HEAD --with-clang 
    brew install qt5 --with-d-bus          # add --verbose if long waits with a stale screen drive you crazy as well

Ubuntu
------

Ниже приведены инструкции по сборке для новейших версий Ubuntu. На декабрь 2014-? года лучше всего поддерживается 64-разрядная Ubuntu 14.04 с как минимум 2 ГБ оперативной памяти. Все наши тесты выполняются в этой версии. Вклады сообщества с инструкциями для других версий приветствуются!

Установка зависимостей:

Прежде чем вы сможете собрать проект из исходных файлов, вам нужно несколько инструментов и зависимостей для приложения.

Первым делом обновите свои репозитории. Не все пакеты предоставляются через основной репозиторий Ubuntu, и те вы получите из Ethereum PPA и архива LLVM.

.. note::

    Пользователям Ubuntu 14.04 потребуется новейшая версия cmake. Чтобы добвить ее, введите следующую команду:
    `sudo apt-add-repository ppa:george-edison55/cmake-3.x`

Теперь добавьте остальное:

.. code-block:: bash

    sudo apt-get -y update
    sudo apt-get -y install language-pack-en-base
    sudo dpkg-reconfigure locales
    sudo apt-get -y install software-properties-common
    sudo add-apt-repository -y ppa:ethereum/ethereum
    sudo add-apt-repository -y ppa:ethereum/ethereum-dev
    sudo apt-get -y update
    sudo apt-get -y upgrade

Для Ubuntu 15.04 (Vivid Vervet) или более старых версий введите следующую команду, чтобы добавить пакеты разработки:

.. code-block:: bash

    sudo apt-get -y install build-essential git cmake libboost-all-dev libgmp-dev libleveldb-dev libminiupnpc-dev libreadline-dev libncurses5-dev libcurl4-openssl-dev libcryptopp-dev libjson-rpc-cpp-dev libmicrohttpd-dev libjsoncpp-dev libedit-dev libz-dev

Для Ubuntu 15.10 (Wily Werewolf) или более новых версий используйте вместо этого следующую команду:

.. code-block:: bash

    sudo apt-get -y install build-essential git cmake libboost-all-dev libgmp-dev libleveldb-dev libminiupnpc-dev libreadline-dev libncurses5-dev libcurl4-openssl-dev libcryptopp-dev libjsonrpccpp-dev libmicrohttpd-dev libjsoncpp-dev libedit-dev libz-dev
    
Причина изменения в том, что `libjsonrpccpp-dev` доступна в более новых версиях Ubuntu в universe repository.

Сборка
--------

Если вы хотите установить только Solidity, выполните следующую команду; ошибки в конце проигнорируте, потому что они относятся только к Alethzero и Mix

.. code-block:: bash

    git clone --recursive https://github.com/ethereum/webthree-umbrella.git
    cd webthree-umbrella
    ./webthree-helpers/scripts/ethupdate.sh --no-push --simple-pull --project solidity # update Solidity repo
    ./webthree-helpers/scripts/ethbuild.sh --no-git --project solidity --all --cores 4 -DEVMJIT=0 # build Solidity and others
                                                                                #enabling DEVMJIT on OS X will not build
                                                                                #feel free to enable it on Linux 

Если вы хотите установить Alethzero и Mix, выполните:

.. code-block:: bash

    git clone --recursive https://github.com/ethereum/webthree-umbrella.git
    cd webthree-umbrella && mkdir -p build && cd build
    cmake ..

Если вы хотите помочь разрабатывать Solidity, вам следует форкнуть Solidity и добавить свой личный форк как второй remote:

.. code-block:: bash

    cd webthree-umbrella/solidity
    git remote add personal git@github.com:username/solidity.git

Имейте в виду, что webthree-umbrella использует субмодули, так что solidity это собственный репозиторий git, но его параметры хранятся не в файле `.git/config`, а в файле `webthree-umbrella/.git/modules/solidity/config`.


