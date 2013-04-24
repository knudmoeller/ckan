Doing a CKAN Release
====================

These are the steps followed by CKAN developers to do a release. To get an
overview of CKAN releases, check :doc:`release-cycle`.


Doing a Beta release
--------------------

Beta releases are branched off a certain point in master and will eventually
become stable releases.

1. Create a new release branch::

        git checkout -b release-v1.8

   Update ``ckan/__init__.py`` to change the version number to the new version
   with a *b* after it, e.g. *1.8b*.
   Commit the change and push the new branch to GitHub::

        git commit -am "Update version number"
        git push origin release-v1.8

   You will probably need to update the same file on master to increase the
   version number, in this case ending with an *a* (for alpha).

2. During the beta process, all changes to the release branch must be
   cherry-picked from master (or merged from special branches based on the
   release branch if the original branch was not compatible).

3. As in the master branch, if some commits involving CSS changes are
   cherry-picked from master, the less compiling command needs to be run on
   the release branch. This will update the ``main.css`` file::

        ./bin/less --production
        git commit -am "Rebuild CSS"
        git push

   There will be a final front-end build before the actual release.

4. The beta staging site (http://beta.ckan.org, currently on s084) should be
   updated regularly to allow user testing.

5. Once the translation freeze is in place (ie no changes to the translatable
   strings are allowed), strings need to be extracted and uploaded to
   Transifex_:

   a. Install the Babel and Transifex libraries if necessary::

        pip install --upgrade Babel
        pip install transifex-client

   b. Create a ``~/.transifexrc`` file if necessary with your login details
      (``token`` should be left blank)::

        [https://www.transifex.com]
        hostname = https://www.transifex.com
        username = <username>
        password = <password>
        token =

   c. Extract new strings from the CKAN source code into the ``ckan.pot``
      file. The pot file is a text file that contains the original,
      untranslated strings extracted from the CKAN source code.::

        python setup.py extract_messages

   d. Pull new and updated translations from Transifex into the ``ckan.po``
      files::

        tx pull --all --force

      The po files are text files, one for each language CKAN is translated to,
      that contain the translated strings next to the originals. Translators edit
      the po files (on Transifex) to update the translations. We never edit the
      po files locally.

      ``--force`` tells Transifex to update all ``ckan.po`` files, regardless of the
      modification time.

   e. Run our script that checks for mistakes in the ckan.po files::

        pip install polib
        paster check-po-files ckan/i18n/*/LC_MESSAGES/ckan.po

      If the script finds any mistakes correct them on Transifex and then run the
      tx pull command again, don't edit the files directly. Repeat until the
      script finds no mistakes.

   f. Edit ``.tx/config``, on line 4 to set the Transifex 'resource' to the new
      major release name (if different), using dashes instead of dots.
      For instance v1.2, v1.2.1 and v1.2.2 all share: ``[ckan.1-2]``.

   g. Update the ``ckan.po`` files with the new strings from the ``ckan.pot`` file::

        python setup.py update_catalog --no-fuzzy-matching

      Any new or updated strings from the CKAN source code will get into the po
      files, and any strings in the po files that no longer exist in the source
      code will be deleted (along with their translations).

      We use the ``--no-fuzzy-matching`` option because fuzzy matching often
      causes problems with Babel and Transifex.

   h. Create a new resource in the CKAN project on Transifex by pushing the new
      pot and po files::

        tx push --source --translations --force

      Because it reads the new version number in the ``.tx/config`` file, tx will
      create a new resource on Transifex rather than updating an existing
      resource (updating an existing resource, especially with the ``--force``
      option, can result in translations being deleted from Transifex).

   i. Update the ``ckan.mo`` files by compiling the po files::

        python setup.py compile_catalog

      The mo files are the files that CKAN actually reads when displaying
      strings to the user.

   j. Commit all the above changes to git and push them to GitHub::

        git commit -am " Update strings files before CKAN X.Y call for translations"
        git push

   k. Announce that strings for the new release are ready for translators. Send
      an email to the mailing lists, tweet or post it on the blog. Make sure to
      post a link to the correct Transifex resource (like
      `this one <https://www.transifex.com/projects/p/ckan/resource/2-0/>`_)
      and tell users that they can register on Transifex to contribute.

   l. A week before the translations will be closed send a reminder email.

   m. Once the translations are closed, pull the updated strings from Transifex,
      check them, compile and push as described in the previous steps::

        tx pull --all --force
        paster check-po-files ckan/i18n/*/LC_MESSAGES/ckan.po
        python setup.py compile_catalog
        git commit -am " Update translations from Transifex"
        git push


Doing a Proper release
----------------------

Once the release branch has been thoroughly tested and is stable we can do
a release.

1. Run the most thorough tests::

        nosetests ckan/tests --ckan --ckan-migration --with-pylons=test-core.ini

2. Do a final build of the front-end and commit the changes::

        paster front-end-build
        git commit -am "Rebuild front-end"

3. Update the CHANGELOG.txt with the new version changes:

   * Add the release date next to the version number
   * Add the following notices at the top of the release, reflecting whether
     updates in requirements, database or Solr schema are required or not::

        Note: This version requires a requirements upgrade on source installations
        Note: This version requires a database upgrade
        Note: This version does not require a Solr schema upgrade

   * Check the issue numbers on the commit messages for information about
     the changes. These are some helpful git commands::

        git branch -a --merged > merged-current.txt
        git branch -a --merged ckan-1.8.1 > merged-previous.txt
        diff merged-previous.txt merged-current.txt

        git log --no-merges release-v1.8.1..release-v2.0
        git shortlog --no-merges release-v1.8.1..release-v2.0

4. Check that the docs compile correctly::

        rm build/sphinx -rf
        python setup.py build_sphinx

5. Remove the beta letter in the version number in ``ckan/__init__.py``
   (eg 1.1b -> 1.1) and commit the change::

        git commit -am "Update version number for release X.Y"

6. Tag the repository with the version number, and make sure to push it to
   GitHub afterwards::

        git tag -a -m '[release]: Release tag' ckan-X.Y
        git push --tags

7. Upload the release to PyPI::

        python setup.py sdist upload

   You will need a PyPI account with admin permissions on the ckan package,
   and your credentials should be defined on a ``~/.pypirc`` file, as described
   `here <http://docs.python.org/distutils/packageindex.html#pypirc>`_
   If you make a mistake, you can always remove the release file on PyPI and
   re-upload it.

8. Enable the new version of the docs on Read the Docs (you will need an admin
   account):

   a. Go to the `versions page <http://readthedocs.org/dashboard/ckan/versions/>`
      and enable the relevant release (make sure to use the tag, ie ckan-X.Y,
      not the branch, ie release-vX.Y).

   b. If it is the latest stable release, set it to be the Default Version and
      check it is displayed on http://docs.ckan.org.

9. Write a `CKAN Blog post <http://ckan.org/wp-admin>`_ and send an email to
   the mailing list announcing the release, including the relevant bit of
   changelog.


.. _Transifex: https://www.transifex.com/projects/p/ckan
.. _ReadTheDocs: http://readthedocs.org/dashboard/ckan/versions/