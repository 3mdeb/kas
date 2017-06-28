User Guide
==========

Dependencies & installation
---------------------------

This projects depends on

- Python 3
- distro Python 3 package
- PyYAML Python 3 package (optional, for yaml file support)

If you need Python 2 support consider sending patches. The most
obvious place to start is to use the trollius package intead of
asyncio.

To install kas into your python site-package repository, run::

    $ sudo pip3 install .


Usage
-----

There are three options for using kas:

- Install it locally via pip to get the ``kas`` command.
- Use the docker image. In this case run the commands in the examples
  below within ``docker run -it <kas-image> sh`` or bind-mount the project into
  the container.
- Use the **run-kas** wrapper from this directory. In this case replace ``kas``
  in the examples below with ``path/to/run-kas``.

Start build::

    $ kas build /path/to/kas-project.yml

Alternatively, experienced bitbake users can invoke usual **bitbake** steps
manually, e.g.::

    $ kas shell /path/to/kas-project.yml -c 'bitbake dosfsutils-native'

kas will place downloads and build artifacts under the current directory when
being invoked. You can specify a different location via the environment
variable `KAS_WORK_DIR`.

Command line usage
~~~~~~~~~~~~~~~~~~

.. argparse::
    :module: kas.kas
    :func: kas_get_argparser
    :prog: kas

Environment variables
~~~~~~~~~~~~~~~~~~~~~

=============================================  ================================
Environment variable name                      Description
=============================================  ================================
``KAS_WORK_DIR``                               The path of the kas work
                                               directory, current work
                                               directory is the default.
``KAS_REPO_REF_DIR``                           The path to the repository
                                               reference directory.
                                               Repositories in this directory
                                               are used as references when
                                               cloning. In order for kas to
                                               find those repositories, they
                                               have to be named correctly.
                                               Those names are derived from the
                                               repo url in the kas config.
                                               (E.g. url:
                                               "https://github.com/siemens/meta-iot2000.git"
                                               resolves to the name
                                               "github.com.siemens.meta-iot2000.git")
``KAS_DISTRO`` ``KAS_MACHINE`` ``KAS_TARGET``  This overwrites the respective
                                               setting in the configuration
                                               file.
``SSH_PRIVATE_KEY``                            Path to the private key file,
                                               that should be added to an
                                               internal ssh-agent. This key
                                               cannot be password protected.
                                               This setting is useful for
                                               CI building server. On desktop
                                               machines, a ssh-agent running
                                               outside the kas environment is
                                               more useful.
``http_proxy`` ``https_proxy`` ``no_proxy``    This overwrites the proxy
                                               configuration in the
                                               configuration file.
``SSH_AGENT_PID``                              SSH agent process id. Used for
                                               cloning over SSH.
``SSH_AUTH_SOCK``                              SSH authenication socket. Used
                                               for cloning over SSH.
``SHELL``                                      The shell to start when using
                                               the `shell` plugin.
``TERM``                                       The terminal options used in the
                                               `shell` plugin.
=============================================  ================================

Use Cases
---------

1.  Initial build/setup::

    $ mkdir $PROJECT_DIR
    $ cd $PROJECT_DIR
    $ git clone $PROJECT_URL meta-project
    $ kas build meta-project/kas-project.yml

2.  Update/rebuild::

    $ cd $PROJECT_DIR/meta-project
    $ git pull
    $ kas build kas-project.yml


Project Configuration
---------------------

Two types of configuration file formats are supported.

For most purposes the static configuration should be used.
In case this static configuration file does not provide enough options for
customization, the dynamic configuration file format can be used.

Static project configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Currently JSON and YAML is supported as the base file format. Since YAML is
arguable easier to read, this documentation focuses on the YAML format.

.. code-block:: yaml

    # Every file needs to contain a header, that provides kas with information
    # about the context of this file.
    header:
      # The `version` entry in the header describes for which kas version this
      # file was created. It is used by kas to figure out if it is compatible
      # with this file. Every version x.y.z should be compatible with
      # the configuration file version x.y. (x, y and z are numbers)
      version: "x.y"
    # The machine as it is written into the `local.conf` of bitbake.
    machine: qemu
    # The distro name as it is written into the `local.conf` of bitbake.
    distro: poky
    repos:
      # This entry includes the repository where the config file is located
      # to the bblayers.conf:
      meta-custom:
      # Here we include a list of layers from the poky repository to the
      # bblayers.conf:
      poky:
        url: "https://git.yoctoproject.org/git/poky"
        refspec: 89e6c98d92887913cadf06b2adb97f26cde4849b
        layers:
          meta:
          meta-poky:
          meta-yocto-bsp:

A minimal input file consist out of the ``header``, ``machine``, ``distro``,
and ``repos``.

Additionally, you can add ``bblayers_conf_header`` and ``local_conf_header``
which are strings that are added to the head of the respective files
(``bblayers.conf`` or ``local.conf``):

.. code-block:: yaml

    bblayers_conf_header:
      meta-custom: |
        POKY_BBLAYERS_CONF_VERSION = "2"
        BBPATH = "${TOPDIR}"
        BBFILES ?= ""
    local_conf_header:
      meta-custom: |
        PATCHRESOLVE = "noop"
        CONF_VERSION = "1"
        IMAGE_FSTYPES = "tar"

``meta-custom`` in these examples should be a unique name (in project scope)
for this configuration entries. We assume that your configuration file is part
of a ``meta-custom`` repository/layer. This way its possible to overwrite or
append entries in files that include this configuration by naming an entry the
same (overwriting) or using a unused name (appending).

Including in-tree configuration files
.....................................

Its currently possible to include kas configuration files from the same
repository/layer like this:

.. code-block:: yaml

    header:
      version: "x.y"
      includes:
        - base.yml
        - bsp.yml
        - product.yml

The specified files are addressed relative to your current configuration file.

Including configuration files from other repos
..............................................

Its also possible to include configuration files from other repos like this:

.. code-block:: yaml

    header:
      version: "x.y"
      includes:
        - repo: poky
          file: kas-poky.yml
        - repo: meta-bsp-collection
          file: hw1/kas-hw-bsp1.yml
        - repo: meta-custom
          file: products/product.yml
    repos:
      meta-custom:
      meta-bsp-collection:
        url: "https://www.example.com/git/meta-bsp-collection"
        refspec: 3f786850e387550fdab836ed7e6dc881de23001b
        layers:
          # Additional to the layers that are added from this repository
          # in the hw1/kas-hw-bsp1.yml, we add here an additional bsp
          # meta layer:
          meta-custom-bsp:
      poky:
        url: "https://git.yoctoproject.org/git/poky"
        refspec: 89e6c98d92887913cadf06b2adb97f26cde4849b
        layers:
          # If `kas-poky.yml` adds the `meta-yocto-bsp` layer and we
          # do not want it in our bblayers for this project, we can
          # overwrite it by setting:
          meta-yocto-bsp: exclude

The files are addressed relative to the git repository path.

The include mechanism collects and merges the content from top to buttom and
depth first. That means that settings in one include file are overwritten
by settings in a latter include file and entries from the last include file can
be overwritten by the current file. While merging all the dictionaries are
merged recursive while preserving the order in which the entries are added to
the dictionary. This means that ``local_conf_header`` entries are added to the
``local.conf`` file in the same order in which they are defined in the
different include files. Note that the order of the configuration file entries
is not preserved within one include file, because the parser creates normal
unordered dictionaries.

Static configuration reference
..............................

* ``header``: dict [required]
    The header of every kas configuration file. It contains information about
    context of the file.

  * ``version``: string [required]
      Lets kas check if it is compatible with this file.

  * ``includes``: list [optional]
      A list of configuration files this current file is based on. They are
      merged in order they are stated. So a latter one could overwrite
      settings from previous files. The current file can overwrite settings
      from every included file. An item in this list can have one of two types:

    * item: string
        The path to a kas configuration file, relative to the current file.

    * item: dict
        If files from other repositories should be included, choose this
        representation.

      * ``repo``: string [required]
          The id of the repository where the file is located. The repo
          needs to be defined in the ``repos`` dictionary as ``<repo-id>``.

      * ``file``: string [required]
          The path to the file relative to the root of the repository.

* ``machine``: string [optional]
    Contains the value of the ``MACHINE`` variable that is written into the
    ``local.conf``. Can be overwritten by the ``KAS_MACHINE`` environment
    variable and defaults to ``qemu``.

* ``distro``: string [optional]
    Contains the value of the ``DISTRO`` variable that is written into the
    ``local.conf``. Can be overwritten by the ``KAS_DISTRO`` environment
    variable and defaults to ``poky``.

* ``target``: string [optional]
    Contains the target to build by bitbake. Can be overwritten by the
    ``KAS_TARGET`` environment variable and defaults to ``core-image-minimal``.

* ``repos``: dict [optional]
    Contains the definitions of all available repos and layers.

  * ``<repo-id>``: dict [optional]
      Contains the definition of a repository and the layers, that should be
      part of the build. If the value is ``None``, the repository, where the
      current configuration file is located is defined as ``<repo-id>`` and
      added as a layer to the build.

    * ``name``: string [optional]
        Defines under which name the repository is stored. If its missing
        the ``<repo-id>`` will be used.

    * ``url``: string [optional]
        The url of the git repository. If this is missing, no git operations
        are performed.

    * ``refspec``: string [optional]
        The refspec that should be used. Required if an ``url`` was specified.

    * ``path``: string [optional]
        The path where the repository is stored.
        If the ``url`` and ``path`` is missing, the repository where the
        current configuration file is located is defined.
        If the ``url`` is missing and the path defined, this entry references
        the directory the path points to.
        If the ``url`` as well as the ``path`` is defined, the path is used to
        overwrite the checkout directory, that defaults to ``kas_work_dir``
        + ``repo.name``.

    * ``layers``: dict [optional]
        Contains the layers from this repository that should be added to the
        ``bblayers.conf``. If this is missing or ``None`` or and empty
        dictionary, the path to the repo itself is added as a layer.

      * ``<layer-path>``: enum [optional]
          Adds the layer with ``<layer-path>`` that is relative to the
          repository root directory, to the ``bblayers.conf`` if the value of
          this entry is not in this list: ``['disabled', 'excluded', 'n', 'no',
          '0', 'false']``. This way it is possible to overwrite the inclusion
          of a layer in latter loaded configuration files.

* ``bblayers_conf_header``: dict [optional]
    This contains strings that should be added to the ``bblayers.conf`` before
    any layers are included.

  * ``<bblayers-conf-id>``: string [optional]
      A string that is added to the ``bblayers.conf``. The entry id
      (``<bblayers-conf-id>``) should be unique if lines should be added and
      can be the same from another included file, if this entry should be
      overwritten. The lines are added to ``bblayers.conf`` in the same order
      as they are included from the different configuration files.

* ``local_conf_header``: dict [optional]
    This contains strings that should be added to the ``local.conf``.

  * ``<local-conf-id>``: string [optional]
      A string that is added to the ``local.conf``. It operates in the same way
      as the ``bblayers_conf_header`` entry.

* ``proxy_config``: dict [optional]
    Defines the proxy configuration bitbake should use. Every entry can be
    overwritten by the respective environment variables.

  * ``http_proxy``: string [optional]
  * ``https_proxy``: string [optional]
  * ``no_proxy``: string [optional]

Dynamic project configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The dynamic project configuration is plain Python with following
mandatory functions which need to be provided:

.. code-block:: python

    def get_machine(config):
        return 'qemu'


    def get_distro(config):
        return 'poky'


    def get_repos(target):
        repos = []

        repos.append(Repo(
            url='URL',
            refspec='REFSPEC'))

        repos.append(Repo(
            url='https://git.yoctoproject.org/git/poky',
            refspec='krogoth',
            layers=['meta', 'meta-poky', 'meta-yocto-bsp'])))

        return repos

Additionally, ``get_bblayers_conf_header()``, ``get_local_conf_header()`` can
be added.

.. code-block:: python

    def get_bblayers_conf_header():
        return """POKY_BBLAYERS_CONF_VERSION = "2"
    BBPATH = "${TOPDIR}"
    BBFILES ?= ""
    """


    def get_local_conf_header():
        return """PATCHRESOLVE = "noop"
    CONF_VERSION = "1"
    IMAGE_FSTYPES = "tar"
    """

Furthermore, you can add pre and post hooks (``*_prepend``, ``*_append``) for
the exection steps in kas core, e.g.

.. code-block:: python

    def build_prepend(config):
        # disable distro check
        with open(config.build_dir + '/conf/sanity.conf', 'w') as f:
            f.write('\n')


    def build_append(config):
        if 'CI' in os.environ:
            build_native_package(config)
            run_wic(config)

TODO: Document the complete configuration API.

