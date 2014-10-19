---
layout: page
title: Welcome to IRMA's Development & Engineering headquarters
---

This is the place to find out more about the code that actually drives the IRMA project.

This page has just been created and will be under development for some time. In the end it will contain a guide/manual for developers to develop new IRMA applications/services or to integrate IRMA support into existing applications and services.

## Credentials API

To start we need to create a working directory (replace `<workdir>` with a name of your choice).

    $ mkdir <workdir>
    $ cd <workdir>

Within this working directory we will check out the various projects and set up the IRMA development environment. Most of the projects provide both Eclipse project files as well as Ant build files. The eclipse project files are usually located in the `dev/eclipse` directory, so they should be copied to the project root before use. When using the Ant build file, running the `ant` command, will produce three `.jar` files:

 * `<projectname>.lib.jar` or `<projectname>.app.jar`: The binaries of the library or application. Applications can be run using the `java -jar <projectname>.app.jar` command.
 * `<projectname>.src.jar`: The sources of the project combined into a single archive.
 * `<projectname>.dev.jar`: A combination of the above, which is convenient for developers.

<h4>
<a name="scuba" class="anchor" href="#scuba"><span class="octicon octicon-link"></span></a>SCUBA</h4>

For the communication with the IRMA card we use the <a href="http://scuba.sf.net">SCUBA library</a> to provide a layer of abstraction.

    $ git clone https://github.com/credentials/scuba.git
    $ cd scuba
    $ ant
    $ cd ..

<h4>
<a name="idemix" class="anchor" href="#idemix"><span class="octicon octicon-link"></span></a>Idemix</h4>

    $ git clone https://github.com/credentials/idemix_library.git
    $ git clone https://github.com/credentials/idemix_terminal.git

The next step is to prepare these sources. For this we first need to <a href="https://prime.inf.tu-dresden.de/idemix/">download the Idemix 2.3.4 sources</a> from IBM and store them in the idemix_library created by the clone command. These sources need to be patched, hence the additional `ant prepare` command. Note: Idemix 2.3.40 and newer are not supported.

    $ cd idemix_library
    $ ant prepare; ant
    $ cd ..

<!-- -->

    $ cd idemix_terminal/lib
    $ ln -s ../../scuba/scuba.dev.jar
    $ ln -s ../../idemix_library/idemix_library.dev.jar
    $ cd ..
    $ ant
    $ cp dev/eclipse/.project dev/eclipse/.classpath .
    $ cd ..

#### IRMA

    $ git clone https://github.com/credentials/irma_configuration.git
    $ git clone https://github.com/credentials/credentials_api.git
    $ git clone https://github.com/credentials/credentials_idemix.git

The `irma_configuration` project contains the data structure used within the IRMA project by the credentials API. The actual API consists of two projects, `credentials_api` which contains the generic code and `credentials_idemix` which contains the Idemix specific instantiations of the API.

    $ cd credentials_api/lib
    $ ln -s ../../scuba/scuba.dev.jar
    $ cd ..
    $ ant
    $ cp dev/eclipse/.project dev/eclipse/.classpath .
    $ cd ..

<!-- -->

    $ cd credentials_idemix/lib
    $ ln -s ../../scuba/scuba.dev.jar
    $ ln -s ../../idemix_library/idemix_library.dev.jar
    $ ln -s ../../idemix_terminal/idemix_terminal.dev.jar
    $ ln -s ../../credentials_api/credentials_api.dev.jar
    $ cd ..
    $ ant
    $ cp dev/eclipse/.project dev/eclipse/.classpath .
    $ ln -s ../irma_configuration
    $ cd ..

## Developing an IRMA application

Now you can start developing your own IRMA application. The easiest for this is to create your own project directory `<project>` within the `<workdir>`. Within this directory add all relevant libraries and add the IRMA configuration such that the Credentials API can find it.

    $ mkdir <project>
    $ cd <project>
    $ ln -s ../irma_configuration
    $ mkdir lib; cd lib
    $ ln -s ../../scuba/scuba.dev.jar
    $ ln -s ../../idemix_library/idemix_library.dev.jar
    $ ln -s ../../idemix_terminal/idemix_terminal.dev.jar
    $ ln -s ../../credentials_api/credentials_api.dev.jar
    $ ln -s ../../credentials_idemix/credentials_idemix.dev.jar

### Support or Contact

Having trouble with the Credentials API or IRMA development? Contact <a href="mailto:pim@cs.ru.nl">pim@cs.ru.nl</a> and we’ll help you sort it out.
