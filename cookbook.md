# Contributing to nixpkgs as an R programmer: a cookbook

## Introduction

So you've discovered Nix, and started using it for your day-to-day work, and now
you would like to help package CRAN and Bioconductor packages for `nixpkgs`, but
don't know where to start?

This guide will help you help us! It lists N common recipes to start fixing R
packages for inclusion to `nixpkgs`. Every package available on CRAN and
Bioconductor gets built on Hydra and made available through the `rPackages`
packageset. However, some packages don't build successfully, and require manual
fixing. Most of them are very quick, one-line fixes, but others require a bit
more work. The goal of this cookbook is to make you quickly familiar with the
main reasons a package may be broken and explain to you how to fix it, and why
certain packages get fixed in certain ways.

## *Mise en place*

We first need to get our tools and ingredients in order before cooking.
Fork the [`nixpkgs` reposiory](https://github.com/NixOS/nixpkgs) and clone it
to your computer. Then, add the original repository as a remote:

```
git checkout master

# Add upstream as a remote
git remote add upstream git@github.com:NixOS/nixpkgs.git
```

This way, each time you want to fix a package, you can fetch the latest updates
from upstream and merge them to your local copy:

```
# Fetch latest updates
git fetch upstream

# Merge latest upstream/master to your master
git merge upstream/master
```

Make sure to merge the latest commits from upstream regularly, because `nixpkgs`
gets updated very frequently each day. We can now look for a package to fix.

## The starter: where to find packages to fix

The first step to help fix a package is to find a package to fix: you should
visit the latest `rPackages` evaluation over [here](https://hydra.nixos.org/jobset/nixpkgs/r-updates).
Click on the "Still failing jobs" tab to see which packages' builds didn't succeed and
click on a package. You should see something like this:

![AIUQ build steps](images/AIUQ_failing.png)

From there, you can see that `{AIUQ}`'s build failed because of another
package, `{SuperGauss}`, so fixing `{SuperGauss}` will likely fix this one
as well.

From here, you can look for `{SuperGauss}` in the "Still failing jobs" tab, and
see why `{SuperGauss}` failed, or you could check out a little dashboard I built
that you can find
[here](https://raw.githack.com/b-rodrigues/nixpkgs-r-updates-fails/targets-runs/output/r-updates-fails.html).
This dashboard shows essentially the same information you find on the "Still
failing jobs" tab from before, but with several added niceties. First of all,
there's a column called `fails_because_of` that shows the name of the package
that caused another to fail. So in our example with `{AIUQ}`, `{SuperGauss}`
would be listed there (if a package fails for another reason, like a missing
system-level dependency, then its own name is listed there). You can type
`{SuperGauss}` on the little box there to filter and see all the packages that
fail because of of it. Then, you can also see the package's *package rank*. This
rank is computed using the `{packageRank}` package, and the table is sorted by
lowest rank (low ranks indicate high popularity and there's also the
`percentile` column that indicates the percentage of packages with higher
downloads). Finally, there's a direct link to a PR fixing the package (if it has
been opened) and also the PR's status: has it been merged already or not?

Having a link to the PR is quite useful, because it immediately tells you if
someone already tried fixing it. If the PR has been merged, simply try to fix
another package. If the PR is open and not yet merged, this is a great
opportunity to help review it (more on that below)!

Let's go back to fixing `{SuperGauss}`. If you go back on Hydra, you can
see the error message that was thrown during building:

![Check out the logfile](images/SuperGauss_log.png)

Click on the logfile (either `pretty`, `raw` or `tail`) to see what happened.
Here's what we see:

```
checking for pkg-config... no
checking for FFTW... configure: error: in `/build/SuperGauss':
configure: error: The pkg-config script could not be found or is too old.  Make sure it
is in your PATH or set the PKG_CONFIG environment variable to the full
path to pkg-config.

Alternatively, you may set the environment variables FFTW_CFLAGS
and FFTW_LIBS to avoid the need to call pkg-config.
See the pkg-config man page for more details.

To get pkg-config, see <http://pkg-config.freedesktop.org/>.
See `config.log' for more details
ERROR: configuration failed for package 'SuperGauss'
* removing '/nix/store/jxv5p85x24xmfcnifw2ibvx9jhk9f2w4-r-SuperGauss-2.0.3/library/SuperGauss'
```

So the issue is that some system-level dependencies are missing, `pkg-config`
and `FFTW`, so we need to add these to fix the build. Which brings us the
first recipe of this cookbook!

## Recipe 1: packages that need dependencies to build

Fixing packages that require system-level dependencies is a matter of adding
one, maybe two lines, in the right place in the expression that defines the
whole of the `rPackages` set. You can find this file
[here](https://github.com/NixOS/nixpkgs/blob/master/pkgs/development/r-modules/default.nix).

In there, you will find a line that starts with `packagesWithNativeBuildInputs =
{` and another that starts with `packagesWithBuildInputs = {` which define a
long list of packages. The differences between `NativeBuildInputs` and
`BuildInputs` is that dependencies that are needed for compilation get listed
into `NativeBuildInputs` (so things like compilers, packages such as
`pkg-config`) and dependencies that are needed at run-time (dynamic libraries)
get listed under `BuildInputs`. For R, actually, you could put everything under
`NativeBuildInputs` and it would still work, but we try to pay attention to this
and do it properly. In case of doubt, put everything under `NativeBuildInputs`:
when reviewing your PR, people will then tell you where to put what.

Now try to build the package. The following line will drop you in an interactive
Nix shell with the package, if build succeeds (run the command at the root of the
cloned `nixpkgs` directory):

```
nix-shell -I nixpkgs=. -p R rPackages.SuperGauss
```

if you see the same error as on Hydra, and made sure that no PR is opened, then
you can start fixing the package.

So, we need to add two dependencies. Let's read the relevant lines in the error
message again:

```
configure: error: The pkg-config script could not be found or is too old.  Make sure it
is in your PATH or set the PKG_CONFIG environment variable to the full
path to pkg-config.

Alternatively, you may set the environment variables FFTW_CFLAGS
and FFTW_LIBS to avoid the need to call pkg-config.
See the pkg-config man page for more details.
```

If you look inside the two lists that define the packages that need
`nativeBuildInputs` and `buildInputs`, you'll see that many of them
have `pkg-config` listed there. So let's add the following line in the
`packagesWithNativeBuildInputs`

```
SuperGauss = [ pkgs.pkg-config ];
```

and this one under `packagesWithBuildInputs`:

```
SuperGauss = [ pkgs.fftw.dev ];
```

This is because `pkg-config` is only needed to compile `{SuperGauss}`
and `fftw.dev` is needed at run-time as well.

Try to build a shell with `{SuperGauss}` again:

```
nix-shell -I nixpkgs=. -p R rPackages.SuperGauss
```

If it worked, start R and load the library. Sometimes packages can build
successfully but fail to launch, so taking the time to load it avoids
wasting your reviewer's time. Ideally, try to run one or several
examples from the package's vignette or help files. This also makes sure
that everything is working properly. If your testing succeeded, you can
now open a PR!

Before committing, make sure that you are on a seprate branch for this fix:

```
git checkout -b fix_supergauss
```

From there, make sure that only the `default.nix` file changed:

```
git status
```


```
user@computer:~/Documents/nixpkgs(fix_supergauss *)$ git status
On branch fix_supergaus
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   pkgs/development/r-modules/default.nix

no changes added to commit (use "git add" and/or "git commit -a")
```

Great, so add it and write following commit message:

```
git add .
git commit -m "rPackages.SuperGauss: fixed build"
```

This commit message follows `nixpkgs` [contributing guidelines](https://nixos.wiki/wiki/Nixpkgs/Contributing).
Format all your messages like this.

Now push your changes:

```
git push origin fix_supergauss
```

and go on your fork's repository to open a PR. 

Congratulations, you fixed your first package!

## Recipe 2: packages that need a home, X, or simple patching

Some package may require a `/home` directory during their installation process. They
usually fail with a message that looks like this:

```
Warning in normalizePath("~") :
  path[1]="/homeless-shelter": No such file or directory
```

so add the package to the list named `packagesRequiringHome` and
try rebuilding.

See this PR as an example: https://github.com/NixOS/nixpkgs/pull/292336

Some packages require `X`, as in `X11`, the windowing system on Linux
distributions. In other words, these pacakges must be installed on a machine
with a graphical session running. So because that's not the case on Hydra, this
needs to be mocked. Simply add the package to the list named
`packagesRequiringX` and try rebuilding.

See this PR https://github.com/NixOS/nixpkgs/pull/292347 for an example.

Finally, some packages that must compiled need first to be configured. This is a
common step when compiling software. This configuration step ensures that needed
dependencies are found (among other things). Because Nix works the way it does,
it can happen that this configuration step fails because dependencies are not in
the usual `/usr/bin` or `/bin`, etc, paths. So this needs to be patched before
the configuration step. To fix this, the `configuration` file that lists the
different dependencies to be found needs to be patched, and this can be done
by overriding one of the phases before the configure phase. We now override
the `postPatch` phase like this:

```
RcppCGAL = old.RcppCGAL.overrideAttrs (attrs: {
  postPatch = "patchShebangs configure";
});
```

Read more about `patchShebangs`
[here](https://github.com/NixOS/nixpkgs/blob/master/pkgs/build-support/setup-hooks/patch-shebangs.sh).

See this PR for an example: https://github.com/NixOS/nixpkgs/pull/289775

## Recipe 3: packages that require their attributes to be overridden

https://github.com/NixOS/nixpkgs/pull/292329

## Recipe 4: packages that require internet access to build

## Recipe 5: packages that need a dependency that must be overridden

https://github.com/NixOS/nixpkgs/pull/293081

https://github.com/NixOS/nixpkgs/pull/291004

https://github.com/NixOS/nixpkgs/pull/292149

## Solving conflicts