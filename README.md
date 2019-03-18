# Application Development for AWS Lambda

Developing Lambda applications for AWS Lambda using Docker.

## Introduction

Lambda is a component of the _serverless_ toolkit that allows execution of code without the headache of server management. Applications are deployed to a sandboxed environment and must complete their task within 15 minutes. This type of behavior is ideal when composing a fleet of microservices that each perform a unit of work.

Developing for Lambda can be a challenge because the Lambda runtime is a walled-garden. Using docker compose, docker volumes, and the `lambci/lambda` images this process can be significantly streamlined.

## Development Stages

The development process can be broken down into six stages:
- **Lock** where the full list of dependencies is locked to support deterministic builds
- **Build** where the application's environment is constructed
- **Package** where the application's environment is zipped
- **Deploy** where the application's package(s) are persisted on S3
- **Test** where the application is tested in an environment that nearly replicates the Lambda runtime

### Setup

Let's begin by creating four files for a simple Python project:

```
/
├─┬ my_lambda_function/
│ ├── __init__.py
│ └── index.py
├── .env
├── docker-compose.yml
├── Makefile
└── setup.py
```

`my_lambda_function` will contain the Python project.
`.env` will contain environmental variables that might be needed for a stage. This file may contain sensitive information so it's important to remember to omit from source control.
`docker-compose.yml` will define our development step.
`Makefile` will assemble docker-compose steps into stages.
`setup.py` will help define our application package.

Add your AWS keys to `.env` as well as any other environment variables your application might require.

```bash
# ./.env
AWS_ACCESS_KEY_ID=<aws-access-key-id>
AWS_SECRET_ACCESS_KEY=<aws-secret-access-key>
AWS_DEFAULT_REGION=us-east-1
```

A simple `setup.py` might look like this:

```python
# ./setup.py
from setuptools import setup

setup(
    name='my-lambda-function',
    packages=['my_lambda_function'],
    version='0.1.0',
)
```

### Lock

Inside our docker-compose configuration we will define a `lock` service and volume:

```yaml
# ./docker-compose.yml
version: '3'
services:

  # Lock dependencies
  lock:
    entrypoint: pipenv lock
    environment:
      WORKON_HOME: /var/lock
    image: lambci/lambda:build-python3.7
    volumes:
      - lock:/var/lock
      - ./:/tmp/task

volumes:
  lock:
```

We will use the built-in installation of pipenv to manage our dependencies. By defining `WORKON_HOME` to be the mounted location of the `lock` volume we can re-use the pipenv virtual environment for subsequent runs.

Update the Makefile with instructions on locking:

```Makefile
# ./Makefile
lock:
  docker-compose run --rm lock
  docker-compose run --rm lock -r > requirements.txt
  docker-compose run --rm lock -r -d > requirements-dev.txt
```

Run `make lock` to generate a `Pipefile`, `Pipfile.lock`, `requirements.txt`, and `requirements-dev.txt`:

```
  /
  ├─┬ my_lambda_function/
  │ ├── __init__.py
  │ └── index.py
  ├── .env
  ├── docker-compose.yml
  ├── Makefile
* ├── Pipfile
* ├── Pipfile.lock
* ├── requirements.txt
* ├── requirements-dev.txt
  └── setup.py
```

Let's assume our project will depend on the `pandas` and `requests` libraries. Update the `Pipfile` to require them:

```toml
[[source]]
name = "pypi"
url = "https://pypi.org/simple"
verify_ssl = true

[dev-packages]

[packages]
pandas = ">=0.24"
requests = ">=2.21"

[requires]
python_version = "3.7"
```

Run `make lock` again to lock this new dependency and update `Pipfile.lock` and `requirements.txt`.

Any time additional dependencies are added to `Pipfile`, re-run `make lock` to lock the dependencies.

### Build

Now that we have fully locked requirements, we can define our build stage in our compose configuration. We will split the build process into two: one for building the dependencies and one for building the application:

```yaml
# ./docker-compose.yml
version: '3'
services:
  # (Lock stage omitted for brevity)

  # Build
  build:
    entrypoint: pip install
    image: lambci/lambda:build-python3.7
    volumes:
      - lambda:/var/task
      - layer:/opt/python
      - ./:/tmp/build
    working_dir: /tmp/build

volumes:
  lock:
  lambda:
  layer:
```

Update the Makefile with instructions on building:

```Makefile
# ./Makefile
lock: # (omitted for brevity)

build:
  # Install the lambda code to /var/task (with no dependencies)
  docker-compose run --rm build -t /var/task --no-deps .
  # Install the dependency layer to /opt/python
  docker-compose run --rm build -t /opt/python -r requirements.txt
```

Run `make build` to install your application to the `lambda` and `layer` volumes.

_Note: Splitting the application and layer in this manner is not required, but it's a handy way to separate your application logic from its core dependencies. Assuming your application codebase is small, updating your Lambda function becomes fairly trivial._

### Package

Add a packaging service that simply zips the working directory to stdout.

```yaml
# ./docker-compose.yml
version: '3'
services:

  # (Lock/Build stages omitted for brevity)

  # Package
  package:
    entrypoint: zip -r - .
    image: lambci/lambda:build-python3.7
    volumes:
      - lambda:/var/task
      - layer:/opt/python

volumes:
  lock:
  lambda:
  layer:
```

Update the Makefile with instructions on packaging:

```Makefile
# ./Makefile
lock: # (omitted for brevity)
build: # (omitted for brevity)

package:
  mkdir -p dist
  docker-compose run --rm -T package > dist/lambda.zip
  docker-compose run --rm -T -w /opt package > dist/layer.zip
```

Run `make package` to create your zip packages under `./dist`.

### Deploy

Not to be confused with deploying the Lambda, the deploy stage is used to upload your packages to S3.

Add a deploy service to your compose configuration, being sure to include AWS credentials in the environment:

```yaml
# ./docker-compose.yml
version: '3'
services:

  # (Lock/Build/Package stages omitted for brevity)

  # Deploy lambda/layer to S3
  deploy:
    entrypoint: aws s3
    environment:
      AWS_ACCESS_KEY_ID:
      AWS_SECRET_ACCESS_KEY:
      AWS_DEFAULT_REGION:
    image: lambci/lambda:build-python3.7
    volumes:
      - ./dist:/var/task

volumes:
  lock:
  lambda:
  layer:
```

Update the Makefile with instructions on deploying to S3:

```Makefile
# ./Makefile
lock: # (omitted for brevity)
build: # (omitted for brevity)
package: # (omitted for brevity)

deploy:
  docker-compose run --rm deploy sync . s3://my-bucket/path/to/prefix/
```

Run `make deploy` to send your zip packages to S3.

### Test

In order to test your application in a Lambda-like runtime, add a test configuration to compose:

```yaml
# ./docker-compose.yml
version: '3'
services:

  # (Lock/Build/Package/Deploy stages omitted for brevity)

  # Test function
  test:
    env_file: .env
    image: lambci/lambda:python3.7
    volumes:
      - lambda:/var/task
      - layer:/opt/python

volumes:
  lock:
  lambda:
  layer:
```

Update the Makefile with instructions on running the application:

```Makefile
# ./Makefile
lock: # (omitted for brevity)
build: # (omitted for brevity)
package: # (omitted for brevity)
deploy: # (omitted for brevity)

test:
  docker-compose run --rm test my_lambda_function.index.handler '{}'
```

Run `make test` to see your application in action!

### Clean

Finally, update your Makefile to clean your environment:

```Makefile
# ./Makefile
lock: # (omitted for brevity)
build: # (omitted for brevity)
package: # (omitted for brevity)
deploy: # (omitted for brevity)
test: # (omitted for brevity)

clean:
  docker-compose down --volumes
```

Run `make clean` to remove any networks, containers, and volumes from your system.
