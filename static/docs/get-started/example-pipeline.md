# Example: Pipelines

To show DVC in action, let's play with an actual machine learning scenario.
Let's explore the natural language processing
([NLP](https://en.wikipedia.org/wiki/Natural_language_processing)) problem of
predicting tags for a given StackOverflow question. For example, we want one
classifier which can predict a post that is about the Python language by tagging
it `python`. This is a short version of the [Tutorial](/doc/tutorial).

In this example, we will focus on building a simple ML pipeline that takes an
archive with StackOverflow posts and trains the prediction model and saves it as
an output. See [Get Started](/doc/get-started) to see links to other examples,
tutorials, use cases if you want to cover other aspects of the DVC. The pipeline
itself is a sequence of transformation we apply to the data file:

![](/static/img/example-flow-2x.png)

DVC helps to describe these transformations and capture actual data involved -
input dataset we are processing, intermediate results (useful if some
transformations take a lot of time to run), output models. This way we can
capture what data and code were used to produce a specific model in a sharable
and reproducible way.

## Initialize

Okay, let's first download the code and set up a Git repository. This step has
nothing to do with DVC so far, it's just a simple preparation:

<details>

### Expand to learn how to download on Windows

Windows doesn't include the `wget` utility by default, so you'll need to use a
browser to download `pipeline.zip`. Save it into the `example` directory.
(Right-click [this link](https://code.dvc.org/tutorial/nlp/pipeline.zip) and
click `Save link as`(Chrome) or `Save object as`(Firefox)).

</details>

```dvc
$ mkdir example && cd example
$ git init
$ wget https://code.dvc.org/tutorial/nlp/pipeline.zip
$ unzip pipeline.zip -d code
$ rm -f pipeline.zip
$ git add code/
$ git commit -m "download and initialize code"
```

Now let's install the requirements. But before we do that, we **strongly**
recommend creating a virtual environment with a tool such as
[virtualenv](https://virtualenv.pypa.io/en/stable/):

```dvc
$ virtualenv -p python3 .env
$ source .env/bin/activate
$ echo ".env/" >> .gitignore
$ pip install -r requirements.txt
```

Next, we will create a pipeline step-by-step, utilizing the same set of commands
that are described in earlier [Get Started](/doc/get-started) chapters.

> Note that its possible to define more than one pipeline in each <abbr>DVC
> project</abbr>. This will be determined by the interdependencies between
> DVC-files, mentioned below.

Initialize DVC repository (run it inside your Git repository):

```dvc
$ dvc init
$ git commit -m "initialize DVC"
```

Download an input dataset to the `data/` directory and take it under DVC
control:

```dvc
$ mkdir data
$ wget -P data https://data.dvc.org/tutorial/nlp/25K/Posts.xml.zip
$ dvc add data/Posts.xml.zip
```

When we run `dvc add` `Posts.xml.zip`, DVC creates a
[DVC-file](/doc/user-guide/dvc-file-format).

<details>

### Expand to learn more about DVC internals

`dvc init` created a new directory `example/.dvc/` with `config`, `.gitignore`
files and the cache directory. These files and directories are hidden from users
in general. Users don't interact with these files directly. See
[DVC Files and Directories](/doc/user-guide/dvc-files-and-directories) to learn
more.

Note that the DVC-file created by `dvc add` has no dependencies, a.k.a. an
"_orphan_ stage file":

```yaml
md5: 4dbe7a4e5a0d41b652f3d6286c4ae788
outs:
  - cache: true
    md5: ce68b98d82545628782c66192c96f2d2
    path: Posts.xml.zip
```

This is the file that should be committed into a version control system instead
of the data file itself.

Actual data file `Posts.xml.zip` is linked into the `.dvc/cache` directory,
under the `.dvc/cache/ce/68b98d82545628782c66192c96f2d2` name and is added to
`.gitignore`. Even if you remove it in the workspace, or checkout a different
branch/commit the data is not lost if a corresponding DVC-file is committed.
It's enough to run `dvc checkout` or `dvc pull` to restore data files.

</details>

Commit the changes to Git repository:

```dvc
$ git add data/Posts.xml.zip.dvc data/.gitignore
$ git commit -m "add dataset"
```

## Define stages

Each [stage](/doc/commands-reference/run) – the parts of a pipeline – is
described by providing a command to run, input data it takes and a list of
output files. DVC is not Python or any other language specific and can wrap any
command runnable via CLI.

The first stage is to extract XML from the archive. Note that we don't need to
run `dvc add` on `Posts.xml` below, `dvc run` saves the data automatically
(commits into the cache, takes the file under DVC control):

```dvc
$ dvc run -d data/Posts.xml.zip \
          -o data/Posts.xml \
          -f extract.dvc \
          unzip data/Posts.xml.zip -d data
```

Similar to `dvc add`, `dvc run` creates a
[DVC-file](/doc/user-guide/dvc-file-format) (or "stage file").

<details>

### Expand to learn more about DVC internals

Here's what the DVC-file (stage file, with dependencies `deps`) looks like:

```yaml
cmd: ' unzip data/Posts.xml.zip -d data'
deps:
  - md5: ce68b98d82545628782c66192c96f2d2
    path: data/Posts.xml.zip
md5: abaf651846ec4fb7a4a8e1a685546ed9
outs:
  - cache: true
    md5: a304afb96060aad90176268345e10355
    path: data/Posts.xml
```

This file is using the same technique (checksums that point to the cache) to
describe and version control dependencies and outputs. Output `Posts.xml` file
is automatically added to the `.gitignore` file and a link is created into a
cache `.dvc/cache/a3/04afb96060aad90176268345e10355` to save it.

Two things are worth noticing here. First, by analyzing dependencies and outputs
that DVC-files describe, we can restore the full series of commands (pipeline
stages) we need to apply. This is important when you run `dvc repro` to
reproduce the final or intermediate result.

Second, you should see by now that the actual data is stored in the `.dvc/cache`
directory, each file having a name in a form of an md5 hash. This cache is
similar to Git's internal objects store but made specifically to handle large
data files.

> **Note!** For performance with large datasets, DVC can use file links from the
> cache to the workspace to avoid copying actual file contents. Refer to
> [File link types](/docs/user-guide/large-dataset-optimization#file-link-types-for-the-dvc-cache)
> to learn which options exist and how to enable them.

</details>

Next stage: let's convert XML into TSV to make feature extraction easier:

```dvc
$ dvc run -d code/xml_to_tsv.py -d data/Posts.xml \
          -o data/Posts.tsv \
          -f prepare.dvc \
          python code/xml_to_tsv.py data/Posts.xml data/Posts.tsv
```

Split training and test datasets. Here `0.2` is a test dataset split ratio,
`20170426` is a seed for randomization. There are two output files:

```dvc
$ dvc run -d code/split_train_test.py -d data/Posts.tsv \
          -o data/Posts-train.tsv -o data/Posts-test.tsv \
          -f split.dvc \
          python code/split_train_test.py data/Posts.tsv 0.2 20170426 \
                                          data/Posts-train.tsv data/Posts-test.tsv
```

Extract features and labels from the data. Two TSV as inputs with two pickle
matrices as outputs:

```dvc
$ dvc run -d code/featurization.py -d data/Posts-train.tsv -d data/Posts-test.tsv \
          -o data/matrix-train.pkl -o data/matrix-test.pkl \
          -f featurize.dvc \
          python code/featurization.py data/Posts-train.tsv data/Posts-test.tsv \
                                          data/matrix-train.pkl data/matrix-test.pkl
```

Train ML model on the training dataset. 20170426 is a seed value here:

```dvc
$ dvc run -d code/train_model.py -d data/matrix-train.pkl \
          -o data/model.pkl \
          -f train.dvc \
          python code/train_model.py data/matrix-train.pkl 20170426 data/model.pkl
```

Finally, evaluate the model on the test dataset and get the metrics file:

```dvc
$ dvc run -d code/evaluate.py -d data/model.pkl -d data/matrix-test.pkl \
          -M auc.metric \
          -f evaluate.dvc \
          python code/evaluate.py data/model.pkl data/matrix-test.pkl auc.metric
```

<details>

### Expand to learn more about DVC internals

By analyzing dependencies and outputs in DVC-files, we can generate a dependency
graph: a series of commands DVC needs to execute. `dvc repro` does this in order
to restore a pipeline and reproduce its intermediate or final results.

`dvc pipeline show` helps to visualize pipelines (run it with `-c` option to see
actual commands instead of DVC-files):

```dvc
$ dvc pipeline show --ascii evaluate.dvc

       .------------------------.
       | data/Posts.xml.zip.dvc |
       `------------------------'
                    *
                    *
                    *
            .-------------.
            | extract.dvc |
            `-------------'
                    *
                    *
                    *
            .-------------.
            | prepare.dvc |
            `-------------'
                    *
                    *
                    *
              .-----------.
              | split.dvc |
              `-----------'
                    *
                    *
                    *
            .---------------.
            | featurize.dvc |
            `---------------'
             **           ***
           **                **
         **                    **
.-----------.                    **
| train.dvc |                  **
`-----------'                **
             **           ***
               **       **
                 **   **
            .--------------.
            | evaluate.dvc |
            `--------------'
```

</details>

## Check results

> Since the dataset for this example is an extremely simplified to make it
> simpler to run this pipeline, exact metric number may vary sufficiently
> depending on Python version you are using and other environment parameters.

An easy way to see metrics across different branches:

```dvc
$ dvc metrics show
  auc.metric: 0.620091
```

It's time to save our pipeline. You can confirm that we do not save pickle model
files or raw datasets into Git using the `git status` command. We are just
saving a snapshot of the DVC-files that describe data and code versions and
relationships between them.

```dvc
$ git add *.dvc auc.metric
$ git commit -am "create pipeline"
```

## Reproduce

All stages could be automatically and efficiently reproduced even if any source
code files have been modified. For example:

Let's improve the feature extraction algorithm by making some modification to
the `code/featurization.py`:

```dvc
$ vi code/featurization.py
```

Specify `ngram` parameter in `CountVectorizer` (lines 72–73):

```python
bag_of_words = CountVectorizer(stop_words='english',
                              max_features=5000,
                              ngram_range=(1, 2))
```

Reproduce all required stages to get our target metrics file:

```dvc
$ dvc repro evaluate.dvc
```

> Since the dataset for this example is extremely simplified to make it simpler
> to run this pipeline, exact metric numbers may vary significantly depending on
> the Python version you are using and other environment parameters.

Take a look at the target metric improvement:

```dvc
$ dvc metrics show -a
master:
  auc.metric: 0.666618
```

## Conclusion

By wrapping your commands with `dvc run` it's easy to integrate DVC into your
existing ML development pipeline/processes without any significant effort to
rewrite your code.

The key detail to notice is that DVC automatically derives the dependencies
between the defined stages by building dependency graphs that represent data
pipelines.

Not only can DVC streamline your work into a single, reproducible environment,
it also makes it easy to share this environment by Git including the
dependencies — an exciting collaboration feature which gives the ability to
reproduce the research easily in a myriad of environments.
