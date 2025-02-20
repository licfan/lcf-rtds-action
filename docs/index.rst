Test lcf Interface ReadTheDocs and GitHub Actions
========================================

I like to use `ReadTheDocs <https://readthedocs.org/>`__ to build (and
version!) my docs, but I *also* like to use `Jupyter
notebooks <https://jupyter.org/>`__ to write tutorials. Unfortunately,
this has always meant that I needed to check executed notebooks (often
with large images) into my git repository, causing huge amounts of
bloat. Futhermore, the executed notebooks would often get out of sync
with the development of the code. **No more!!**

*This library avoids these issues by executing code on* `GitHub
Actions <https://github.com/features/actions>`_, *uploading build
artifacts (in this case, executed Jupter notebooks), and then (only
then!) triggering a ReadTheDocs build that can download the executed
notebooks.*

There is still some work required to set up this workflow, but this
library has three pieces that make it a bit easier:

1. A GitHub action that can be used to trigger a build for the current
   branch on ReadTheDocs.
2. A Sphinx extension that interfaces with the GitHub API to download
   the artifact produced for the target commit hash.
3. Some documentation that shows you how to set all this up!

Usage
-----

1. Set up ReadTheDocs
~~~~~~~~~~~~~~~~~~~~~

1. First, you’ll need to import your project as usual. If you’ve already
   done that, don’t worry: this will also work with existing ReadTheDocs
   projects.
2. Next, go to the admin page for your project on ReadTheDocs, click on
   ``Integrations`` (the URL is something like
   ``https://readthedocs.org/dashboard/YOUR_PROJECT_NAME/integrations/``).
3. Click ``Add integration`` and select
   ``Generic API incoming webhook``.
4. Take note of the webhook ``URL`` and ``token`` on this page for use
   later.

You should also edit your webhook settings on GitHub by going to
``https://github.com/USERNAME/REPONAME/settings/hooks`` and clicking "Edit"
next to the ReadTheDocs hook. On that page, you should un-check the
``Pushes`` option.

2. Set up GitHub Actions workflow
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In this example, we’ll assume that we have tutorials written as Jupyter
notebooks, saved as Python scripts using
`Jupytext <https://jupytext.readthedocs.io/en/latest/introduction.html>`__
(because that’s probably what you should be doing anyways!) in a
directory called ``docs/tutorials``.

First, you’ll need to add the ReadTheDocs webhook URL and token that you
recorded above as “secrets” for your GitHub project by going to the URL
``https://github.com/USERNAME/REPONAME/settings/secrets``. I’ll call
them ``RTDS_WEBHOOK_URL`` (include the ``https``!) and
``RTDS_WEBHOOK_TOKEN`` respectively.

For this use case, we can create the workflow
``.github/workflows/docs.yml`` as follows:

.. code:: yaml

   name: Docs
   on: [push, release]

   jobs:
     notebooks:
       name: "Build the notebooks for the docs"
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v2

         - name: Set up Python
           uses: actions/setup-python@v2
           with:
             python-version: 3.8

         - name: Install dependencies
           run: |
             python -m pip install -U pip
             python -m pip install -r .github/workflows/requirements.txt

         - name: Execute the notebooks
           run: |
             jupytext --to ipynb --execute docs/tutorials/*.py

         - uses: actions/upload-artifact@v2
           with:
             name: notebooks-for-${{ github.sha }}
             path: docs/tutorials

         - name: Trigger RTDs build
           uses: licfan/lcf-rtds-action@v1
           with:
             webhook_url: ${{ secrets.RTDS_WEBHOOK_URL }}
             webhook_token: ${{ secrets.RTDS_WEBHOOK_TOKEN }}
             commit_ref: ${{ github.ref }}

Here, we’re also assuming that we’ve added a ``pip`` requirements file
at ``.github/workflows/requirements.txt`` with the dependencies required
to execute the notebooks. Also note that in the ``upload-artifact`` step
we give our artifact that depends on the hash of the current commit.
This is crucial! We also need to take note of the ``notebooks-for-``
prefix because we’ll use that later.

It’s worth emphasizing here that the only “special” steps in this
workflow are the last two. You can do whatever you want to generate your
artifact in the previous steps (for example, you could use ``conda``
instead of ``pip``) because this workflow is not picky about how you get
there!

3. Set up Sphinx
~~~~~~~~~~~~~~~~

Finally, you can edit the ``conf.py`` for your Sphinx documentation to
add support for fetching the artifact produced by your action. Here is a
minimal example:

.. code:: python

   import os

   extensions = [... "rtds_action"]

   # The name of your GitHub repository
   rtds_action_github_repo = "USERNAME/REPONAME"

   # The path where the artifact should be extracted
   # Note: this is relative to the conf.py file!
   rtds_action_path = "tutorials"

   # The "prefix" used in the `upload-artifact` step of the action
   rtds_action_artifact_prefix = "notebooks-for-"

   # A GitHub personal access token is required, more info below
   rtds_action_github_token = os.environ["GITHUB_TOKEN"]

Where we have added the custom extension and set the required
configuration parameters.

You’ll need to provide ReadTheDocs with a GitHub personal access token
(it only needs the ``public_repo`` scope if your repo is public). You
can generate a new token by going to `your GitHub settings
page <https://github.com/settings/tokens>`__. Then, save it as an
environment variable (called ``GITHUB_TOKEN`` in this case) on
ReadTheDocs.

Examples
--------

Here are some example tutorials! See `the GitHub repository
<https://github.com/licfan/lcf-rtds-action/>`_ for the source of this example site.

.. toctree::
   :maxdepth: 2
   :caption: Tutorials

   tutorials/tutorial
   tutorials/another-tutorial
