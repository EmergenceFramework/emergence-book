# Introduction

## Why another framework?

* Already today, every organization needs an array of web applications serving them persistently from the cloud. This need has already started spreading to individuals too and soon persistent personal web applications will be as commonplace as smartphone apps.
* Building on top of others' work should be effortless, as should sharing yours in-turn
* Experimenting shouldn't cost anything, locally or in the open
* Deploying an app: The forking problem
* Deploying an app: The plugin problem
* Plumbing sucks
* Move infrastructure out of your code

## What does it do?

### Kernel

Managed services

### Runtime

Turn piles of files into living things that handle HTTP requests

### Skeleton

A starting point with emergence-optimized MVC, starter design, and user system

## Why PHP?

* Execution model
  * Lends itself well to hosting many applications with frequently changing parts under one runtime
  * Worker pools are ready to go and scale well
  * Live and die by the request, no continuous state, keeps things easy to scale
  * OpCache eliminates disk and parsing overhead efficiently
  * APCU provides fast persistent memory
* But I heard it's dead
  * Not really: facebook, etsy, wordpress, wikipedia
  * We just have to flush the first 10 years of its life down the toilet
  * php-fig, composer, packagist, PHP7/HHVM
* Why not Node?
  * PHP seemed to lend itself better to the model, but there's no reason any other language couldn't be adopted by someone who understands its runtime well enough

