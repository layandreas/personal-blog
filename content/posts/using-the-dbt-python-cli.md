+++
author = "Andreas Lay"
title = "dbt: Programmatic Invocation via dbtRunner"
date = "2024-08-06"
description = "A Python dbtRunner template to get ready for self-hosting dbt"
tags = ["dbt", "Python"]
categories = ["Data Engineering"]
ShowToc = true
TocOpen = true
+++

## Introduction

[dbt](https://docs.getdbt.com/) is a great tool for building & organising your ELT data pipelines. When deploying dbt yourself you can invoke dbt either through [dbt core cli](https://docs.getdbt.com/reference/node-selection/syntax) or through Python via [dbtRunner](https://docs.getdbt.com/reference/programmatic-invocations). I will give you an example template on how to use the latter. You can find the full example in the [Full Example](#full-example) section.

**Note: This example was build on `dbt-core==1.8.3`. `dbtRunner` may be subject to breaking changes so there's no guarantee the provided code works as is with other dbt versions.**

## Imports

Let's start with the imports:

```python
"""This module demonstrate how to invoke the dbt command line
through the Python interface"""

import logging

from dbt.artifacts.schemas.freshness import FreshnessResult
from dbt.artifacts.schemas.results import FreshnessStatus, RunStatus, TestStatus
from dbt.cli.main import RunExecutionResult, dbtRunner, dbtRunnerResult
from devtools import pformat

logger = logging.getLogger(__name__)
```

`dbtRunner` is the main entrypoint to invoke `dbt` through Python. `RunExecutionResult` and `dbtRunnerResult` are types which are returned by the `dbtRunner`.

We also import  `FreshnessStatus`, `RunStatus`, `TestStatus` to check which dbt nodes failed (e.g. due to SQL syntax errors) or raised errors or warnings (for example failed dbt tests). This will allow us to use the Python `logging` module to correctly log warnings and errors.

## Preparation: Checking dbt Results

Let's start by writing a function which checks the result of our `dbtRunner` invocation. Note that this function _does not raise errors_ but uses the Python `logging` module to log warnings and errors with the correct log level. If you're deploying `dbt` for example on any cloud provider you could then automatise `Slack` or `MS Teams` notifications based on observed dbt logs log levels. Usually you want to be notified about any errors or warnings raised by `dbt`. While you will see all dbt logs printed to `stdout` when invoking the `dbtRunner`, the runner doesn't use the Python `logging` module and you won't be able to distinguish `info`, `warning` and `error`log levels.

```python
def _check_dbt_result_and_log_error_or_warning(
    result: RunExecutionResult | FreshnessResult,
) -> None:
    """Check the results for errors, test failures and warnings. In case of
    errors, test failures and / or warnings we log those with
    logger.info and logger.error respectively.
    This way we guarantee the correct log levels such that these will show up
    correctly in for example cloud logs. The correct log level is important
    so that we can catch errors / warnings and notify us e.g. via Teams."""
    for node_exec_result in result:
        node_result_status = node_exec_result.status
        node_result_message = node_exec_result.message or ""

        # The node key should be in the node_exec_result however the type
        # signature of `BaseResult` doesn't really show this. To be sure
        # we catch any errors occuring here
        node_name = "<could_not_determine>"
        try:
            node_name = str(node_exec_result.node.name)
        except Exception:
            pass

        # Log any errors and / or test failures
        if node_result_status in (
            RunStatus.Error,
            TestStatus.Error,
            TestStatus.Fail,
            FreshnessStatus.Error,
            FreshnessStatus.RuntimeErr,
        ):
            logger.error(
                f"Error / failure in dbt node {node_name}." f"{node_result_message}"
            )
        # Log warnings
        elif node_result_status in (
            TestStatus.Warn,
            FreshnessStatus.Warn,
        ):
            logger.warning(f"Warning in dbt node {node_name}. {node_result_message}")
```

## Step 1: Freshness Checks

A reasonable first step in a `dbt` pipeline is checking the freshness of your sources. The following function does this for us:

```python
def _dbt_source_freshness(dbt: dbtRunner, profile: str) -> dbtRunnerResult:
    cli_args_freshness = [
        "source",
        "freshness",
        "-t",
        profile,
    ]
    dbt_runner_result_freshness = dbt.invoke(cli_args_freshness)
    run_execution_result_freshness = dbt_runner_result_freshness.result  # type: ignore

    # The returned type should be `FreshnessResult` when invoking dbt source
    # freshness
    # If it's a different type we raise an error which will interrupt further
    # execution
    # Can also return `NoneType` in case of dbt model parsing errors
    if run_execution_result_freshness is None:
        dbt_runner_exception_freshness = dbt_runner_result_freshness.exception
        logger.error(
            f"Exception in dbt source freshness: {dbt_runner_exception_freshness}"
        )
        raise RuntimeError("Exception in dbt source freshness execution")
    elif not isinstance(run_execution_result_freshness, FreshnessResult):
        raise RuntimeError(
            "Expected `FreshnessResult`, got " f"{type(run_execution_result_freshness)}"
        )

    _check_dbt_result_and_log_error_or_warning(result=run_execution_result_freshness)

    return dbt_runner_result_freshness
```

The `dbtRunner` will return a `FreshnessResult` which we then pass on to our previously defined `_check_dbt_result_and_log_error_or_warning` function.

## Step 2: Source Tests

Now let's run all tests on our sources. Let's first define a function to run `dbt` tests:

```python
def _dbt_test(
    dbt: dbtRunner, profile: str, test_args: tuple[str, ...]
) -> dbtRunnerResult:
    logger.info(f"Running dbt test on:\n{pformat(test_args)}")

    cli_args = ["test", "-t", profile, *test_args]
    dbt_runner_result = dbt.invoke(cli_args)
    run_execution_result = dbt_runner_result.result

    # The returned type should `RunExecutionResult` when invoking dbt build
    # If it's a different type we raise an error which will interrupt further
    # execution
    # Can also return `NoneType` in case of dbt model parsing errors
    if run_execution_result is None:
        dbt_runner_exception = dbt_runner_result.exception
        logger.error(f"Exception in dbt build: {dbt_runner_exception}")
        raise RuntimeError("Exception in dbt build execution")
    elif not isinstance(run_execution_result, RunExecutionResult):
        raise RuntimeError(
            f"Expected `RunExecutionResult`, got {type(run_execution_result)}"
        )

    _check_dbt_result_and_log_error_or_warning(result=run_execution_result)

    return dbt_runner_result
```

Note that we use our `_check_dbt_result_and_log_error_or_warning` function to check our test results. We can now invoke the tests on our sources by running the following:


```python
# Run tests on sources now
logger.info("Running dbt tests on sources")
_dbt_test(
    dbt=dbt,
    profile=profile,
    test_args=(
        "--select",
        "source:*",
    ),
)
logger.info("Finished running dbt tests on sources")
```

## Step 3: dbt Build

Now we can build our models & run tests on our models by invoking dbt build. Let's first define a function which runs the build for us and checks its results:


```python
def _dbt_build(
    dbt: dbtRunner, profile: str, build_args: tuple[str, ...]
) -> dbtRunnerResult:
    cli_args = ["build", "-t", profile, *build_args]
    dbt_runner_result = dbt.invoke(cli_args)
    run_execution_result = dbt_runner_result.result

    # The returned type should `RunExecutionResult` when invoking dbt build
    # If it's a different type we raise an error which will interrupt further
    # execution
    # Can also return `NoneType` in case of dbt model parsing errors
    if run_execution_result is None:
        dbt_runner_exception = dbt_runner_result.exception
        logger.error(f"Exception in dbt build: {dbt_runner_exception}")
        raise RuntimeError("Exception in dbt build execution")
    elif not isinstance(run_execution_result, RunExecutionResult):
        raise RuntimeError(
            f"Expected `RunExecutionResult`, got {type(run_execution_result)}"
        )

    _check_dbt_result_and_log_error_or_warning(result=run_execution_result)

    return dbt_runner_result
```

In our main module we invoke it by adding this code snippet:

```python
# Start dbt build
build_args = ("-m", "<SOME_SUBSET_OF_MODELS>") # this is optional
logger.info(f"{build_args=}")
logger.info("Starting dbt build")
dbt_runner_result = _dbt_build(dbt=dbt, profile=profile, build_args=build_args)
logger.info("Finished dbt build")
```

You can optionally pass some `build_args` if you only want to build a subset of your models. You can pass an empty tuple here to simply build all of your models.

## Step 4: Generate & Deploy Documentation

Last but not least we will generate [dbt's HTML docs](https://docs.getdbt.com/reference/commands/cmd-docs):

```python
def _dbt_docs_generate_and_upload(dbt: dbtRunner, profile: str) -> None:
    logger.info("Generating dbt docs")
    cli_args = ["docs", "generate", "-t", profile]
    dbt_runner_result = dbt.invoke(cli_args)

    if not dbt_runner_result.success:
        raise DbtFailedException("dbt docs generate failed")

    logger.info("Uploading dbt docs")
    # Add here a function which uploads all files in the "./target" directory
    # to your deployment target (nginx, s3, Google Cloud Storage, ...)
```

We generally want to upload the generated static files to a static file server. Your upload target will depend on your deployment target / hosting provider. Again we invoke this function in our main module:

```python
try:
    logger.info("Generating and uploading dbt docs")
    _dbt_docs_generate_and_upload(dbt=dbt, profile=profile)
except Exception as e:
    logger.error(f"Error while generating/uploading dbt docs: {e}")
```

## Full Example

Here's the full example combining our code snippets from above:

```python
"""This module demonstrate how to invoke the dbt command line
through the Python interface"""

import logging

from dbt.artifacts.schemas.freshness import FreshnessResult
from dbt.artifacts.schemas.results import FreshnessStatus, RunStatus, TestStatus
from dbt.cli.main import RunExecutionResult, dbtRunner, dbtRunnerResult
from devtools import pformat

logger = logging.getLogger(__name__)


def main() -> None:
    dbt = dbtRunner()

    # Check source freshness first
    logger.info("Running dbt source freshness check")
    profile = "<YOUR_DBT_PROFILE>"  # Set this to your profile
    dbt_runner_result_freshness = _dbt_source_freshness(dbt=dbt, profile=profile)
    logger.info("Finished dbt source freshness check")

    # Run tests on sources now
    logger.info("Running dbt tests on sources")
    _dbt_test(
        dbt=dbt,
        profile=profile,
        test_args=(
            "--select",
            "source:*",
        ),
    )
    logger.info("Finished running dbt tests on sources")

    if not dbt_runner_result_freshness.success:
        raise DbtFailedException("dbt source freshness check failed")

    # Start dbt build
    build_args = ("-m", "<SOME_SUBSET_OF_MODELS>") # this is optional
    logger.info(f"{build_args=}")
    logger.info("Starting dbt build")
    dbt_runner_result = _dbt_build(dbt=dbt, profile=profile, build_args=build_args)
    logger.info("Finished dbt build")

    # Raise error on dbt build failure
    if not dbt_runner_result.success:
        raise DbtFailedException("dbt build failed")

    try:
        logger.info("Generating and uploading dbt docs")
        _dbt_docs_generate_and_upload(dbt=dbt, profile=profile)
    except Exception as e:
        logger.error(f"Error while generating/uploading dbt docs: {e}")

    logger.info("Finished dbt job")


class DbtFailedException(Exception):
    pass


def _dbt_build(
    dbt: dbtRunner, profile: str, build_args: tuple[str, ...]
) -> dbtRunnerResult:
    cli_args = ["build", "-t", profile, *build_args]
    dbt_runner_result = dbt.invoke(cli_args)
    run_execution_result = dbt_runner_result.result

    # The returned type should `RunExecutionResult` when invoking dbt build
    # If it's a differe type we raise an error which will interrupt further
    # execution
    # Can also `NoneType` in case of dbt model parsing errors
    if run_execution_result is None:
        dbt_runner_exception = dbt_runner_result.exception
        logger.error(f"Exception in dbt build: {dbt_runner_exception}")
        raise RuntimeError("Exception in dbt build execution")
    elif not isinstance(run_execution_result, RunExecutionResult):
        raise RuntimeError(
            f"Expected `RunExecutionResult`, got {type(run_execution_result)}"
        )

    _check_dbt_result_and_log_error_or_warning(result=run_execution_result)

    return dbt_runner_result


def _dbt_source_freshness(dbt: dbtRunner, profile: str) -> dbtRunnerResult:
    cli_args_freshness = [
        "source",
        "freshness",
        "-t",
        profile,
    ]
    dbt_runner_result_freshness = dbt.invoke(cli_args_freshness)
    run_execution_result_freshness = dbt_runner_result_freshness.result  # type: ignore

    # The returned type should be `FreshnessResult` when invoking dbt source
    # freshness
    # If it's a different type we raise an error which will interrupt further
    # execution
    # Can also return `NoneType` in case of dbt model parsing errors
    if run_execution_result_freshness is None:
        dbt_runner_exception_freshness = dbt_runner_result_freshness.exception
        logger.error(
            f"Exception in dbt source freshness: " f"{dbt_runner_exception_freshness}"
        )
        raise RuntimeError("Exception in dbt source freshness execution")
    elif not isinstance(run_execution_result_freshness, FreshnessResult):
        raise RuntimeError(
            "Expected `FreshnessResult`, got " f"{type(run_execution_result_freshness)}"
        )

    _check_dbt_result_and_log_error_or_warning(result=run_execution_result_freshness)

    return dbt_runner_result_freshness


def _dbt_test(
    dbt: dbtRunner, profile: str, test_args: tuple[str, ...]
) -> dbtRunnerResult:
    logger.info(f"Running dbt test on:\n{pformat(test_args)}")

    cli_args = ["test", "-t", profile, *test_args]
    dbt_runner_result = dbt.invoke(cli_args)
    run_execution_result = dbt_runner_result.result

    # The returned type should `RunExecutionResult` when invoking dbt build
    # If it's a differe type we raise an error which will interrupt further
    # execution
    # Can also `NoneType` in case of dbt model parsing errors
    if run_execution_result is None:
        dbt_runner_exception = dbt_runner_result.exception
        logger.error(f"Exception in dbt build: {dbt_runner_exception}")
        raise RuntimeError("Exception in dbt build execution")
    elif not isinstance(run_execution_result, RunExecutionResult):
        raise RuntimeError(
            f"Expected `RunExecutionResult`, got {type(run_execution_result)}"
        )

    _check_dbt_result_and_log_error_or_warning(result=run_execution_result)

    return dbt_runner_result


def _dbt_docs_generate_and_upload(dbt: dbtRunner, profile: str) -> None:
    logger.info("Generating dbt docs")
    cli_args = ["docs", "generate", "-t", profile]
    dbt_runner_result = dbt.invoke(cli_args)

    if not dbt_runner_result.success:
        raise DbtFailedException("dbt docs generate failed")

    logger.info("Uploading dbt docs")
    # Add here a function which uploads all files in the "./target" directory
    # to your deployment target (nginx, s3, Google Cloud Storage, ...)


def _check_dbt_result_and_log_error_or_warning(
    result: RunExecutionResult | FreshnessResult,
) -> None:
    """Check the results for errors, test failures and warnings. In case of
    errors, test failures and / or warnings we log those with
    logger.info and logger.error respectively.
    This way we guarantee the correct log levels such that these will show up
    correctly in for example cloud logs. The correct log level is important
    so that we can catch errors / warnings and notify us e.g. via Teams."""
    for node_exec_result in result:
        node_result_status = node_exec_result.status
        node_result_message = node_exec_result.message or ""

        # The node key should be in the node_exec_result however the type
        # signature of `BaseResult` doesn't really show this. To be sure
        # we catch any errors occuring here
        node_name = "<could_not_determine>"
        try:
            node_name = str(node_exec_result.node.name)
        except Exception:
            pass

        # Log any errors and / or test failures
        if node_result_status in (
            RunStatus.Error,
            TestStatus.Error,
            TestStatus.Fail,
            FreshnessStatus.Error,
            FreshnessStatus.RuntimeErr,
        ):
            logger.error(
                f"Error / failure in dbt node {node_name}." f"{node_result_message}"
            )
        # Log warnings
        elif node_result_status in (
            TestStatus.Warn,
            FreshnessStatus.Warn,
        ):
            logger.warning(f"Warning in dbt node {node_name}. {node_result_message}")


if __name__ == "__main__":
    main()

```