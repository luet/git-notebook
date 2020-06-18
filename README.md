---
Using Git with Jupyter Notebooks
---

-   [Script to remove the metadata and cells
    output](#script-to-remove-the-metadata-and-cells-output)
-   [Git configuration](#git-configuration)
    -   [clean filter](#clean-filter)
    -   [Ignoring checkpoint directory](#ignoring-checkpoint-directory)
-   [A note on the behavior of this
    setting](#a-note-on-the-behavior-of-this-setting)
-   [References](#references)

Jupyter Notebooks cannot easily be kept under version control in Git
because running notebooks adds metadata and cell outputs to the source
code so that a notebook will appear to be changed even if the source
code has not been modified. In effect the source code and the results of
the code are kept in the same file.

One solution to this problem is to strip the metadata and cells outputs
before adding the change in Git. This involves two components:

1.  Removing the extraneous information from the `ipynb` file: Jupyter
    Notebooks are JSON files, so we can strip the metadata from the
    notebooks by using a \"grep-like\" code for JSON called
    [jq](https://stedolan.github.io/jq/).
2.  Executing this command automatically when adding changes to the
    staging area of Git. This is done with [the `clean` filter from
    Git](https://git-scm.com/book/en/v2/Customizing-Git-Git-Attributes).

Script to remove the metadata and cells output
==============================================

-   Install `jq` and put it somewhere that is in your `PATH`.

-   Create a script called `nbstrip_jq` that is a wrapper for `jq` to
    strip the notebooks of the metadata and cell outputs. There are some
    decisions to be made as to what you want to keep and you should
    experiment with it and modify the script to your needs.

    ``` {.shell}
    #! /bin/bash
    jq --indent 1 '(.cells[] | select(has("outputs")) | .outputs) = [] | (.cells[] | select(has("execution_count")) | .execution_count) = null | .metadata = {"language_info": {"name": "python", "pygments_lexer": "ipython3"}}' "$1"
    ```

-   I also created a script that is not called bu Git but that I like to
    call by hand. It just applies the filter above to all the notebooks
    in a directory.

    ``` {.shell}
    #! /bin/bash
    SAVEIFS=$IFS
    IFS=$(echo -en "\n\b")

    for nbfile in *.ipynb; do
    echo "$( nbstrip_jq $nbfile )" > $nbfile
    done

    IFS=$SAVEIFS
    ```

Git configuration
=================

clean filter
------------

-   This approach uses the `clean` filter from Git. Because I want to
    apply this filter to all the Git repo containing Jupyter Notebooks,
    I define those filter and settings globally for my account. You can
    do that by creating a file that I called
    `$HOME/.gitattributes_global` and that contains:

    ``` {.conf}
    *.ipynb filter=nbstrip_full
    ```

-   Then you need to configure Git to use that file:

    ``` {.example}
    git config --global core.attributesfile $HOME/.gitattributes_global
    ```

    this should add a entry to your `$HOME/.gitcongig`:

    ``` {.conf}
    [core]
        attributesfile = /Users/luet/.gitattributes_global
    ```

-   The next step is to define the filter `nbstrip_full`.

    ``` {.example}
    git config --global filter.nbstrip_full.clean "nbstrip_jq %f"
    git config --global filter.nbstrip_full.smudge cat
    ```

    the use of `cat` for the `smudge` filter simply means that you
    don\'t change the file when you take it from staging area to the
    working directory, for instance when you checkout a file.

Ignoring checkpoint directory
-----------------------------

-   In addition to adding metadata to the notebook files, Jupyter
    creates a directory for checkpointing the notebooks called
    `.ipynb_checkpoints/`. These should be excluded from versioning as
    well. I do that by creating a file `$HOME/.gitignore_global` that
    looks like:

    ``` {.conf}
    .ipynb_checkpoints/
    ```

    and this file needs to be added to the Git configuration with:

    ``` {.example}
    git config --global core.attributesfile $HOME/.gitignore_global
    ```

A note on the behavior of this setting
======================================

-   Once you have the setup described above in place, it will be run
    automatically, but there are some behaviors that can give pause.
-   Let\'s assume that you have a Jupyter notebook called `test.ipynb`
    free of metadata in your Git repo.
-   You then execute it, without changing the source code, and save it.
-   The command `git status` will tell you that the file `test.ipynb`
    has been modified, but `git diff` will show no difference, and `git
     add -A` followed by `git commit` will tell you that nothing needs
    to be committed. This is because the `clean` filter is not called by
    `git status`. This is in part why I created the `nbstrip_all` above.
    This way, a `git status` will really show me what has changed.

References
==========

-   This was heavily inspired by this post by Tim Staley: [Making Git
    and Jupyter Notebooks play nice - Tim
    Staley](http://timstaley.co.uk/posts/making-git-and-jupyter-notebooks-play-nice/)
-   Smudge and clean filters: [Git - Git
    Attributes](https://git-scm.com/book/en/v2/Customizing-Git-Git-Attributes)
-   Jq program: [jq](https://stedolan.github.io/jq/).
