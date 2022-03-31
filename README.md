# tartufo-scan-test

This repo provides a simple example of an issue with tartufo which will be used to support an issue and subseqent PR against tartufo.

# Background

As of Tartufo 3.0.0 to the current 3.1.2 an option exists to exclude by entropy patterns when scanning with tartufo.

Tartufo also offers 2 main repo scanning modes:

- `scan-local-repo` which scans a repository already cloned to your local system
- `scan-remote-repo` which automatically clones and scans a remote git repository

# Problem Statement

When using the scan-remote-repo option, tartufo will clone the remote repo and then scan it. In this mode tartufo will use the remote repo's tartufo.toml if it exists, which can define various exclusions as well as additional rule patterns for the given repo.

The various exclusion types are:

- exclude-entropy-patterns
- exclude-path-patterns
- exclude-signatures

Testing appears to show that exclude-entropy-patterns in a remote repo's tartufo.toml are not actually respected when running a scan using the scan-remote-repo option.

# Example

Using this repo you can test to see the problem.

The following versions were used in testing:

- Tartufo 3.1.2
- Python 3.9.11

## Steps to Reproduce

1. Clone this repo locally: `git clone https://github.com/mgaspar-godaddy/tartufo-scan-test.git`
2. `cd tartufo-scan-test`
3. Run `tartufo scan-local-repo .`

You should receive an **All clear. No secrets detected.** message.

4. Change up one directory: `cd ..`
5. Ensure there is no local tartufo.toml in the current directory that might interfere with the test
6. Run `tartufo scan-remote-repo https://github.com/mgaspar-godaddy/tartufo-scan-test.git`

You should receive output like the following:

```
~~~~~~~~~~~~~~~~~~~~~
Reason: High Entropy
Filepath: test-file.yaml
Signature: ac8ac0301f886766fe6273419ae4bb84287a031fdce68e17521a1e5d4281cb80
Commit time: 2022-03-31 09:21:35
Commit message: Add test file containing a high entropy string

Commit hash: 5212f424ab1882974f8f7cfd802bd0589eb8061e
Branch: main
diff --git a/test-file.yaml b/test-file.yaml
new file mode 100644
index 0000000..e3b8bc3
--- /dev/null
+++ b/test-file.yaml
@@ -0,0 +1 @@
+sha512: 1bd8532bb72a1af92c9cdcfbc5d7a0dc7c16ad473c7550d26b6e6e887155bd496ba166be21c630dcdb3c54af822aa2d1570d8f203eb9b4a60764029f29eab7c8

~~~~~~~~~~~~~~~~~~~~~
```

When looking at the [remote tartufo.toml](https://github.com/mgaspar-godaddy/tartufo-scan-test/blob/main/tartufo.toml#L7) we see that this entropy pattern should have been excluded.

# Test Conclusion

It's clear when cloning the repo locally and running a scan-local-repo the entropy pattern is excluded as expected. However, when running a scan-remote-repo against the same repo with the same tartufo.toml, the entropy pattern is not excluded.

# Solution

This appears to be due to a section of the [tartufo/scanner.py](https://github.com/godaddy/tartufo/blob/06c249930ab4c6ee813df723ea84dee08372a61b/tartufo/scanner.py#L283) which does not append excluded entropy patterns.

I believe the solution is to modify that line of code as follows:

```diff
diff --git a/tartufo/scanner.py b/tartufo/scanner.py
index fee3e6e..7b0d8b9 100755
--- a/tartufo/scanner.py
+++ b/tartufo/scanner.py
@@ -280,7 +280,9 @@ class ScannerBase(abc.ABC):  # pylint: disable=too-many-instance-attributes
         """Get a list of regexes used as an exclusive list of paths to scan."""
         if self._excluded_entropy is None:
             self.logger.info("Initializing excluded entropy patterns")
-            patterns = list(self.global_options.exclude_entropy_patterns or ())
+            patterns = list(self.global_options.exclude_entropy_patterns or ()) + list(
+                self.config_data.get("exclude_entropy_patterns", ())
+            )
             self._excluded_entropy = config.compile_rules(patterns) if patterns else []
             self.logger.debug(
                 "Excluded entropy was initialized as: %s", self._excluded_entropy
```

# Links

Issue on Tartufo: https://github.com/godaddy/tartufo/issues/343
PR on Tartufo: https://github.com/godaddy/tartufo/pull/344

