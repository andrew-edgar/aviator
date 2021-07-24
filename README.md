# Aviator

[![GoDoc](https://godoc.org/github.com/JulzDiverse/aviator/cockpit?status.svg)](https://godoc.org/github.com/JulzDiverse/aviator/cockpit)

Aviator is a tool to template & merge YAML files in a convenient fashion based on a configuration file called `aviator.yml`. Aviator utilizes [Spruce](https://github.com/geofffranks/spruce) for templating and merging and therefore enables you to use all the Spruce operators in your YAML files.

If you have to handle rather complex YAML files (for Kubernetes, Concourse, or Bosh), you just provide the flight plan (`aviator.yml`), the Aviator flies you there.

**Reads:**

- [Using Aviator for Concourse Pipelines](https://medium.com/@julian.skupnjak/create-a-concourse-pipeline-for-your-cloud-foundry-apps-with-ease-98ceaa055be7)
- [Using Aviator as a alternitive to Helm](https://medium.com/@julian.skupnjak/an-alternative-to-helm-aviator-7099d50f2d28)


## Installation

### OS X

```
$ wget -O /usr/local/bin/aviator https://github.com/JulzDiverse/aviator/releases/download/v1.8.1/aviator-darwin-amd64 && chmod +x /usr/local/bin/aviator
```

**Via Homebrew**

```
$ brew tap julzdiverse/tools  
$ brew install aviator
```

### Linux

```
$ wget -O /usr/bin/aviator https://github.com/JulzDiverse/aviator/releases/download/v1.8.1/aviator-linux-amd64 && chmod +x /usr/bin/aviator
```

### Windows (NOT TESTED)

```
https://github.com/JulzDiverse/aviator/releases/download/v1.8.1/aviator-win
```

## Usage

To run Aviator navigate to a directory that contains an `aviator.yml` and run:

```
$ aviator
```

OR

Specify an AVIATOR YAML file  with the [--file|-f] option:

```
$ aviator -f myAviatorFile.yml
```

## Configure an `aviator.yml`

- [Configure an `aviator.yml`](#configure-an-aviatoryml)
	- [Spruce Section](#spruce-section)
		- [Base (`string`)](#base-string)
		- [Prune (`Array`)](#prune-array)
		- [cherry_pick (`array`)](#cherrypick-array)
		- [go_patch (`bool`)](#gopatch-bool)
		- [Merge (`Array`)](#merge-array)
		- [skip_eval (`bool`)](#skipeval-bool)
		- [To (`string`)](#to-string)
		- [ForEach](#foreach)
		- [Read From and Write To Internal Data Store](#read-from-and-write-to-internal-datastore)
		- [Environment Variables](#environment-variables)
		- [Variables](#variables)
		- [Modifier](#modifier)
	- [Squash Section](#squash-section)
		- [Squashing specific files](#squashing-specific-files)
		- [Squash files from a directory](#squash-files-from-a-directory)
	- [Executors](#executors)
		- [The `kubectl` executor](#kubectl-executor)
		- [The `fly` executor](#fly-executor)
		- [The Generic Executor](#generic-executor)
	- [CLI Options](#cli-options)
		- [`--curly-braces`](#--curly-braces)
		- [`--silent`](#--silent)
		- [`--verbose`](#--verbose)
		- [`--dry-run`](#--dry-run)
		- [`--var`](#--var)
- [Development](#development)

Aviator provides a verbose style of configuration. It is the result of configuring a spruce merge plan and optionally an execution plan (e.g `fly`).

Example for a simple aviator file:

```yaml
spruce:
- base: path/to/base.yml
  merge:
  - with:
      files:
      - top.yml
  to: result.yml
```

### Spruce Section

The `spruce` section is an array of merge steps. It provides different parameters to provide high flexibility when merging YAML files. You can:

- specify specific files to include into your merge
- specify a specific directory to include into your merge
- specify a specific directory including all subdirectories to include into your merge

However, this is not enough. Additionally you can use *regular expressions*, *environment-variables*, and more. Read about all parameters and what they do in this section.

#### Base (`string`)

The `base` property specifies the path to the base YAML file. All other YAML files will be merged on top of this YAML file.

---

#### Prune (`Array`)

`prune` defines YAML properties which will be pruned during the merge. For more information check the `spruce` [merge semantics](https://github.com/geofffranks/spruce/blob/master/doc/merging.md#order-of-operations).

Example:

```yaml
spruce:
- base: base.yml
  prune:
  - meta
  - properties
  merge:
  - with:
      files:
      - top.yml
  to: result.yml
```

In this case `meta` and `properties` will be pruned during merge.

#### cherry_pick (`array`)

Enables [Spruce](https://github.com/geofffranks/spruce/blob/master/doc/merging.md#order-of-operations) `cherry pick` option: With the `cherry_pick` property you can specify specific YAML subtrees you want to have in your restulting YAML file (opposite of `prune`)  

Example:

```yaml
spruce:
- base: path/to/base.yml
  cherry_pick:
  - properties
  merge:
  - with_in: path/to/dir/
  - with:
      files:
      - top.yml
  regexp: ".*.(yml)"
  skip_eval: true
  to: result.yml
```
---

#### go_patch (`bool`)

To use spruce in conjuction with the `go-patch` format it can be enabled within the aviator `spruce` section as a toplevel bool property:

```
spruce:
- base: some.yml
   go_patch: true
   merge:
   - with:
        files:
        - some/ops/file.yml
   to: result.yml
```

Read more about it [here](https://github.com/geofffranks/spruce/blob/master/doc/merging-go-patch-files.md)

---

#### Merge (`Array`)

You can configure three different merge types inside the `merge` section: `with`, `with_in`, `with_all_in`:

**with**

`with` specifies specific files you want to include into the merge.

- `files` (required): List of paths to YAML files

- `in_dir` (optional): If all of the files you want to include into the merge are in one specific directory, you can specify the directoyr path and list only file names in the `files` list. _Note: Whenever a directory is defined, the path requires a trailing "/"!!!_

- `skip_non_existing` (optional): Setting this property to `true` will skip non existing files that are specified in the `files` list rather then returning an error. This is useful, if a file is not necessarely there.

Example:

```yaml
spruce:
- base: path/to/base.yml
  merge:
  - with:
      files:
      - top.yml
      - top2.yml
      - top3.yml
    in_dir: path/to/
    skip_non_existing: true
  to: result.yml
```

**with_in** (`string`)

`with_in` specifies a path (do not forget the trailing "/") to a directory. All files  within this directory (but not subdirectories) will be included in the merge.

Example:

```yaml
spruce:
- base: path/to/base.yml
  merge:
  - with_in: path/to/dir/
  to: result.yml
```

`except` (`array`)

With `except` you can specify a list of files you want to exclude from the path specified in `with_in`

Example:

```yaml
spruce:
- base: path/to/base.yml
  merge:
  - with_in: path/to/dir/
    except:
    - file1
    - file2
  to: result.yml
```

This will exclude `path/to/dir/file1` and `path/to/dir/file2` from the merge.


**with_all_in**

`with_all_in` specifies a path (do not forget the trailing "/") to a directory. All files within this directory -including all subdirectories - will be included in the merge.

Example:

```yaml
spruce:
- base: path/to/base.yml
  merge:
  - with_all_in: path/to/dir/
    except:
    - someFiles.yml
    - youWant.yml
    - toExclude.yml
  to: result.yml
```

*NOTE: `except` also works for `with_all_in`*

**regexp** (`string`(quoted))

Only files matching the regular expression will be included in the merge. It can be specified for all three merge types `with`, `with_in`, and `with_all_in`. This could be required if the target directory contains other then only YAML files.

Example:

```yaml
spruce:
- base: path/to/base.yml
  merge:
  - with_in: path/to/dir/
    regexp: ".*.(yml)"
  - with:
      files:
      - top.yml
    regexp: ".*.(yml)"
  - with_all_in: path/to/another/dir/
    regexp: ".*.(yml)"
  to: result.yml
```

---

#### skip_eval (`bool`)

Enabling this skip-eval will merge without resolve spruce expressions. For more information check [Spruce doc](https://github.com/geofffranks/spruce/blob/master/doc/merging.md#order-of-operations)

Example:

```yaml
spruce:
- base: path/to/base.yml
  merge:
  - with_in: path/to/dir/
  - with:
      files:
      - top.yml
  regexp: ".*.(yml)"
  skip_eval: true
  to: result.yml
```

---
#### To (`string`)


`to` specifies the target file, where the merged files should be saved to. It can be used only in combination with the basic merge types `files`, `with_in`, and `with_all_in`.

---

#### ForEach

On top of the basic `merge` you can do more complex merges with `for_each`. More precisely, you can execute the basic `merge` for multiple files specified in `for_each`. When specifying files with `for_each` you need to use `to_dir` instead of `to` to specify a target directory instead of a target file.    

**files**

`files` specifies a list of files that will be included in your merge seperately.

Example:

```yaml
spruce:
- base: path/to/base.yml
  merge:
  - with:
      files:
      - top.yml
    regexp: ".*.(yml)"
  for_each:
    files:
    - env.yml
    - env2.yml
  to_dir: results/
```

This merge step will execute two merges and generate two files. It will merge `base.yml` and `top.yml` with `env.yml`, write it to `results/` and do the same with `env2.yml`.

**in**

`in` is basically the same as `files` with the difference that it will merge all files for a given path sperately

Example:

```yaml
spruce:
- base: path/to/base.yml
  merge:
  - with:
      files:
      - top.yml
    regexp: ".*.(yml)"
  for_each:
    in: path/to/dir/
  to_dir: results/
```

**Except**

`except` works in combination with `in`: list of files that you want to exclude from the merge.

Example:

```yaml
spruce:
- base: path/to/base.yml
  merge:
  - with:
      files:
      - top.yml
  regexp: ".*.(yml)"
  for_each_in: path/to/dir/
  except:
  - some.yml
  to_dir: results/
```

**include_sub_dirs**

`include_sub_dirs` includes all files including files in all subdirectories of a directory into the merge seperately.

```yaml
spruce:
- base: path/to/base.yml
  merge:
  - with:
      files:
      - top.yml
  regexp: ".*.(yml)"
  for_each:
    in: path/to/dir
    include_sub_dirs: true
    enable_matching: true
    copy_parents: true
  to_dir: results/
```

When `include_sub_dirs` is defined you can specify further properties:

- `enable_matching`: this will only include files in the merge, that contains the same substring as the parent directory.

- `copy_parents`: setting this property to `true` (default `false`) will copy the parent folder of a file to the target directory (in the above example `results/`)

**regexp**

The `regexp` property can also be set in combination with `for_each`, `for_each_in`, and `walk_through` to only include files matching the regular expression.

```yaml
spruce:
- base: path/to/base.yml
  merge:
  - with:
      files:
      - top.yml
  regexp: ".*.(yml)"
  for_each:
    in: path/to/dir
    include_sub_dirs: true
    enable_matching: true
    copy_parents: true
    regexp: ".*.(yml)"
  to_dir: results/
```
---

#### Read From and Write To Internal Datatsore

Sometimes it is required to do more than one merge step, which creates intermediate YAML files. In this case you can save merge results to internal datastore/cache which you can write/read by surrounding your location with double courly braces `{{file|dir}}`. Internal cache also work as directories and can be used with `to_dir`.

Example:

```yaml
spruce:
- base: path/to/base.yml
  merge:
  - with_in: path/to/dir/
  to: {{result}}

- base: {{result}}
  merge:
  - with_in: another/path/
  to: final.yml
```

#### Environment Variables

Aviator supports to read _Environment Variables_. Environment variables can be set with `$VAR` or `${VAR}` at an arbitrary place in the `aviator.yml`.

Example:

```yaml
spruce:
- base: $BASE_PATH/app-${NUMBER}.yml
  merge:
  - with_in: path/to/dir/
  to: {{result}}

- base: {{result}}
  merge:
  - with_in: $TARGET_PATH
  to: $RESULT_YAML
```

Executing `aviator` as follows:

```
$ BASE_PATH=/tmp/ NUMBER=1 RESULT_YAML=result.yml aviator
```

will resolve:

```yaml
spruce:
- base: /tmp/app-1.yml
  merge:
  - with_in: path/to/dir/
  to: {{result}}

- base: {{result}}
  merge:
  - with_in: $TARGET_PATH
  to: result.yml
```

#### Variables

You can provide variables to aviator files using the `--var` flag. Basic CLI usage:

`$ aviator --var key=value`

In you aviator file you need to specify the name of the variable you want to interpolate. The syntax for a variable is the following `(( varName ))` (note the space before and after the variable name!)

Example:

```yaml
---
spruce:
- base: (( key ))
  ...
```

In this example the variable `key` will be replaced by the value `value`. So the result would look like this:

```yaml
---
spruce:
- base: value
  ...
```

Values for aviator variables can be multi-line

#### Modifier

With modifier you can modify the resulting (merged) YAML file. You can either delete, set, or update a property. The modifier will always be applied on the result. If you use `for_each` it will be applied on each `for_each` merge step.

Consider a resulting YAML from a merge process `result.yml`, which has a property `person.name`:

```yaml
---
person:
  name: Julz
```

1. the property can be deleted

  ```yaml
  spruce:
  - base: base.yml
    merge:
    - with:
        files:
        - top.yml
    modify:
      delete:
      - "person.name"
    to: result.yml
  ```

  It deletes a property only if it exists. There will be no error if a proerty does NOT exist.

2. the property can be updated:

  ```yaml
  spruce:
  - base: base.yml
    merge:
    - with:
        files:
        - top.yml
    modify:
      update:
      - path: person.name
        value: newName
    to: result.yml
  ```

  Using update will update existing properties only.

3. Other properties can be added/updated with set:

  ```yaml
  spruce:
  - base: base.yml
    merge:
    - with:
        files:
        - top.yml
    modify:
      set:
      - path: person.name
        value: NewName
    to: result.yml
  ```

  Set updates or adds a property to an array. If a property exists it will be overwritten, if the property does not exist it will be added (works only for maps not arrays).

Aviator uses [goml](https://github.com/JulzDiverse/goml) as YAML modifier. If you want to read more about `update`, `delete`, and `set`, check the README.

---

### Squash Section

You can squash multiple files into one single YAML file using the `squash` section.


#### Squashing specific files

```yaml
squash:
  contents:
  - files:
    - deployment.yml
    - service.yml
  to: app.yml
```

#### Squash files from a directory

```yaml
squash:
  contents:
  - dir: my/dir/
  to: app.yml
```

### Executors

Executors execute executables installed on the OS that Aviator is running on. The following three executors are currently supported by Aviator:

- `kubectl` Executor
- `fly` Executor
- Generic Executor: Runs an any specified executable. 

#### `kubectl` executor 

Execute `kubectl apply` with the following options: 

*Supported flags*

- **file**: Name of the file to `kubectl apply`
- **force**: calls `kubectl apply` with the `--force` flag
- **dry_run**: calls `kubectl apply` with the `--dry-run` flag
- **overwrite**: calls `kubectl apply` with the `--overwrite` flag
- **recursive**: calls the `kubectl apply` with the `--recursive` flag
- **output**: calls the `kubectl apply` with the `--output=<desired-ouput>` parameter
- **kustomize**: calls the `kubectl apply` with the `--kustomazation/-k` flag rather than with `--filename/-f`.
  - Read more about `kubectl apply` + kustomization [here](https://github.com/kubernetes-sigs/kustomize) and [here](https://kubectl.docs.kubernetes.io/pages/app_management/apply.html)

You can read about the details of the flags of `kubectl apply` [here](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply)

Example:

```yaml
kubectl
  apply:
    file: deployment.yml
    force: true
    dry_run: true
    overwrite: true
    recursive: true
    output: yaml
```

**Kustomize Example:**

When `kustomize` flag in aviator is set to true it will call `kubectl apply` with the `--kustomization/-k` flag, which expects a directory path. The directory paht needs to be specified in the `file` property of `kubectl.apply`:

```yaml
kubectl:
  apply:
    file: kustomization/path/
    kustomize: true
```

#### Fly Executor

An executor for the Concourse Fly CLI. The supported commands are `set-pipeline`, `validate-pipeline`, `format-pipeline`, and `expose-pipeline/hide-pipeline`.
The `set-pipeline` and `hide-pipeline` commands are executed by default. Here is the list of commands and flags that can be used with `aviator`:

- **name**: Name of the pipeline
- **target**: Target short name (`fly` target)
- **config (string):** the pipeline config file (yml)
- **load_vars_from (array):** List of all property files (-l)
- **vars (map):** Map of variables (--var)
- **non_interactive (bool):** Enables non-interactive mode (-n)
- **expose (bool):** Exposes the pipeline (expose-pipeline)
- **validate_pipeline (bool):** Validate local pipeline configuration (failes on errors)
- **strict (bool):** causes `validate_pipeline` to fail on errors AND warnings
- **format_pipeline (bool):** format a pipeline config in a "canonical" form (prints to stdout).
- **write:** update the formatted pipeline config file in-place.
- **team_name (string):** the team name to fly for set-pipeline

More detailed description of the flags can be found [here](https://concourse-ci.org/pipelines.html)

Example - set-pipeline:

```yaml
fly:
  name: myPipelineName
  target: myFlyTarget
  team_name: concourse-team
  config: pipeline.yml
  non_interactive: true
  check_creds: true
  load_vars_from:
  - credentials.yml
  vars:
    var1: myvar
    var2: myvar2
  expose: true
```

Example - validate-pipeline:

```yaml
fly:
  config: myconfig.yml
  validate_pipeline: true
  strict: true
```

Example - format-pipeline:

```yaml
fly:
  config: myconfig.yml
  format_pipeline: true
  write: true
```


_NOTE: You will need to fly login first, before executing `aviator`_

#### Generic Executor

The Generic Executor executes any specified executable. Here is how to define an Generic Executor in the `aviator.yml`:

```yaml
exec:
- executable: some-executable # required
  global_options: # optional
  - name: --option
    value: option-value
  command: # optional
    name: command-name
    options:
    - name: --command-option
      value: command-option-value
  args:
  - arg1
  - arg2
```

This will executed as `$ some-executable --option option-value command-name --command-option command-option-value arg1 arg2`

Example: `cp`

```yaml
exec:
- executable: cp
  global_options:
  - name: -r
  args:
  - dir/
  - destination/
```

This calls `cp` as follows: `$ cp -r dir/ destination/`

---

### CLI Options

#### `--curly-braces`

Some YAML based tools (like concourse in the past) are using `{{}}` sytnax. This is not YAML conform. Using the `--curly-braces` option you can allow this syntax.

#### `--silent`

This option will output no infromation to stdout.

#### `--verbose`

This option prints which files are excluded from a merge.

#### `--dry-run`

This option prints contents to `stdout` rather than writing it to files. This flag also omits any defined executor.

#### `--var`

You can provide variables to the aviator file.

---

# Development

```
$ go get github.com/JulzDiverse/aviator
```

## Build Aviator

To build the aviator binary you can navigate to `cmd/aviator`:

```
$ cd ./cmd/aviator`
```

Now you can build with the go cli:

```
$ go build 
```

For more information on the `$ go build` command run `$ go help build`. 
