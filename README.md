# UEFI-OS-build


Ubuntu 20.04(Virtual Box上)

ビルド手順

1. ビルド環境の構築
2. OS のソースコードの入手
3. ブートローダーのビルド
4. OS のビルド

## ビルド環境の構築

ブートローダーおよび OS 本体のビルドに必要なツールやファイルを揃える。

### リポジトリのダウンロード

まずは Git をインストールして，このリポジトリをダウンロードする。

    $ sudo apt update
    $ sudo apt install git
    $ cd $HOME
    $ git clone https://github.com/murata0531/UEFI-OS-build.git　osbook

(余談) ansible playbookに準えてosbookと命名している

### 開発ツールの導入

次に Clang，Nasm といった開発ツールや，EDK IIのセットアップを行う。
`ansible_provision.yml` に必要なツールが記載されている。
Ansible を使ってセットアップを行うと楽。

注意）ansible_provision.yml は LLVM7 をデフォルトに設定する。これは Ubuntu の alternatives という仕組みを使い、/usr/bin 以下にリンクを張ることで実現する。

    $ sudo apt install ansible
    $ cd $HOME/osbook/devenv
    $ ansible-playbook -K -i ansible_inventory ansible_provision.yml

セットアップが上手くいけば `iasl` というコマンドがインストールされ，`$HOME/edk2` というディレクトリが生成されている。

    $ iasl -v
    $ ls $HOME/edk2

### 標準ライブラリのライセンスについて

加えて，上記の `ansible-playbook` コマンドはビルド済みの標準ライブラリを `$HOME/os/devenv/x86_64-elf` にダウンロードする。

このディレクトリに含まれるファイルは [Newlib](https://sourceware.org/newlib/)，[libc++](https://libcxx.llvm.org/)，[FreeType](https://www.freetype.org/) をビルドしたもの。
それらのライセンスはそれぞれのライブラリ固有のライセンスに従う。

次のファイル群は Newlib 由来。ライセンスは `x86_64-elf/LICENSE.newlib` を参照。

    x86_64-elf/lib/
        libc.a
        libg.a
        libm.a
    x86_64-elf/include/
        c++/ を除くすべて

次のファイル群は libc++ 由来。ライセンスは `x86_64-elf/LICENSE.libcxx` を参照。

    x86_64-elf/lib/
        libc++.a
        libc++abi.a
        libc++experimental.a
    x86_64-elf/include/c++/v1/
        すべて

次のファイル群は FreeType 由来。ライセンスは `x86_64-elf/LICENSE.freetype` を参照。

    x86_64-elf/lib/
        libfreetype.a
    x86_64-elf/include/freetype2/
        すべて

## OS のソースコードの入手

Git で入手。

    $ git clone https://github.com/murata0531/UEFI-OS.git

最後の `git clone` によって、カレントディレクトリに EUFI-OS ディレクトリが生成され、そこに OS のソースコードがダウンロードされる。

## ブートローダーのビルド

EDK II のディレクトリに EUFI-OSブートローダーのディレクトリをリンクする。

    $ cd $HOME/edk2
    $ ln -s /path/to/UEFI-OS/MikanLoaderPkg ./

`/path/to/EUFI-OS` は先ほど `git clone` でダウンロードした UEFI-OSディレクトリへのパスを指定する。
ブートローダーのソースコードが正しく見られたらリンク成功。

    $ ls MikanLoaderPkg/Main.c

次に，`edksetup.sh` を読み込むことで EDK II のビルドに必要な環境変数を設定する。

    $ source edksetup.sh

`edksetup.sh` ファイルを読み込むと，環境変数が設定される他に `Conf/target.txt` が自動的に生成される。
このファイルをエディタで開き，次の項目を修正する。

| 設定項目        | 設定値                            |
|-----------------|-----------------------------------|
| ACTIVE_PLATFORM | MikanLoaderPkg/MikanLoaderPkg.dsc |
| TARGET          | DEBUG                             |
| TARGET_ARCH     | X64                               |
| TOOL_CHAIN_TAG  | CLANG38                           |

設定が終わったらブートローダーをビルドする。

    $ build
    
※「ModuleNotFoundError: No module named 'distutils.util'」というようなエラーが出る可能性がある。その際は `sudo apt install python3-distutils` として、python3-distutils パッケージをインストールしてから、再度 `build` を実行する。

Loader.efi ファイルが出力されていればビルド成功。

    $ ls Build/MikanLoaderX64/DEBUG_CLANG38/X64/Loader.efi

## OS のビルド

ビルドに必要な環境変数を読み込む。

    $ source $HOME/osbook/devenv/buildenv.sh

ビルドする。

    $ cd /path/to/UEFI-OS
    $ source build.sh

QEMU で起動するには `./build.sh` に `run` オプションを指定。

    $ source build.sh run

apps ディレクトリにアプリ群を入れ、フォントなどのリソースをも含めたディスクイメージを作るには APPS_DIR と RESOURCE_DIR 変数を指定。

    $ APPS_DIR=apps RESOURCE_DIR=resources source build.sh run
