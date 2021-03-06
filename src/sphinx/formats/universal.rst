.. _Universal:

Universal Plugin
================

Universal packaging just takes a plain ``mappings`` configuration and generates various 
package files for distribution.  It allows you to provide your users a distribution
that is not tied to any particular platform, but may require manual labor to set up.


.. contents:: 
  :depth: 2
  
.. raw:: html

  <div class="alert alert-info" role="alert">
    <span class="glyphicon glyphicon-info-sign" aria-hidden="true"></span>
    The universal plugin dependens on the sbt-native-packager plugin. For general settings read the 
    <a href="np-plugin.html">SBT Native Packager Plugin Documentation</a>
  </div>

Related Plugins
---------------

.. toctree::
   :maxdepth: 1
   
   linux.rst
   windows.rst
   docker.rst


Requirements
------------

Depending on what package format you want to use, you need one of the following applications installed

* zip (if native)
* gzip
* xz
* tar
* hdiutil (for dmg)

Build
-----

There is a task for each output format

  **Zip**
  
  .. code-block:: bash
  
    sbt universal:packageBin
  
  **Tar**
  
  .. code-block:: bash

    sbt universal:packageZipTarball

  **Xz**

  .. code-block:: bash

    sbt universal:packageXzTarball

  **Dmg**

  .. code-block:: bash
  
    sbt universal:packageOsxDmg


Required Settings
~~~~~~~~~~~~~~~~~

A universal has no mandatory fields.


1.0 or higher
~~~~~~~~~~~~~

Enable the universal plugin

.. code-block:: scala

  enablePlugins(UniversalPlugin)


0.8 or lower
~~~~~~~~~~~~

For this versions universal packaging is automatically activated.
See the :doc:`Getting Started </gettingstarted>` page for informations
on how to enable sbt native packager.


Configurations
--------------

Settings and Tasks inherited from parent plugins can be scoped with ``Universal``.

Universal packaging provides three Configurations:

  ``universal``
    For creating full distributions
  ``universal-docs``
    For creating bundles of documentation
  ``universal-src``
    For creating bundles of source.

.. code-block:: scala

    name in Universal := name.value
    
    name in UniversalDocs <<= name in Universal
    
    name in UniversalSrc <<= name in Universal
    
    packageName in Universal := packageName.value

Settings
--------
As we showed before, the Universal packages are completely configured through the use of the mappings key.  Simply
specify the desired mappings for a given configuration.  For Example:

.. code-block:: scala

    mappings in Universal <+= packageBin in Compile map { p => p -> "lib/foo.jar" }

However, sometimes it may be advantageous to customize the files for each archive separately.  For example, perhaps 
the .tar.gz has an additional README plaintext file in additon to a README.html.  To add this just to the .tar.gz file,
use the task-scope feature of sbt:

.. code-block:: scala

    mappings in Universal in package-zip-tarball += file("README") -> "README"
    
Besides ``mappings``, the ``name``, ``sourceDirectory`` and ``target`` configurations are all respected by universal packaging.

**Note: The Universal plugin will make anything in a bin/ directory executable.  This is to work around issues with JVM
and file system manipulations.**

Tasks
-----

  ``universal:package-bin``
    Creates the ``zip`` universal package.
  
  ``universal:package-zip-tarball``
    Creates the ``tgz`` universal package.
    
  ``universal:package-xz-tarball``
    Creates the ``txz`` universal package.  The ``xz`` command can get better compression
    for some types of archives.

  ``universal:package-osx-dmg``
    Creates the ``dmg`` universal package.  This only work on OSX or systems with ``hdiutil``.
    
  ``universal-docs:package-bin``
    Creates the ``zip`` universal documentation package.
  
  ``universal-docs:package-zip-tarball``
    Creates the ``tgz`` universal documentation package.
    
  ``universal-docs:package-xz-tarball``
    Creates the ``txz`` universal documentation package.  The ``xz`` command can get better compression
    for some types of archives.
    
Customize
---------

Getting Started with Universal Packaging
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
By default, all files found in the ``src/universal`` directory are included in the distribution.  So, the first step
in creating a a distribution is to place files in this directory in the layout you would like in the distributed zip file.

To add build generated files to the distribution, simple add a *mapping* to the ``mappings in Universal`` setting.  Let's
look at an example where we add the packaged jar of a project to the lib folder of a distribution:

.. code-block:: scala

    mappings in Universal <+= (packageBin in Compile) map { jar =>
      jar -> ("lib/" + jar.getName)
    }

The above does two things:  

1. It depends on ``packageBin in Compile`` which will generate a jar file form the project.
2. It creates a *mapping* (a ``Tuple2[File, String]``) which denotes the file and the location in the distribution as a string.

You can use this to add anything you desire to the package.

**Note**

..

    If you are using an ``application archetype`` or the ``playframework``, the jar mapping is already defined and
    you should not include these in your ``build.sbt``. `issue 227`_
    
.. _issue 227: https://github.com/sbt/sbt-native-packager/issues/227


Universal Conventions
~~~~~~~~~~~~~~~~~~~~~
This plugin has a set of conventions for universal packages that enable the automatic generation of native packages.  The
universal convention has the following package layout:


.. code-block:: none

    bin/
       <scripts and things you want on the path>
    lib/
       <shared libraries>
    conf/
       <configuration files that should be accessible using platform standard config locations.>
    doc/
       <Documentation files that should be easily accessible. (index.html treated specially)>

If your plugin matches these conventions, you can enable the settings to automatically generate native layouts based on your universal package.  To do
so, add the following to your build.sbt:


.. code-block:: scala

    mapGenericFilesToLinux

    mapGenericFilesToWinows


In Linux, this mapping creates symlinks from platform locations to the install location of the universal package.  For example, 
given the following packaging:


.. code-block:: none

    bin/
       cool-tool
    lib/
       cool-tool.jar
    conf/
       cool-tool.conf


The ``mapGenericFilesToLinux`` settings will create the following package (symlinks denoted with ``->``):


.. code-block:: none

    /usr/share/<pkg-name>/
       bin/
         cool-tool
       lib/
         cool-tool.jar
       conf/
         cool-tool.conf
    /usr/bin/
         cool-tool  -> /usr/share/<package-name>/bin/cool-tool
    /etc/<pkg-name> -> /usr/share/<package-name>/conf

The ``mapGenericFilesToWindows`` will construct an MSI that installs the application in ``<Platform Program Files>\<Package Name>`` and include
the ``bin`` directory on Windows ``PATH`` environment variable (optionally disabled).  While these mappings provide a great start to nice packaging, it still
may be necessary to customize the native packaging for each platform.   This can be done by configuring those settings directly.

For example, even using generic mapping, debian has a requirement for changelog files to be fully formed.  Using the above generic mapping, we can configure just this
changelog in addition to the generic packaging by first defining a changelog in ``src/debian/changelog`` and then adding the following setting:


.. code-block:: scala

    linuxPackageMappings in Debian <+= (name in Universal, sourceDirectory in Debian) map { (name, dir) =>
      (packageMapping(
        (dir / "changelog") -> "/usr/share/doc/sbt/changelog.gz"
      ) withUser "root" withGroup "root" withPerms "0644" gzipped) asDocs()
    }

Notice how we're *only* modifying the package mappings for Debian linux packages.  For more information on the underlying packaging settings, see
:ref:`Windows` and :ref:`Linux` documentation.


MappingsHelper
~~~~~~~~~~~~~~

The `MappingsHelper`_ class provides a set of helper functions to make mapping directories easier.

  **sbt 0.13.5 and plugin 1.0.x or higher**
  
  .. code-block:: scala
    
      import NativePackagerHelper._
    
  **plugin  version 0.8.x or lower**
  
  .. code-block:: scala
    
    import com.typesafe.sbt.SbtNativePackager._
    import NativePackagerHelper._


You get a set of methods which will help you to create mappings very easily.

.. code-block:: scala

    mappings in Universal ++= directory("src/main/resources/cache")
    
    mappings in Universal ++= contentOf("src/main/resources/docs")
    
    mappings in Universal <++= sourceDirectory map (src => directory(src / "main" / "resources" / "cache"))
    
    mappings in Universal <++= sourceDirectory map (src => contentOf(src / "main" / "resources" / "docs"))


.. _MappingsHelper: http://www.scala-sbt.org/sbt-native-packager/latest/api/#com.typesafe.sbt.packager.MappingsHelper$

Mapping Examples
~~~~~~~~~~~~~~~~

SBT provides and IO and `Path`_ API, which
lets you define custom mappings easily. The files will appear in the generate universal zip, but also in your
debian/rpm/msi/dmg builds as described above in the conventions.

.. _Path: http://www.scala-sbt.org/0.13.1/docs/Detailed-Topics/Paths.html

The ``packageBin in Compile`` dependency is only needed, if your files get generated
during the ``packageBin`` command or before. For static files you can remove it.

Mapping a complete directory
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: scala

    mappings in Universal <++= (packageBin in Compile, target ) map { (_, target) =>
        val dir = target / "scala-2.10" / "api"
        (dir.***) pair relativeTo(dir.getParentFile)
    } 

This maps the ``api`` folder directly to the generate universal zip. ``dir.***`` is a short way for
``dir ** "*"``, which means _select all files including *dir*. ``relativeTo(dir.getParentFile)``
generates a function with a ``file -> Option[String]`` mapping, which tries to generate a relative
string path from ``dir.getParentFile`` to the passed in file. ``pair`` uses the ``relativeTo``
function to generate a mapping ``File -> String``, which is *your file* to *relative destination*.

It exists some helper methods to map a complete directory in more human readable way.

.. code-block:: scala

    //For dynamic content, e.g. something in the target directory which depends on a Task
    mappings in Universal <++= (packageBin in Compile, target) map { (_, target) =>
      directory(target / "scala-2.10" / "api")
    }

    //For static content it can be added to mappings directly
    mappings in Universal ++= directory("SomeResourcesToInclude")


Mapping the content of a directory
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: scala

    mappings in Universal <++= (packageBin in Compile, target ) map { (_, target) =>
        val dir = target / "scala-2.10" / "api"
        (dir.*** --- dir) pair relativeTo(dir)
    }
    
The ``dir`` gets excluded and is used as root for ``relativeTo(dir)``.

Filter/Remove mappings
^^^^^^^^^^^^^^^^^^^^^^

If you want to remove mappings, you have to filter the current list of mappings.
This example demonstrates how to build a fat jar with sbt-assembly, but using all
the convenience of the sbt native packager archetypes.

tl;dr how to remove stuff

.. code-block:: scala

    // removes all jar mappings in universal and appends the fat jar
    mappings in Universal := {
        // universalMappings: Seq[(File,String)]
        val universalMappings = (mappings in Universal).value 
        val fatJar = (assembly in Compile).value
        // removing means filtering
        val filtered = universalMappings filter { 
            case (file, name) =>  ! name.endsWith(".jar") 
        }
        // add the fat jar
        filtered :+ (fatJar -> ("lib/" + fatJar.getName))
    }
    
    // sbt 0.12 syntax
    mappings in Universal <<= (mappings in Universal, assembly in Compile) map { (universalMappings, fatJar) => /* same logic */}


The complete ``build.sbt`` should contain these settings if you want a single assembled fat jar.

.. code-block:: scala

    // the assembly settings
    assemblySettings

    // we specify the name for our fat jar
    jarName in assembly := "assembly-project.jar"

    // using the java server for this application. java_application would be fine, too
    packageArchetype.java_server

    // removes all jar mappings in universal and appends the fat jar
    mappings in Universal := {
        val universalMappings = (mappings in Universal).value 
        val fatJar = (assembly in Compile).value
        val filtered = universalMappings filter { 
            case (file, name) =>  ! name.endsWith(".jar") 
        }
        filtered :+ (fatJar -> ("lib/" + fatJar.getName))
    }

    // the bash scripts classpath only needs the fat jar
    scriptClasspath := Seq( (jarName in assembly).value )
    

