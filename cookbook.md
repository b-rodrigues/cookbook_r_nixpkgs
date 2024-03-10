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

## The starter: where to find packages to fix

The first step to help fix a package is to find a package to fix: you should
visit the latest `rPackages` evaluation over [here](https://hydra.nixos.org/jobset/nixpkgs/r-updates).
Visit the "Still failing jobs" tab to see which packages' builds didn't succeed and
click on a package:

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
system-level dependency, that its own name is listed there). You can type
`{SuperGauss}` on the little box there to filter and see all the packages that
fail because of of it. Then, you can also see the package's *package rank*. This
rank is computer using the `{packageRank}` package, and the table is sorted by
lowest rank (low ranks indicate high popularity and there's also the
`percentile` column that indicate the percentage of packages with higher
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

In there, you will find a line that starts with `packagesWithNativeBuildInputs = {`
and another that starts with `packagesWithBuildInputs = {` which define a
long list of packages. The differences between `NativeBuildInputs` and
`BuildInputs` is that dependencies that are needed for compilation get listed
into `NativeBuildInputs` (so things like compiler, or `pkg-config`) and
dependencies that are needed at run-time (dynamic libraries) are `BuildInputs`.
For R, actually, you could put everything under `NativeBuildInputs` and it would
still work, but we try to pay attention to this and do it properly. In case of
doubt, put everything under `NativeBuildInputs`: when reviewing your PR, people
will then tell you where to put what.

Before trying anything, try to install the package and test it. Assuming you have
cloned your fork of the `nixpkgs` repository, make sure to be on the very latest
commit of master:

```
git checkout master

# Add upstream as a remote
git remote add upstream git@github.com:NixOS/nixpkgs.git

# Fetch latest updates
git fetch upstream

# Merge latest upstream/master to your master
git merge upstream/master
```

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

If you look inside the two lists, that define the packages that need
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


## Recipe 2: packages that need a home, X, or simple patching

## Recipe 3: packages that require their attributes to be overridden

https://github.com/NixOS/nixpkgs/pull/292329

## Recipe 4: packages that require internet access to build

## Recipe 5: packages that need a dependency that must be overridden

https://github.com/NixOS/nixpkgs/pull/293081

https://github.com/NixOS/nixpkgs/pull/291004

https://github.com/NixOS/nixpkgs/pull/292149
