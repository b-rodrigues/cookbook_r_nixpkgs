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


## Recipe 1: packages that need dependencies to build

## Recipe 2: packages that need a home, X, or simple patching

## Recipe 3: packages that internet access to build

## Recipe 4: packages that need a dependency that must be overridden
