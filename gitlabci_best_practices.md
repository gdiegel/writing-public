# Running acceptance tests on GitlabCI

This article will provide an introduction of how to run automated acceptance tests on [GitlabCI][1] and best practices when designing your pipelines.

## Paradigm shifts when coming from Jenkins

If you're coming from Jenkins and want to use GitlabCI to run your acceptance tests there may be a few things to consider. Some concepts don't map 1-to-1 from Jenkins to GitlabCI and you might have to re-think the structure of your test jobs.

### Jenkins Jobs

In Jenkins, many times a build job is the first level of abstraction. This job may contain `n` build steps and post-build actions. Jobs may be run in parallel and may also be chained by configuring one job as "down-stream" of another job. In addition, each job has it's own build trigger. Usually, this is a schedule configurable by a cron-like syntax.

Jobs usually check out a project from VCS, package it and run the tests. The build logs and reports are interpreted by Jenkins and a test overview displayed. The job status reflects the status of the test run. A blue job means successful tests, an orange or red job indicates test failures or a build problem, respectively.

### Pipeline design in GitlabCI

A pipeline in GitLabCI is configured by a file called [.gitlab-ci.yml][2] placed at the repository's root. The scripts set in this file are executed by the GitLab runner. A pipeline is the first level of abstraction in GitlabCI.

Pipelines comprise:

* Jobs that define what to run.
* Stages that define when and how to run.

Multiple jobs in the same stage are executed by runners in parallel, if there are enough runners.

If all the jobs in a stage either:

* Succeed, the pipeline moves on to the next stage.
* Fail, the next stage is not (usually) executed and the pipeline ends early.

Pipelines may run in parallel independently of each other, triggered manually or by arbitrary triggers like pushes to the repository, schedules or webhooks.

* It's a good idea to create separate files for each type of test jobs and include the in the main `.gitlab-ci.yml` file. The file may other wise become very large very quickly and it might be hard to get a good understanding of the structure of the resulting pipeline.
* Forgo excessive templating and use of YAML anchors and aliases. You should DRY your script as much as necessary, but no more. It may be difficult to figure out what a job does and how it is configured while having to jump to different places over separate files.

## GitlabCI runners

A GitlabCI runner is what actually runs the pipeline defined in `gitlab-ci.yml`. If the runner that's available to you uses a Docker executor, you may choose a Docker image and GitlabCI will run the pipeline in a separate and isolated container based on the Docker image. The job then has access to all dependencies contained in the Docker image.

* Choose a base image for your job or pipeline that has most of the dependencies you need. You can always install additional dependencies during the build.
* Start any services you may need with the [services][3] keyword and configure the resulting address in the artifact itself. You can use this e.g. with the [standalone browser images][4] provided by Selenium.
* If you need tighter control over versions of browser, certificates etc. you may use your own Docker image.

Find information about runners on [this][5] page.

## Strategies for browser automation with Selenium WebDriver

When running Selenium-WebDriver-based tests one generally has the choice of configuring a selenium server or using a direct connection to a locally pre-installed browser. Based on this approach, you may either use the [standalone Docker images][6] provided by Selenium to start a Selenium server as a GitlabCI service or use a custom GitlabCI runner image to run the job and rely on the pre-installed browsers.

### Standalone Selenium-Server

The Docker images provided by Selenium may be started with the `services` element in a job configuration or at the root of the CI configuration for all jobs:

```yaml
services:
    - name: selenium/standalone-firefox:latest
      alias: selenium-firefox
```

This will download the Docker image with the latest tag and start a container with a Selenium server on the default port. By supplying an alias one may specify the hostname of the selenium server. The driver configuration would look like the following:

```java
final var driver = new RemoteWebDriver(new URL("http://selenium-firefox:4444/wd/hub"), new FirefoxOptions());
```

Pay attention to the class of the `WebDriver` implementation. We have to use a remote driver because we are connecting to a remote Selenium server. The configuration for Chrome is analogous to this one. The WebDriver version may be controlled by the tag of the Docker image.

### Custom GitlabCI runner image

As mentioned before it is a good idea to create a custom Docker image as a base image for a GitlabCI job. For automated frontend tests, this image usually contains latest versions of Node.js, Firefox and Chrome. It also may contain `xvfb` for running headful tests with a virtual frame buffer. It is recommended to regulary update the versions of the dependencies and build the Docker image nightly. It may then be used in a job configuration with the `image` element:

```yaml
image: internal/frontend-tests-ci-runner:latest
```

### Headless browser or virtual frame buffer

Selenium WebDriver allows running most browsers in a headless mode, i.e. there will be no actual GUI or graphical output, the browser will render everything in memory. For most use cases this works exactly as if the browser were to run headful. If you need to run a test headful for whatever reason you may use [xvfb][7], the virtual framebuffer. `xvfb-run` is a wrapper for the `xvfb` command which simplifies the task of running commands (typically an X client, or a script containing a list of clients to be run) within a virtual X server environment. `xvfb-run` starts the `xvfb` X server as a background process. The specified command is then run using the X display corresponding to the `xvfb` server just started. When the command exits, its status is saved, the `xvfb` server is killed. `xvfb-run` then exits with the exit status of command, except in error conditions. Example of using `xvfb-run` (Substitute <COMMAND> by actual command to run test artifact):

```bash
xvfb-run --auto-servernum --server-args '-screen 0 1920x1080x24' <COMMAND>
```

[1]: https://docs.gitlab.com/ee/ci/README.html
[2]: https://docs.gitlab.com/ee/ci/yaml/README.html
[3]: https://docs.gitlab.com/ee/ci/docker/using_docker_images.html#define-image-and-services-from-gitlab-ciyml
[4]: https://hub.docker.com/u/selenium
[5]: https://docs.gitlab.com/ee/ci/runners/README.html
[6]: https://hub.docker.com/u/selenium
[7]: https://packages.debian.org/en/sid/xvfb
