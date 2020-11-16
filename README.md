# ansible-tomcat-sample 
[![Drone Pipeline](https://drone.zuh0.com/api/badges/zuh0/ansible-tomcat-sample-app/status.svg)](https://drone.zuh0.com/zuh0/ansible-tomcat-sample-app)

## Introduction

This repository contains an simple and straightforward [Ansible][1] playbook
to deploy the [Apache Tomcat][2] [sample application][3] in a [Docker][4] container.

The play is cut into 5 parts:

* installing the necessary dependencies
* testing if the server is already up and exiting if so
* building the container image
* creating a container from that image
* testing the server was deployed correctly

Each of these phases is configurable through ansible variables.

The default configuration will build an image based on the [`tomcat:9.0-slim`
image][5], tag it `tomcat-sample:latest` and
deploy it on port `8080` naming the running container `tomcat-sample`.

All configuration is done through Ansible variables and is pretty
straightforward. For a quick overview of the configuration options, you can
take a look at the `group_vars/webservers` file and the `hosts` file.

[1]: https://www.ansible.com/
[2]: https://tomcat.apache.org/
[3]: https://tomcat.apache.org/tomcat-9.0-doc/appdev/sample/
[4]: https://docker.com/
[5]: https://hub.docker.com/_/tomcat

## Ansible dependencies

Before launching the playbook you will need to fetch the `community.general`
Ansible collection as it contains the modules we use to interact with the
Docker API:

* `docker_image` to build the container image
* `docker_image_info` to check the image is available before trying to launch
  it
* `docker_container` to actually create the container from its image

To install `community.general` you can use the `requirements.yml` file which
can be found in the root directory of this project.

```
$ ansible-galaxy collection install -r requirements.yml
```

## Usage

The easiest way to run the playbook is to just use its default configuration.
With the default variables, superuser access will be needed to install packages
using `pip` and to talk to the Docker socket. Privilege escalation will be
handled with `become`.

```
$ ansible-playbook site.yml
```

If you don't need privilege escalation because your user can already access the
Docker socket or the Python packages are already installed or can be installed
user-locally you can set the `need_become` variable to `false`.

## Python Dependencies

This playbook relies on 2 Python packages:

* [`docker`][6] to interact with the Docker API
* [`requests`][7] to test the HTTP endpoint

Both dependencies will be installed by the playbook if needed.

You can pass options to `pip` using the `pip_extra_args` variable. For example,
to install the packages using the `--user` flag you can run the playbook with
`-e 'pip_extra_args=--user'`.

To comply with Ansible best practices, if these packages are already installed
on the target machine they will **not** be updated.

[6]: https://github.com/docker/docker-py
[7]: https://requests.readthedocs.io/en/master/

## Image Build

When building, the Dockerfile used to actually build the image is generated
from a simple template which can be found at
`roles/build_image/templates/Dockerfile.j2`. The Dockerfile will be generated
in a temporary directory which will be removed whether the build succeeds or
fails.

```dockerfile
{{ ansible_managed | comment }}

FROM {{ build.base_image.name }}:{{ build.base_image.tag }}

RUN mkdir -p {{ build.base_image.catalina_base }}/webapps

ADD {{ build.war_uri }} {{ build.base_image.catalina_base }}/webapps
```

By default the generated file will look like this.

```dockerfile
#
# Ansible managed
#

FROM tomcat:9.0-slim

RUN mkdir -p /usr/local/tomcat/webapps

ADD https://tomcat.apache.org/tomcat-9.0-doc/appdev/sample/sample.war /usr/local/tomcat/webapps
```

To configure the image build you can use this variable structure:

```yaml
build:
  base_image:
    name: name of the base image to use
    tag: tag of the image to use
    catalina_base: path to the catalina base in which to unpack the WAR file
  war_uri: uri to the downloadable WAR file we wish to serve
```

Also, the name and tag of the built image can be configured. By default, the
built image will be `tomcat-sample:latest`.

```yaml
built_image:
  name: name of the image
  tag: tag of the image
```

## Starting the Container

Once the image is built, a container will be created and started using it. You
can set the name of the container as well as the port on which the Tomcat will
be accessible. The syntax for the variables is as follows.

```yaml
deploy:
  container_name: name of the container
  exposed_port: port to expose
```

By default, the container name will be `tomcat-sample` and it will be exposed
on port `8080`.

## Deployment Test

Tests are run twice during a play run:

* at the beggining to check if we actually need to run the rest
* at the end to check if the deployment was successful

To run tests, a Python test is generated as a temporary file and run on the
remote host.

`pytest` is used as a test framework and `requests` is used to actually do the
HTTP GET requests.

The template for the script can be found at
`roles/test_container/templates/http_get_test.py.j2` and contains this:

```python
import hashlib
import sys

from typing import List, Tuple

import pytest
import requests

test_cases: List[Tuple[str, int, bool, str]] = [
{% for test in test_input.http_get_tests %}
    (
        "http://{}:{}{}{}".format(
            "{{ test_input.host }}",
            {{ deploy.exposed_port }},
            "{{ test_input.path_prefix }}",
            "{{ test.path }}"
        ),
        {{ test.status_code | default(200) }},
        {{ test.allow_redirects | default(True) }},
        "{{ test.content_hash | default() }}"
    ),
{% endfor %}
]


@pytest.mark.parametrize(
    "uri,status_code,allow_redirects,content_hash", test_cases
)
def test_http_get(uri, status_code, allow_redirects, content_hash):
    resp: requests.Response = requests.get(
        url=uri, allow_redirects=allow_redirects
    )
    assert resp.status_code == status_code
    if content_hash:
        assert hashlib.sha256(resp.content).hexdigest() == content_hash

if __name__ == "__main__":
    sys.exit(pytest.main([sys.modules["__main__"].__file__, "-qq"]))
```

The test inputs are dynamic and given through Ansible variables (you can check
out the end of the `group_vars/webservers` file for a simpler overview). For
each test case, an HTTP GET request is done on the server.

The port on which to do the request is given by the same variable used to start
the Docker image.

The host is global to all tests and is set by the `test_input.host` variable as
well a prefix to apply to all paths `test_input.path_prefix` (the default is
"/sample").

The bulk of the test cases is in the `test_input.http_get_tests` variable which
is an array of entries.

Each entry has 1 mandatory field:

* `path` the path to GET

And 3 optional fields:

* `status_code` the expected HTTP status code the request will yield (defaults
  to 200)
* `allow_redirects` a boolean value whether or not to follow redirections
  (defaults to `true`)
* `content_hash` a string of the SHA2-256 hash of the request's body which will
  be checked if specified

For example to test a page at `http://my.awesome.tld:9000/service/test.txt`
containing a single line `Hello World!` (with a newline at the end) and that
`http://my.awesome.tld:9000/service/my_redir` redirects to
`http://my.awesome.tld:9000/service/test.txt` using a 302 redirection we could
have our tests look like this.

```
deploy:
  name: my_awesome_container_name
  exposed_port: 9000

test_input:
  host: my.awesome.tld
  path_prefix: /service
  http_get_tests:
    - path: /test.txt
      content_hash: 03ba204e50d126e4674c005e04d82e84c21366780af1f43bd54a37816b6ab340
    - path: /my_redir
      allow_redirects: false
      status_code: 302
    - path: /my_redir
      content_hash: 03ba204e50d126e4674c005e04d82e84c21366780af1f43bd54a37816b6ab340
```

# Contributing

If you wish to contribute to this project, you are welcome to do so as long as
you keep the playbook clean by following these rules:

* YAML files must respect the default configuration for `yamllint`
* The playbook itself must pass `ansibl-lint` with its default configuration
* Python code must be correctly typed

To install `yamllint` and `ansible-lint` you can use the `requirements-dev.txt`
file like this.

```
$ pip install -r requirements-dev.txt
```

There is a simple CI which runs the lints on push which can be viewed [here][8].

[8]: https://drone.zuh0.com/zuh0/ansible-tomcat-sample-app/
