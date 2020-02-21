<!--
.. title: PyTest, coverage reports and Gitlab runners
.. slug: pytest-coverage-reports-and-gitlab-runners
.. date: 2020-02-21 21:52:45 UTC+01:00
.. tags: python,testing,pytest,minio,coverage,gitlab
.. category: 
.. link: 
.. description: 
.. type: text
-->

A friend of mine says that he doesn't need tests, he just writes bug-free code. Everyone thinks that his or her code is without bugs, while I think that a bunch of non-trivial amount of code almost definitely contains a bug or two, if not tested properly. And even then, one cannot be sure that no hidden bug lurks somewhere. I was not always like that. Once upon a time I too thought that testing is a waste of time. Why would you test something if you just know it works? Such is the current state in research, good software development practices are rarely followed and it is up to the few of us to try and force the good practices throughout the research organizations.

I am still learning about what can and should be done. Testing is one of the things I try to enforce with fellow researchers and students, when working on common projects. The other ones are: automation, documentation, and reproducibility. In this post I will describe our testing set-up I established. Since we work mainly with Python, we use popular [PyTest](https://docs.pytest.org/en/latest/) framework for testing and the [pytest-cov](https://pytest-cov.readthedocs.io/en/latest/), a PyTest plug-in using [coverage](https://coverage.readthedocs.io/en/latest/) for measuring the code coverage of Python programs. After the test coverage report generation, the html report is uploaded to a self-hosted [Minio](https://min.io/) instance for viewing.
<!-- TEASER_END -->

First thing first, all the code should live in a remote Git repository. At JSI we are fortunate enough that our IT department provides us with a self-hosted [Gitlab Community Edition](https://about.gitlab.com/install/?version=ce) instance. Gitlab comes with a ton of features, however, not all of them are enabled in our instance. For the answer as to why is that, I am still waiting for a response from the IT department to the email I have sent half a year ago. Yes, they are not very quick about user questions or requests.

# Gitlab runners
Anyway, one of the features that is enabled is the CI / CD pipelines. These pipelines enables us to run scripts using the Gitlab runners on every code push to the Gitlab instance. I have hacked together a script for our Gitlab runners virtual machine, that enables us to start a Gitlab runner for each project/organization separately.

```bash
#!/bin/bash

# This is a script for managing gitlab-runners
#
# Usage:
#
#    $ sudo bash ./start-runner.sh RUNNER-NAME [update,init,start]
#
# RUNNER-NAME  -  the name of the runner you want to manage
# COMMAND:
#    update    -  update the gitlab-runner container version
#    init      -  run when you want to configure a runner
#    start     -  start a runner using existing configuration
#
# Example:
#
#    $ sudo bash ./start-runner.sh my-awesome-project init
#    $ ...enter credentials as asked
#    $ sudo bash ./start-runner.sh my-awesome-project start

RUNNER_NAME="$1"
COMMAND="$2"

DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" >/dev/null 2>&1 && pwd )"
VOLUME_DIR="$DIR/volumes"
CONFIG_DIR="$VOLUME_DIR/$RUNNER_NAME/config"

if [ ! -f "$CONFIG_DIR" ]; then
    mkdir -p "$CONFIG_DIR"
fi

update () {
  docker pull gitlab/gitlab-runner
}

init () {
  update
  # register runner
  docker run -it --rm -v "$CONFIG_DIR":/etc/gitlab-runner gitlab/gitlab-runner:latest register
}

start () {
  docker rm -f gitlab-runner-"$RUNNER_NAME"

  # RUN GitLab Runner
  docker run -d --name "gitlab-runner-$RUNNER_NAME" --restart unless-stopped \
         -v "$CONFIG_DIR":/etc/gitlab-runner \
         -v /var/run/docker.sock:/var/run/docker.sock \
         gitlab/gitlab-runner:latest
}

"$COMMAND"
```

When a Gitlab runner is run and registered with the organization or a repository it can receive and handle jobs, specified in the [`.gitlab-ci.yml`](https://docs.gitlab.com/ee/ci/yaml/) file. One type of such jobs are usually tests.

# Tests
Testing with the PyTest framework is really easy. Most of the time you are just writing the `assert` statements, where your expectation as to how the code should work is tested. I found [Python testing with pytest](https://pragprog.com/book/bopytest/python-testing-with-pytest) a useful reference although the PyTest documentation is itself quite good already.

In order to get an idea on how much of your code is covered with tests we use the `pytest-cov` package. It track what statements were tested and which were missed and can generate a succinct report presented in the terminal or an interactive html report (among others), where you can view exactly which lines  are tested and which are not. Using Gitlab you can specify a regex to catch the overall coverage, which can give you a nice looking badge to look at and feel proud. Sometimes.

![pipeline badge](/images/pipeline-badges.png "pipeline badge"){: style="display:block;margin-left:auto;margin-right:auto"}

At the beginning you will write your test and see the coverage metric go up-up-up. After a while though, you may start to wonder which parts of your code remain uncovered. That is when you will need to produce the html report, like this:

```bash
pytest --cov-report term --cov-report html --cov=app -v tests/
``` 

You need the `term` (terminal) report, so the Gitlab can display the overall coverage info and the `html` report so you can see what needs to be tested next. 

# Vieving the coverage reports
When running the tests locally, you can view the report without problems:

```
$ cd htmlcov
$ python -m http.server
```

However, when the report is generated by Gitlab runner, viewing the report becomes trickier. I have searched for a way to enable uploading the serving the reports automatically after they become available. I have come up with the following solution. We alread have a firewalled testing machine that runs Gitlab runners. We can use the same machine to run a self-hosted [Minio](https://min.io/) instance (self-hosted S3 alternative for object storage). When configured correctly it can host static sites with ease. In the end, I hacked together the following script:

```bash
#!/bin/sh

HTMLCOV_LOCATION="htmlcov"
MINIO_BASE={YOUR-MINIO-SERVER-HOST}
MINIO_NAME="minio"

RANDOM_FOLDER_NAME=$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | head -c 64)

./mc config host add "$MINIO_NAME" "$MINIO_BASE" {YOUR-MINIO-ACCESS-KEY} {YOUR-MINIO-SECRET-KEY}

MINIO_FOLDER="/coverage/$RANDOM_FOLDER_NAME"

./mc cp --recursive "$HTMLCOV_LOCATION/" "$MINIO_NAME/$MINIO_FOLDER/"
./mc policy set public "$MINIO_NAME/$MINIO_FOLDER"

echo " "
echo "========================================================================="
echo " "
echo " "
echo "Coverage report is available at the following link"
echo " "
echo "    $MINIO_BASE$MINIO_FOLDER/index.html"
```

This is run after the test and uploads the generated html report to the Minio instance using the official [Minio client](https://docs.min.io/docs/minio-client-complete-guide) - `mc`. Yes, this adds 11MB to your repo. I have tried using `curl` for uploading the files, however, I constantly ran into problems with it. In the end I just settled for the client.

One of the key line in the script is the following

```bash
./mc policy set public "$MINIO_NAME/$MINIO_FOLDER"
```

This enables the hosting of the folder as a static site, which can be viewed by following the random link generated within the script. Now, when a test is run, the coverage report is generated, Gitlab reports the overall statistics and a link is provided in the test job output, which can be followed to see what tests have to be written next. And everyone on the team is on the same page at any time.


