.. This Source Code Form is subject to the terms of the Mozilla Public
.. License, v. 2.0. If a copy of the MPL was not distributed with this
.. file, You can obtain one at http://mozilla.org/MPL/2.0/.

.. _pipeline:

===================================
Continuous Integration & Deployment
===================================

Bedrock runs a series of automated tests as part of continuous integration workflow and
`Deployment Pipeline`_. You can learn more about each of the individual test suites
by reading their respective pieces of documentation:

* Python unit tests (see :ref:`run-python-tests`).
* JavaScript unit tests (see :ref:`testing`).
* Redirect tests (see :ref:`testing-redirects`).
* Functional tests (see :ref:`testing`).

Tests in the lifecycle of a change
----------------------------------

Below is an overview of the tests during the lifecycle of a change to bedrock:

Local development
~~~~~~~~~~~~~~~~~

The change is developed locally, and all integration tests can be executed against a
locally running instance of the application.

Pull request
~~~~~~~~~~~~

Once a pull request is submitted, `CircleCI`_ will run both the Python and  JavaScript
unit tests, as well as the smoke suite of redirect headless HTTP(s) response checks.

Push to master branch
~~~~~~~~~~~~~~~~~~~~~

Whenever a change is pushed to the master branch, the smoke suite of
headless (see :ref:`testing-redirects`) and UI tests (see :ref:`smoke-functional-tests`)
are run against Firefox on Linux. If successful, the change is pushed to the dev environment,
and the full suite of headless and UI tests are then run against
Firefox on Windows 10 using `Sauce Labs`_. This is handled by the pipeline, and is subject
to change according to the settings in the `jenkins.yml file`_ in the repository.

.. _tagged-commit:

Push to prod branch (tagged)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When a tagged commit is pushed to the prod branch, everything that happens during a
untagged push is still run. In addition, the full suite of UI tests is run against
Chrome and Internet Explorer on Windows 10, and the sanity suite is run against older
versions of Internet Explorer (currently IE6 & IE7). If successful, the change is
pushed to staging, tested, and then to production and the same tests are then run against
production. As with untagged pushes, this is all handled by the pipeline, and is subject
to change according to the settings in the `jenkins.yml file`_ in the repository.

**Push to prod cheat sheet**

#. Check out the ``master`` branch
#. Make sure the ``master`` branch is up to date with ``mozilla/bedrock master``
#. Check that dev deployment is green:
    #. View `deployment pipeline <https://ci.us-west.moz.works/blue/organizations/jenkins/bedrock_multibranch_pipeline/branches/>`_
       and look at ``master`` branch
#. Tag and push the deployment by running ``bin/tag-release.sh --push``

.. note::

    By default the ``tag-release.sh`` script will push to the ``origin`` git remote. If you'd
    like for it to push to a different remote name you can either pass in a ``-r`` or
    ``--remote`` argument, or set the ``MOZ_GIT_REMOTE`` environment variable. So the following
    are equivalent::

        $ bin/tag-release.sh --push -r mozilla
        $ MOZ_GIT_REMOTE=mozilla bin/tag-release.sh --push

    And if you'd like to just tag and not push the tag anywhere, you may omit the ``--push``
    parameter.

Pipeline integration
--------------------

Our `Jenkinsfile`_ will run the integration tests based on information in our `jenkins.yml file`_.
This file specifies various test names per branch that will cause it to load different
parameters, allowing it to be called in many different ways to cover the testing
needs. The job finds the parameters file and executes `this script <https://github.com/mozilla/bedrock/blob/master/docker/jenkins/run_integration_tests.sh>`_,
which then runs `this Docker image <https://github.com/mozilla/bedrock/blob/master/docker/dockerfiles/bedrock_integration_tests>`_,
and ultimately runs `another script <https://github.com/mozilla/bedrock/blob/master/bin/run-integration-tests.sh>`_.
The two scripts can also be executed locally to replicate the way Jenkins operates.

During the **Test Images** stage, the Test Runner job is called without a ``BASE_URL``. This means
that a local instance of the application will be started, and the URL of this instance
will be used for testing. The ``DRIVER`` parameter is set to ``Remote``, which causes a
local instance of Selenium Grid to be started in Docker and used for the browser-based
functional UI tests.

The test scripts above will be run once for each properties file specified in the `jenkins.yml file`_
for the branch being built and tested. Pushes to `master` will run different tests than pushes to `prod`
for example.

Configuration
~~~~~~~~~~~~~

Many of the options are configured via environment variables passed from the initial
script, to the Docker image and onto the final script. This means that global defaults
can be `configured in Jenkins`_. Note that admin access is required to make changes to the
global configuration, and there is a known issue that may cause Jenkins to `become
unresponsive`_ after a configuration change.

Updating Selenium
~~~~~~~~~~~~~~~~~

There are two components for Selenium, which are independently versioned. The first is
the Python client, and this can be updated via the `test dependencies`_. The other
component is the server, which in the pipeline is either provided by a Docker container
or `Sauce Labs`_. The ``SELENIUM_VERSION`` environment variable controls both of these, and
they should ideally use the same version, however it’s possible that availability of
versions may differ. You can check the `Selenium Docker versions`_ available. If needed, the global
default can be set and then can be overridden in the individual job configuration.

Adding test runs
~~~~~~~~~~~~~~~~

Test runs can be added by creating new `properties files`_ with the parameters of the new
test run. These are simply bash syntax files that set environment variables.
 For example, if you wanted to run tests in Firefox on both Windows 10 and
OS X, you could create the following files

win10-firefox.properties
........................

.. code-block:: none

    export DRIVER=SauceLabs
    export BROWSER_NAME=firefox
    export PLATFORM="Windows 10"
    export MARK_EXPRESSION="not headless""

osx-firefox.properties
......................

.. code-block:: none

    export DRIVER=SauceLabs
    export BROWSER_NAME=firefox
    export PLATFORM="OS X 10.11"
    export MARK_EXPRESSION="not headless"

You can use `Sauce Labs platform configurator`_ to help with the parameter values.

If you have an account on our Jenkins server, you can build the `bedrock_integration_tests_runner`_
job and pass in the ``BASE_URL`` and other parameters of your choosing. This is also useful for
testing against deployed demo environments. For a good baseline, use the values from ``win10-firefox.properties``_
above.

Known issues in Jenkins
-----------------------

Jenkins stalls after global configuration changes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When using the IRC plugin for notifications, global configuration changes can cause
Jenkins to become unresponsive. To make such changes it can be necessary to first
restart Jenkins, as this issue only appears some time after Jenkins has been started.
A `bug for the IRC plugin`_ has been raised.

.. _Deployment Pipeline: https://ci.us-west.moz.works/view/Bedrock%20Pipeline/
.. _CircleCI: https://circleci.com/
.. _Sauce Labs: https://saucelabs.com/
.. _Jenkinsfile: https://github.com/mozilla/bedrock/tree/master/Jenkinsfile
.. _jenkins.yml file: https://github.com/mozilla/bedrock/tree/master/jenkins.yml
.. _properties files: https://github.com/mozilla/bedrock/tree/master/docker/jenkins/properties/integration_tests
.. _bedrock_integration_tests_runner: https://ci.us-west.moz.works/view/Bedrock/job/bedrock_integration_tests_runner/
.. _configured in Jenkins: https://ci.us-west.moz.works/configure
.. _become unresponsive: https://issues.jenkins-ci.org/browse/JENKINS-28175
.. _test dependencies: https://github.com/mozilla/bedrock/blob/master/requirements/test.txt
.. _Selenium Docker versions: https://hub.docker.com/r/selenium/hub/tags/
.. _Sauce Labs platform configurator: https://wiki.saucelabs.com/display/DOCS/Platform+Configurator/
.. _enhancement request: https://issues.jenkins-ci.org/browse/JENKINS-26210
.. _bug for the IRC plugin: https://issues.jenkins-ci.org/browse/JENKINS-28175
