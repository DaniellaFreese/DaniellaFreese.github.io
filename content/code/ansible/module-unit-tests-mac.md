---
title: "How to Write an Ansible Module Unit Test in the Ansible Collection Paradigm"
date: 2022-10-27
draft: false

categories: ["ansible"]
tags: ["ansible", "mac", "module", "unit test"]
toc: false
summary: "This tutorial will cover two major issues. **One** - how to setup your local workstation to build and test an ansible collection, and **two** - resolving a mac error related to cgroups. We will finish with a helloworld unit test to validate."
author: "djfreese"
---

**Two major issues** came up for me when it came just getting my local mac setup to write unit tests for an ansible module.

**First** one was ansible expects a very specific file structure layout, without the `ansible-test` command would fail.

**Second** was there is an open Docker on Mac related issue that I needed to resolve. With these two in issues resolved I could successful run some simple unit tests. 

So here I will step through my setup and then the quick helloworld unit test I used to validate everything was working. This super quick tutorial assumes you have an ansible collection you are already working on.

## How to Setup your Local ENV to Develop Ansible-Collections

With ansible collections, ansible is expecting the collection to have a very specific path of `/ansible_collections/{collection_namespace}/{collection}`.

If you look in your `~/.ansible/collections` directory you'll find all the collections that you've downloaded and maybe in use for writing actual playbooks.

The problem is unless you have the right file structure that ansible expects running the `ansible-tests` commmands fails but I did not want to contaminate my `.ansible` directory with my dev code.

### The Actual Setup

To resolve the file structure concern this is my current setup:

**First** - In my projects folder (where I typically store all my code projects) I created a new folder ansible_collections

```console
mkdir $HOME/Projects/ansible_collections/{collection_namespace}/{collection_name}
```

**Second** - clone down the git repo to my Projects folder.

```bash
git clone ansible-collection-hello.git
```

**And Last** - copied all the content into the collection_directory

```bash
cp -r ansible-collection-hello/ ansible_collections/{collection_namespace}/{collection_name}/
```

With that file structure place, I could ***almost*** run ` ansible-test units --docker -v --python 3.10` with no errors

## Mac Issue

There is an open issue with Docker and Mac: https://github.com/ansible/ansible/issues/77903.

I had to shut down Docker.

Open `~/Library/Group\ Containers/group.com.docker/settings.json` and update the deprecatedCgroupv1 to true: `"deprecatedCgroupv1": true`

With Docker restart, the error was resolved.

## Build a Hello World Module Unit Test

Within the collection, the modules are located in:

```bash
{collection-name}/plugins/modules/{module-name}.py
```

So the unit test file structure is going to match that pattern but for testing. which will look like

```bash
{collection-name}/tests/unit/plugins/modules/{module-name}.py
```

But this is just the convention, you can write whatever python file name you need. So lets do a hello world unit test together.

### Hello World Unit test

**First** - create a helloworld.py in the modules directory:

```bash
mkdir my-sample-collection/tests/unit/plugins/modules/helloworld.py
```

**Second** - open helloworld.py to write the unit test. In python we will be using the unittest package

```python
from __future__ import absolute_import, division, print_function
__metaclass__ = type

from unittest import TestCase


# Tests Module Arguments onlys
class TestHelloworld(TestCase):

    def test_hello(self):
        self.assertTrue(True)
```

With the `unittest` python package, any method in the classes that you want to execute as a test needs to be prefixed with `test_{name of test}`.

In the above example `def test_hello(self)`. If the function is `def hello(self)`, no dice the function will not run when the tests command is excuted.

So, save that unit test and you can now execute the following:

```bash
ansible-test units --docker -v --python 3.10 tests/unit/plugins/modules/helloworld.py
```

### Next Steps

So that is the basic setup for writing units testing for an ansible module that is maintained in an ansible collection.

Next you need to write an actual test for the module, and figure out how to mock the module calls.

We can talk about this in the next tutorial, but for now! Find your favorite [ansible-collection in github](https://github.com/ansible-collections) as a reference point.
