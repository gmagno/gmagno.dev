---
date: "2019-01-27"
draft: false
showpagemeta: true
showcomments: true
title: "Building python wheels for many linux-es"
imgurl: "https://unsplash.com/photos/xyDQNmT6vSs"
---

> On building and pushing python wheels for Linux. An example is shown with [nb-cpp](https://github.com/gmagno/nb-cpp), a python package implemented in C++ with [pybind11](https://github.com/pybind/pybind11).

{{< figure src="/jots/building-linux-wheels/austrian-national-library-tuzqM1L1l-Q-unsplash.jpg" caption="Photo by [Austrian National Library](https://unsplash.com/photos/tuzqM1L1l-Q)" link="https://unsplash.com/photos/tuzqM1L1l-Q" >}}

Packaging and distributing is certainly one crucial step in the process of creating an app/lib. This is the moment you expose your app to the world,hopefully in a way that people can easily install.
However, python is not known for its reputation with packaging, specifically for Linux based platforms where glibc version may differ among distros and building and distributing portable binaries can be challenging. Effort to simplify this process led to the creation of [manylinux](https://github.com/pypa/manylinux) and [PEP513](https://www.python.org/dev/peps/pep-0513), which will be referred in the following sections.

For now let's start with how the app/lib directory should be structured.

### Project directory structure

Although not mandatory, the project directory should follow a standard structure. Here's the dir structure being followed:

```bash
    proj-name/
    ├─ proj_name/
    │  ├─ __init__.py
    │  ├─ proj_name.py
    │  ├─ foo.py
    │  ├─ bar.py
    │  ├─ ...
    │  └─ tests/
    │     └─ ...
    ├─ setup.py
    └─ .git/
```

Essentially `proj-name/` can be whatever pleases you and potentially matches the name of the git repo. This is not relevant for the packaging though.

On the other hand `proj_name` should match the param `name` set in the `setup.py::setup(name='proj_name')` function. This is where all the source code will sit.
Once the package is pushed to PyPI it will be installable with `pip install proj_name`.

### Setuptools setup.py

To package our app/lib we start by populating the `setup.py` script that has the metadata which will be pushed to PyPI along with the actual binaries and also describes how to build eventual extensions. In this post an example of building a C++ extension, that uses pybind11, is shown. The script should call a `setup()` function which at the bare minimum passes the arguments `name`, `version` and `packages`. The first two are self-explanatory, and the latter is a list of module names that should be
packaged. Instead of manually listing the modules, one can use the function `setuptools.findpackages` which will conveniently do exactly that.
Follows an example of a `setup()` function with a few more arguments:

```python
setup(
    name='proj_name',
    version='0.0.1',
    author='author',
    author_email='user@domain.com',
    description='A short description of the app/lib',
    long_description='A longer one',
    ext_modules=[CMakeExtension('proj_name')],
    cmdclass=dict(build_ext=CMakeBuild),
    packages=find_packages(),
    classifiers=[
        'Development Status :: 3 - Alpha',
        'Intended Audience :: Developers',
        'License :: OSI Approved :: MIT License',
        'Topic :: Scientific/Engineering',
        'Programming Language :: Python :: 3',
    ],
    zip_safe=False,
)
```

### Packaging

The actual building and packaging is done with the command `pip wheel`:

```bash
pip wheel /dir/to/proj-name/ -w /dir/where/wheels/are/written/
```

which will generate a file with extension .whl, the so called binary or wheel, for example: `proj_name-0.0.1-cp35-cp35m-linux_x86_64.whl`. This is effectively a zip file with the python modules and built extensions.
This package has a few problems though:

1. If your linux glibc is not old enough, there is a good chance the wheels you are building won't run on distros with older glibc's,
2. Similarly if the app/lib depends on external shared libraries, different linux distros might have versions of these shared libs that are incompatible
with the one the project was built against,
3. The file name has a tag, `linux_x86_64` (assuming a 64bits architecture), that PyPI rejects when trying to upload,

The first issue can be sorted by running the building process in a linux distro with a relatively old glibc, e.g CentOS5. This can be done conveniently with a docker image, e.g: `quay.io/pypa/manylinux1_x86_64`. For details on this check [manylinux](https://github.com/pypa/manylinux).

To address the second and third issues mentioned above, a tool named `auditwheel` was created. This tool acts on a previously built wheel, and generates a new and self-contained one, with all external shared libs inside, and with its name appropriately tagged `manylinux1_x86_64`. Example of usage:

```bash
auditwheel repair /dir/where/wheel/was/created -w /dir/with/manylinux/wheel/
```

which would generate the new wheel:
`proj_name-0.0.1-cp35-cp35m-manylinux1_x86_64.whl`.

Finally, and to make the wheel available to public, we need to upload it to PyPI. This can be done with another tool, `twine`, as follows:

```bash
twine upload proj_name-0.0.1-cp35-cp35m-manylinux1_x86_64.whl
```

For additional options I recommend reading twine
[documentation](https://twine.readthedocs.io).

I would highlight the following ones though:

```bash
--repository-url REPOSITORY_URL
    The repository (package index) URL to upload the package to. This
    overrides --repository. (Can also be set via TWINE_REPOSITORY_URL
    environment variable.)
-u USERNAME, --username USERNAME
    The username to authenticate to the repository (package index) as.
    (Can also be set via TWINE_USERNAME environment variable.)
-p PASSWORD, --password PASSWORD
    The password to authenticate to the repository (package index) with.
    (Can also be set via TWINE_PASSWORD environment variable.)
--skip-existing
    Continue uploading files if one already exists. (Only valid when
    uploading to PyPI. Other implementations may not support this.)
```

Specifying a different repository can be handy, for instance the test.pypi repo `https://test.pypi.org/legacy/` can be used for testing purposes, this way you know you won't be polluting the main repository.
In order to upload the wheel to PyPI you need to be authenticated. If the user and password are not specified either through flags or environment variables, you will be prompted by the tool which is an issue when automating this process.
Finally uploading the same version of a wheel more than once leads to an error. To avoid it, the flag `--skip-existing` may be set. Once again this can be useful in an automated build.

### Example: Newton Basins C++/Pybind11 automated with Travis-CI

Follows an example of a simple python lib, `nb-cpp` which stands for Newton Basins in C++, that implements the generation of
[Newton Fractals](https://en.wikipedia.org/wiki/Newton_fractal), written in C++ with python bindings, using [pybind11](https://github.com/pybind/pybind11).

The project is hosted on github [here](https://github.com/gmagno/nb-cpp)".

It is not the aim of this post to elaborate on the implementation of `nb-cpp` or how pybind11 works, that could be the subject of another post.

#### setup.py

The setup.py is mainly the definition of a couple of classes that handle the cmake-ing and make-ing of the C++ project, and a setup() function whose purpose was mentioned above.

#### .travis.yml

If you are not familiar with Travis, please check their
[website](https://travis-ci.org).
In short Travis is a Continuous Integration service used to build and test software projects hosted at Github.

The .travis.yml file defines a list of configurations and instructions that tell how the job is setup and run.
Follows the .travis.yml we will be using:

```yaml
notifications:
  email: false
if: tag IS present
matrix:
  include:

- sudo: required
    services:
  - docker
    env:
  - DOCKER_IMAGE=quay.io/pypa/manylinux1_x86_64
- sudo: required
    services:
  - docker
    env:
  - DOCKER_IMAGE=quay.io/pypa/manylinux1_i686
  - PRE_CMD=linux32
install:
- docker pull $DOCKER_IMAGE
script:
- docker run --rm -v `pwd`:/io -e TWINE_USERNAME -e TWINE_PASSWORD $DOCKER_IMAGE $PRE_CMD /io/travis/build-wheels.sh
- ls wheelhouse/
```

I would highlight the `matrix` field which allows us to run two instances of the job in parallel, one for i686 builds and the other for x86_64.
Each job will spawn a docker container of a
[modified image](https://github.com/pypa/manylinux/tree/master/docker) of a CentOS5 that will run the `build-wheels.sh` script which will do the actual building and push the wheels to PyPI:

```bash
# !/bin/bash
set -e -x

export PYHOME=/home
cd ${PYHOME}

/opt/python/cp37-cp37m/bin/pip install twine cmake
ln -s /opt/python/cp37-cp37m/bin/cmake /usr/bin/cmake

# Compile wheels

for PYBIN in /opt/python/cp3*/bin; do
    "${PYBIN}/pip" wheel /io/ -w wheelhouse/
    "${PYBIN}/python" /io/setup.py sdist -d /io/wheelhouse/
done

# Bundle external shared libraries into the wheels and fix naming

for whl in wheelhouse/*.whl; do
    auditwheel repair "$whl" -w /io/wheelhouse/
done

# Test

for PYBIN in /opt/python/cp3*/bin/; do
    "${PYBIN}/pip" install -r /io/requirements-test.txt
    "${PYBIN}/pip" install --no-index -f /io/wheelhouse nb-cpp
    (cd "$PYHOME"; "${PYBIN}/pytest" /io/test/)
done

# Upload

for WHEEL in /io/wheelhouse/nb_cpp*; do
    /opt/python/cp37-cp37m/bin/twine upload \
        --skip-existing \
        -u "${TWINE_USERNAME}" -p "${TWINE_PASSWORD}" \
        "${WHEEL}"
done
```

In other words, the script is building wheels for each python3.x version and generating an sdist archive with the source code and CMakeLists.txt. It runs `auditwheel repair` for the reasons mentioned in the previous sections, followed by a couple of tests to ensure the wheels work as expected. Then it uploads all the wheels and the sdist archive to PyPI, which are publicly
available at the project [download files](https://pypi.org/project/nb-cpp/#files) page.

And `pip install nb-cpp` will just work:

```bash
Collecting nb-cpp
Downloading https://files.pythonhosted.org/packages/.../nb_cpp-0.0.8-cp35-cp35m-manylinux1_x86_64.whl (99kB)
    100% |████████████████████████████████| 102kB 764kB/s
Installing collected packages: nb-cpp
Successfully installed nb-cpp-0.0.8
```

Finally let's check our lib runs as expected:

```bash
import matplotlib as mpl
import matplotlib.pyplot as plt
import numpy as np
​
import nb_cpp
​
def main():
    hsv = nb_cpp.compute(
        imw=512, imh=512,
        coefs=np.array([1, 0, 0, 0, 0, 0, 1], dtype=np.float),
        crmin=-5.0, crmax=5.0,
        cimin=-5.0, cimax=5.0,
        itmax=30, tol=1e-6
    )
    rgb = mpl.colors.hsv_to_rgb(hsv)
    plt.figure('Example: Newton Basins 6th order poly.')
    plt.imshow(rgb)
    plt.show()
​
if __name__ == '__main__':
    main()
```

![newton basins](/jots/building-linux-wheels/nb6.png)
