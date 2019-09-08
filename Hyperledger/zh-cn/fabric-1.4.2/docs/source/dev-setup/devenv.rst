设置开发环境
--------------------------------------

概览
~~~~~~~~

Prior to the v1.0.0 release, the development environment utilized Vagrant
running an Ubuntu image, which in turn launched Docker containers as a
means of ensuring a consistent experience for developers who might be
working with varying platforms, such as macOS, Windows, Linux, or
whatever. Advances in Docker have enabled native support on the most
popular development platforms: macOS and Windows. Hence, we have
reworked our build to take full advantage of these advances. While we
still maintain a Vagrant based approach that can be used for older
versions of macOS and Windows that Docker does not support, we strongly
encourage that the non-Vagrant development setup be used.

Note that while the Vagrant-based development setup could not be used in
a cloud context, the Docker-based build does support cloud platforms
such as AWS, Azure, Google and IBM to name a few. Please follow the
instructions for Ubuntu builds, below.

前置工作
~~~~~~~~~~~~~

-  `Git 客户端 <https://git-scm.com/downloads>`__
-  `Go <https://golang.org/dl/>`__ - 版本 1.11.x
-  (macOS)
   `Xcode <https://itunes.apple.com/us/app/xcode/id497799835?mt=12>`__
   必须安装
-  `Docker <https://www.docker.com/get-docker>`__ - 17.06.2-ce 或者更高
-  `Docker Compose <https://docs.docker.com/compose/>`__ - 1.14.0 或者更高
-  (macOS) 你必须安装gnutar,macOS默认使用bsdtar，
   但是构建使用了一些gnutar的标识。 
   你可以使用Homebrew来安装它:

::

    brew install gnu-tar --with-default-names

-  (macOS) `Libtool <https://www.gnu.org/software/libtool/>`__ 。
    你可以使用Homebrew来安装它：

::

    brew install libtool

-  (只有使用Vagrant的需要安装) - `Vagrant <https://www.vagrantup.com/>`__ -
   1.9 或者更高
-  (只有使用Vagrant的需要安装) -
   `VirtualBox <https://www.virtualbox.org/>`__ - 5.0 或者更高
-  BIOS 支持虚拟化 - 因硬件而异

-  注意: BIOS虚拟化设置可能在BIOS的CPU或者Security setting里

``pip``
~~~~~~

::

    pip install --upgrade pip


步骤
~~~~~

设置GOPATH
^^^^^^^^^^^^^^^

确保你已经设置了你主机的 
`GOPATH 环境变量 <https://github.com/golang/go/wiki/GOPATH>`__ 。
只有设置了环境变量才能在你的主机或者虚拟机中编译构建。

如果你安装的Go不在默认位置，确保你设置了
 `GOROOT环境变量 <https://golang.org/doc/install#install>`__ 。

Windows用户请注意
^^^^^^^^^^^^^^^^^^^^^

如果你用的是Windows系统，在运行 ``git clone`` 命令之前，请先运行下列命令。

::

    git config --get core.autocrlf

如果 ``core.autocrlf`` 设置 ``true``, 你必须将它设置为 ``false`` 

::

    git config --global core.autocrlf false

如果你继续设置 ``core.autocrlf`` 为 ``true`` ，
``vagrant up`` 命令将会报错:


``./setup.sh: /bin/bash^M: bad interpreter: No such file or directory``

克隆Hyperledger Fabric项目源代码
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

因为Hyperledger Fabric是用 ``Go`` 写的，所以你需要将它克隆到$GOPATH/src目录。
如果你的$GOPATH有多个，那么请选择第一个$GOPATH。
这里有一些需要设置：

::

    cd $GOPATH/src
    mkdir -p github.com/hyperledger
    cd github.com/hyperledger

重申一下，我们使用 ``Gerrit`` 来做代码的控制， ``Gerrit`` 内部有自己的git仓库。
因此我们需要从
:doc:`Gerrit <../Gerrit/gerrit>` 来克隆。

::

    git clone ssh://LFID@gerrit.hyperledger.org:29418/fabric && scp -p -P 29418 LFID@gerrit.hyperledger.org:hooks/commit-msg fabric/.git/hooks/

**注意:** 当然你要用将 ``LFID`` 替换为你的
:doc:`Linux Foundation ID <../Gerrit/lf-account>` 。

Bootstrapping the VM using Vagrant
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If you are planning on using the Vagrant developer environment, the
following steps apply. **Again, we recommend against its use except for
developers that are limited to older versions of macOS and Windows that
are not supported by Docker for Mac or Windows.**

::

    cd $GOPATH/src/github.com/hyperledger/fabric/devenv
    vagrant up

Go get coffee... this will take a few minutes. Once complete, you should
be able to ``ssh`` into the Vagrant VM just created.

::

    vagrant ssh

Once inside the VM, you can find the source under
``$GOPATH/src/github.com/hyperledger/fabric``. It is also mounted as
``/hyperledger``.

Building Hyperledger Fabric
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Once you have all the dependencies installed, and have cloned the
repository, you can proceed to :doc:`build and test <build>` Hyperledger
Fabric.

Notes
~~~~~

**NOTE:** Any time you change any of the files in your local fabric
directory (under ``$GOPATH/src/github.com/hyperledger/fabric``), the
update will be instantly available within the VM fabric directory.

**NOTE:** If you intend to run the development environment behind an
HTTP Proxy, you need to configure the guest so that the provisioning
process may complete. You can achieve this via the *vagrant-proxyconf*
plugin. Install with ``vagrant plugin install vagrant-proxyconf`` and
then set the VAGRANT\_HTTP\_PROXY and VAGRANT\_HTTPS\_PROXY environment
variables *before* you execute ``vagrant up``. More details are
available here: https://github.com/tmatilai/vagrant-proxyconf/

**NOTE:** The first time you run this command it may take quite a while
to complete (it could take 30 minutes or more depending on your
environment) and at times it may look like it's not doing anything. As
long you don't get any error messages just leave it alone, it's all
good, it's just cranking.

**NOTE to Windows 10 Users:** There is a known problem with vagrant on
Windows 10 (see
`hashicorp/vagrant#6754 <https://github.com/hashicorp/vagrant/issues/6754>`__).
If the ``vagrant up`` command fails it may be because you do not have
the Microsoft Visual C++ Redistributable package installed. You can
download the missing package at the following address:
http://www.microsoft.com/en-us/download/details.aspx?id=8328

**NOTE:** The inclusion of the miekg/pkcs11 package introduces
an external dependency on the ltdl.h header file during
a build of fabric. Please ensure your libtool and libltdl-dev packages
are installed. Otherwise, you may get a ltdl.h header missing error.
You can download the missing package by command:
``sudo apt-get install -y build-essential git make curl unzip g++ libtool``.

.. Licensed under Creative Commons Attribution 4.0 International License
   https://creativecommons.org/licenses/by/4.0/

